SplineMeshSceneProxy.h中

利用宏DECLARE_VERTEX_FACTORY_TYPE
```cpp
DECLARE_VERTEX_FACTORY_TYPE(FSplineMeshVertexFactory);
```

宏的内容如下：
```cpp
/**
 * A macro for declaring a new vertex factory type, for use in the vertex factory class's definition body.
 */
#define DECLARE_VERTEX_FACTORY_TYPE(FactoryClass) \
	public: \
	static FVertexFactoryType StaticType; \
	virtual FVertexFactoryType* GetType() const override;
```

还有一个静态函数
```cpp
static FVertexFactoryShaderParameters* ConstructShaderParameters(EShaderFrequency ShaderFrequency);
```

SplineMeshComponent.cpp中

利用宏IMPLEMENT_VERTEX_FACTORY_TYPE
```cpp
IMPLEMENT_VERTEX_FACTORY_TYPE(FSplineMeshVertexFactory, "/Engine/Private/LocalVertexFactory.ush", true, true, true, true, true);
```

宏的内容如下：
```cpp
/**
 * A macro for implementing the static vertex factory type object, and specifying parameters used by the type.
 * @param bUsedWithMaterials - True if the vertex factory will be rendered in combination with a material.
 * @param bSupportsStaticLighting - True if the vertex factory will be rendered with static lighting.
 */
#define IMPLEMENT_VERTEX_FACTORY_TYPE(FactoryClass,ShaderFilename,bUsedWithMaterials,bSupportsStaticLighting,bSupportsDynamicLighting,bPrecisePrevWorldPos,bSupportsPositionOnly) \
	FVertexFactoryShaderParameters* Construct##FactoryClass##ShaderParameters(EShaderFrequency ShaderFrequency) { return FactoryClass::ConstructShaderParameters(ShaderFrequency); } \
	FVertexFactoryType FactoryClass::StaticType( \
		TEXT(#FactoryClass), \
		TEXT(ShaderFilename), \
		bUsedWithMaterials, \
		bSupportsStaticLighting, \
		bSupportsDynamicLighting, \
		bPrecisePrevWorldPos, \
		bSupportsPositionOnly, \
		Construct##FactoryClass##ShaderParameters, \
		FactoryClass::ShouldCompilePermutation, \
		FactoryClass::ModifyCompilationEnvironment, \
		FactoryClass::SupportsTessellationShaders \
		); \
		FVertexFactoryType* FactoryClass::GetType() const { return &StaticType; }
```

对ConstructShaderParameters函数进行封装，初始化了之前声明的静态成员StaticType，封装后的函数Construct##FactoryClass##ShaderParameters被存储到FVertexFactoryType类型的StaticType中