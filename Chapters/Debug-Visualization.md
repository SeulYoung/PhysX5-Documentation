# Debug Visualization

## Introduction

通过 OmniPVD（见 Omniverse Visual Debugger），NVIDIA 提供了一个工具来记录模拟 PhysX 场景的信息，并在远程查看器应用程序中对这些信息进行可视化。然而，有时最好是将可视化调试信息直接整合到应用程序的视图中。为此，PhysX 提供了一个接口，可将可视化调试信息提取为一组基本的渲染基元，即点、线、三角形和文本。然后，这些基元可以被渲染并与应用程序的渲染对象叠加。

## Usage

要启用调试可视化，首先必须将全局可视化比例设置为正值：

```cpp
PxScene* scene = ...
scene->setVisualizationParameter(PxVisualizationParameter::eSCALE, 1.0f);
```

然后，需要可视化的各个属性可以同样通过设置正值来启用：

```cpp
scene->setVisualizationParameter(PxVisualizationParameter::eACTOR_AXES, 2.0f);
```

在这个例子中，角色的世界坐标轴将被可视化。用于可视化的比例将是全局比例（本例中为 1.0）和属性比例（本例中为 2.0）的乘积。请注意，对于某些属性来说，比例系数并不适用：例如，形状几何不会被缩放，因为其尺寸是由用户应用程序定义的。此外，对于某些对象，可视化必须在相应的对象实例上显式启用（见 PxActorFlag::eVISUALIZATION、PxShapeFlag::eVISUALIZATION、...）。

在模拟步骤之后，可以按如下方式提取可视化基元：

```cpp
const PxRenderBuffer& rb = scene->getRenderBuffer();

for(PxU32 i=0; i < rb.getNbPoints(); i++)
{
    const PxDebugPoint& point = rb.getPoints()[i];
    // render the point
}

for(PxU32 i=0; i < rb.getNbLines(); i++)
{
    const PxDebugLine& line = rb.getLines()[i];
    // render the line
}

// etc ...
```

> **Note**
> 不要在模拟运行时提取渲染基元。

调试可视化的数据量可能过于庞大，对于大型场景来说，无法有效地创建。在只对局部区域感兴趣的情况下，可以选择通过 PxScene::setVisualizationCullingBox() 使用剔除框进行调试可视化。

注意，简单地启用调试可视化（PxVisualizationParameter::eSCALE）会对性能产生重大影响，即使所有其他单独的可视化标志被禁用。因此，确保调试可视化在你的最终/发布版本中被禁用。
