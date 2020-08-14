---
title: Protocol Buffers Efficient Encoding
toc: true
date: 2020-03-14 11:39:26
tags: 
- Protocol Buffers
---
## Protocol如何实现高效编码
&emsp;当看到proto协议的时候, 会觉得和struct和class的定义差不多.那么它是不是以一定的内存对齐原则,排列在stream里面的呢?但是如果int类型都是占4个字节的话,1和10000在stream中的长度都是一样的,岂不是很浪费, 都说使用protocol buffers可以节省内存,但具体是如何实现的呢? 
<!-- more -->  
### Protocol不同类型编码方式介绍
| 类型id | 含义 | 使用 |
| --- | --- | --- |
| 0 | Varint | int32,int64,uint32,uint64,sint32,sint64,bool,enum|
| 1 | 64-bit | fixed64, sfixed64, double|
| 2 | Length-delimited | string,bytes,embedded messages, packed repeated fields|
| 3 | Start group | groups(弃用) |
| 4 | End group | groups(弃用) |
| 5 | 32-bit | fixed32, sfixed32, float |

&emsp;解码器是如何知道即将解码的对象是什么类型的呢?
&emsp;protocol buffers的数据流中,数据是是以流的形式串连起来的,而每一个数据是由key(filed_number << 3|wire_type)-value组成, 形式如下:
&emsp;key1-value1|key2-value2|-------|keyn-valuen|
&emsp;接下来优先看一下Varint这种可变长存储整形数据的类型.
#### Varint
&emsp;我不知道其他人怎么翻译这个词, 从它的的作用来看的话它就是一类可变长的变量. 和内存对齐不同, 这种变量类型在protocol编码过程当中是可变长的,那么自然的1和10000的编码所占字节数也是不同的.
&emsp;Varint类型数据是由一连串单字节数据构成, 既然是一连串, 那么必然需要一个串联信息. protocol将8位数据分成了两部分:1是数据本身,占7位(取值范围0~127);2就是时候还有后续数据串联,占1位(0没有 1有). 
##### uint编码 1和10000
###### 1
&emsp;对应的二进制为:0000 0001由于1个字节已经够了所有只占用1个自己
###### 10000
&emsp;先看10000原本的二进制编码为 0010 0111 0001 0000. 由于占用了两个字节, 也就是说第一个字节在解码的时候需要告诉解码器, 这个数据还需要有后续字节, 所有最终在protocol中的编码为:
11001110 00010000
1 100 1110  -> 0 001 0000 -> 10011100010000 = 10000
Ok现在我们发现1000和1占用空间并不一样了!
##### int和sint的区别
&emsp;google官网的解释是:sint和int类型在数值为负数时编码上有较大的差异；int将占用10个字节，而sint将使用更为高效的zigzag编码。
> there is an important difference between the signed int types (sint32 and sint64) and the "standard" int types (int32 and int64) when it comes to encoding negative numbers. If you use int32 or int64 as the type for a negative number, the resulting varint is always ten bytes long – it is, effectively, treated like a very large unsigned integer. If you use one of the signed types, the resulting varint uses ZigZag encoding, which is much more efficient.
&emsp;接下来就重点分析两个问题：int类型负值编码为什么始终会占用10个字节和ZigZag编码为何更加高效。
###### int负数编码为何始终占用10个字节
&emsp;下面看一下protocol int32编码c#源码：
````c#
/// <summary>
/// Writes an int32 field value, without a tag, to the stream.
/// </summary>
/// <param name="value">The value to write</param>
public void WriteInt32(int value)
{
    if (value >= 0)
    {
        WriteRawVarint32((uint) value);
    }
    else
    {
        // Must sign-extend.
        WriteRawVarint64((ulong) value);
    }
}

internal void WriteRawVarint64(ulong value)
{
    while (value > 127 && position < limit)
    {
        buffer[position++] = (byte) ((value & 0x7F) | 0x80);
        value >>= 7;
    }
    while (value > 127)
    {
        WriteRawByte((byte) ((value & 0x7F) | 0x80));
        value >>= 7;
    }
    if (position < limit)
    {
        buffer[position++] = (byte) value;
    }
    else
    {
        WriteRawByte((byte) value);
    }
}
````
&emsp;可以看到int32在编码负数时，会将类型转为ulong(64位)，然后还是按照7位串联的方式进行编码。
所以负数最终会占用 math.ceil(64 / 7) = 10字节。
###### sint之ZigZag编码
&emsp;ZigZag编码是将负数重新映射到了正数范围,之后就可以继续按照7位串联的方式进行编码,下面就是ZIgZag编码的映射表:

| 原始数值 | 映射后数值 |
| --- | --- | 
| 0 | 0 |
| -1| 1 |
| 1| 2 |
| -2 | 3 |
| 2147483647 | 4294967294|
| -2147483648 | 4294967295|
&emsp;在学习int和sint的编码区别之后, 现在sint负数编码高效的原因也就清除了~
#### Length-delimited
&emsp;对于Length-delimited类型字段需要一个额外参数来描述该字符串的长度, 该长度参数使用的varint类型的编码方式(7位串联）。
&emsp; key(file_number<<3 | wire_type) + varint_length + data_value
##### string
&emsp;下面就来看一个string类型的例子：
````c++
message Test1 
{  
     string b = 2;
}
test1.b = "testing"

//编码：12 07 74 65 73 74 69 6e 67
````
 key:12 = （0010 << 3 | 0010） = 00010010
 length = 7
 value = 74 65 73 74 69 6e 67
##### embedded mesaages
````c++
message Test2
{  
     int32 a = 1;
}
test2.a = 150
//编码：08 96 01

message Test3 
{  
    Test1 c = 3;
}
//编码：1a 03 08 96 01
````
&emsp;由此可见protocol buffer通过varint类型7位串联对数值编码，使得数值占用空间减少；使用key-value串联的形式使得数据排列更加紧凑。
