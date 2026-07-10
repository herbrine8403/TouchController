# 构建流水线

在了解构建系统前，我们先需要了解如何构建一个 Minecraft 模组。这些构建流程常常被隐藏在 Gradle 插件中，经常被开发者忽略。
而且不同加载器的模组构建流程也不同，更是为配置增加了复杂度。更麻烦的是，到了 Bazel 里，我们需要显示指定所有流程，才能构建出一个模组。

综上所述，对 Minecraft 模组的构建流程了解是非常有必要的。这篇文章会介绍每个加载器的构建流程，从而为编写对应 Bazel 规则做好准备。

## 命名和映射

传统上来说（26.1 以前），Minecraft 发布时是以混淆的形式发布的，因此我们需要把 Minecraft 的各种符号名重映射到一个可读的名称下，才可以进行开发。

我们用以下的称呼表示不同的命名：

- official：官方 minecraft.jar 内的符号名，在 26.1 之前是混淆的
- intermediary：Fabric 采用的中间名，在 26.1 及其之后不存在
- named：开发时采用的符号名，对于 1.14.4 及其之后的版本采用 Mojang 官方符号名，对于 1.14.4 之前的版本采用 Legacy Yarn / Biny 混淆名

## Fabric

Fabric 的构建流程相对简单且固定，主要分为以下几个步骤：

- 映射处理
- 重映射 Minecraft 从 official 到 named
- 重映射依赖从 intermediary 到 named
- 编译 named 命名的模组
- 在 named 命名下进行测试
- 重映射模组从 named 到 intermediary
- 发布！

要注意的是，由于 Minecraft 26.1 开始取消了混淆，因此一切映射处理在 26.1 及其之后的版本都不需要了，步骤简化为：

- 编译模组
- 进行测试
- 发布！

接下来介绍一些关键步骤：

### 映射处理

在 26.1 之前的版本，我们有以下映射表：

- Mojang 官方映射表（1.14.4 及其之后的版本）/ Legacy Yarn（老版本）/ Biny（Beta 1.7.3）
- Intermediary 映射表
- Parchment 映射表（可能没有）

在开发时，为了方便考虑，我们需要把所有映射表合并到一个文件中，项目中有一个工具 `mapping_merger` 可以完成这个工作。

### 重映射 JAR

Minecraft 和依赖模组的重映射，和发布前对我们自己的模组的重映射由项目中的 `tiny_remapper_worker` 来完成。

## NeoForge / Forge

NeoForge 和 Forge 的构建流程稍微复杂一些，但步骤基本是一样的：

- 映射处理
- 准备 NeoForm / MCP 工件
- 对之前得到的工件加入 NeoForge / Forge 并重编译
- 编译 named 命名的模组
- 在 named 命名下进行测试
- 重映射模组从 named 到 srg（在 1.20.1 之后不再需要）
- 发布！

### 准备 NeoForm / MCP 工件

相对于 Fabric 的固定步骤，NeoForge 和 Forge 采用动态的配置文件来对 minecraft.jar 进行处理，具体的实现在 `neoform.bzl` 内。

一般来说，配置文件里会定义以下步骤：

- 反编译 minecraft.jar，得到 minecraft.srcjar
- 对反编译后的 minecraft.srcjar 的符号重映射到 named
- 对重映射后的 minecraft.srcjar 打补丁，确保代码能重编译

因为涉及到反编译，这个阶段的速度会很慢。

### 对之前得到的工件加入 NeoForge / Forge 并重编译

这个阶段的实现在 `neoforge.bzl` 内。和 Fabric API 的运行时注入不一样，NeoForge / Forge 在编译时直接对代码打补丁，因此我们需要对
minecraft.srcjar 内打上补丁，然后重新编译得到最终可用的工件。

因为涉及到编译整个 Minecraft，这个阶段的速度也很慢。

其他的步骤和 Fabric 都是一致的。
