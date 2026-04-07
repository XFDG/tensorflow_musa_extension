# TensorFlow 2.15.x 迁移排查与计划

更新时间：2026-04-07

## 目标与范围

本文档用于回答两个问题：

1. 当前 `tensorflow_musa_extension` 从 TensorFlow 2.6.1 走向 TensorFlow 2.15.x，哪些地方必须改。
2. 在“不直接改现有实现代码”的前提下，先把后续迁移计划拆出来，便于逐步推进。

本文档基于两类证据：

- 仓库当前实现与构建/测试/CI 配置的静态排查。
- TensorFlow 官方 2.15 发布说明、官方构建文档、官方 PluggableDevice 设计资料，以及 TensorFlow 2.15.1 包元数据。

## 结论先行

这次升级不是单纯“改几个 include”。

当前工程和 TF 2.6.1 的耦合点主要集中在以下 4 个高风险区域：

- 构建链强绑定旧环境：CMake 直接固定 `gcc/g++`、`C++14`、`_GLIBCXX_USE_CXX11_ABI=0`，并依赖当前 shell 中的 `python3`。
- 设备接入强绑定 TensorFlow 内部 C++ 运行时：`DeviceFactory`、`LocalDevice`、`StreamExecutor` C++ 内部接口、`BFCAllocator`、`GpuDeviceInfo` 都直接在插件里实现。
- 图优化强绑定 Grappler 内部 C++ 注册机制：`CustomGraphOptimizer` + `REGISTER_GRAPH_OPTIMIZER_AS`。
- 测试和 CI 仍以 TF 2.6 风格为中心：大量 `tf.compat.v1`/`Session`/`GraphDef` 测试，CI 也没有版本矩阵和隔离环境。

如果只追求“先能在 2.15 上编译和加载”，可以走一条兼容层 + 版本分支路径。

如果目标是“2.15 起步，并尽量减少未来 2.16+ 再大改”，则设备层应逐步向 TensorFlow 官方推荐的 PluggableDevice/StreamExecutor C API 方向收敛。

## 当前基线

### 1. 版本与依赖基线

仓库文档当前仍写死为 TF 2.6.1：

- `README.md` 中声明 `Python >= 3.7`、`TensorFlow == 2.6.1`、`protobuf == 3.20.3`、`pettytable >= 3.0.0`。
- 这与 TensorFlow 2.15.x 的官方 Python/工具链/依赖要求已经不一致。

相关证据：

- `README.md:46-54`

### 2. 构建链基线

当前构建链的关键特征：

- 直接固定 `gcc/g++`。
- 主机侧 `.cc` 编译标准固定为 `C++14`。
- 强制剥离并重新写入 `-D_GLIBCXX_USE_CXX11_ABI=0`。
- 通过当前 shell 的 `python3` 动态读取 `tf.sysconfig`，但并没有锁定目标 TensorFlow 环境。
- `.mu` 编译已经单独使用 `-std=c++17`，主机与设备侧标准不一致。

这说明当前构建脚本本质上是“TF 2.6.1 + 当时 wheel ABI 假设”的定制方案，不适合作为 TF 2.15.x 的通用构建入口。

相关证据：

- `CMakeLists.txt:4-9`
- `CMakeLists.txt:35-55`
- `CMakeLists.txt:79-92`
- `CMakeLists.txt:119-130`
- `build.sh`

### 3. 设备接入基线

当前 MUSA 设备接入不是走稳定插件边界，而是直接实现 TensorFlow 内部设备栈：

- `REGISTER_LOCAL_DEVICE_FACTORY("MUSA", ...)`
- 自己实现 `MusaDeviceFactory`
- 自己实现 `MusaPlatform : public Platform`
- 自己实现 `MusaExecutor : public internal::StreamExecutorInterface`
- 自己实现 `MusaStream : public internal::StreamInterface`
- 自己实现 `MusaEvent : public internal::EventInterface`
- `MusaDevice` 直接继承 `Device`
- 直接填充 `GpuDeviceInfo`
- 直接使用 `BFCAllocator`

这类实现方式在 TF 2.6.1 可行，但会直接受到 TensorFlow 内部 C++ API/命名空间/头文件布局变化影响。

相关证据：

- `musa_ext/mu/device_register.cc:8-73`
- `musa_ext/mu/device/musa_platform.cc:3-91`
- `musa_ext/mu/device/musa_executor.h:11-339`
- `musa_ext/mu/device/musa_stream.h`
- `musa_ext/mu/device/musa_event.h`
- `musa_ext/mu/device/musa_device.h:33-127`
- `musa_ext/mu/device/musa_device.cc:400-516`

### 4. 图优化/融合基线

当前图优化器直接建立在 Grappler 内部 C++ 扩展点之上：

- 直接依赖 `tensorflow/core/grappler/...`
- `MusaGraphOptimizer : public CustomGraphOptimizer`
- 直接操作 `GraphDef` / `NodeDef`
- 使用 `REGISTER_GRAPH_OPTIMIZER_AS`
- 文件中注释仍明确提到“TF 2.6.1 CustomGraphOptimizer interface”

这部分在 2.15 上未必完全不可用，但它属于内部接口，迁移风险高于普通 `REGISTER_KERNEL_BUILDER` 内核。

相关证据：

- `musa_ext/mu/optimizer/musa_graph_optimizer.cc:27-39`
- `musa_ext/mu/optimizer/musa_graph_optimizer.cc:307-440`
- `musa_ext/mu/optimizer/musa_graph_optimizer.cc:442-447`
- `musa_ext/mu/optimizer/musa_graph_optimizer.cc:810-821`

### 5. 测试基线

测试体系当前分成两层：

- 通用算子测试：多数是 eager 风格。
- 融合/图优化测试：大量依赖 `tf.compat.v1`、`Session`、`GraphDef`。

本次排查中，至少发现：

- `test/fusion` 下有 14 个 Python 文件仍依赖 `tf.compat.v1` / `Session` / `GraphDef` / `disable_eager_execution`。
- `test/ops` 下有 6 个 Python 文件仍依赖这些旧式图模式入口。

这意味着 2.15 迁移不能只做编译适配，还需要补测试分层，否则很难区分“设备层坏了”“图优化坏了”“只是 compat.v1 测试写法老了”。

相关证据：

- `test/musa_test_utils.py:36-93`
- `test/fusion/*.py`
- `test/ops/apply_gradient_descent_op_test.py`
- `test/ops/apply_rmsprop_op_test.py`
- `test/ops/apply_resource_op_test.py`
- `test/ops/remove_isolated_nodes_graph_test.py`
- `test/ops/convert_graphdef_pbtxt_to_pb.py`
- `test/ops/shifted_affine_map_benchmark.py`

### 6. CI 基线

当前 CI 仍是单环境、自托管 runner、无版本矩阵模式：

- 直接 `./build.sh`
- 直接 `python test_runner.py --fusion`
- 依赖 runner 机器上的登录环境
- 构建阶段甚至会从 runner 固定路径拷贝 `CMakeLists.txt`

这对单版本开发可行，但不适合做 TF 2.6.1 / 2.15.x 双线支持，更不适合排查 ABI/依赖/编译器差异。

相关证据：

- `.github/workflows/pr-validation.yml:124-160`

## TensorFlow 2.15.x 目标约束

### 官方版本/工具链约束

根据 TensorFlow 官方构建文档：

- TensorFlow 2.15.0 在 Linux 上的官方测试构建组合为：
- Python `3.9-3.11`
- Compiler `Clang 16.0.0`
- Build tools `Bazel 6.1.0`
- GPU 组合为 `cuDNN 8.9` / `CUDA 12.2`

与 TF 2.6.0 相比，2.15 的官方构建矩阵已经明显升级：

- TF 2.6.0: Python `3.6-3.9`，GCC `7.3.1`，Bazel `3.7.2`
- TF 2.15.0: Python `3.9-3.11`，Clang `16.0.0`，Bazel `6.1.0`

这意味着“旧构建脚本继续靠经验固定 gcc/c++14/ABI=0”不是稳妥路径。

### 官方扩展方向约束

根据 TensorFlow 官方 PluggableDevice RFC：

- 新的第三方设备集成，推荐走标准化的 `PluggableDevice` 机制。
- 该机制建立在 `StreamExecutor C API` 之上，而不是要求插件继续直接实现 TensorFlow 内部的 `StreamExecutor` C++ 细节。

这并不等于“必须一步到位重写全部设备层”，但至少说明：

- 如果我们只做面向 2.15 的最小兼容补丁，短期也许能跑。
- 但如果继续扩展 2.16+ / 2.20+，当前设备层写法会越来越脆。

### TensorFlow 2.15.1 包元数据约束

TensorFlow 2.15.1 的包元数据显示：

- `Requires-Python: >=3.9`
- `numpy >=1.23.5,<2.0.0`
- `protobuf >=3.20.3,<5.0.0dev`
- `ml-dtypes ~=0.3.1`
- `keras >=2.15.0,<2.16`
- `tensorboard >=2.15,<2.16`

因此 README 中“`Python >=3.7` + `TensorFlow ==2.6.1` + `protobuf ==3.20.3`”这套写法必须拆成版本矩阵，而不是继续用一套环境描述覆盖所有版本。

## 需要改哪些东西 / 需要补哪些东西

以下按优先级拆分。

### P0：不做就很难开始支持 TF 2.15.x

#### 1. 补版本矩阵与隔离环境

必须补：

- 明确支持矩阵：`TF 2.6.1`、`TF 2.15.x` 分别对应的 Python 版本、MUSA SDK 版本、编译器、构建方式。
- 独立环境文件：至少要有 `requirements` / `constraints` / `conda env` 之一，不能继续只靠 README 文字说明。
- 独立构建入口：要能显式选择目标 Python 和目标 TensorFlow 环境，不能继续默认使用当前 shell 的 `python3`。

原因：

- 当前仓库没有任何 `requirements*.txt`、`pyproject.toml`、`environment.yml` 之类的环境定义文件。
- 只靠当前 shell 环境，会导致“以 2.20 的头文件/ABI 去编 2.15 目标”这种误用。

#### 2. 重做构建参数管理

必须改：

- 不再全局强制 `gcc/g++`。
- 不再全局固定 `C++14`。
- 不再无条件覆盖 `_GLIBCXX_USE_CXX11_ABI=0`。
- 主机侧 `.cc` 与设备侧 `.mu` 的标准和 ABI 策略需要统一、版本化。
- 所有编译/链接参数以目标 TF wheel 的 `tf.sysconfig` 为主，仓库本地只补版本特定修正。

原因：

- 当前本机默认 `python3` 对应的 TensorFlow 2.20.0 已经返回 `--std=c++17` 和 `_GLIBCXX_USE_CXX11_ABI=1`。
- 当前本机 TensorFlow 2.20.0 头文件中，`tensorflow/stream_executor/multi_platform_manager.h` 与 `tensorflow/stream_executor/lib/status*.h` 已经不存在，而 `tsl/platform/status*.h` 存在，这说明现有 include 方式缺乏长期前向兼容性。
- 即使 2.15 目标最终未必完全等同，至少也说明现在这套“固定 ABI=0”的思路不能再当成通用真理。

#### 3. 增加 TensorFlow 兼容层

必须补：

- 新建统一兼容层目录，例如 `musa_ext/compat/`。
- 把版本敏感的内容集中进去：
- `Status` / `StatusOr` / `OkStatus` 适配
- `absl` / `tsl` / `tensorflow::` 命名空间差异
- 头文件路径差异
- 条件编译宏，例如按 TF 版本切换 include 和 API 包装

原因：

- 现在版本敏感引用是散落的。
- 设备层、图优化层、部分工具头直接引用 TensorFlow 内部头，后续如果一处处修，成本会很高。

#### 4. 先把设备层迁移策略定下来

必须补：

- 先明确路线，不然实现会来回返工。

建议路线：

- 短期路线：保留旧设备层思路，先尝试让 TF 2.15 能编译、能加载、能枚举 MUSA、能跑基础 eager 算子。
- 中期路线：把设备层逐步迁移到 TensorFlow 官方推荐的 PluggableDevice / StreamExecutor C API 模式。

原因：

- 当前实现深度依赖 `LocalDevice` 和 `StreamExecutor` C++ 内部接口。
- 官方推荐路线已经是稳定插件边界，不是继续直接实现 TensorFlow 内部 C++ 设备栈。

#### 5. 对 Grappler/融合单独建兼容任务

必须补：

- 不要把“设备能跑起来”和“图优化/融合可用”当成同一个里程碑。
- 需要单独定义：
- `无融合` 的 2.15 基础可运行目标
- `有融合` 的 2.15 图优化可运行目标

原因：

- 当前融合逻辑全部压在 `CustomGraphOptimizer` + `GraphDef` 手工改写上。
- 这是迁移中第二高风险区域，仅次于设备层。

### P1：为了把 2.15 真正跑稳、跑完整

#### 6. 做算子/内核分层审计

需要补：

- 按目录对 `musa_ext/kernels` 做分层：
- 纯 `REGISTER_KERNEL_BUILDER` 且只依赖常规 `op_kernel.h` 的
- 依赖 `tensorflow/core/util` / `register_types.h` / `core/kernels` 的
- 依赖设备上下文、资源变量、IO、图模式行为的

目标：

- 先迁移低风险 eager 算子，再迁移资源变量、训练、图优化相关算子。

#### 7. 补测试分层

需要补：

- `smoke`：插件能加载、设备能枚举、`tf.config.list_physical_devices('MUSA')` 正常。
- `eager-op`：基础算子 eager 路径。
- `graph-compat`：保留 `tf.compat.v1` 的图模式回归。
- `fusion`：图优化/融合。
- `benchmark`：性能与精度。

原因：

- 现在测试层级不够清晰，定位问题时容易混在一起。

#### 8. 补版本化构建产物

需要补：

- 按 TF 版本区分 build 目录或插件产物名。
- 例如 `build/tf261/`、`build/tf215/`，或者插件名称带版本后缀。

原因：

- 双版本并行支持时，同一个 `build/libmusa_plugin.so` 很容易相互覆盖。

### P2：为了后续 2.16+ / 长期演进

#### 9. 向 PluggableDevice 架构收敛

建议补：

- 评估把设备发现、平台注册、stream/event/memory 分配逐步替换为基于 C API 的实现。
- 保留现有算子实现，但降低对 TensorFlow 内部设备运行时的耦合。

原因：

- 这是减少未来版本迁移成本的关键工程项。

#### 10. 把图优化逻辑和 TensorFlow 集成层解耦

建议补：

- 将 pattern match / graph rewrite 逻辑与 TensorFlow 注册入口分离。
- 让 `FusionPatternManager`、pattern 本体尽量少依赖具体 TF 版本的注册/状态类型。

原因：

- 当前逻辑主体和 TensorFlow 注册接口缠得较紧，不利于跨版本维护。

## 迁移计划

## Phase 0：冻结基线，先把目标边界说清楚

目标：

- 明确“第一阶段的 2.15 支持”到底包含什么。

建议验收标准：

- 能在 TF 2.15.x 环境中完成编译。
- 插件可加载。
- MUSA 设备可枚举。
- 至少一批基础 eager 算子可通过。
- 暂时允许图优化/融合先关闭。

本阶段输出：

- 版本矩阵文档
- 环境定义文件
- 迁移任务清单

## Phase 1：构建系统改造

目标：

- 让仓库具备“按目标 TF 版本构建”的能力。

工作项：

- 引入版本化环境定义
- 引入版本化构建入口
- 清理固定编译器/ABI/标准
- 增加 TF 兼容宏与兼容头

验收标准：

- 同一仓库可显式选择 `TF 2.6.1` 或 `TF 2.15.x` 构建
- 构建日志能清楚输出所用 Python/TF/compiler/ABI

## Phase 2：设备层先跑通 2.15

目标：

- 在不启用融合的情况下，让设备层先工作。

工作项：

- 处理设备注册、平台注册、allocator、stream/event 相关 API 兼容问题
- 增加基础 smoke test
- 增加设备枚举与最小 tensor copy 测试

验收标准：

- `tf.load_library()` 成功
- `tf.config.list_physical_devices('MUSA')` 返回设备
- 基础 eager 算子可在 `/device:MUSA:0` 上跑通

## Phase 3：基础算子逐批迁移

目标：

- 先保通主路径算子，再扩展到训练/资源变量/IO。

建议顺序：

- 数学基础算子
- array/data movement
- nn 常用前向算子
- state/resource variable
- training 优化器相关
- IO / string / control flow

验收标准：

- eager 基础算子覆盖形成稳定回归集
- 关键训练链路至少有一条可以跑通

## Phase 4：图优化与融合恢复

目标：

- 在 2.15 中重新启用 MUSA graph optimizer 和 fusion path。

工作项：

- 验证 `CustomGraphOptimizer` 接口与注册路径
- 恢复 layout/AMP/fusion pass
- 单独处理 `remove_isolated_nodes` 与 GraphDef rewrite 回归
- 分开维护“无融合正确性”和“有融合正确性”

验收标准：

- 关键 fusion case 跑通
- 关闭融合时正确，开启融合时也正确
- 图优化失败时可回退到非融合路径

## Phase 5：CI、文档、发布收口

目标：

- 把“个人环境里能跑”变成“仓库持续可验证”。

工作项：

- CI 增加 TF 版本矩阵
- 分离不同版本的 build/test job
- 更新 README 与调试文档
- 输出已支持/未支持算子清单

验收标准：

- CI 至少覆盖 `TF 2.6.1` 和 `TF 2.15.x`
- 文档能指导他人独立构建与测试

## 建议的任务拆分顺序

建议按下面顺序推进，避免大面积返工：

1. 先补环境矩阵和版本化构建入口。
2. 再做 TF 兼容层，集中管理 include/status/宏差异。
3. 先跑通设备加载和基础 eager 算子。
4. 再做资源变量、训练链路。
5. 最后恢复 Grappler/Fusion。
6. 等 2.15 稳定后，再评估向 PluggableDevice 方向收敛。

## 风险判断

高风险：

- 设备层：`DeviceFactory` / `StreamExecutor` / allocator / event/stream 生命周期
- 图优化层：`CustomGraphOptimizer` / `GraphDef` rewrite / fusion 注册

中风险：

- 资源变量、训练优化器、状态类算子
- CI 与环境隔离

低到中风险：

- 纯基础算子内核
- README/文档修正

## 本次排查建议的里程碑定义

建议不要把“支持 TF 2.15.x”一次性定义得过大，而是拆成 3 个里程碑：

- Milestone A：`build + load + enumerate + eager basic ops`
- Milestone B：`resource/training + broader op coverage`
- Milestone C：`grappler/fusion + CI matrix + documentation`

这样推进时更容易判断风险真正集中在哪。

## 参考资料

官方资料：

- TensorFlow 2.15 发布说明：<https://raw.githubusercontent.com/tensorflow/tensorflow/r2.15/RELEASE.md>
- TensorFlow 官方构建文档：<https://www.tensorflow.org/install/source>
- TensorFlow GPU / device plugin 资料入口：<https://www.tensorflow.org/install/gpu_plugins>
- TensorFlow PluggableDevice RFC：<https://github.com/tensorflow/community/blob/master/rfcs/20200624-pluggable-device-for-tensorflow.md>
- TensorFlow Create Op Guide：<https://www.tensorflow.org/guide/create_op>
- TensorFlow custom-op 官方仓库：<https://github.com/tensorflow/custom-op>

包元数据：

- TensorFlow 2.15.1 PyPI metadata：<https://pypi.org/project/tensorflow/2.15.1/>

## 本次排查产出说明

本次仅新增文档，没有修改现有实现代码。
