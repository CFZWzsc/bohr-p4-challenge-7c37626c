---
name: formula-structure-reasoner
description: Use when formulas from a paper must be parsed into symbols, definitions, derivation steps and structural or numeric checks.
---

# 公式解析、符号定义与推导校验

## 用途与适用条件

把论文公式还原为结构可校验的 LaTeX 与符号表，关联 where 定义、单位与出处，按等式拆出可检查的推导步骤，并在变量齐全时做安全数值代入。

适用：`/input/case.json`（或 `--case` 指定），可带 `/input/task.md` 与同目录材料，包括 `artifacts/publisher-source.xml` 这类出版商 XML。输入含 `formulas`/`equations`、正文公式行、MathML 节点或变量数值。

不适用：不做符号代数证明；缺少变量定义时不臆造含义；不对不可信表达式执行任意代码。

## 输入

- 主输入 `/input/case.json`：只读取其中真实的公式、符号定义段落与变量值，不根据文件名、标题或领域标签补造内容。
- 任务说明 `/input/task.md`：存在时作为任务语义依据；缺失时仅用 `case.json` 的显式字段。
- MathML/出版商 XML：按节点还原分式、上下标、括号层级与来源 ID；公式目录只是候选清单，不是线性化答案，也不等于推导。
- 路径解析顺序：`/input` → 案例目录 → 当前目录 → `/app`；找不到的路径写入 `errors`/`uncertainty`，不静默替换。
- 支持 TXT/MD/CSV/TSV/JSON/BibTeX/RIS/XML/HTML、DOCX/PPTX/XLSX（解包 XML）与 PDF（最小流解析）；无文本层或公式图像只登记状态，不伪装成已读。
- 仅用 CPython 3.9+ 标准库；不联网、不读取环境变量中的令牌、不修改 `/input`，只写 `--out` 与 `--work`。

## 处理流程

1. **公式抽取**：读取 `formulas`/MathML，并从正文识别含等号与上下标/希腊字母的行；去除公式编号与标签。
2. **LaTeX 还原**：为每个公式给出保留分式、上下标、括号层级、希腊字母与单位的 LaTeX，并记录 `source_id`/段落定位；图像或无文本层只能标未还原，不猜写。
3. **符号解析**：区分希腊命令、单字母与带下标符号；忽略 sin/cos/log 等函数名。
4. **定义关联**：用 where/is/denotes/为/表示 等模式匹配定义与单位，保留段落与逐字引文，并区分原文定义、上下文推断与常识补充。
5. **结构检查**：检查括号/花括号平衡、等号两侧非空、符号是否全部有定义。
6. **推导与数值校验**：按等号拆分左右侧与中间表达式，标明假设、近似与边界条件；仅当变量齐全且表达式通过安全 AST 白名单时代入，否则说明未执行。
7. **契约落盘**：保留 `case_id` 与契约字段，未解析符号标 `unresolved`。

## 输出契约

写入 `/app/output/result.json`（UTF-8 JSON，确定性排序，无随机、无时间戳）。顶层必须保留：

- `case_id`：与输入完全一致（缺失时为 `null`）；
- 输入 `output_contract` 声明的每一个字段名，原样保留；无法确定性映射的写 `null`，并在 `contract_notes` 说明原因；
- `status`：`success` / `partial` / `insufficient_input` / `failed`；
- `result`、`evidence`、`uncertainty`、`errors`、`stats`、`contract_notes`、`skill`、`challenge_id`。

主要交付段与必填子字段：

| 段 | 必填内容 |
|---|---|
| `result.formulas[]` | `formula_id`、`latex`、`source_id`、`status`（`verified`/`needs_review`）、`symbols`、`variables`、`checks`、`derivation_steps`、`numeric_check` |
| `result.variable_table[]` | `symbol`、`definition`、`unit`、`status`（`defined`/`unresolved`）、`paragraph_id`、`quote` |
| `result.derivation_steps[]` | `formula_id`、`steps[]`（`step`、`expression`、`kind`、`assumptions`） |
| `result.checks[]` | `formula_id`、`check`、`passed`、`detail` |
| `result.coverage` | `formula_candidates`、`defined_symbols`、`unresolved_symbols`、`paragraph_units` |
| `evidence[]` | 每条含 `formula_id`、`paragraph_id`、`quote`、`source_id`、`locator` |
| `needs_human_confirmation[]` | `item`、`locator`、`reason`：符号歧义、公式截断、单位缺失、无可核验来源时登记 |

字段名以实际 `output_contract` 为准；不删除、不改名、不放宽契约字段。

## 交付成品要求

1. **交付成品而不是流程说明**：脚本只产出结构骨架；调用 Agent 必须补齐每个公式的 `latex`、符号定义、推导步骤与核验结论，禁止 `pending`、`TODO` 或空交付段。
2. **覆盖全部公式**：输入中的每个公式节点与公式行都要有条目，不得只处理首行或首个节点；真正缺失的写入 `uncertainty` 并说明原因。
3. **不反推、不补零**：案例没有公式表示、逐条预测或可复算数据时，报告 `partial` 或 `insufficient_input`；不反推公式，不把未知填 0，也不把目录标签当公式。
4. **自检**：`case_id` 与输入一致；`output_contract` 每字段存在或 `null`+`contract_notes`；`status` 取上述枚举；LaTeX 与符号表、推导、核验结论互相自洽；重复运行结果相同。
5. **参数等价**：`--out` 与 `--output` 等价；默认 `/input/case.json` → `/app/output/result.json`。

## 还原口径

- 分式、上下标、括号层级、希腊字母与单位逐项对应原文，不合并、不丢项、不错位。
- 每个符号定义必须指回原文段落；找不到定义标 `unresolved`，不按常识补。
- 推导只重排原文等式，不跳步、不新增变换；假设、近似与边界条件单独列出。

## 异常与降级

| 情况 | 处理 | 输出表现 |
|---|---|---|
| `case.json` 缺失或非法 JSON | 不改写输入，写出结构化失败结果 | `status: insufficient_input`（解析错误为 `failed`），`errors` 记原因 |
| 配套材料缺失 | 继续处理可用材料，不猜测清单 | `errors` 记录缺失文件，相关段为空并说明 |
| PDF 无文本层、文件损坏或无权限 | 登记状态并跳过，继续其余材料 | `read_status`/`uncertainty` 标注 |
| 公式截断、符号严重歧义 | 保留原样并标记，不补全 | `needs_review`，`needs_human_confirmation` 登记 |
| 符号找不到定义 | 保留 `null`，不推断 | `status: unresolved`，`uncertainty` 说明 |
| 多来源数值冲突 | 保留全部候选值与定位，不平均不择一 | 标 `source_conflict`，`evidence` 给出两处定位 |
| 运行异常 | 捕获后输出结构化失败 | `status: failed`，`errors` 含异常类型 |

进程正常结束（含失败分支）退出码恒为 `0`，失败类别只看 `status`，不要用退出码判断成败。

## 限制

仅做结构与代入校验，不做量纲代数或符号证明；复杂嵌套分式、矩阵与多行对齐公式可能解析不全。

## 最小调用示例（示例，非官方案例结果）

```bash
python scripts/run_skill.py --case /input/case.json --out /app/output/result.json --work /app/work
```

示意输入（合成，仅演示接口）：

```json
{"case_id":"synthetic-smoke","output_contract":{"formulas":"array"},"formulas":["E = m c^2"],"variable_values":{"m":2,"c":3}}
```

示意输出（合成，仅演示字段形状，不代表任何真实案例或评分）：

```json
{"case_id":"synthetic-smoke","formulas":[{"formula_id":"F01","latex":"E = m c^2","status":"verified","numeric_check":{"left":18.0,"right":18.0,"consistent":true}}],"status":"success","result":{},"evidence":[],"uncertainty":[],"errors":[],"stats":{},"contract_notes":{}}
```

## 自查清单

- [ ] `case_id` 与输入一致；`output_contract` 每个字段都存在，或记为 `null` 并写入 `contract_notes`。
- [ ] 每个公式的 LaTeX 与原文结构一致，每个符号能指回段落定位；无编造作者、年份、DOI、页码或引文。
- [ ] 未还原、截断、歧义、冲突与无定义项均已登记，未写成“全部完成”。
- [ ] 重复运行结果相同；未联网、未修改 `/input`；未声称平台评分或官方案例结果。
