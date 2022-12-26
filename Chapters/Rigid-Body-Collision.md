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

触发器形状在场景的模拟中不发挥任何作用（尽管它们可以被配置为参与场景查询）。相反，它们的作用是报告与另一个形状的重叠情况。接触报告并不会为交叉点生成接触，因此接触报告不适用于触发器形状。此外，由于触发器在模拟中不发挥作用，SDK 不允许 PxShapeFlag::eSIMULATION_SHAPE 和 PxShapeFlag::eTRIGGER_SHAPE 标志同时被配置；也就是说，如果一个标志被配置，那么试图配置另一个标志将会被拒绝并产生一个错误流。

触发器形状可以用来实现传感器。例如，它们可以被用来确定玩家是否已经到达检查点区域，或者当一个物体在门前移动时自动打开门。在这样的例子中，检查点或门周围的空间区域将被表示为一个具有独特形状（通常是一个盒子或一个球体）的角色，该角色被配置为一个触发器形状：

```cpp
PxShape* sensorShape;
gSensorActor->getShapes(&sensorShape, 1);
sensorShape->setFlag(PxShapeFlag::eSIMULATION_SHAPE, false);
sensorShape->setFlag(PxShapeFlag::eTRIGGER_SHAPE, true);
```

通过用户定义的 PxSimulationEventCallback 对象，特别是通过 PxSimulationEventCallback::onTrigger() 的实现，报告角色与触发器的重叠情况：

```cpp
void MySimulationEventCallback::onTrigger(PxTriggerPair* pairs, PxU32 count)
{
    for(PxU32 i=0; i<count; i++)
    {
        // ignore pairs when shapes have been deleted
        if(pairs[i].flags & (PxTriggerPairFlag::eREMOVED_SHAPE_TRIGGER | PxTriggerPairFlag::eREMOVED_SHAPE_OTHER))
            continue;

        // Detect for example that a player entered a checkpoint zone
        if((&pairs[i].otherShape->getActor() == gPlayerActor) &&
            (&pairs[i].triggerShape->getActor() == gSensorActor))
        {
            gCheckpointReached = true;
        }
    }
}
```

上面的代码遍历了所有涉及触发器形状的重叠形状对。如果发现玩家触及了检查点传感器，那么标志 gCheckpointReached 被设置为 true。

## Broad-phase Collision Detection

广义阶段是碰撞管道的第一部分。它被称为广义阶段是因为它检测轴对齐后的边投影之间的重叠情况，即它只报告潜在的碰撞而不是实际的碰撞。实际的碰撞是由物理学管道的下一个阶段检测的，名为狭义阶段。

### Broad-phase Algorithms

PhysX支持多种广义阶段的碰撞算法：

- sweep-and-prune (SAP)
- multi box pruning (MBP)
- automatic box pruning (ABP)
- parallel automatic box pruning (PABP)
- GPU broadphase (GPU)

PxBroadPhaseType::eSAP 是一个很好的通用选择，当许多对象处于睡眠状态时性能很好。不过，当所有的物体都在移动，或者大量的物体被加入或移出广义阶段时，性能会明显下降。这个算法不需要定义世界边界就可以工作。

PxBroadPhaseType::eMBP 是 PhysX 3.3 中引入的一种算法。它是一种替代性的广义阶段算法，当所有物体都在移动或插入大量物体时，不会出现 eSAP 那样的性能问题。然而，当许多物体处于睡眠状态时，其通用性能可能不如 eSAP，而且它需要用户定义世界边界（broadphase regions）才能工作。

PxBroadPhaseType::eABP 是 PhysX 4 中引入的 PxBroadPhaseType::eMBP 的重新实现，它自动管理世界边界和广义阶段区域，从而提供 PxBroadPhaseType::eSAP 的便利性，再加上 PxBroadPhaseType::eMBP 的性能。虽然 PxBroadPhaseType::eSAP 在大多数对象处于睡眠状态时可以保持较快的速度，而 PxBroadPhaseType::eMBP 在使用大量正确定义的区域时可以保持较快的速度，但 PxBroadPhaseType::eABP 往往在平均性能和内存使用上都是最好的。它是广义阶段的一个很好的默认选择。

PxBroadPhaseType::ePABP 是 PhysX 5 中引入的 PxBroadPhaseType::eABP 的重新实现。 它与 PxBroadPhaseType::eABP 相同，但利用了多线程的优势。因为单线程的 ABP 实现本身就非常快，其多线程版本只对大型场景更快，对小型场景不一定，且它也会使用更多的内存。

PxBroadPhaseType::eGPU 是增量扫描和修剪方法的一个 GPU 实现。此外，它使用了 ABP 风格的初始对生成方法，以避免插入形状时出现大峰值。它不仅具有传统 SAP 方法的优势，在许多对象处于睡眠状态时很好，而且由于是完全并行的，在大量形状移动或运行时对插入和删除的支持也很好。

所需的广义阶段算法是由 PxSceneDesc::broadPhaseType 控制的。

### Regions of Interest

一个 region of interest 是一个世界空间的AABB包围盒，围绕着一个由广义阶段控制的空间体积。包含在这些区域内的物体会被广义阶段正确处理。落在这些区域之外的物体将失去所有的碰撞检测。理想情况下，这些区域应该覆盖整个模拟空间，同时限制所覆盖的空白空间的数量。

区域之间可以重叠，但为了获得最大的效率，建议尽可能地减少区域之间的重叠。请注意，AABBs 刚好接触的两个区域不被认为是重叠的。例如，PxBroadPhaseExt::createRegionsFromWorldBounds() 辅助函数通过简单地将一个给定世界的 AABB 细分为一个规则的二维网格来创建一些不重叠的区域边界。

区域可以由 PxBroadPhaseRegion 结构和分配给它们的用户数据来定义。它们可以在场景创建或运行时使用 PxScene::addBroadPhaseRegion() 来定义。SDK 会返回分配给新创建的区域的句柄，以后可以使用PxScene::removeBroadPhaseRegion() 来删除这些区域。

新添加的区域可能会与已有的对象重叠。如果 PxScene::addBroadPhaseRegion() 调用中的 populateRegion 参数被设置，SDK可以自动将这些对象添加到新区域。但是这个操作并相当昂贵，而且可能会对性能产生很大的影响，尤其是在同一帧中添加多个区域时。因此，建议尽可能地禁用它。这样区域就会被创建为空，并且只会把区域创建后被添加到场景中的对象填充进去，或者在更新时（即移动时）填充之前已经创建并存在的对象。

注意，只有 PxBroadPhaseType::eMBP 需要定义区域，其他算法则不需要。这些信息被记录在 PxBroadPhaseCaps 结构中，该结构列出了每个算法的信息和能力。这个结构可以通过调用 PxScene::getBroadPhaseCaps() 来检索。

关于当前区域的运行时信息可以通过 PxScene::getNbBroadPhaseRegions() 和 PxScene::getBroadPhaseRegions() 函数进行检索。

目前区域的最大数量被限制在256个。

### Broad-phase Callback

可以在 PxSceneDesc 构中定义一个与广义阶段有关的事件回调。这个 PxBroadPhaseCallback 对象将在发现物体超出指定的关注区域时被调用，也就是"出界"。SDK 将禁用这些物体的碰撞检测。一旦物体重新进入一个有效的区域，它就会自动重新启用。

用户可以自行决定如何处理界外物体。典型的选择是：

- 删除这些物体
- 让它们继续运动而不发生碰撞，直到它们重新进入一个有效区域
- 人为地将它们传送回一个有效位置

这个回调主要用于 PxBroadPhaseType::eMBP。

## Interactions

SDK 会在内部为广义阶段报告的每一对重叠物体创建一个交互对象。这些对象不仅是为碰撞的刚体对创建，也为重叠的触发器对创建。一般来说，用户应该认为这些对象的创建与所涉及的对象类型（刚体、触发器等）和所涉及的 PxFilterFlag 标志无关。

PhysX 广义阶段的操作对象是形状，而不是角色。这意味着，一个交互是为一对形状创建的，而两个碰撞的复杂角色可以在内部产生多个交互对象。在这种情况下，可以使用聚合来减少交互次数（见Aggregates）。

## Collision Filtering

todo...
