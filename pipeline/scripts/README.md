# pipeline/scripts/ — 处理脚本

> 流程规范（7 步流水线）见 `../README.md` 。本文件只管**这个目录里有什么脚本、怎么调用、什么时候能用**。

## 目录

- [保留的工具脚本](#保留的工具脚本)
- [常用命令速查](#常用命令速查)
- [脚本的生成物](#脚本的生成物)

## 保留的工具脚本

> ⚠️ **这里多数是一次性工具**：文件名常含课程缩写或轮次（如 `_fix_ti_round2.py` ），是为**特定清洗阶段**写的。
> 再次运行前先确认它的**目标文件与阶段是否仍然适用**——不要因为「脚本还在仓库里」就把它当通用工具用。
> **临时脚本不要留在这里**：一次性的调试/扫描脚本放仓库根 `tmp/` ，用完即弃（见 `AGENTS.md` 红线 8）。

| 脚本 | 用途 |
|---|---|
| `_resplit_6books.py` | 408 四科 + 数学 30讲从 books 一次性拆分，图片指向 books |
| `_crop_liang_pdfs.py` | 扫描 PDF 预裁剪**参考实现**（写在李良概统上）：页眉按需裁（如 7.5%，章首页不裁）+ tesseract 逐页精确裁页码，按 基础/强化→章节 分章合成 PDF。**新增扫描书时参考此方法另写/复制修改**（PDF 路径、章边界、无页眉页清单为该书专属） |
| `_calc_pageno_pos.py` | 页码精确定位验证：tesseract 逐页读页码，输出页码 top 位置分布（初裁前跑，确认页码带/章边界） |
| `_verify_integrity.py` | 字数完整度 + 图片引用匹配校验 |
| `_verify_fingerprint.py` | 行指纹多提取/少提取校验 |
| `_validate_dir.cjs` / `validate_single.js` | MathJax 公式渲染校验 |
| `_scan_origin_structure.py` | books 各书结构扫描（章/节/习题区标题异常） |
| `_fix_origin_structure.py` | 修复结构异常（章/节/习题区/子节标题） |
| `_del_ads.py` / `_del_orphan_div.py` | 删广告块 / 删孤儿 `<div>` 残留 |
| `_check_chapter_exams.py` / `_check_exam_gaps.py` / `_check_sec_exam_gaps.py` | 习题完整性检查（每章有习题 / 习题区对比 / 无习题节回 books 核实） |
| `_validate_blocks.cjs` | 跨行 `$$` 块 KaTeX 校验（**补 `_validate_dir.cjs` 盲区**） |
| `_validate_all.cjs` | **全文 KaTeX 校验**（状态机提取所有公式含跨行行内，终极权威，校验阶段必跑） |
| `_show_inline_bb.py` / `_scan_crossblock.py` | 定位行内嵌 `$$` / 跨行 `$...$` 块 |
| `_scan_dollar_balance2.py` | 公式边界解析（跳过 `\$` 转义、块内 `$` 不算边界）→ 应"边界完整" |
| `_scan_glue_heading.py` | 标题紧贴 HTML 行（无空行）扫描 → 命中即在标题前插空行，否则大纲吞标题 |
| `_check_ti_continuity.py` | 30讲数学 例/题/答编号连续性+题答对称交叉验证（题目进大纲后必跑） |
| `_add_ti_headings_408.py` | 408×4 课后习题题目化（题/答 `NN.` → `##### NN`，顺序期望+重同步，幂等） |
| `_check_ti_continuity_408.py` | 408×4 习题编号连续性+题答对称交叉验证（题目化后必跑） |
| `_fix_ti_sjjg_round2.py` / `_fix_ti_os_round2.py` / `_fix_ti_jz_round2.py` / `_fix_ti_jw_round2.py` | 408 断档收尾修复（13/9/12/3 处，内容锚定+断言，已应用，可删） |
| `_scan_dollar_lines.py` | 逐行单 `$` 配对检查（跳过代码块/`$$`/`\$`） |
| `_scan_dollar_unmatched.py` | 定位孤立 `$`/`$$`（承受点） |
| `_scan_unknown_cmds.py` | 未知 LaTeX 命令（OCR 截断）扫描 |
| `_fix_options.py` | 选择题选项乱码修复（`\qua　`/`(D　`/`nfty`/`rac{`/补 `(C)`） |
| `_norm_408_headings.py` / `_norm_408_pass2.py` | 408 标题层级统一 + 上下文防跳级 |
| `_norm_math_headings.py` / `_norm_math_pass2.py` | 30讲数学标题层级统一 |
| `_norm_1000_headings.py` | 1000题标题层级统一 |
| `_remove_toc.py` | 移除 books 头部 `📑 快速跳转` 目录块（已弃用该功能） |
| `_fix_unclosed_div.py` / `_check_div_balance.py` | 修复未闭合 `<div>`（img 补闭合/孤儿删除）/ 收支校验 |
| `_fix_eq_pairs.py` | 成对 `==` 假高亮修复（C 表达式包反引号） |
| `_fix_final.py` | 句中广告残留 + 跨行公式边界空格 + `\xrightarrow` 乱码修复 |
| `_final_scan.py` / `_scan_bounds.py` / `_scan_artifacts.py` / `_scan_div.py` / `_scan_unclosed_divs.py` | 内容终检扫描（广告/边界空格/反斜杠/重复行/div 分类） |
| `_scan_headings.py` / `_dump_headings.py` / `_categorize_408.py` | 标题扫描/导出/分类诊断 |


## 常用命令速查

```bash
# —— 第 0 步：OCR 前 PDF 预裁剪（去页眉页脚，扫描版新书必做）——
python pipeline/scripts/_calc_pageno_pos.py 10 127        # 页码定位验证（基础篇 10~127）
python pipeline/scripts/_crop_liang_pdfs.py --all          # 全书裁剪+分章合成（基础/强化→第N章.pdf）
python pipeline/scripts/_crop_liang_pdfs.py --all --book 基础   # 只跑基础篇
python pipeline/scripts/_scan_headings.py              # 标题层级跳级检查（应全部"(无)"，必跑）
python pipeline/scripts/_norm_408_headings.py && python pipeline/scripts/_norm_408_pass2.py   # 408 标题层级统一
python pipeline/scripts/_norm_math_headings.py && python pipeline/scripts/_norm_math_pass2.py # 30讲数学标题层级统一
python pipeline/scripts/_norm_1000_headings.py         # 1000题标题层级统一
python pipeline/scripts/_resplit_6books.py --dry    # 生成到 _new_split/ 对比
python pipeline/scripts/_resplit_6books.py --apply  # 正式覆盖拆分稿
python pipeline/scripts/_verify_integrity.py        # 字数+图片校验
python pipeline/scripts/_verify_fingerprint.py      # 指纹校验
node pipeline/scripts/_validate_dir.cjs             # 公式校验
python pipeline/scripts/_scan_origin_structure.py   # books 结构扫描（拆分前必跑）
python pipeline/scripts/_check_chapter_exams.py     # 每章习题文件检查
python pipeline/scripts/_check_exam_gaps.py         # 习题区 vs 习题文件对比
node pipeline/scripts/_validate_blocks.cjs <md>     # 跨行 $$ 块 KaTeX 校验（补盲区，必跑）
python pipeline/scripts/_scan_dollar_balance2.py    # 公式边界解析 → 应"边界完整"
```

## 脚本的生成物

部分脚本把结果写到 `pipeline/` 顶层（**不是 `11408/`** —— 内容层不放流程产物）：

- `_gen_badlist.py` → `pipeline/公式待修清单.md` ：用 KaTeX 逐行校验 `11408/split/数学/30讲-*/` 的公式，列出渲染失败的行。
- `_gen_origin_list.py` → `pipeline/公式待修清单-origin定位.md` ：把上面的清单映射到 books 行号，便于对照原书补公式。

两者都要在**仓库根目录**执行（脚本内部用相对路径）。**当前两份清单均为 0 处**——历史上那批公式问题已全部修复。