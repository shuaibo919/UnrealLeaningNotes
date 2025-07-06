## 向虚幻引擎中添加自定义的顶点工厂

> 尚未完成，完成后添加对各个部分的分析

- 问题记录
  - 已解决：添加自定义ush后引起的Shader编译Crush,添加SceneData和`#include "VertexFactoryDefaultInterface.ush"`
  - 待解决：在蓝图Actor下添加MeshComponent触发Crush,疑似LOD在RenderProxy设置有问题？
![alt text](image-AddCustomVertexFactoryCrush.png)


### 1. Step1 添加对应ush着色器文件
```hlsl
#include "/Engine/Private/VertexFactoryCommon.ush"

// 1.定义FVertexFactoryInput，根据需要填充不同的数据
struct FVertexFactoryInput
{
	float4	Position	: ATTRIBUTE0;
	uint VertexId : SV_VertexID;
	uint InstanceId	: SV_InstanceID;
};

// 2. VS到PS的数据
struct FVertexFactoryInterpolantsVSToPS
{
	half4 Color	: COLOR0;
	float2 TexCoords	: TEXCOORD0;
};

// 3. Intermediates的临时数据填充
struct FVertexFactoryIntermediates
{
	float3	Position;
	half4	Color;

	FSceneDataIntermediates SceneData;
};

// 4. 填充Intermediates
FVertexFactoryIntermediates GetVertexFactoryIntermediates(FVertexFactoryInput Input)
{
	FVertexFactoryIntermediates Intermediates = (FVertexFactoryIntermediates)0;
	Intermediates.SceneData = VF_GPUSCENE_GET_INTERMEDIATES(Input);
	Intermediates.Position = Input.Position.xyz;
	Intermediates.Color = half4(0,0,0,1);
	
	return Intermediates;
}

// 5. Converts from vertex factory specific interpolants to a FMaterialPixelParameters, which is used by material inputs.
FMaterialPixelParameters GetMaterialPixelParameters(FVertexFactoryInterpolantsVSToPS Interpolants, float4 SvPosition)
{
	// GetMaterialPixelParameters is responsible for fully initializing the result
	FMaterialPixelParameters Result = MakeInitializedMaterialPixelParameters();
#if NUM_MATERIAL_TEXCOORDS
	UNROLL
	for (int CoordinateIndex = 0; CoordinateIndex < NUM_MATERIAL_TEXCOORDS; CoordinateIndex++)
	{
		Result.TexCoords[CoordinateIndex] = Interpolants.TexCoords;
	}
#endif
	
	Result.VertexColor = Interpolants.Color;
	Result.TwoSidedSign = 1;
	
	return Result;
}

// 6. Converts from vertex factory specific input to a FMaterialVertexParameters, which is used by vertex shader material inputs
FMaterialVertexParameters GetMaterialVertexParameters(FVertexFactoryInput Input, FVertexFactoryIntermediates Intermediates, float3 WorldPosition, half3x3 TangentToLocal)
{
	FMaterialVertexParameters Result = MakeInitializedMaterialVertexParameters();
	
	Result.WorldPosition = WorldPosition;
	Result.TangentToWorld = mul(TangentToLocal, GetLocalToWorld3x3());
	Result.VertexColor = Intermediates.Color;
	Result.SceneData = Intermediates.SceneData;

	return Result;
}

// 7. 用于从顶点着色器调用来获得世界空间的顶点位置。这个TransformLocalToTranslatedWorld是在VertexFactoryCommon里定义好的，直接调用
float4 VertexFactoryGetWorldPosition(FVertexFactoryInput Input, FVertexFactoryIntermediates Intermediates)
{
	return TransformLocalToTranslatedWorld(Intermediates.Position.xyz);
}

// 8. 直接返回即可，暂时对光栅化后的位置不做修改
float4 VertexFactoryGetRasterizedWorldPosition(FVertexFactoryInput Input, FVertexFactoryIntermediates Intermediates, float4 InWorldPosition)
{
	return InWorldPosition;
}

// 9. 组装Vertex Shader 到Pixel Shader中需要插值的变量
FVertexFactoryInterpolantsVSToPS VertexFactoryGetInterpolantsVSToPS(FVertexFactoryInput Input, FVertexFactoryIntermediates Intermediates, FMaterialVertexParameters VertexParameters)
{
	FVertexFactoryInterpolantsVSToPS Interpolants;
	Interpolants.Color = Intermediates.Color;
	Interpolants.TexCoords = float2(0,0);
	return Interpolants;
}

// 10. 获取上一帧世界位置，暂时直接返回
float4 VertexFactoryGetPreviousWorldPosition(FVertexFactoryInput Input, FVertexFactoryIntermediates Intermediates)
{
	return VertexFactoryGetWorldPosition(Input, Intermediates);
}

// 11. 直接抄的Engine/Plugins/Experimental/Water/Shaders/Private/WaterMeshVertexFactory.ush
half3x3 VertexFactoryGetTangentToLocal( FVertexFactoryInput Input, FVertexFactoryIntermediates Intermediates )
{
	return half3x3(1,0,0,0,1,0,0,0,1);
}

// 12. 直接抄的Engine/Plugins/Experimental/Water/Shaders/Private/WaterMeshVertexFactory.ush
float3 VertexFactoryGetWorldNormal(FVertexFactoryInput Input, FVertexFactoryIntermediates Intermediates)
{
	// TODO: Central differencing to figure out the normal
	return float3(0.0f, 0.0f, 1.0f);
}

FInstanceSceneData GetInstanceData(FVertexFactoryIntermediates Intermediates)
{
	return Intermediates.SceneData.InstanceData;
}

#include "VertexFactoryDefaultInterface.ush"
```

### 2. Step2 添加对应的自定义MeshComponent文件
   
```cpp
#pragma once

#include "CoreMinimal.h"
#include "UObject/ObjectMacros.h"
#include "Engine/EngineTypes.h"
#include "Engine/EngineBaseTypes.h"
#include "Components/SceneComponent.h"

#include "MyCustomMeshComponent.generated.h"



class FMyCustomVertexFactory: public FVertexFactory
{
	DECLARE_VERTEX_FACTORY_TYPE(FMyCustomVertexFactory)
public:
	FMyCustomVertexFactory(ERHIFeatureLevel::Type InFeatureLevel, const char* InDebugName)
		: FVertexFactory(InFeatureLevel)
	{
	}
	

	//~ Begin  FRenderResource interface
	virtual void InitRHI(FRHICommandListBase& RHICmdList) override
	{
		// 顶点流， 类似于VertexShaderInputLayout的说明
		FVertexStream PositionVertexStream;
		PositionVertexStream.VertexBuffer = VertexBuffer;
		PositionVertexStream.Stride = sizeof(FVector);
		PositionVertexStream.Offset = 0;
		PositionVertexStream.VertexStreamUsage = EVertexStreamUsage::Default;

		const FVertexElement VertexPositionElement(Streams.Add(PositionVertexStream), 0, VET_Float3, 0, PositionVertexStream.Stride, false);

		// 顶点声明
		FVertexDeclarationElementList Elements;
		Elements.Add(VertexPositionElement);

		InitDeclaration(Elements);
	}
	
	virtual void ReleaseRHI() override
	{
		UniformBuffer.SafeRelease();
		FVertexFactory::ReleaseRHI();
	}
	//~ End  FRenderResource interface

	static bool ShouldCompilePermutation(const FVertexFactoryShaderPermutationParameters& Parameters);

	FVertexBuffer *VertexBuffer = nullptr;
	
private:
	FUniformBufferRHIRef UniformBuffer;
	
};


UCLASS(Blueprintable, ClassGroup=(Rendering, Common), hidecategories=(Object, "Mesh|CustomAssetMesh"), config=Engine, editinlinenew, meta=(BlueprintSpawnableComponent), MinimalAPI)
class UMyCustomMeshComponent : public UMeshComponent
{
	GENERATED_BODY()
	public:
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Components|CustomAssetMesh")
	bool bIsVisible;

	UFUNCTION(BlueprintCallable, Category = "Components|CustomAssetMesh")
	void SetIsVisible(const bool bNewIsVisible) { bIsVisible = bNewIsVisible; }

	UPROPERTY(EditAnywhere, Category="Components|CustomAssetMesh")
	TObjectPtr<UStaticMesh> StaticMesh;

	UPROPERTY()
	TObjectPtr<UMaterialInterface> Material;

	//~ Begin UPrimitiveComponent Interface
	ENGINE_API virtual FPrimitiveSceneProxy* CreateSceneProxy() override;
	ENGINE_API virtual FBoxSphereBounds CalcBounds(const FTransform& LocalToWorld) const override;
	ENGINE_API virtual int32 GetNumMaterials() const override;
	ENGINE_API virtual void GetUsedMaterials(TArray<UMaterialInterface*>& OutMaterials, bool bGetDebugMaterials = false) const override;
	//~ End UPrimitiveComponent Interface

};
```

```cpp
#include "Components/MyCustomMeshComponent.h"

#include "DataDrivenShaderPlatformInfo.h"
#include "LandscapeGizmoActiveActor.h"
#include "MaterialDomain.h"
#include "MeshMaterialShader.h"
#include "Materials/MaterialRenderProxy.h"
#include "Serialization/JsonTypes.h"

bool FMyCustomVertexFactory::ShouldCompilePermutation(const FVertexFactoryShaderPermutationParameters& Parameters)
{
	return RHISupportsManualVertexFetch(Parameters.Platform);
}
IMPLEMENT_VERTEX_FACTORY_TYPE(FMyCustomVertexFactory, "/Engine/Private/MyCustomVertexFactory.ush", EVertexFactoryFlags::UsedWithMaterials)

class FMyCustomSceneProxy final : public FPrimitiveSceneProxy
{
public:
	SIZE_T GetTypeHash() const override
	{
		static size_t UniquePointer;
		return reinterpret_cast<size_t>(&UniquePointer);
	}
	
	FMyCustomSceneProxy(UMyCustomMeshComponent* InComponent)
		: FPrimitiveSceneProxy(InComponent), bIsVisible(InComponent->bIsVisible), Material(	InComponent->Material),
	MaterialRelevance(InComponent->GetMaterialRelevance(GetScene().GetFeatureLevel())), StaticMesh(InComponent->StaticMesh),
	VertexFactory(GetScene().GetFeatureLevel(), "FCustomVertexFactory"),
	Component(InComponent)
	{
		if (StaticMesh)
		{
			UpdateStaticMesh(StaticMesh);
		}
	}

	virtual void GetDynamicMeshElements(const TArray<const FSceneView*>& Views, const FSceneViewFamily& ViewFamily, uint32 VisibilityMap, FMeshElementCollector& Collector) const override
	{
		if (!bIsVisible || !StaticMesh)
		{
			return;
		}

		// 生成MaterialProxy和线框
		const bool bIsWireframe = AllowDebugViewmodes() && ViewFamily.EngineShowFlags.Wireframe;
		FMaterialRenderProxy* WireframeMaterial = nullptr;
		if (bIsWireframe)
		{
			WireframeMaterial = new FColoredMaterialRenderProxy(
				GEngine->WireframeMaterial->GetRenderProxy(),
				FLinearColor(0.2f, 0.5f, 1.0f));

			Collector.RegisterOneFrameMaterialProxy(WireframeMaterial);
		}
		
		if (StaticMesh)
		{
			const FStaticMeshRenderData* RenderData = StaticMesh->GetRenderData();
			// 假设只有LOD0
			for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
			{
				const FSceneView* View = Views[ViewIndex];
				if (IsShown(View) && VisibilityMap & (1 << ViewIndex))
				{
					for (int32 LODIndex = 0; LODIndex < RenderData->LODResources.Num(); LODIndex++)
					{
						const FStaticMeshLODResources& LODModel = RenderData->LODResources[LODIndex];
						for (int32 SectionIndex = 0; SectionIndex < LODModel.Sections.Num(); SectionIndex++)
						{
							UMaterialInterface* SectionMaterial = Component->GetMaterial(LODModel.Sections[SectionIndex].MaterialIndex) == nullptr ? UMaterial::GetDefaultMaterial(MD_Surface): Component->GetMaterial(SectionIndex);
							FMaterialRenderProxy* MaterialRenderProxy = bIsWireframe ? WireframeMaterial : SectionMaterial->GetRenderProxy();

							FMeshBatch& MeshBatch = Collector.AllocateMesh();
							FMeshBatchElement& BatchElement = MeshBatch.Elements[0];
							BatchElement.IndexBuffer = &LODModel.IndexBuffer;
							MeshBatch.bWireframe = bIsWireframe;
							MeshBatch.VertexFactory = &VertexFactory;
							MeshBatch.MaterialRenderProxy = MaterialRenderProxy;

							bool bHasPrecomputedVolumetricLightMap;
							FMatrix PreviousLocalToWorldMatrix;
							int32 SingleCaptureIndex;
							bool bOutputVelocity;
							GetScene().GetPrimitiveUniformShaderParameters_RenderThread(GetPrimitiveSceneInfo(), bHasPrecomputedVolumetricLightMap, PreviousLocalToWorldMatrix, SingleCaptureIndex, bOutputVelocity);
							FDynamicPrimitiveUniformBuffer& DynamicPrimitiveUniformBuffer = Collector.AllocateOneFrameResource<FDynamicPrimitiveUniformBuffer>();
							DynamicPrimitiveUniformBuffer.Set(Collector.GetRHICommandList(), GetLocalToWorld(), PreviousLocalToWorldMatrix, GetBounds(), GetLocalBounds(), true, bHasPrecomputedVolumetricLightMap, bOutputVelocity);
							BatchElement.PrimitiveUniformBufferResource = &DynamicPrimitiveUniformBuffer.UniformBuffer;

							BatchElement.FirstIndex = 0;
							BatchElement.NumPrimitives = LODModel.IndexBuffer.GetNumIndices() / 3;
							BatchElement.MinVertexIndex = 0;
							BatchElement.MaxVertexIndex = LODModel.VertexBuffers.PositionVertexBuffer.GetNumVertices() - 1;
							MeshBatch.ReverseCulling = IsLocalToWorldDeterminantNegative();
							MeshBatch.Type = PT_TriangleList;
							MeshBatch.DepthPriorityGroup = SDPG_World;
							MeshBatch.bCanApplyViewModeOverrides = false;
							Collector.AddMesh(ViewIndex, MeshBatch);
						}
					}
				}
			}
		}
	}

	virtual FPrimitiveViewRelevance GetViewRelevance(const FSceneView* View) const override
	{
		FPrimitiveViewRelevance Result;
		Result.bDrawRelevance = IsShown(View) && bIsVisible;
		Result.bShadowRelevance = false;
		Result.bDynamicRelevance = true;
		Result.bRenderInMainPass = true;
		Result.bUsesLightingChannels = false;
		Result.bRenderCustomDepth = false;
		
		MaterialRelevance.SetPrimitiveViewRelevance(Result);
		
		return Result;
	}

	virtual void CreateRenderThreadResources(FRHICommandListBase& RHICmdList) override
	{
		VertexFactory.InitResource(RHICmdList);
	}

	void UpdateStaticMesh(UStaticMesh* InMesh)
	{
		if(InMesh)
		{
			StaticMesh = InMesh;
			VertexFactory.VertexBuffer = &StaticMesh->GetRenderData()->LODResources[0].VertexBuffers.PositionVertexBuffer;
		}
	}
	
	virtual uint32 GetMemoryFootprint(void) const override { return(sizeof(*this) + GetAllocatedSize()); }
    	
	uint32 GetAllocatedSize(void) const { return( FPrimitiveSceneProxy::GetAllocatedSize() ); }

private:
	bool bIsVisible;

	UMaterialInterface* Material;
	UStaticMesh* StaticMesh;
	FMaterialRelevance MaterialRelevance;
	FMyCustomVertexFactory VertexFactory;
	UMyCustomMeshComponent* Component;
};


FPrimitiveSceneProxy* UMyCustomMeshComponent::CreateSceneProxy()
{
	return new FMyCustomSceneProxy(this);
}

FBoxSphereBounds UMyCustomMeshComponent::CalcBounds(const FTransform& LocalToWorld) const
{
	if (StaticMesh != nullptr)
	{
		// Graphics bounds.
		FBoxSphereBounds NewBounds = StaticMesh->GetBounds().TransformBy(LocalToWorld);
		NewBounds.BoxExtent *= BoundsScale;
		NewBounds.SphereRadius *= BoundsScale;

		return NewBounds;
	}
	else
	{
		return FBoxSphereBounds(LocalToWorld.GetLocation(), FVector::ZeroVector, 0.f);
	}
}

int32 UMyCustomMeshComponent::GetNumMaterials() const
{
	return 1;
}

void UMyCustomMeshComponent::GetUsedMaterials(TArray<UMaterialInterface*>& OutMaterials, bool bGetDebugMaterials) const
{
	if (Material!=nullptr)
	{
		OutMaterials.Add(Material);
	}
}
```