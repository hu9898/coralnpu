# platforms 目录详解

## 1. 目录概览

`platforms/` 是 Bazel 构建系统中的平台定义目录，负责声明 CoralNPU 的目标硬件平台和运行环境约束。Bazel 通过这些定义来选择正确的交叉编译工具链，确保代码被编译为 CoralNPU（RISC-V）而非宿主机（x86_64）的二进制。

```text
platforms/
├── BUILD.bazel          # 顶层平台定义（组合 CPU + OS 约束）
├── cpu/
│   └── BUILD            # CPU 约束定义（coralnpu_v2）
├── os/
│   └── BUILD            # OS 约束定义（semihosting）
└── platforms目录详解.md  # 本文件
```

## 2. 各文件详解

### 2.1 cpu/BUILD — CPU 约束

```python
constraint_setting(name = "cpu")

constraint_value(
    name = "coralnpu_v2",
    constraint_setting = ":cpu",
)
```

定义了一个自定义 CPU 约束维度 `cpu`，并声明了一个具体值 `coralnpu_v2`。

这是 Bazel 平台机制的基础。CoralNPU 是基于 RISC-V 的自定义处理器，不属于 Bazel 内置的 `@platforms//cpu` 标准值（如 `x86_64`、`arm`），因此需要自行定义。

### 2.2 os/BUILD — OS 约束

```python
constraint_setting(name = "os")

constraint_value(
    name = "semihosting",
    constraint_setting = ":os",
)
```

定义了自定义 OS 约束维度，声明了 `semihosting` 值。

Semihosting 是嵌入式开发中的一种调试机制：目标处理器通过特殊指令（HTIF）将 I/O 操作（如 `printf`）转发给宿主机的调试器处理。当不使用 semihosting 时，平台使用 Bazel 内置的 `@platforms//os:none`（裸机/无 OS）。

### 2.3 BUILD.bazel — 平台组合定义

这是核心文件，将 CPU 和 OS 约束组合为完整的平台：

```python
# 标准裸机平台
platform(
    name = "coralnpu_v2",
    constraint_values = [
        "//platforms/cpu:coralnpu_v2",   # 自定义 CPU
        "@platforms//os:none",            # 无操作系统（裸机）
    ],
)

# Semihosting 调试平台
platform(
    name = "coralnpu_v2_semihosting",
    constraint_values = [
        "//platforms/cpu:coralnpu_v2",
        "//platforms/os:semihosting",     # 启用 semihosting I/O
    ],
)

# 配置匹配规则（用于 select() 条件编译）
config_setting(
    name = "coralnpu_config",
    constraint_values = [
        "//platforms/cpu:coralnpu_v2",
        "@platforms//os:none",
    ],
)
```

| 平台名称 | CPU | OS | 用途 |
| ---- | --- | -- | ---- |
| `coralnpu_v2` | coralnpu_v2 | none（裸机） | 正式构建、FPGA 部署 |
| `coralnpu_v2_semihosting` | coralnpu_v2 | semihosting | 仿真调试（Verilator/VCS） |

`coralnpu_config` 是一个 `config_setting`，允许在 BUILD 文件中使用 `select()` 进行条件编译：

```python
# 示例：根据平台选择不同的源文件
cc_library(
    srcs = select({
        "//platforms:coralnpu_config": ["impl_coralnpu.cc"],
        "//conditions:default": ["impl_host.cc"],
    }),
)
```

## 3. 与项目其他模块的联动关系

### 3.1 与 .bazelrc 的关系

`.bazelrc` 中定义了构建配置快捷方式：

```text
build:coralnpu_v2 --platforms=//platforms:coralnpu_v2
```

执行 `bazel build --config=coralnpu_v2 //examples:...` 时，Bazel 会自动使用 `coralnpu_v2` 平台，从而触发交叉编译工具链的选择。

### 3.2 与 toolchain/ 的关系

`toolchain/BUILD.bazel` 中注册了两套 C++ 工具链，通过 `target_compatible_with` 与平台约束绑定：

```text
toolchain(cc_coralnpu_v2_toolchain)
  ├── target: //platforms/cpu:coralnpu_v2 + @platforms//os:none
  └── 使用标准 newlib-nano 运行时

toolchain(cc_coralnpu_v2_semihosting_toolchain)
  ├── target: //platforms/cpu:coralnpu_v2 + //platforms/os:semihosting
  └── 使用 semihosting 运行时（支持 HTIF I/O）
```

当 Bazel 解析到目标平台为 `coralnpu_v2` 时，自动选择对应的 RISC-V 交叉编译工具链（而非宿主机的 GCC/Clang）。

### 3.3 与 rules/coralnpu_v2.bzl 的关系

构建规则 `coralnpu_v2_binary` 使用 Bazel transition 机制，根据 `semihosting` 参数自动切换平台：

```python
CORALNPU_V2_PLATFORM = "//platforms:coralnpu_v2"
CORALNPU_V2_SEMIHOSTING_PLATFORM = "//platforms:coralnpu_v2_semihosting"

def _coralnpu_v2_transition_impl(_settings, attr):
    if attr.semihosting:
        return {"//command_line_option:platforms": CORALNPU_V2_SEMIHOSTING_PLATFORM}
    else:
        return {"//command_line_option:platforms": CORALNPU_V2_PLATFORM}
```

这意味着在使用 `coralnpu_v2_binary` 宏时，只需设置 `semihosting = True/False`，平台和工具链会自动匹配。

### 3.4 整体联动流程图

```text
用户执行 bazel build
        │
        ▼
  .bazelrc 设置 --platforms=//platforms:coralnpu_v2
        │
        ▼
  platforms/BUILD.bazel 解析平台约束
  (cpu=coralnpu_v2, os=none 或 semihosting)
        │
        ▼
  Bazel 工具链解析（toolchain resolution）
  匹配 toolchain/BUILD.bazel 中的工具链
        │
        ▼
  选择 RISC-V 交叉编译器（而非宿主机编译器）
        │
        ▼
  rules/coralnpu_v2.bzl 执行编译
  生成 .elf / .bin / .vmem 产物
```

## 4. 两种平台的使用场景

### 4.1 coralnpu_v2（裸机）

适用于最终部署的固件构建：

```bash
# 构建裸机二进制
bazel build //examples:coralnpu_v2_hello_world_add_floats
```

程序使用 newlib-nano 精简运行时，无 I/O 转发能力，适合烧录到实际硬件或 FPGA。

### 4.2 coralnpu_v2_semihosting（调试）

适用于仿真环境下的调试：

```bash
# 在 Verilator 仿真器中运行，支持 printf 输出
bazel run //tests/verilator_sim:core_mini_axi_sim -- \
    --binary path/to/semihosting_binary.elf
```

程序通过 HTIF（Host-Target Interface）将 `printf` 等调用转发到宿主机终端，方便调试。

## 5. 扩展指南

如果需要添加新的平台变体（例如支持新的内存配置或外设），步骤如下：

1. 在 `cpu/BUILD` 或 `os/BUILD` 中添加新的 `constraint_value`（如有需要）
2. 在 `BUILD.bazel` 中组合新的 `platform()` 定义
3. 在 `toolchain/BUILD.bazel` 中注册对应的工具链
4. 在 `rules/coralnpu_v2.bzl` 中添加 transition 逻辑（如有需要）
