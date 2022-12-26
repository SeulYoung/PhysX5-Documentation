# Geometry

## Introduction

本节将讨论 PhysX 的几何体类。几何图形用于构建刚体的形状，作为碰撞触发器，以及作为 PhysX 场景查询系统中的体积。PhysX 还提供了独立的函数，用于测试几何体之间的交集，对它们进行 raycast 检测，以及对一个几何体进行 sweep 操作。

几何体是一种值类型，并继承自一个共同的基类**PxGeometry**。每个几何体类都定义了一个具有固定位置和方向的体积或表面。变换（Transform）指定了几何体被解释的框架。对于平面和胶囊体类型，PhysX 提供了辅助函数，以便从常见的替代表示中构建这些变换。

几何图形可分为两类。

- Primitives（PxBoxGeometry, PxSphereGeometry, PxCapsuleGeometry, PxPlaneGeometry），几何对象包含了所有的数据
- 网格或高度场（PxConvexMeshGeometry, PxTriangleMeshGeometry, PxHeightFieldGeometry），其中几何对象包含一个指向更大对象的指针（分别为PxConvexMesh, PxTriangleMesh, PxHeightField）你可以在引用这些对象的每个PxGeometry类型中使用不同的尺度。较大的对象必须使用*cooking*过程来创建，下面对每种类型进行描述。

当传入和传出SDK作为模拟几何时，几何体会被复制到 PxShape 类中。在这种不知道几何体具体类型的情况下检索几何体可能会很棘手，因此 PhysX 提供了一个类似联合的包装类（PxGeometryHolder），可以用来按值传递任何几何体类型。每个网格（或高度场）都有一个引用计数，用于跟踪几何体引用该网格的 PxShapes 的数量。

## Geometry Types

### Spheres

![Geom Type Sphere](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomTypeSphere.png)

一个 PxSphereGeometry 由一个属性指定，即它的半径，并以原点为中心。

### Capsules

![Geom Type Capsule](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomTypeCapsule.png)

一个 PxCapsuleGeometry 是以原点为中心的。它由一个半径和一个半高值指定，它的轴沿着正负X轴延伸。

要创建一个动态角色，其几何形状是一个直立的胶囊体，这个形状需要一个相对变换，将其围绕Z轴旋转四分之一圈。通过这样做，胶囊将沿着角色的Y轴而不是X轴延伸。设置形状和角色的方法与设置球体的方法相同。

函数 PxTransformFromSegment() 将定义胶囊体的中心轴线段转换为一个变换（Transform）和半高。

### Boxes

![Geom Type Box](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomTypeBox.png)

一个PxBoxGeometry有三个属性，三个边长的一半。

```cpp
PxShape* aBoxShape = PxRigidActorExt::createExclusiveShape(*aBoxActor, PxBoxGeometry(a/2, b/2, c/2), aMaterial);
```

其中a、b和c是生成的Box的边长。

### Planes

![Geom Type Plane](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomTypePlane.png)

平面将空间分为"上方"和"下方"。一切"低于"平面的东西都会与之相撞。

平面位于三位坐标轴的YZ平面上，"上方"指向正X方向。要从平面方程转换为等效变换，使用函数 PxTransformFromPlaneEquation()。PxPlaneEquationFromTransform()可以进行反向转换。

一个 PxPlaneGeometry 没有属性，因为形状的姿势完全定义了平面的碰撞体积。

带有 PxPlaneGeometry 的形状只能为静态角色创建。

### Convex Meshes

![Geom Type Convex Mesh](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomTypeConvex.png)

todo...

### Triangle Meshes

![Geom Type Triangle Mesh](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomTypeMesh.png)

### Height Fields

![Geom Type Height Field](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/GeomTypeHeightField.png)

## Deformable meshes

## Mesh Scaling

## PxGeometryHolder

## Vertex and Face Data
