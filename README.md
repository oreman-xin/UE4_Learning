# UE4_Learning
UE渲染模块源码阅读的简单记录，目前看的比较粗，有些地方可能有错误，主要是了解下源码中各个类是如何使用和交互的。

## UPrimitiveComponent被添加到FScene

每个UPrimitiveComponent被添加到UWorld中后，会将当前Component渲染相关的信息添加到FScene的各种DrawList中。见[UE4_AddPrimitive](UE4_AddPrimitive.md)。

## 渲染时如何调用

记录了[BassPass](UE4_BassPass.md)和[InitViews](UE4_Render.md)