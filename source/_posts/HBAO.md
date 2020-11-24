---
title: 浅谈HBAO
toc: true
date: 2020-03-15 21:18:50
tags:
- HBAO
- AO
- PostProcess
- Unity
---
### Horizon-Based Ambient Occlusion 算法说明
&emsp; Ambient Occlusion(漫反射遮罩)，是用来描述反射光线被周围物体遮挡的一种光照模型。先看下面的公式：是描述物体表面任一一点的遮罩情况：
![](/images/HBAO/1.png)
<!-- more -->
&emsp; 上式中V代表该某一特定方向上的遮挡情况，该点延w方向被遮挡时V=1，未被遮挡时V=0。W则是一个简单的线性衰减函数。
&emsp; 接下来进一步分析Horizon-Based Ambient Occlusion是如何计算遮挡的。
![](/images/HBAO/2.png)
&emsp; 先看上图，其中弯曲的红线是p点半球面延某一特定方向t的切片上(半球上砍一刀的横截面)的高度场。
&emsp; 在切片范围内高度场的最大值点与P点连线方向(黄色)与方向t的夹角为h(θ); 
&emsp; 垂直于视线平面(位于该平面上的范围才计算遮挡)与方向t的夹角为t(θ)。
&emsp; 近似上诉切片上被遮挡的值=cose(α)。在t(θ)~h(θ)这个范围对遮挡值求定积分，接着在整个半球范围对遮挡值积分，得到整个半球面的遮挡值。以上即使Horizontal-Based Ambient Occlusion算法的核心，公式如下。
![](/images/HBAO/3.png)
&emsp; 上式中：
&emsp; ![](/images/HBAO/4.png)
&emsp; 进一步简化正弦函数（Ps:GPU普通指令运算单元数量远多于三角函数这类复杂运算单元）

![](/images/HBAO/5.png)
&emsp; 通过上面的转换，将正弦函数转化为常用运算指令。
````c++
inline float SimpleAO(float3 pos, float3 stepPos, float3 normal, inout float top)
{
    float3 h = stepPos - pos;
    float dist = sqrt(dot(h, h));
    float sinBlock = dot(normal, h) / dist;
    float diff = max(sinBlock - top, 0);
    top = max(sinBlock, top);
    return saturate(diff) * saturate(FallOff(dist));
}
````
&emsp; 以上则是AO近似计算的代码，sinBlock-top对应sin(h)-sin(t), Falloff(dist)是一个随距离线性衰减的函数对应W(w)部分。
### 基于Unity实现
&emsp; HBAO是在后处理阶段，逐像素构建半球面。采用蒙特卡洛采样计算像素对应物体坐标点的AO值。下图是nvddia官方给出的HBAO实现流程。
![](/images/HBAO/6.png)
&emsp; 根据算法介绍我们知道，需要重建对应点坐标和法线，计算AO再进行模糊处理，最终和原始图像合并得到最终效果。以下是HBAO基于u3d前向渲染的实现。

#### 屏幕空间坐标重建
&emsp; 下图中θ为相机的fov, 均是分析对应p屏幕空间的y坐标（其中v0, v1, v2对应uv值中的v映射至-1~1）。

![](/images/HBAO/7.png)
&emsp; 对应代码如下：
````c#
    var tanHalfFovY = Mathf.Tan(mCamera.fieldOfView * 0.5f * Mathf.Deg2Rad);
    var tanHalfFovX = tanHalfFovY * ((float)mCamera.pixelWidth / mCamera.pixelHeight);

    //计算相机空间:x = (2* u - 1) * tanHalfFovX * depth  (2u - 1将坐标映射到-1,1)
    mMaterial.SetVector(ShaderProperties.UV2View, 
        new Vector4(2 * tanHalfFovX, 2 * tanHalfFovY, -tanHalfFovX, -tanHalfFovY));
````
````c++
    inline float3 FetchViewPos(float2 uv)
    {
        float depth = LinearEyeDepth(FetchDepth(uv));
        return float3((uv * _UV2View.xy + _UV2View.zw) * depth, depth);
    }
````
#### 计算HBAO
&emsp; 上面已经介绍了半球某一特定方向上切片遮挡的计算方法，接下来使用蒙特卡洛积分，采样半球不同方向上的切片的遮挡，模拟HBAO积分结果。
&emsp; 为了避免明显的交错感，每次采样时进行一次随机，让采样方向产生一点偏移。如下图所示：
![](/images/HBAO/8.png)
&emsp; 随机算法采用了the book of shader 11中提到的value-nosie。
````c++
inline float random(float2 uv) {
    return frac(sin(dot(uv.xy, float2(12.9898, 78.233))) * 43758.5453123);
}
````
&emsp; 下图时使用python matplotlib库生成的value-nosie的噪声图。
![](/images/HBAO/9.png)

&emsp; 以下则是使用蒙特卡洛方法计算HBAO的主要代码
````c++
    for (int i = 0; i < DIRECTION; ++i)
    {
        float angle = delta * (float(i) + rnd);
        float cos, sin;
        sincos(angle, sin, cos);
        float2 dir = float2(cos, sin);
        float rayPixel = 1;
        float top = _AngleBias;
        UNITY_UNROLL
        for(int j = 0; j < STEPS; ++j)
        {
            float2 stepUV = round(rayPixel * dir) * _TexelSize.xy + input.uv;
            float3 stepViewPos = FetchViewPos(stepUV);
            ao += SimpleAO(viewPos, stepViewPos, normal, top);
            rayPixel += stepSize;
        }
    }
    ao /= STEPS * DIRECTION;
`````
&emsp; 由于采样数量和高度场步进数有限，生成的AO图上会有一定交错感和噪点，使用模糊算法（本文代码分别使用了均值模糊和高斯模糊）对AO图像进行处理后，与原始图像叠加既可以得到最终效果。以上是个人学习过程中的总结，如果有问题还望看到的同学指出。
&emsp; 源代码：https://github.com/shadylyf321/HBAO

参考：
[Image-Space Horizon-Based Ambient Occlusion](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.214.6686&rep=rep1&type=pdf)
[An alternative implementation for HBAO](https://www.derschmale.com/2013/12/20/an-alternative-implementation-for-hbao-2/)
[NVIDIA HBAO+ 2.4](https://docs.nvidia.com/gameworks/content/gameworkslibrary/visualfx/hbao/product.html)
[the book of shader 11](https://thebookofshaders.com/11/)
