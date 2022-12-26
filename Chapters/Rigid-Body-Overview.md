# Rigid Body Overview

## Introduction

本章将介绍使用 NVIDIA PhysX 引擎模拟刚体动力学的基本原理。

## Rigid Body Object Model

PhysX 使用自上而下分层的刚体对象/角色模型。

| Class | 扩展自 | 功能性 |
| --- | --- | --- |
| PxBase | N/A | 反射/查询对象类型。 |
| PxActor | PxBase | Actor名称、Actor标志、支配（dominance）、clients、聚合（aggregates）、查询世界边界。 |
| PxRigidActor | PxActor | 形状、变换和查询约束。 |
| PxRigidBody | PxRigidActor | 质量、惯性、速度、阻尼、力、Body标志。 |
| PxRigidStatic | PxRigidActor | 场景中的静态体的接口，具有隐含的无限质量/惯性。 |
| PxRigidDynamic | PxRigidBody | 场景中动态刚体的接口。引入了对运动学目标和对象睡眠的支持。 |
| PxArticulationLink | PxRigidBody | PxArticulation 中动态刚体链接的接口，引入了查询衔接和相邻链接的支持。 |
| PxArticulationReducedCoordinate | PxBase | 定义 PxArticulation 的接口，实际上是一个引用多个 PxArticulationLink 刚体的容器。 |

下图显示了刚体继承和实现中涉及的主要类型之间的关系。
![Rigid Body Overview](https://nvidia-omniverse.github.io/PhysX/physx/5.1.2/_images/RigidBodyOverview.PNG)

## Creating Rigid Actors

刚体及其形状可以通过 PxPhysics 实例创建。

```cpp
PxShapeFlags shapeFlags = PxShapeFlag::eVISUALIZATION | PxShapeFlag::eSCENE_QUERY_SHAPE | PxShapeFlag::eSIMULATION_SHAPE;
PxMaterial* materialPtr = &material;

//plane rigid static
PxRigidStatic* rigidStatic = mPhysics.createRigidStatic(PxTransformFromPlaneEquation(PxPlane(PxVec3(0.f, 1.f, 0.f), 0.f)));
{
        PxShape* shape = mPhysics.createShape(PxPlaneGeometry(), &materialPtr, 1, true, shapeFlags);
        rigidStatic->attachShape(*shape);
        shape->release(); // this way shape gets automatically released with actor
}

//single shape rigid dynamic
PxRigidDynamic* rigidDynamic = mPhysics.createRigidDynamic(PxTransform(PxVec3(0.f, 2.5f, 0.f)));
{
        PxShape* shape = mPhysics.createShape(PxBoxGeometry(2.f, 2.f, 2.f), &materialPtr, 1, true, shapeFlags);
        rigidDynamic->attachShape(*shape);
        shape->release(); // this way shape gets automatically released with actor
}

mScene.addActor(*rigidStatic);
mScene.addActor(*rigidDynamic);
```

关于碰撞几何和形状的更多细节在 Rigid Body Collision 中提供。如果是单一形状的Actor，可以使用PxRigidActorExt::createExclusiveShape()函数，该函数来自PxRigidActorExt.h。

```cpp
//plane rigid static
PxRigidStatic* rigidStatic = mPhysics.createRigidStatic(PxTransformFromPlaneEquation(PxPlane(PxVec3(0.f, 1.f, 0.f), 0.f)));
PxRigidActorExt::createExclusiveShape(*rigidStatic, PxPlaneGeometry(), material);

//single shape rigid dynamic
PxRigidDynamic* rigidDynamic = mPhysics.createRigidDynamic(PxTransform(PxVec3(0.f, 2.5f, 0.f)));
PxRigidActorExt::createExclusiveShape(*rigidDynamic, PxBoxGeometry(2.f, 2.f, 2.f), material);

mScene.addActor(*rigidStatic);
mScene.addActor(*rigidDynamic);
```

PxSimpleFactory.h 中的辅助函数可以进一步简化常见情况的设置。

```cpp
//plane rigid static
PxRigidStatic* rigidStatic = PxCreatePlane(mPhysics, PxPlane(PxVec3(0.f, 1.f, 0.f), 0.f), material);

//single shape rigid dynamic
PxRigidDynamic* rigidDynamic = PxCreateDynamic(mPhysics, PxTransform(PxVec3(0.f, 2.5f, 0.f)), PxBoxGeometry(2.f, 2.f, 2.f), material, 1.f);

mScene.addActor(*rigidStatic);
mScene.addActor(*rigidDynamic);
```

PxSimpleFactory.h 中的 helpers 进一步支持创建运动学角色、克隆形状、角色和缩放角色。

## Simulation

在实际的模拟中，刚体物体需要被添加到一个场景中，该场景将随着时间被向前推进。见"The Simulation Loop"一节。
