# 面向 AI 工具的协作模板：在自研 3D 芯片上实现类 CUDA 计算程序

> 适用场景：面试/笔试开放题、技术方案设计题，例如
> **"在自研 3D 芯片上，实现一个类似 CUDA 的计算程序，应该如何实现？"**
>
> 本文给出两件东西：
> 1. **Prompt 模板（直接喂给 ChatGPT / Copilot Chat / Claude / Gemini）** —— 让 AI 一次性给出完整、结构化、可落地的回答
> 2. **Skill 文件模板（`SKILL.md` / `.instructions.md` / `.prompt.md`）** —— 把这套领域知识固化成可复用的 Copilot/Claude Skill

---

## 一、问题拆解：先自己想清楚再问 AI

> "在自研 3D 芯片上实现类 CUDA 计算程序"实际上是在问 **"如何从 0 设计一个异构计算软件栈"**。
> AI 的回答质量与你提问的颗粒度成正比——把下面这些维度提前列给 AI，回答就不会泛泛而谈。

**6 大子问题（喂给 AI 时按层次列出）**：

1. **硬件抽象层 HAL / KMD（Kernel Mode Driver）**
   - 通过 PCIe 枚举设备、加载 firmware
   - 暴露 BAR / MMIO / DMA / 中断（MSI-X）
   - Linux 字符设备 + ioctl + mmap
2. **用户态运行时 Runtime / UMD（User Mode Driver）**
   - 设备初始化、上下文 Context、流 Stream、事件 Event
   - 设备内存分配器（VA↔PA 映射、UVA / UVM）
   - 命令队列 Command Queue / Ring Buffer / Doorbell / Fence
   - Host↔Device 传输（DMA、pinned memory、P2P、零拷贝）
3. **编程模型 Programming Model**
   - Grid / Block / Thread / Warp 抽象（3D 芯片可能引入第 4 维或 tile/cluster）
   - Kernel 启动语法（仿照 `<<<grid,block>>>`，或宏/模板封装）
   - 内存层级：global / shared / local / register / constant
4. **编译工具链 Compiler Toolchain**
   - 前端：Clang / 自定义 DSL → MLIR / LLVM IR
   - 后端：LLVM target / 自研 codegen → 设备 ISA
   - 离线 / 在线（JIT）编译，类似 NVRTC
5. **库 & 上层框架对接**
   - BLAS / DNN / Collective（对标 cuBLAS / cuDNN / NCCL）
   - PyTorch / TensorFlow / vLLM 后端适配（dispatcher、Aten op 注册）
6. **测试 / 调试 / Profiling**
   - 单元测试、模拟器（functional simulator）、时序仿真
   - profiler（对标 nsys / nvprof）、性能计数器、trace
   - 错误码、debug build、ASAN/TSAN、内核 dmesg

**3D 芯片的"特殊性"必须单独提**：
- "3D" 通常指 **3D 堆叠（die-stacking, HBM, chiplet）** 或 **3D 互联拓扑（torus/mesh）**，**不是图形 3D**
- 内存层级多了一层 **HBM / on-die SRAM / NoC 层间带宽**
- 编程模型可能新增 **第三维线程 / tile / cluster** 抽象
- 任务调度要考虑 **die 间数据局部性、热点均衡、跨 die 同步开销**

---

## 二、Prompt 模板（直接复制给 AI）

### 模板 A：一次性获得完整方案（架构题）

````markdown
# 角色
你是异构计算基础软件资深架构师，熟悉 NVIDIA CUDA、AMD ROCm、Intel oneAPI、寒武纪 CNRT、华为 CANN 的内部实现。

# 任务
我需要在一款**自研 3D 堆叠 AI 算力芯片**上，从 0 实现一套**对标 CUDA**的计算软件栈，使上层 PyTorch/vLLM 可调用。
请给出一份**可落地的技术方案**。

# 硬件假设（如果你有信息可替换）
- 3D 堆叠：N 颗计算 die + HBM3，die 间 NoC 互联
- PCIe Gen5 x16 接入 host
- 每个 die 有 M 个 SM-like 计算单元，warp 宽度 32
- 支持 MSI-X 中断、IOMMU
- 提供裸 ISA 与汇编器，没有现成编译器

# 输出要求（严格按以下结构）
1. **整体架构图**（用 Mermaid `graph TD` 画出 5 层：App → Framework → Runtime → KMD → HW）
2. **逐层职责与关键数据结构**（每层 ≤ 200 字，列出核心结构体名）
3. **Kernel 启动全链路时序图**（用 Mermaid `sequenceDiagram`，从 `myKernel<<<g,b>>>(...)` 到设备执行完成、host 收到完成事件）
4. **设备内存管理设计**（VA 空间、页大小、UVM、与 HBM 多 die 的亲和性策略）
5. **Stream / Event / 同步语义**（如何对标 cudaStream_t、cudaEvent_t）
6. **3D 拓扑特有的设计权衡**（至少 3 条：编程模型、调度、互连感知）
7. **MVP 里程碑**（用任务列表 `- [ ]` 给出 4 个阶段，每阶段交付物）
8. **风险与对标 CUDA 的差距**（表格：维度 | CUDA 现状 | 我方初版 | 缩短策略）

# 约束
- 代码示例用 ```cpp 块；公式用 KaTeX；图必须 Mermaid
- 不要泛泛而谈，所有结论给出"为什么这样选"
- 用中文回答
````

### 模板 B：聚焦单点深挖（追问）

```markdown
继续：针对上一个回答的【设备内存管理】部分，请详细说明：
1. 如何在 KMD 中通过 IOMMU 建立 host VA → device VA 的映射？给出 ioctl 接口签名
2. UVM 缺页迁移的状态机（用 Mermaid stateDiagram）
3. 多 die HBM 的 first-touch / interleave / explicit affinity 三种策略对比表
4. 与 cudaMallocManaged 的语义差异
```

### 模板 C：要可运行 demo（落地题）

````markdown
请给出一个**最小可运行**的"vector_add"完整示例，包含：
1. KMD 侧：字符设备框架 + ioctl 处理 ALLOC/FREE/LAUNCH/WAIT 4 个命令
2. UMD 侧：runtime API 头文件（仿 cuda_runtime.h），实现 myMalloc/myMemcpy/myLaunchKernel
3. Kernel 侧：用伪 ISA 或 LLVM IR 描述 vector_add
4. App 侧：```cpp 写 main，分配 host/device 内存，启动 kernel，校验结果
所有代码加注释，关键 syscall/ioctl 标出对应 CUDA API。
````

### 模板 D：让 AI 出"考官视角"反问（备战面试）

```markdown
假设你是算苗科技/沐曦/摩尔线程的面试官，针对"自研芯片实现类 CUDA 软件栈"这个题，
请抛出 10 个最可能追问的细节问题（每个问题给出"考察点"和"标准答案要点"），用 Markdown 表格输出。
```

---

## 三、提问的"黄金原则"（提高 AI 回答质量）

| 原则 | 反例 | 正例 |
|---|---|---|
| 给角色 | "怎么实现 CUDA？" | "你是异构计算架构师……" |
| 给约束 | "讲讲 runtime" | "用 Mermaid 画出 kernel launch 时序图，≤ 30 行" |
| 给输出格式 | "回答详细点" | "按 8 节结构输出，每节用二级标题" |
| 给硬件背景 | "我们有个芯片" | "3D 堆叠 + HBM3 + PCIe Gen5……" |
| 拒绝泛泛 | （不约束） | "所有结论给出'为什么这样选'，不要列教科书概念" |
| 增量追问 | 一次问 20 个问题 | 先要框架，再针对某模块深挖 |
| 给参考语料 | （让 AI 自由发挥） | "以 LDD3 + CUDA Programming Guide 风格作答" |
| 让它自查 | （直接采纳） | "回答完后，列出你不确定的 3 处、需要我补充的信息" |

---

## 四、把这套方法固化为 Skill 文件

> Skill = 把"领域知识 + 提示词模式 + 工具调用规则"打包成一个 Markdown 文件，
> Copilot Chat / Claude Code / Cursor 等工具自动识别并在合适场景调用。

### 4.1 Skill 文件类型与位置

| 文件 | 适用工具 | 默认位置 |
|---|---|---|
| `SKILL.md` | Claude Code、Copilot Chat（skills 目录） | `<extension>/assets/prompts/skills/<skill-name>/SKILL.md` 或仓库 `.github/skills/` |
| `*.instructions.md` | VS Code Copilot（按文件匹配自动注入） | `.github/instructions/` |
| `*.prompt.md` | VS Code Copilot 斜杠命令 `/foo` | `.github/prompts/` 或用户 prompts 目录 |
| `copilot-instructions.md` | 整个仓库全局规则 | `.github/copilot-instructions.md` |
| `AGENTS.md` | Cursor / Aider / 通用 agent | 仓库根目录 |

### 4.2 SKILL.md 模板（推荐这套题用）

把以下文件保存为 `.github/skills/heterogeneous-runtime/SKILL.md`：

````markdown
---
name: heterogeneous-runtime
description: |
  **WORKFLOW SKILL** — 设计或评审"对标 CUDA 的异构计算软件栈"方案。
  USE FOR: 自研 AI/GPU/3D 芯片的 KMD、UMD、Runtime、编程模型、编译栈设计；
  类 CUDA Stream/Event/Memory API 仿写；PyTorch/vLLM 后端适配方案。
  DO NOT USE FOR: 纯 CUDA 应用编程（直接用默认 agent）；图形渲染管线；
  与算力栈无关的 Linux 驱动问题。
  INVOKES: 文件搜索（找现有驱动/runtime 代码）、Mermaid 渲染、代码生成。
applyTo:
  - "**/runtime/**"
  - "**/driver/**"
  - "**/*.{c,cpp,h,hpp,cu,cl,mlir}"
tools:
  - read_file
  - grep_search
  - renderMermaidDiagram
---

# 异构计算软件栈设计技能

## 触发条件
当用户问题包含以下关键词时启用：
- "类 CUDA"、"对标 CUDA"、"自研芯片"、"异构计算 runtime"
- "KMD / UMD / HAL / Stream / Event / Kernel launch"
- "PyTorch backend"、"vLLM 适配"

## 标准回答结构（必须遵守）
回答任何相关方案题时，**始终按 8 节结构输出**：

1. 整体架构图（Mermaid graph TD，5 层）
2. 各层职责 + 核心数据结构
3. Kernel 启动时序图（Mermaid sequenceDiagram）
4. 设备内存管理（VA/UVM/页表/HBM 亲和）
5. Stream / Event / 同步语义
6. 硬件特殊性权衡（如 3D 堆叠 / chiplet / NoC）
7. MVP 里程碑（任务列表 `- [ ]`）
8. 与 CUDA 差距对照表

## 领域知识背景
- CUDA 模型：Grid → Block → Warp → Thread；SIMT；6 种内存
- Runtime API vs Driver API：cudaXxx vs cuXxx
- KMD 通过 PCIe BAR + MSI-X + DMA + IOMMU 与硬件通信
- Command Submission：用户态写 ring buffer，doorbell 通知硬件
- 同步原语：fence、semaphore、interrupt + eventfd
- AI 编译栈：MLIR / LLVM / Triton / TVM / PyTorch dispatcher

## 输出风格约束
- 代码块必须标语言；图必须用 Mermaid
- 中文回答；技术名词保留英文（Stream、Kernel、Doorbell）
- 每个设计决策给出"为什么"，不要罗列教科书概念
- 不确定的地方明确说"需要更多信息"，列出问题

## 反模式（不要这样做）
- ❌ 把图形 3D 渲染管线（OpenGL/DirectX）当成"3D 芯片"答
- ❌ 直接照搬 CUDA 文档而不结合自研硬件假设
- ❌ 上来就写代码，没有架构图
- ❌ 忽略 KMD 与 UMD 边界，把所有逻辑塞进一层
````

### 4.3 `.instructions.md` 模板（按文件类型自动注入）

把以下保存为 `.github/instructions/runtime.instructions.md`：

```markdown
---
applyTo: "**/runtime/**/*.{c,cpp,h,hpp}"
---

# Runtime 代码风格与约定

- 所有公开 API 以 `my` 前缀（仿 `cuda` 前缀），返回 `myError_t`
- 句柄类型用不透明指针 + `_st` 后缀：`typedef struct myStream_st* myStream_t;`
- 错误码定义在 `my_error.h`，新增错误必须同时更新 `myGetErrorString`
- ioctl 命令号用 `_IOWR(MY_IOC_MAGIC, n, struct ...)` 宏定义在共享头文件
- 不允许在 host API 路径上做隐式同步（除非函数名含 `Synchronize`）
- 所有 Mermaid 图、KaTeX 公式必须在头文件 doxygen 注释中给出
```

### 4.4 `.prompt.md` 模板（斜杠命令 `/design-runtime`）

把以下保存为 `.github/prompts/design-runtime.prompt.md`：

````markdown
---
mode: agent
description: 生成对标 CUDA 的异构计算软件栈方案
---

阅读当前工作区的硬件规格文档（如果存在 `docs/hw-spec.md`），然后：

1. 按 8 节结构（架构图 / 各层职责 / 时序图 / 内存 / 同步 / 特殊性 / MVP / 差距）输出方案
2. 所有图用 Mermaid，所有代码用 ```cpp
3. 在最后追加"我不确定的 3 件事"列表，让用户补充
4. 如果发现工作区已有 `runtime/` 目录，先用 grep 摘要现有 API 命名风格，保持一致
````

### 4.5 Skill 文件写作要点（避坑）

| 要点 | 说明 |
|---|---|
| `description` 字段写"USE FOR / DO NOT USE FOR" | LLM 靠它判断要不要触发 skill |
| `applyTo` 用 glob | 限制作用域，避免在无关文件触发 |
| 标准回答结构要硬约束 | 用"必须按 X 节结构"而不是"建议" |
| 领域知识写"背景"而非"教程" | 让 LLM 拿来即用，不浪费 token |
| 列反模式 | 比正面规则更能防止跑偏 |
| 测试方法 | 写完后开新对话，故意问相关/不相关问题各 3 个，看是否正确触发 |
| 版本控制 | Skill 入仓库，PR 评审，避免私改 |

---

## 五、一键使用流程

1. 把 **§二·模板 A** 复制进 ChatGPT / Copilot Chat → 得到方案初稿
2. 用 **模板 B/C** 针对薄弱点追问 → 补全细节
3. 用 **模板 D** 让 AI 扮考官 → 反向找出自己没准备的点
4. 把稳定下来的"提问范式"按 **§四** 固化为 SKILL.md，下次开新项目直接 `/design-runtime`

---

## 附：可直接喂 AI 的"超短版"提问（不背模板时用）

```
你是异构计算架构师。我要在自研 3D 堆叠 AI 芯片上做对标 CUDA 的软件栈。
请按 8 节输出：
(1) Mermaid 架构图 5 层
(2) 各层职责+核心结构体
(3) Mermaid kernel launch 时序图
(4) 设备内存(UVM+多 die HBM)
(5) Stream/Event 语义
(6) 3D 堆叠特殊设计 3 条
(7) MVP 4 阶段任务列表
(8) 与 CUDA 差距对照表
所有结论说"为什么"，中文，代码用 ```cpp。
```
