<!-- TOC -->

- [Component-Constructor](#component-constructor)
- [Render-InitViews](#render-initviews)

<!-- /TOC -->

FMeshBatch中保存着VertexFactory，FLocalVertexFactory类中存储Data的是FDataType，继承自FStaticMeshDataType
```cpp
struct FDataType : public FStaticMeshDataType
{
};

FDataType Data;
```
FStaticMeshData中以FVertexStreamComponent存储顶点信息
```cpp
struct FStaticMeshDataType
{
	/** The stream to read the vertex position from. */
	FVertexStreamComponent PositionComponent;

	/** The streams to read the tangent basis from. */
	FVertexStreamComponent TangentBasisComponents[2];

	/** The streams to read the texture coordinates from. */
	TArray<FVertexStreamComponent, TFixedAllocator<MAX_STATIC_TEXCOORDS / 2> > TextureCoordinates;

	...
};
```
FVertexStreamComponent中保存着FVertexBuffer信息
```cpp
/** The vertex buffer to stream data from.  If null, no data can be read from this stream. */
const FVertexBuffer* VertexBuffer = nullptr;
```
## Component-Constructor

在SceneProxy的构造函数中，调用InitWithDummyData将VertexBuffers与VertexFactory进行绑定
* Constructor
```cpp
FXXSceneProxy(FXXComponent* Component)
{
    ...
    VertexBuffers.InitWithDummyData(&VertexFactory, GetRequiredVertexCount());
    ...
}
```
InitWithDummyData中进行bind
* FStaticMeshVertexBuffers::InitWithDummyData
```cpp
void FStaticMeshVertexBuffers::InitWithDummyData(FLocalVertexFactory* VertexFactory, uint32 NumVerticies, uint32 NumTexCoords, uint32 LightMapIndex)
{
    ...
	FLocalVertexFactory::FDataType Data;
	Self->PositionVertexBuffer.BindPositionVertexBuffer(VertexFactory, Data);
	Self->StaticMeshVertexBuffer.BindTangentVertexBuffer(VertexFactory, Data;
	Self->StaticMeshVertexBuffer.BindPackedTexCoordVertexBuffe(VertexFactory, Data);
	Self->StaticMeshVertexBuffer.BindLightMapVertexBuffer(VertexFactory,Data, LightMapIndex);
	Self->ColorVertexBuffer.BindColorVertexBuffer(VertexFactory, Data);
	VertexFactory->SetData(Data);
    ...
}
```

## Render-InitViews
FDeferredShadingSceneRenderer::Render中，InitViews阶段会对每个View初始化RHIResources
```cpp
{
	QUICK_SCOPE_CYCLE_COUNTER(STAT_InitViews_InitRHIResources);
	// initialize per-view uniform buffer.
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		FViewInfo& View = Views[ViewIndex];

		View.ForwardLightingResources = View.ViewState ? &View.ViewState->ForwardLightingResources : &View.ForwardLightingResourcesStorage;

		// Possible stencil dither optimization approach
		View.bAllowStencilDither = bDitheredLODTransitionsUseStencil;

		// Set the pre-exposure before initializing the constant buffers.
		if (View.ViewState)
		{
			View.ViewState->UpdatePreExposure(View);
		}

		// Initialize the view's RHI resources.
		View.InitRHIResources();
	}
}
```

在每个View的ViewInfo中存储着绘制所需的DynamicResources，FViewInfo::InitRHIResources中依次对它们进行初始化
```cpp
// Initialize the dynamic resources used by the view's FViewElementDrawer.
for (int32 ResourceIndex = 0; ResourceIndex < DynamicResources.Num(); ResourceIndex++)
{
	DynamicResources[ResourceIndex]->InitPrimitiveResource();
}
```

各种DynamicResources有着不同的初始化过程，以FPooledDynamicMeshVertexFactory为例，它重写了父类FRenderResource的InitResource方法，在InitResource中，调用渲染线程，利用对应的VertexBuffer信息初始化自身（继承自FLocalVertexFactory）
```cpp
ENQUEUE_RENDER_COMMAND(InitDynamicMeshVertexFactory)(
	[VertexFactory, PooledVertexBuffer](FRHICommandListImmediate& RHICmdList)
{
	FDataType Data;
	Data.PositionComponent = FVertexStreamComponent(
		&PooledVertexBuffer->PositionBuffer,
		0,
		sizeof(FVector),
		VET_Float3
	);

	Data.NumTexCoords = PooledVertexBuffer->GetNumTexCoords();
	{
		Data.LightMapCoordinateIndex = PooledVertexBuffer->GetLightmapCoordinateIndex();
		Data.TangentsSRV = PooledVertexBuffer->TangentBufferSRV;
		Data.TextureCoordinatesSRV = PooledVertexBuffer->TexCoordBufferSRV;
		Data.ColorComponentsSRV = PooledVertexBuffer->ColorBufferSRV;
		Data.PositionComponentSRV = PooledVertexBuffer->PositionBufferSRV;
	}

	{
		EVertexElementType UVDoubleWideVertexElementType = VET_None;
		EVertexElementType UVVertexElementType = VET_None;
		uint32 UVSizeInBytes = 0;
		if (PooledVertexBuffer->GetUse16bitTexCoords())
		{
			UVSizeInBytes = sizeof(FVector2DHalf);
			UVDoubleWideVertexElementType = VET_Half4;
			UVVertexElementType = VET_Half2;
		}
		else
		{
			UVSizeInBytes = sizeof(FVector2D);
			UVDoubleWideVertexElementType = VET_Float4;
			UVVertexElementType = VET_Float2;
		}

		int32 UVIndex;
		uint32 UvStride = UVSizeInBytes * PooledVertexBuffer->GetNumTexCoords();
		for (UVIndex = 0; UVIndex < (int32)PooledVertexBuffer->GetNumTexCoords() - 1; UVIndex += 2)
		{
			Data.TextureCoordinates.Add
			(
				FVertexStreamComponent(
					&PooledVertexBuffer->TexCoordBuffer,
					UVSizeInBytes * UVIndex,
					UvStride,
					UVDoubleWideVertexElementType,
					EVertexStreamUsage::ManualFetch
				)
			);
		}

		// possible last UV channel if we have an odd number
		if (UVIndex < (int32)PooledVertexBuffer->GetNumTexCoords())
		{
			Data.TextureCoordinates.Add(FVertexStreamComponent(
				&PooledVertexBuffer->TexCoordBuffer,
				UVSizeInBytes * UVIndex,
				UvStride,
				UVVertexElementType,
				EVertexStreamUsage::ManualFetch
			));
		}

		Data.TangentBasisComponents[0] = FVertexStreamComponent(&PooledVertexBuffer->TangentBuffer, 0, 2 * sizeof(FPackedNormal), VET_PackedNormal, EVertexStreamUsage::ManualFetch);
		Data.TangentBasisComponents[1] = FVertexStreamComponent(&PooledVertexBuffer->TangentBuffer, sizeof(FPackedNormal), 2 * sizeof(FPackedNormal), VET_PackedNormal, EVertexStreamUsage::ManualFetch);
		Data.ColorComponent = FVertexStreamComponent(&PooledVertexBuffer->ColorBuffer, 0, sizeof(FColor), VET_Color, EVertexStreamUsage::ManualFetch);
	}
	VertexFactory->SetData(Data);
});
```

然后调用父类的FLocalVertexFactory::InitResource()继续初始化，将之前初始化的Data（FDataType）封装进FVertexDeclarationElementList类型中，创建vertex declaration
```cpp
void FVertexFactory::InitDeclaration(FVertexDeclarationElementList& Elements)
{
	// Create the vertex declaration for rendering the factory normally.
	Declaration = RHICreateVertexDeclaration(Elements);
}
```

Declaration为FVertexFactory类的成员
```cpp
/** The RHI vertex declaration used to render the factory normally. */
FVertexDeclarationRHIRef Declaration;
```
