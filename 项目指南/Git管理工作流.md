# CoralNPU 项目 Git 管理与工作流指南

## 1. 仓库概览

- 远程仓库：`https://github.com/hu9898/coralnpu.git`
- 本地路径：`/home/work/customCoralNPU`
- 构建系统：Bazel 7.4.1
- 许可证：Apache 2.0

## 2. 分支策略

```
origin/main          上游主分支（稳定版本，CI 保护）
  │
  ├── main           本地主分支（与 origin/main 同步）
  │
  └── my             个人日常开发分支（基于 main 创建）
       │
       ├── feature/*  功能分支（从 my 或 main 创建）
       └── fix/*      修复分支（从 my 或 main 创建）
```

| 分支 | 用途 | 生命周期 |
|------|------|----------|
| `main` | 跟踪上游，保持与 origin/main 一致 | 永久 |
| `my` | 个人日常开发、实验、集成 | 永久 |
| `feature/*` | 单个功能开发 | 合并后删除 |
| `fix/*` | Bug 修复 | 合并后删除 |

## 3. 日常开发流程

### 3.1 同步上游

```bash
# 切到 main，拉取最新代码
git checkout main
git pull origin main

# 将更新合并到 my 分支
git checkout my
git merge main
```

### 3.2 在 my 分支上开发

```bash
# 确保在 my 分支
git checkout my

# 编辑代码...

# 查看改动
git status
git diff

# 暂存并提交
git add <文件路径>
git commit -m "简要描述改动"
```

### 3.3 功能分支开发（较大改动推荐）

```bash
# 从 my 创建功能分支
git checkout my
git checkout -b feature/add-new-accelerator

# 开发完成后合并回 my
git checkout my
git merge feature/add-new-accelerator

# 删除已合并的功能分支
git branch -d feature/add-new-accelerator
```

### 3.4 推送到远程

```bash
# 推送 my 分支
git push origin my

# 推送功能分支（如需协作或备份）
git push origin feature/add-new-accelerator
```

## 4. 向上游贡献代码（Pull Request）

```bash
# 1. 确保 main 是最新的
git checkout main
git pull origin main

# 2. 从最新 main 创建 PR 分支
git checkout -b feature/my-contribution main

# 3. 开发并提交

# 4. 推送
git push origin feature/my-contribution

# 5. 在 GitHub 上创建 Pull Request: main ← feature/my-contribution
```

> 注意：上游项目要求签署 CLA（Contributor License Agreement），
> 详见 <https://cla.developers.google.com/>

## 5. CI/CD 说明

项目配置了 GitHub Actions（`.github/workflows/main.yml`）：

- 触发条件：PR、推送到 main、打 tag、手动触发
- 构建目标：`//hdl/chisel/src/coralnpu:rvv_core_mini_axi_cc_library_emit_verilog`
- 产物：`RvvCoreMiniAxi.sv` / `RvvCoreMiniAxi.zip`
- 打 tag 时自动创建 GitHub Release 并附带构建产物

## 6. .gitignore 规则

项目已忽略以下内容，无需手动管理：

| 类别 | 忽略模式 |
| ---- | -------- |
| Bazel 构建产物 | `bazel-*` |
| Robot Framework | `log.html`, `report.html`, `robot_output.xml` |
| VCS 仿真器 | `novas.fsdb`, `ucli.key`, `*vcst*`, `novas*`, `*csrc*` |
| UVM 构建产物 | `tests/uvm/rtl`, `tests/uvm/sim_work`, `tests/uvm/vc_hdrs.h` |
| Python 缓存 | `__pycache__` |

## 7. 提交规范

参考项目已有的提交风格（Conventional Commits 变体）：

```text
<type>(<scope>): <description>

# 示例：
fix(bus): resolve TlulWidthBridge clear/write ordering bug
feat(bus): Support AXI bursts in Axi2TLUL
```

常用 type：

- `feat` — 新功能
- `fix` — Bug 修复
- `refactor` — 重构
- `docs` — 文档
- `test` — 测试
- `chore` — 构建/工具链变更

## 8. 常用 Git 命令速查

```bash
# 查看状态与历史
git status                    # 当前状态
git log --oneline -10         # 最近 10 条提交
git log --oneline --graph     # 图形化历史
git diff                      # 未暂存的改动
git diff --cached             # 已暂存的改动

# 暂存与提交
git add <file>                # 暂存指定文件
git add -p                    # 交互式暂存（按代码块选择）
git commit -m "message"       # 提交
git commit --amend            # 修改最近一次提交

# 分支操作
git branch                    # 列出本地分支
git branch -a                 # 列出所有分支（含远程）
git checkout <branch>         # 切换分支
git checkout -b <new-branch>  # 创建并切换
git branch -d <branch>        # 删除已合并分支

# 远程操作
git fetch origin              # 获取远程更新（不合并）
git pull origin main          # 拉取并合并
git push origin <branch>      # 推送分支

# 临时保存
git stash                     # 暂存当前改动
git stash pop                 # 恢复暂存的改动
```

## 9. 项目目录结构（关键部分）

```text
customCoralNPU/
├── hdl/chisel/       # Chisel 硬件描述（核心源码）
├── hdl/verilog/      # 生成的 Verilog
├── sw/               # 软件组件（模拟器、工具）
├── tests/            # 测试套件（cocotb/verilator/vcs/uvm）
├── examples/         # 示例程序
├── doc/              # 文档
├── fpga/             # FPGA 集成
├── toolchain/        # 编译工具链
├── third_party/      # 第三方依赖
├── 项目指南/          # 本地工作流文档（本文件所在目录）
├── WORKSPACE         # Bazel 工作区定义
└── .bazelrc          # Bazel 构建配置
```
