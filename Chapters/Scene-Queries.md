# Scene Queries

## Introduction

PhysX 在 PxScene 中提供了对场景中的角色和附属形状进行碰撞查询的方法。有三种类型的查询。Raycasts、Sweeps 和 Overlaps，每一种都可以返回单个或多个结果。广义上讲，每个查询都会遍历一个包含场景对象的剔除结构（culling structure）（又称修剪结构），使用几何查询函数（见 Geometry Queries）进行精确测试，并累积结果。过滤可能发生在精确测试之前或之后。

场景使用两个不同的查询结构，一个用于 PxRigidStatic 角色，另一个用于 PxRigidBody 角色（PxRigidDynamic 和 PxArticulationLink）。这两个结构可以被配置为使用不同的剔除（culling）实现，具体取决于所需的速度/空间特性（见 PxPruningStructureType）。

## Basic queries

### Raycasts

PxScene::raycast() 查询将用户定义的射线与整个场景相交。Raycast query 最简单的用例是沿着给定的射线找到最近的命中，如下所示：

```cpp
PxScene* scene;
PxVec3 origin = ...;                 // [in] Ray origin
PxVec3 unitDir = ...;                // [in] Normalized ray direction
PxReal maxDistance = ...;            // [in] Raycast max distance
PxRaycastBuffer hit;                 // [out] Raycast results

// Raycast against all static & dynamic objects (no filtering)
// The main result from this call is the closest hit, stored in the 'hit.block' structure
bool status = scene->raycast(origin, unitDir, maxDistance, hit);
if (status)
    applyDamage(hit.block.position, hit.block.normal);
```

在这段代码中，一个 PxRaycastBuffer 对象被用来接收 raycast 查询的结果。如果命中则 PxScene::raycast() 方法返回 true。如果命中则 hit.hadBlock 也被设置为 true。Raycasts 的距离必须在[0, inf)范围内。

Raycasts 的结果包括位置、法线、命中距离、形状和角色，以及一个带有UV坐标的三角形网格和高度场的面索引。在使用查询结果之前，首先检查 PxHitFlag::ePOSITION、PxHitFlag::eNORMAL、PxHitFlag::eUV 标志，以确保相应的数据可用。例如，如果不需要冲击位置和法线的计算，可以跳过这些计算。用户提供的 PxHitFlags 控制查询过程中的计算内容。

请注意，场景级的射线广播查询返回 PxRaycastHit 结构，而对象级的射线广播查询返回 PxGeomRaycastHit 命中。区别仅仅在于 PxRaycastHit 被增加了 PxRigidActor 和 PxShape 指针。

### Sweeps

todo...
