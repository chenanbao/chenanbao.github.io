---
 layout:     post
 title:      通用渲染管线(URP)HLSL
 subtitle:   HLSL基本语法及库函数
 date:       2021-02-19
 author:     Bob
 header-img: img/post-bg-unity.jpg
 catalog: true
 tags:
     - Unity
---

##### HLSL语义

###### 顶点着色器（Vertex Shader）语义

|输入	|描述	|类型|
| ------ | ------ | ------ |
|BINORMAL[n]	|次法线|	float4|
|BLENDINDICES[n]	|混合指数|	uint|
|BLENDWEIGHT[n]	|混合权重|	float|
|COLOR[n]	|固有色与高光色	|float4|
|NORMAL[n]	|法向量|	float4|
|POSITION[n]	|模型空间中顶点坐标|	float4|
|POSITIONT	|顶点坐标转置|	float4|
|PSIZE[n]	|点大小|	float|
|TANGENT[n]	|切线	|float4|
|TEXCOORD[n]	|纹理坐标|	float4|

|输入	|描述	|类型|
| ------ | ------ | ------ |
|COLOR[n]	|固有色与高光色	|float4|
|FOG	|顶点雾化值|	float|
|POSITION[n]	|齐次空间顶点坐标，将（x,y,z）除以w即可得到屏幕坐标空间，每个顶点Shader必须有该语义输出。|	float4|
|PSIZE	|点大小|	float|
|TESSFACTOR[n]	|曲面细分因子|	float|
|TEXCOORD[n]	|纹理坐标|	float4|


###### 像素着色器（Pixel Shader）语义

|输入	|描述	|类型|
| ------ | ------ | ------ |
|COLOR[n]|	固有色与高光色|	float4|
|TEXCOORD[n]|	纹理坐标	|float4|
|VFACE|	用于标记一个背面图元的浮点标量值，负值表示面向背面，正值表示面向相机，该语义仅在Direct3D 9的Shader Model 3.0中可用，Direct3D 10以后使用SV_IsFrontFace替代|	float|
|VPOS	|屏幕空间像素位置(x,y). 该语义用于Direct3D 9，之后版本请使用SV_Position	|float2|

|输入	|描述	|类型|
| ------ | ------ | ------ |
|COLOR[n]	|输出颜色|	float4|
|DEPTH[n]	|输出深度|	float|

###### 语义变化

|Direct3D 10语义 |	Direct3D 9等价语义 |
| ------ | ------ |
|SV_Depth|	DEPTH||
|SV_Position|	POSITION|
|SV_Target|	COLOR|


##### com.unity.render-pipelines.universal

Core.hlsl
|名称	|说明|
| ------ | ------ |
|GetVertexPositionInputs(float3 positionOS)	|获取输入顶点坐标信息|
|GetVertexNormalInputs(float3 normalOS)	|获取输入顶点法线信息|
|GetVertexNormalInputs(float3 normalOS, float4 tangentOS)	|获取输入顶点法线信息（重载）|
|GetScaledScreenParams()	|获取屏幕缩放参数信息|
|NormalizeNormalPerVertex(real3 normalWS)	|逐顶点法线正交|
|NormalizeNormalPerPixel(real3 normalWS)	|逐像素法线正交|
|ComputeScreenPos(float4 positionCS)	|计算屏幕坐标信息|
|(real)ComputeFogFactor(float z)	|计算雾参数|
|(real)ComputeFogIntensity(real fogFactor)	|计算雾强度|
|(half3)MixFogColor(real3 fragColor, real3 fogColor, real fogFactor)	|混合雾颜色|
|(half3)MixFog(real3 fragColor, real fogFactor)	|混合雾|

Lighting.hlsl

|名称	|说明|
| ------ | ------ |
|DistanceAttenuation(float distanceSqr, half2 distanceAttenuation)	|距离衰减|
|AngleAttenuation(half3 spotDirection, half3 lightDirection, half2 spotAttenuation)	|角度衰减|
|GetMainLight()/GetMainLight(float4 shadowCoord)	|获取主光源|
|GetPerObjectLightIndex(int index)	|获取每个对象灯光Index|
|GetAdditionalLightsCount()	|获取额外灯光数量|
|ReflectivitySpecular(half3 specular)	|高光反射率|
|OneMinusReflectivityMetallic(half metallic)	|OneMinus金属反射率|
|InitializeBRDFData(half3 albedo, half metallic, half3 specular, half smoothness, half alpha, out BRDFData outBRDFData)	|初始化BRDF|
|EnvironmentBRDF(BRDFData brdfData, half3 indirectDiffuse, half3 indirectSpecular, half fresnelTerm)	|环境BRDF|
|DirectBDRF(BRDFData brdfData, half3 normalWS, half3 lightDirectionWS, half3 viewDirectionWS)	|BRDF|
|SampleLightmap(float2 lightmapUV, half3 normalWS)|	光照贴图|
|GlossyEnvironmentReflection(half3 reflectVector, half perceptualRoughness, half occlusion)	|环境光泽反射|
|GlobalIllumination(BRDFData brdfData, half3 bakedGI, half occlusion, half3 normalWS, half3 viewDirectionWS)	|全局光照|
|MixRealtimeAndBakedGI(inout Light light, half3 normalWS, inout half3 bakedGI, half4 shadowMask)	|实时烘培混合|
|LightingLambert(half3 lightColor, half3 lightDir, half3 normal)	|兰伯特模型|
|LightingSpecular(half3 lightColor, half3 lightDir, half3 normal, half3 viewDir, half4 specular, half smoothness)	|高光|
|LightingPhysicallyBased(BRDFData brdfData, half3 lightColor, half3 lightDirectionWS, half lightAttenuation, half3 normalWS, half3 viewDirectionWS)/LightingPhysicallyBased|(BRDFData brdfData, Light light, half3 normalWS, half3 viewDirectionWS)	|基于物理的光照模型|
|VertexLighting(float3 positionWS, half3 normalWS)	|顶点光照颜色|
|LightweightFragmentPBR(InputData inputData, half3 albedo, half metallic, half3 specular,half smoothness, half occlusion, half3 emission, half alpha)	|轻量级片元PBR|
|LightweightFragmentBlinnPhong(InputData inputData, half3 diffuse, half4 specularGloss, half smoothness, half3 emission, half alpha)	|轻量级片元布林·冯|

Shadows.hlsl

|名称	|说明|
| ------ | ------ |
|GetMainLightShadowStrength()	|获取主光源阴影强度|
|GetAdditionalLightShadowStrenth(int lightIndex)	|获取额外光源阴影强度|
|SampleScreenSpaceShadowmap(float4 shadowCoord)	|屏幕空间阴影贴图|
|SampleShadowmap(float4 shadowCoord, TEXTURE2D_SHADOW_PARAM(ShadowMap, sampler_ShadowMap), ShadowSamplingData samplingData, half shadowStrength, bool isPerspectiveProjection = true)	|阴影贴图|
|TransformWorldToShadowCoord(float3 positionWS)	|把顶点的世界坐标转换到阴影坐标|
|MainLightRealtimeShadow(float4 shadowCoord)	|主光源实时阴影|
|AdditionalLightRealtimeShadow(int lightIndex, float3 positionWS)	|额外光源实时阴影|
|GetShadowCoord(VertexPositionInputs vertexInput)	|获取阴影坐标信息|
|ApplyShadowBias(float3 positionWS, float3 normalWS, float3 lightDirection)	|应用阴影偏移|

##### com.unity.render-pipelines.core

SpaceTransforms.hlsl
变换矩阵：

|名称	|说明|
| ------ | ------ |
|TransformObjectToWorld(float3 positionOS)	|当前模型空间转世界空间矩阵，通常用于把顶点/方向矢量从模型空间转到世界空间|
|TransformWorldToObject(float3 positionWS)	|当前世界空间转模型空间矩阵，通常用于把顶点/方向矢量从世界空间转到模型空间|
|TransformWorldToView(float3 positionWS)	|当前世界空间转相机空间矩阵，通常用于把顶点/方向矢量从世界空间转到相机空间|
|TransformObjectToHClip(float3 positionOS)	|当前模型空间转裁剪空间矩阵，通常用于把顶点/方向矢量从模型空间转到裁剪空间|
|TransformWorldToHClip(float3 positionWS)	|当前世界空间转裁剪空间矩阵，通常用于把顶点/方向矢量从世界空间转到裁剪空间|
|TransformWViewToHClip(float3 positionVS)	|当前相机空间转裁剪空间矩阵，通常用于把顶点/方向矢量从相机空间转到裁剪空间|
|TransformObjectToWorldDir(real3 dirOS)	|把方向矢量从模型空间转换到世界空间中|
|TransformWorldToObjectDir(real3 dirWS)	|把方向矢量从世界空间转换到模型空间中|
|TransformWorldToViewDir(real3 dirWS)	|把方向矢量从世界空间转换到相机空间中|
|TransformWorldToHClipDir(real3 directionWS)	|把方向矢量从世界空间转换到裁剪空间中|
|TransformObjectToWorldNormal(float3 normalOS)	|把法线从模型空间转换到世界空间中|
|CreateTangentToWorld(real3 normal, real3 tangent, real flipSign)	|创建一个切线空间转为世界空间的3x3矩阵|
|TransformTangentToWorld(real3 dirTS, real3x3 tangentToWorld)	|当前切线空间转世界空间矩阵，通常用于把顶点/方向矢量从切线空间转到世界空间|
|TransformWorldToTangent(real3 dirWS, real3x3 tangentToWorld)	|当前世界空间转切线空间矩阵，通常用于把顶点/方向矢量从世界空间转到切线空间|
|TransformTangentToObject(real3 dirTS, real3x3 tangentToWorld)	|当前切线空间转模型空间矩阵，通常用于把顶点/方向矢量从切线空间转到模型空间|



##### 参考

[HLSL官方手册](https://docs.microsoft.com/zh-cn/windows/win32/direct3dhlsl/dx-graphics-hlsl-reference?redirectedfrom=MSDN)