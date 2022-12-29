# Geometry Queries

## Introduction

本章介绍了如何使用 PhysX 的碰撞功能来处理单个几何对象。有四种主要的几何体查询方式:

- raycasts ("raycast queries") 测试射线与几何对象的关系。(见 Raycasts)
- sweeps（"sweep queries"）沿着一条线移动一个几何对象，以找到与另一个几何对象的第一个交点。(见 Sweeps)
- overlaps（"overlaps queries"）确定两个几何对象是否相交。(见 Overlaps)
- 穿透深度计算（"最小平移距离查询"，在此缩写为"MTD"）测试两个重叠的几何对象，以找到它们可以沿着最小距离分开的方向。(见 Penetration Depth)

此外，PhysX 还提供了计算几何对象的边界（AABB）（见 Bounds computation）和计算点与几何对象之间的距离（见 Point-distance query）的辅助工具。

在以下所有函数中，几何对象是由其形状（PxGeometry 结构）和姿势（PxTransform 结构）定义的。所有的变换和向量都被解释为在同一空间中，结果也在该空间中返回。

## PxGeometryQueryFlags

以下大多数的几何查询都接受一个 PxGeometryQueryFlags 输入参数。在撰写本文时，这只是一个可以通过使用默认的标志值（PxGeometryQueryFlag::eDEFAULT）来进行忽略的小优化，。实验用户可以通过省略 PxGeometryQueryFlag::eSIMD_GUARD 来利用这些标志，前提是在调用代码中手动处理 SSE 控制字。有关示例，请参考 SnippetPathTracing 及其 OPTIM_SKIP_INTERNAL_SIMD_GUARD 定义。

## Raycasts

![Geom Query Raycast](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomQueryRaycast.png)

Raycast 查询沿着一条线段追踪一个点，直到它碰到一个几何对象。除了 PxParticleSystemGeometry、PxTetrahedronMeshGeometry 和 PxHairSystemGeometry 之外，PhysX支持所有几何体类型的 raycasts。

> **Note**
> 对于 PxCustomGeometry，代码被重新路由到用户定义的 PxCustomGeometry::Callbacks。从本质上讲，要由用户来实现自定义几何体的各种几何体查询。

下面的代码说明了如何使用一个简单的 raycast 查询：

```cpp
PxRaycastHit hitInfo;
const PxU32 maxHits = 1;
const PxHitFlags hitFlags = PxHitFlag::ePOSITION|PxHitFlag::eNORMAL|PxHitFlag::eUV;
PxU32 hitCount = PxGeometryQuery::raycast(origin, unitDir,
                                          geom, pose,
                                          maxDist,
                                          hitFlags,
                                          maxHits, &hitInfo);
```

参数的解释如下：

- origin 是射线的起始点。
- unitDir 是一个定义射线方向的单位矢量。
- maxDist 是沿射线搜索的最大距离。它必须在 [0, inf) 范围内。如果最大距离是0，那么只有当射线形状内部开始时才会返回命中，下面将详细说明每个几何体的情况。
- geom 是要测试的几何体。
- pose 是几何体的姿态。
- hitFlags 指定查询应返回的值，以及处理查询的选项。
- maxHits 是要返回的最大命中数。
- hitInfo 是一个 PxRaycastHit 结构体，Raycast 的结果将被存储到该结构中。

从PhysX 5开始，还提供了一个新的 raycast 查询方式：

```cpp
PxGeomRaycastHit hitInfo;
const PxU32 maxHits = 1;
const PxHitFlags hitFlags = PxHitFlag::ePOSITION|PxHitFlag::eNORMAL|PxHitFlag::eUV;
const PxU32 stride = sizeof(PxGeomRaycastHit);
const PxGeometryQueryFlags queryFlags = PxGeometryQueryFlag::eDEFAULT;
PxRaycastThreadContext* threadContext = NULL;
PxU32 hitCount = PxGeometryQuery::raycast(origin, unitDir,
                                          geom, pose,
                                          maxDist,
                                          hitFlags,
                                          maxHits, &hitInfo,
                                          stride, queryFlags, threadContext);
```

todo...
