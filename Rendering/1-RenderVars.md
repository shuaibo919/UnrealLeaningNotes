## 虚幻渲染
### 1.着色器和顶点数据
所有Unreal着色器的基类是`FShader`。Unreal将着色器分为两大类：`FGlobalShader`适用于只应存在一个实例的情况（可见[添加后处理Pass](../Practice/1-AddPostProcessingPass.md)章节），而`FMaterialShader`则与材料相关的着色器。`FShader`与`FShaderResource`配对，后者跟踪与该特定着色器关联的GPU上的资源。如果多个FShader的编译输出与已有资源匹配，`FShaderResource`可以在多个FShader之间共享。

其中：`FMaterialShader` 和 `FMeshMaterialShader`。这两个类都允许多个实例，每个实例都有自己的 GPU 资源副本。`FMaterialShader` 添加了一个 `SetParameters` 函数，允许你的着色器的 C++ 代码更改绑定的 HLSL 参数的值。参数绑定是通过 `FShaderParameter/FShaderResourceParameter` 类实现的，可以在着色器的构造函数中完成。

### 2.顶点工厂
顶点工厂封装了一个顶点数据源，并且可以链接到顶点着色器。（如果你之前编写过自己的渲染器代码，一个比较典型的实现是创建一个包含所有可能的顶点数据的类）。Unreal则使用顶点工厂，它可以只上传到顶点缓冲区中实际需要的数据。

要理解顶点工厂，有两个具体示例：`FLocalVertexFactory` 和 `FGPUBaseSkinVertexFactory`。`FLocalVertexFactory` 在许多地方都得以应用，因为它提供了一种简单的方式，将明确的顶点属性从局部空间转换到世界空间。如StaticMesh、和Procedural Mesh都使用这种顶点工厂。不过，SkeletalMesh则使用 `FGPUBaseSkinVertexFactory`。接下来，我们探讨与这两个顶点工厂相匹配的着色器数据，其中所包含的数据是有所不同的。

那Unreal中是如何知道在网格中使用哪个顶点工厂的？答案是通过`FPrimitiveSceneProxy`类!
`FPrimitiveSceneProxy`是`UPrimitiveComponent`的在渲染线程中的数据。如果要自定义一些功能，那么就要创建`UPrimitiveComponent`和`FPrimitiveSceneProxy`的子类，并具体实现。这部分的具体内容可参考[博客园 向往0的 《剖析虚幻渲染体系（03）- 渲染机制》](https://www.cnblogs.com/timlly/p/14588598.html)

#### 2.2.1 了解顶点渲染流程
一个顶点工厂对存在一个一个对应的着色器标头文件（.ush）,标头文件中的FVertexFactoryInput 以及一些函数会被其他的虚幻着色器格式（.usf）或标头文件（.ush）所包含。例如BasePassVertexShader.usf具有如下的入口函数:
```hlsl
/** Entry point for the base pass vertex shader. */
void Main(
	FVertexFactoryInput Input,
	out FBasePassVSOutput Output
#if USE_GLOBAL_CLIP_PLANE
	, out float OutGlobalClipPlaneDistance : SV_ClipDistance
#endif
#if INSTANCED_STEREO
	, out uint ViewportIndex : SV_ViewPortArrayIndex
#endif
	)
```
其中宏处理部分，可以暂时不关心，那么就只要一个输入是`FVertexFactoryInput`，一个输出是`FBasePassVSOutput`。其中，后者是创建好的，可以不关心。`FVertexFactoryInput`结构体类型就要求我们必须在顶点工厂的对应的.ush文件中创建，否则Shader编译时就会报错，因为它找不到`FVertexFactoryInput`的定义。

于是，如果我们自己自定义了顶点工厂，就需要创建对应的`.ush`文件，其中又必须实现`FVertexFactoryInput`,例如以下是来自`MeshParticleVertexFactory`的`.ush`中的`FVertexFactoryInput`
```hlsl
// Engine\Shaders\Private\MeshParticleVertexFactory.ush
struct FVertexFactoryInput
{
	float4	Position	: ATTRIBUTE0;
	HALF3_TYPE	TangentX	: ATTRIBUTE1;
	// TangentZ.w contains sign of tangent basis determinant
	HALF4_TYPE	TangentZ	: ATTRIBUTE2;
	HALF4_TYPE	VertexColor : ATTRIBUTE3;
};
```

其它比较重要的部分包括:
|                                    |                                                                                                                                                                                                                       |
| :--------------------------------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|    FVertexFactoryIntermediates     |                                     用于存储高速缓存的中间数据，该数据将在多个顶点工厂函数中使用。一个常用的示例是 TangentToLocal 矩阵，该矩阵必须根据未打包的顶点输入进行计算。                                      |
|  FVertexFactoryInterpolantsVSToPS  |                                                                                    要从顶点着色器传递到像素着色器的顶点工厂数据。                                                                                     |
|   VertexFactoryGetWorldPosition    | 此函数从顶点着色器中调用，用于获取全局空间顶点位置。对于静态网格，此函数只是使用 LocalToWorld 矩阵将局部空间位置从顶点缓冲区转换到全局空间。对于由 GPU 处理皮肤的网格，将首先处理此位置的皮肤，然后再转换到全局空间。 |
| VertexFactoryGetInterpolantsVSToPS |                                                     将 FVertexFactoryInput 转换为 FVertexFactoryInterpolants，后者将由图形硬件进行插值，然后再传递到像素着色器。                                                      |
|     GetMaterialPixelParameters     |                                  此函数在像素着色器中调用，并将特定于顶点工厂的插值 (FVertexFactoryInterpolants) 转换为 FMaterialPixelParameters 结构，该结构由过程像素着色器使用。                                   |
#### 2.1 FLocalVertexFactory


(https://dev.epicgames.com/documentation/zh-cn/unreal-engine/shader-development-in-unreal-engine)
(https://thegraphicguysquall.wordpress.com/2021/11/15/unreal-4-custom-mesh-pass-in-deferred-shading/)
