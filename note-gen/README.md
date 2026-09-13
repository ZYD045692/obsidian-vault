# note-gen

本目录存放**从拆分稿批量生成学习笔记**的脚本与规范。它独立于「pipeline」（后者负责 OCR → books → 拆分稿这条内容生产线），专注于「笔记层」的生成与维护。

## 职责边界

| 目录 | 职责 |
|---|---|
| `pipeline/` | 把扫描书变成拆分稿：清洗 books、拆分、校验 |
| `note-gen/` | 把拆分稿变成复习笔记：扫描任务、调用模型填充、渲染落盘、注入关联图谱 |
| `11408/notes/` | 最终产物：人工可读、可复习的考点总结与闪卡 |

## 架构

```text
拆分稿 ──▶ core/organizers（整理任务、聚合原文、构造 JSON 提示词）
              │
              ▼
        core/agent（只发请求：限速滑动窗口、重试、json 降级）
              │ raw 留档 _notes_runtime/raw/
              ▼
        core/parsers（容错 JSON → 校验 $ 配平 → 渲染模板）
              │
              ▼
        11408/notes/ 对应位置（保留 frontmatter，写 gen:）
```

- **入口**：`make_notes.py` （批量，薄封装调 `core.pipeline.main` ）、`make_one.py` （单粒度：看一节做一节）。
- **模块职责、一次生成的完整时序、JSON Schema、如何新增笔记类型、常见坑**：见 `core/README.md` 。
- **配置、模型选择、提示词约束、408 机械信息**：见 `提示词与模型.md` 。
- **运行时产物**：模型 raw 留档 / 失败清单 / 运行日志统放顶层 `_notes_runtime/`（与 `_vlm/` 分离）。

> ⚠️ **旧脚本保留情况**：`scripts/_build_kg_skeleton.py` （骨架）与 `scripts/_apply_math_graph*.py` （图谱互链）仍沿用旧 `BASE_NOTE` 路径结构，与当前结构不符，**运行前需先核对并适配其内部路径常量**；不改路径时它们多数是无操作，不会破坏现有笔记。

## 快速开始

### 单粒度生成（推荐：看一节做一节）

```bash
python note-gen/make_one.py 高数 1.1            # 高数 1.1 小节
python note-gen/make_one.py 高数 第1讲           # 高数第1讲：讲级 + 全部小节
python note-gen/make_one.py 线代 第2讲           # 线代第2讲讲级笔记
python note-gen/make_one.py 操作系统 1           # 操作系统 01 章全部考点
python note-gen/make_one.py 数据结构 2.2         # 数据结构 2.2 考点
python note-gen/make_one.py --all               # 全量（等价于 make_notes.py）
```

第一个参数是课程/书/科（`高数` 、`线代` 、`30讲-高数` 、`操作系统` 、`数据结构` …），第二个参数是小节号/讲名/章号；已有 `gen:` 的会跳过，加 `--force` 强制重跑。

### 批量入口

```bash
python note-gen/make_notes.py --list           # 只列任务（不调 API），并幂等建空 stub
python note-gen/make_notes.py --pilot          # 试点（408×2 + 数学讲×1 + 小节×1）
python note-gen/make_notes.py                  # 全量
python note-gen/make_notes.py --type 408       # 只跑某类型：408 | math_lecture | math_section
python note-gen/make_notes.py 数据结构          # 按课程/书过滤
python note-gen/make_notes.py --force 操作系统  # 忽略 gen: 断点强制重跑
```

## 目录结构

```text
note-gen/
├── README.md              # 本文件
├── 笔记约定.md             # 笔记格式契约（frontmatter / 区块 / 闪卡标签 / 笔记目标路径 / MOC）
├── 提示词与模型.md          # 配置、模型选择、限速、提示词约束、408 机械信息
├── TODO.md                # 已知遗留问题（暂缓项）
├── notes_config.json      # 模型/限速/线程等配置
├── make_notes.py          # 批量 CLI 入口
├── make_one.py            # 单粒度 CLI 入口（看一节做一节）
├── core/                  # 架构核心（详见 core/README.md）
│   ├── config.py          # 配置加载（notes_config.json + 顶层 _vlm/.env 共享 api key）
│   ├── agent.py           # Agent + RateLimiter：只发请求
│   ├── task.py            # Task（一篇笔记）/ Job（一次模型请求）
│   ├── pipeline.py        # 入口编排：收集→过滤→线程池→重试→落盘→汇总
│   ├── organizers/        # 整理：scan()→[Task]，jobs()→提示词
│   │   ├── k408.py        #   408 考点（聚合真题年份/前后节/习题链接）
│   │   ├── math_lecture.py#   数学讲级（长讲 chunk→synth 多阶段）
│   │   └── math_section.py#   数学小节（仅高数；拆分稿 #### N 切小节，幂等建 stub）
│   └── parsers/           # 解析：JSON→校验→渲染笔记模板
│       ├── base.py        #   容错 JSON 提取、LaTeX 保护、$ 配平、闪卡聚合
│       ├── k408.py
│       ├── math_lecture.py
│       └── math_section.py
└── scripts/               # 保留的旧脚本（骨架 / 图谱，使用时需适配路径）
```

## 闪卡与 SR 复习队列

标签语义（为什么生成物用 `#card` 而不是 `#flashcards` ）见 `笔记约定.md` ；模型侧的卡片结构见 `提示词与模型.md` 。本节只给操作命令。

```bash
# 启用复习（#card → #flashcards）
python note-gen/toggle_card_tag.py --on 408/操作系统/1-计算机系统概述/1.1-操作系统的基本概念.md

# 禁用复习（#flashcards → #card）
python note-gen/toggle_card_tag.py --off 408/操作系统/1-计算机系统概述/1.1-操作系统的基本概念.md

# 批量启用/禁用（glob 模式）
python note-gen/toggle_card_tag.py --on "408/操作系统/**/*.md"
python note-gen/toggle_card_tag.py --off "数学/30讲-高数/第1讲-函数极限与连续/**/*.md"

# 只检查，不写盘
python note-gen/toggle_card_tag.py --dry --on "408/**/*.md"
```

SR 插件默认识别 `#flashcards` ；切换后重新打开 Obsidian 或等待插件扫描即可入队。

## 运行前检查

1. `_vlm/.env` 有有效的 `SILICONFLOW_API_KEY` ；拆分稿已生成且目录结构正确。
2. 确认 `notes_config.json` 的 `model` 与 `models` 限速值（见 `提示词与模型.md` ）。
3. 第一次跑某范围前，先 `--list` 看任务收集是否符合预期，再单点（`make_one.py` ）看单篇效果。
4. 模型生成消耗 token，全量前确认配额。

## 排错

- **`缺少 API key`**：`_vlm/.env` 没有 `SILICONFLOW_API_KEY=...` 行（或环境变量）。
- **任务为 0 / 路径不对**：先 `--list` 核对；确认拆分稿里有 `### 基础内容精讲` / `#### N` 标题（高数小节切分依赖它们）。
- **生成内容为空或格式崩**：去 `_notes_runtime/raw/` 看模型原始输出；拆稿原文 OCR 噪声大时可先清洗 books。
- **LaTeX 显示成 `ε` 字面量 / `\b` 吞字**：模型把反斜杠写成了 `\` 或单反斜杠命令，属于解析层问题——检查 `parsers/base.py` 的 `_protect_latex` 是否覆盖了该模式。
- **频繁 429 / 限速**：`threads` 调低，或核对 `models` 注册表 rpm/tpm 是否过高于实际配额。
- **卡在某篇重试**：`_notes_runtime/logs/run.log` 有每篇的请求与失败记录；`_notes_runtime/repair/notes_manual.md` 是失败清单。

## 相关文档

- 笔记格式契约（frontmatter / 区块 / 闪卡标签 / 目标路径 / MOC）：`笔记约定.md`
- 配置 / 模型 / 提示词：`提示词与模型.md`
- 核心模块与扩展步骤：`core/README.md`
- 已知遗留（暂缓）：`TODO.md`
