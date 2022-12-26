# Rigid Body Collision

## Introduction

本节将介绍刚体碰撞的基本原理。对于更高级的主题，请参考 Advanced Collision Detection 一节。

## Shapes

形状描述了角色的空间范围和碰撞属性。它们在 PhysX 中有三种用途。

- 交集查询，确定刚性物体的接触特征。
- 场景查询，如 raycast，overlap，sweep。
- 定义触发体，当其他形状与之相交时产生通知。

形状是具有引用计数的特点，详见 Reference Counting。

每个形状都包含一个 PxGeometry 对象和一个对 PxMaterial 的引用，它们都必须在创建时被指定。下面的代码创建了一个具有球形几何体和特定材料的形状。

```cpp
PxShape* shape = physics.createShape(PxSphereGeometry(1.0f), myMaterial, true);
myActor.attachShape(*shape);
shape->release();
```

PxRigidActorExt::createExclusiveShape() 方法等同于上面三行代码。

> **Note**
> 关于反序列化形状的引用计数，请参考 Reference Counting of Deserialized Objects。

PxPhysics::createShape() 的参数 "true" 告诉SDK，该形状将不会与其他角色共享。当你有许多具有相同几何形状的角色时，你可以使用形状共享来减少模拟的内存成本，但共享形状有一个非常强的限制：当共享形状连接到一个角色时，你不能更新它的属性。

你可以通过指定 PxShapeFlags 类型的形状标志来配置一个形状。默认情况下，一个形状被配置为：

- 模拟形状（在模拟过程中启用生成接触）
- 场景查询形状（为场景查询启用）
- 如果启用了调试渲染，则被可视化

当为一个形状指定一个几何对象时，该几何对象被复制到该形状中。根据形状标志和父角色的类型，对形状可以指定哪些几何体有一些限制。

- 对于附加到动态角色的模拟形状，不支持 TriangleMesh、HeightField、Plane 几何体，除非动态角色被配置为运动学（kinematic）角色。
- TriangleMesh、HeightField 的几何形状不支持触发形状。

更多细节请看下面的章节。

将形状从Actor上分离，方法如下：

```cpp
myActor.detachShape(*shape);
```

### Simulation Shapes and Scene Query Shapes

形状可以被独立地配置为参与场景查询或接触测试的其中之一。这由 PxShapeFlag::eSIMULATION_SHAPE 和 PxShapeFlag::eSCENE_QUERY_SHAPE 控制。默认情况下，一个形状将同时打开这两种配置。

下面的伪代码配置了一个 PxShape 实例，使其不再参与形状之间的交叉测试：

```cpp
void disableShapeInContactTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSIMULATION_SHAPE, false);
}
```

PxShape 实例可以被配置为参与形状之间的交叉测试，如下所示：

```cpp
void enableShapeInContactTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSIMULATION_SHAPE, true);
}
```

最后，一个 PxShape 实例可以在场景查询测试中被重新启用：

```cpp
void enableShapeInSceneQueryTests(PxShape* shape)
{
    shape->setFlag(PxShapeFlag::eSCENE_QUERY_SHAPE, true);
}
```

> **Note**
> 如果形状的角色移动根本不需要被模拟控制，也就是说，形状只用于场景查询，且只在必要时手动移动，那么可以通过额外禁用角色本身的模拟来节省内存（见 PxActorFlag::eDISABLE_SIMULATION）。

### Kinematic Triangle Meshes (Planes, Heightfields)

可以创建一个基于运动学的 PxRigidDynamic，它可以有一个三角形网格（平面，高度场）形状。如果这个形状具有模拟形状的标志，则这个角色必须保持动态。如果你把标志改为不模拟，你甚至可以切换到运动学标志。

> **个人理解**
> 此处意为，如果创建了一个打开模拟标志位的角色，则这个角色将会根据物理参数进行运动模拟，所以翻译为保持动态；如果关闭了模拟标志位，则这个角色将不会进行运动模拟，至于运动学标志位，会在 Rigid Body Dynamics 中介绍。

要设置运动学的三角形网格，请看下面的代码：

```cpp
PxRigidDynamic* meshActor = getPhysics().createRigidDynamic(PxTransform(1.0f));
PxShape* meshShape;
if(meshActor)
{
    meshActor->setRigidDynamicFlag(PxRigidDynamicFlag::eKINEMATIC, true);

    PxTriangleMeshGeometry triGeom;
    triGeom.triangleMesh = triangleMesh;
    meshShape = PxRigidActorExt::createExclusiveShape(*meshActor,triGeom,
        defaultMaterial);
    getScene().addActor(*meshActor);
}
```

要将运动学三角形网格角色切换为动态角色：

> **个人理解**
> 此处代码似乎不完整？

```cpp
PxRigidDynamic* meshActor = getPhysics().createRigidDynamic(PxTransform(1.0f));
PxShape* meshShape;
if(meshActor)
{
    meshActor->setRigidDynamicFlag(PxRigidDynamicFlag::eKINEMATIC, true);

    PxTriangleMeshGeometry triGeom;
    triGeom.triangleMesh = triangleMesh;
    meshShape = PxRigidActorExt::createExclusiveShape(*meshActor, triGeom,
        defaultMaterial);
    getScene().addActor(*meshActor);

    PxConvexMeshGeometry convexGeom = PxConvexMeshGeometry(convexBox);
    convexShape = PxRigidActorExt::createExclusiveShape(*meshActor, convexGeom,
        defaultMaterial);
```

### Dynamic Triangle Meshes with SDFs

动态三角形网格只有在具有SDF（Signed Distance Field）时才被支持。SDF 可以选择性地在 cooking 过程中生成。建议使用稀疏 SDF，因为在其最简单的形式（密集SDF）下，会使用大量的内存。稀疏和密集 SDF 的碰撞检测性能几乎是相同的。请注意，在静态角色上，SDFs 不被支持，而在运动学和动态角色上，它们在大多数情况下运行良好。在场景中添加动态三角形网格的工作原理与添加运动学网格基本相同。主要的区别是，PxTriangleMesh 必须是用 SDF 制作的，质量和其他动态相关的属性必须用有效的值填充，因为它们对动态物体的行为很重要：

```cpp
void addDynamicTriangleMeshInstance(const PxTransform& transform, PxTriangleMesh* mesh)
{
    PxRigidDynamic* dyn = gPhysics->createRigidDynamic(transform);
    dyn->setLinearDamping(0.2f);
    dyn->setAngularDamping(0.1f);
    PxTriangleMeshGeometry geom;
    geom.triangleMesh = mesh;
    geom.scale = PxVec3(0.1f, 0.1f, 0.1f);

    dyn->setRigidBodyFlag(PxRigidBodyFlag::eENABLE_GYROSCOPIC_FORCES, true);
    dyn->setRigidBodyFlag(PxRigidBodyFlag::eENABLE_SPECULATIVE_CCD, true);

    PxShape* shape = PxRigidActorExt::createExclusiveShape(*dyn, geom, *gMaterial);
    shape->setContactOffset(0.1f);
    shape->setRestOffset(0.02f);

    PxReal density = 100.f;
    PxRigidBodyExt::updateMassAndInertia(*dyn, density);

    gScene->addActor(*dyn);

    dyn->setSolverIterationCounts(50, 1);
    dyn->setMaxDepenetrationVelocity(5.f);
}
```

如果 SDF 碰撞没有产生令人满意的结果，通常可以通过调整一些参数来改善这种情况。

- 增加接触偏移
  - 较大的接触偏移允许求解器对可能发生在后续时间段的碰撞做出更早的反应。过大的接触偏移量会减慢碰撞检测的性能，所以这通常是一个反复寻找较优数值的过程
- 在合理的范围内，让物体更重
  - 重量极轻的物体或质量相差很大的物体接触时，碰撞响应可能不完美，因为这些情况使求解器的收敛更加困难
- 调整 SDF 的分辨率
  - 过低的 SDF 分辨率可能会导致网格中非常薄的部分不发生碰撞，因为 SDF 无法表示/捕捉到它们。
  - 过高的 SDF 分辨率可能会导致内存消耗增加，碰撞检测性能也会变慢。
- 降低最大的冲刺速度
  - 使得碰撞响应速度稍慢，可以帮助避免过冲。
- 增加位置迭代的次数
  - 给予求解器更多的迭代次数以提高收敛性
- 增加摩擦力
  - 有助于接触的物体达到静止状态
- 在生成 SDF 时使用具有良好嵌片的网格
  - 没有自交点或其他缺陷的密闭三角形网格可以简化网格烘焙过程。如果网格有孔，烘焙会将其封闭，但封闭面将由网格烘焙器定义。
- 减少物理模拟的时间步长
  - 只有在所有其他调整都不能带来好结果的情况下，才建议这样做。

### Trigger Shapes

todo...
