# Rigid Body Dynamics

在这一章中，我们涵盖了一些主题，当你熟悉如何设置基本的刚体模拟世界后，理解这些主题也很重要。

## Velocity

刚体的运动分为线速度和角速度。在模拟过程中，PhysX 会根据重力、其他作用力和扭矩，以及各种约束条件（如碰撞或关节）的结果，修改物体的速度。

可以使用以下方法获取物体的线速度和角速度：

```cpp
PxVec3 PxRigidBody::getLinearVelocity();
PxVec3 PxRigidBody::getAngularVelocity();
```

可以使用以下方法设置物体的线速度和角速度：

```cpp
void PxRigidBody::setLinearVelocity(const PxVec3& linVel, bool autowake);
void PxRigidBody::setAngularVelocity(const PxVec3& angVel, bool autowake);
```

## Mass Properties

一个动态的角色需要质量相关的属性：质量、惯性矩和质心坐标轴，它指定了角色的质心和主要惯性轴的位置。计算质量属性的最简单方法是使用 PxRigidBodyExt::updateMassAndInertia() 辅助函数，它将根据角色的形状和一个统一的密度值来设置所有三个属性。这个函数的变体允许组合每个形状的密度和手动指定一些质量属性。更多细节见 PxRigidBodyExt 的参考页。

SnippetMassProperties（注：PhysX中示例代码片段）中摇摆不定的雪人说明了不同质量属性的使用。这些雪人的行为就像不倒翁玩具，通常只是一个空壳，底部填充了一些重物。低质量中心使它们在被倾斜后又回到直立的位置。它们有不同的风格，取决于质量属性如何设置。

- 第一种基本上是无质量的。在雪人角色的底部只有一个质量相对较高的小球。由于所产生的惯性力矩很小，因此会导致相当快的运动。雪人感觉很轻。
- 第二种只使用底部雪球的质量，导致更大的惯性。然后质心被移到角色的底部。这种近似的做法在物理上绝不是正确的，但产生的雪人感觉更充实一些。
- 第三个和第四个雪人使用形状来计算质量。不同的是，一个先计算惯性矩（从真正的质心开始），然后质心被移到底部。另一个计算关于低质心的惯性力矩，我们将其传递给计算程序。请注意，尽管两者的质量相同，但第二种情况下的晃动速度要慢得多。这是因为头部所占的惯性矩（与质心的距离平方）要大得多。
- 最后一个雪人的质量属性是手动设置的。该代码片段使用惯性矩的粗略值来创建一个特定的期望行为。对角张量在X轴上的值较低，在Y和Z轴上的值较高，产生了围绕X轴旋转的低阻力和围绕Y、Z轴的高阻力，因此，雪人将只围绕X轴来回摆动。

如果有一个 3x3 的惯性矩阵（例如，现实生活中物体的惯性张量），请使用 PxDiagonalize() 函数来获得主轴和对角惯性张量，以初始化 PxRigidDynamic 角色。

在手动设置物体的质量/惯性张量时，PhysX 要求质量和每个主惯性轴的数值为正。然而，在这些数值中提供0是合法的。当提供的质量或惯性值为0时，PhysX将其解释为围绕该主轴的质量或惯性是无限的。这可用于创建抵抗所有线性运动，抵抗所有或部分角运动的物体。使用这种方法可以实现的效果的例子是：

- 表现得如同运动学的物体。
- 平移表现为运动学但其旋转是动态的物体。
- 平移是动态的但旋转是运动学的物体。
- 只能围绕一个特定轴线旋转的物体。

下面详细介绍可以实现的一些例子。首先，让我们假设正在创建一个普通的结构 - 一个风车。下面提供了构建作为风车一部分的主体的代码：

```cpp
PxRigidDynamic* dyn = mPhysics.createRigidDynamic(PxTransform(PxVec3(0.f, 2.5f, 0.f)));
PxRigidActorExt::createExclusiveShape(*dyn, PxBoxGeometry(2.f, 0.2f, 0.1f), material);
PxRigidActorExt::createExclusiveShape(*dyn, PxBoxGeometry(0.2f, 2.f, 0.1f), material);
dyn->setActorFlag(PxActorFlag::eDISABLE_GRAVITY, true);
dyn->setAngularVelocity(PxVec3(0.f, 0.f, 5.f));
dyn->setAngularDamping(0.f);
PxRigidStatic* st = mPhysics.createRigidStatic(PxTransform(PxVec3(0.f, 1.5f, -1.f)));
PxRigidActorExt::createExclusiveShape(*st, PxBoxGeometry(0.5f, 1.5f, 0.8f), material);
mScene.addActor(*dyn);
mScene.addActor(*st);
```

上述代码为风车创建了一个静态 box 框架，并创建了一个十字架来代表风车的叶片。我们关闭风车叶片的重力和角阻尼，并给它一个初始角速度。结果，这个风车的叶片将以恒定的角速度无限地旋转。然而，如果另一个物体与风车相撞，我们的风车将停止正常运转，因为风车叶片会被撞飞。有几种方法可以使风车叶片在其他物体与之相互作用时保持正确的位置。其中一种方法可能是使风车具有无限的质量和惯性。在这种情况下，任何与其他物体的相互作用都不会影响风车：

```cpp
dyn->setMass(0.f);
dyn->setMassSpaceInertiaTensor(PxVec3(0.f));
```

这个例子保留了之前风车以恒定角速度无限旋转的行为。然而，现在物体的速度不能受到任何约束的影响，因为物体有无限的质量和惯性。如果一个物体与风车叶片相撞，碰撞的行为就像风车叶片是一个运动物体一样。

另一个选择是使风车具有无限的质量，并将其旋转限制在仅仅围绕自身的局部Z轴。这将提供类似在风车叶片和静态风车框架之间有一个旋转关节（revolute joint）的效果：

```cpp
dyn->setMass(0.f);
dyn->setMassSpaceInertiaTensor(PxVec3(0.f, 0.f, 10.f));
```

在这两个例子中，物体的质量被设置为0，表明物体的质量是无限的，所以它的线速度不能被任何约束所改变。然而，在后面这个例子中，物体的惯性被配置为允许物体的角速度受到围绕一个主轴或惯性的约束影响。这提供了一个类似于引入旋转关节的效果。围绕Z轴的惯性值可以增加或减少，以使风车叶片对运动的阻力更大/更小。

## Applying Forces and Torques

todo...

## Kinematic Actors

有时，使用力或约束条件来控制一个角色的行为是不够稳健、精确或灵活的。例如，移动平台或角色控制器经常需要操纵角色的位置，或让它准确地遵循一个特定的路径。这样的控制方案是由运动学角色（kinematic actors）提供的。

使用 PxRigidDynamic::setKinematicTarget() 函数可以控制运动学角色。在每个模拟步骤中，PhysX 都会将角色移动到其目标位置，而不考虑外力、重力、碰撞等因素。因此，我们必须在每个时间步长中不断调用 setKinematicTarget()，以使每个运动学角色沿着所需路径移动。一个运动学角色的运动会影响到与其相撞的，或与其有关节约束的其他动态角色。运动学角色看起来有无限的质量，并且会把普通的动态角色推开。

要创建一个运动学角色，只需创建一个普通的动态角色，然后设置其运动型标志：

```cpp
PxRigidBody::setRigidBodyFlag(PxRigidBodyFlag::eKINEMATIC, true);
```

使用相同的函数将一个运动学角色转换回一个普通的动态角色。虽然确实需要为运动学角色提供一个质量（就像所有的动态角色一样），但当其处于运动学模式时，这个质量实际上不会被用于任何事情。

注意事项：

- 在这里理解 PxRigidDynamic::setKinematicTarget() 和 PxRigidActor::setGlobalPose() 之间的区别很重要。虽然 setGlobalPose() 也会将角色移动到所需的位置，但它不会使该角色与其他对象正确的互动。特别是，使用 setGlobalPose()，运动学角色不会推开其路径上的其他动态角色，相反，它会直接穿过它们。不过，如果人们只是想把运动学角色传送到一个新的位置，仍然可以使用 setGlobalPose() 函数。
- 运动学角色可以推开动态对象，但没有任何东西能把角色本身推回来。因此，一个运动学角色可以很容易地将动态角色挤压在静态角色或另一个动态角色上。结果会导致被挤压的物体可能深深地嵌入被推入的几何体中。
- 运动学角色和静态角色之间没有互动或碰撞。然而，可以通过 PxSceneDesc::kineKineFilteringMode 和 PxSceneDesc::staticKineFilteringMode 来请求发生碰撞等这些情况时的关联信息。

### Kinematic Surface Velocities

独立于运动学角色的运动，人们可能希望模拟一个表面速度，这意味着通过碰撞与运动学角色相互作用的物体，将表现得好像运动学角色在移动一样，尽管运动学角色的姿势实际上没有变化。这种机制可以用来创建传送带和旋转表面。请参考  Contact Modification 部分以获得详细的解释。

## Active Actors

todo...
