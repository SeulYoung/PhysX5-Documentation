# Building with PhysX

## Generating projects with CMake

PhysX 使用 CMake 来生成所有支持平台的构建配置文件（如 Microsoft Visual Studio 解决方案）。PhysX 发行版带有默认的 CMake 参数集，称为 **presets** 。构建配置文件可以通过发行版 "physx/" 目录下的脚本 **generate_projects** 来生成。该脚本可以提示预设输入，也可以接受 presets 作为命令行参数。产生的构建配置文件在 "physx/compiler/" 目录下。

PhysX 的二进制文件（包括静态库）被放置在 "bin/\<platform>.\<target>.\<compiler>.\<runtime>/\<configuration>/" 目录中，例如 vc16win64 检查通过后将二进制文件放在 "bin/win.x86_64.vc142.mt/checked/" 目录中。

## Build Configurations

SDK 有四种可用的构建配置，为开发和部署的不同阶段而设计。

- *Debug* 构建对于错误分析非常有用，但是包含了用于 SDK 开发的断言，一些用户可能会发现这些断言对于日常使用来说太具干扰性。在 Debug 配置下，优化功能将被关闭。
- *Checked* 构建包含检测无效参数、API Race 条件和其他不正确使用 API 的代码，这些问题可能导致奇怪的崩溃现象或导致模拟失败。
- *Profile* 构建省略了检查，但仍包含 PVD 和内存检测。
- *Release* 构建是为了实现最小的占用空间和最大的运行速度，它省略了大部分的检查。

模拟（Simulation）在这四种不同的构建方式中以相同的方式工作，并且都以高优化级别编译（调试配置除外）。

> **Note**
> 我们强烈建议你使用 Checked 构建作为日常开发和 QA 的主要配置。
>
> **Note**
> 不同构建配置的 PhysX 库（例如，DEBUG 版本的 PhysXVehicle 和 CHECKED 版本的 PhysXVisualDebuggerSDK）不应混用在一个应用程序中，因为这将导致 CRT 冲突。

## Header Files

要构建自己的 PhysX 应用程序，需要在项目的 makefile 或 IDE 中 include 一些路径和库。

用户应该在构建配置的附加 include 和库目录中分别指定 root "include/" 和相应的 "bin/" 文件夹。有一个包含基本 API 的 header 可以快捷使用：

```cpp
#include "PxPhysicsAPI.h"
```

这将包括整个 PhysX API，包括核心、扩展、车辆等。如果愿意，也可以包括 SDK 的子集，例如：

```cpp
#include "vehicle/PxVehicleSDK.h"
```

## Redistribution

在Windows平台上，需要将我们的一些 DLL 作为你的应用程序的一部分重新发布给最终用户。

- PhysXCommon_*.dll - will always be needed.
- PhysX_*.dll - will always be needed.
- PhysXFoundation_*.dll - will always be needed.
- PhysXCooking_*.dll - you only need to bundle if your application cooks geometry data on the fly.
- PhysXGPU_*.dll - is only needed if your application runs some simulation on the GPU.

其中 * 表示一个特定的平台后缀，例如32或64。需要哪一个版本的DLL，取决于你的应用程序是否在64位下构建。

## Customize CMake presets

可以定制 CMake presets，配置存储在 "physx/buildtools/presets/public/" 中。每个XML文件代表一个预设，根据平台的不同，一些CMake的配置会被打开或关闭。

通用配置：

- PX_BUILDSNIPPETS - 指定是否应将PhysX Snippets添加到构建配置中 - 默认为True。

Windows配置：

- PX_GENERATE_STATIC_LIBRARIES - 将PhysX构建切换为输出静态库而不是DLLs - 默认为False。
- NV_USE_STATIC_WINCRT - 设置静态或动态运行时使用 - 默认为 True
- NV_USE_DEBUG_WINCRT - 指定是否应使用调试 CRT - 默认为 True
- PX_FLOAT_POINT_PRECISE_MATH - 切换到精确数学（precise math）而不是快速数学（fast math），这对机器人相关的项目十分适用，因为这些项目确实需要更高的精度 - 默认为False

一些配置如 PX_GENERATE_STATIC_LIBRARIES 可能导致需要在你的应用程序中设置额外的定义。在这种情况下，可以在你的应用程序中包括PxConfig.h头文件，该头文件是在生成项目时生成的。
