# graphMet 任务集：从论文补充材料提取并按当前项目适配

> 来源：`media-1 (3).pdf`，重点为表 S13（第 16-22 页）和表 S14（第 22-25 页）；评价维度参考表 S1-S2（第 1-3 页）。  
> 适配基线：2026-07-21 的本地 graphMet 仓库。本文是任务规格，不代表所有任务已经实现。

## 1. 结论摘要

原 PDF 的任务不是项目开发待办，而是一组用于评估大语言模型处理 genome-scale metabolic model（GEM/GSM）能力的提示词。原始任务可归并为六类：

1. 模型理解与领域知识；
2. FBA/FVA 通量分析；
3. 代谢通路补全与整合；
4. 代谢通量优化与敲除分析；
5. 论文驱动的案例复现；
6. 模型错误检测。

为适应 graphMet，本文做了以下核心修改：

- 不再把完整 JSON 嵌入提示词，统一使用会话上传文件、附件 ID 或会话内模型路径，避免上下文窗口溢出。
- 通用任务默认使用仓库现有的 `models/ENGRO2_annotated.xml`；快速测试可使用 `models/e_coli_core.xml`。
- iML1515、Yeast8/yeast850、iJR904、iCW773 等物种特异任务不替换成人体模型，改为“用户上传后执行”的外部模型任务。
- 所有培养基、氧条件、碳源、目标反应和产物交换反应必须显式确认；禁止凭名称猜测反应 ID。
- 模型修改采用“候选生成 -> 证据核验 -> 人工确认 -> 会话副本修改 -> 前后对比验证”的闭环，不能直接修改仓库基准模型。
- 计算结果必须包含求解状态、输入与约束审计、关键数值、可复现 Notebook、结构化表格和证据来源。

## 2. graphMet 当前能力边界

### 2.1 可直接复用的能力

- Agent 编排：Supervisor、Reviewer、Retrieval、Execution。
- 模型文件：SBML/XML 与 COBRA JSON 加载。
- GEM 工具：`load_gem_model`、`run_fba`、`run_fva`、`find_dead_end_metabolites`、`find_blocked_reactions`、`suggest_gap_fill`、`gene_knockout_analysis`。
- 自定义分析：会话隔离的 `run_python`/Jupyter，支持 Notebook 与 artifact 输出。
- 证据检索：Neo4j、KEGG/外部数据库工具、PubMed 摘要与开放全文。
- 交互：上传模型、表格、论文 PDF；缺少培养基或高风险修改条件时可触发 human-in-the-loop。

### 2.2 需要自定义代码或新增封装的能力

- pFBA、批量单/双基因敲除、产率计算、growth-coupling 判定、FVSEOF。
- 模型质量检查的统一报告（质量/电荷平衡、参考模型差异、连通性损失）。
- 通路写入、导出修改后 SBML，以及修改前后验证流水线。
- iBridge 的标准化实现及与论文/参考代码的一致性测试。

### 2.3 当前缺少的输入资产

- iML1515、Yeast8/yeast850、iJR904、iCW773 模型。
- PDF 案例中用于计算产率、iBridge、FVSEOF 的具体论文与参考代码。
- `excluded_metabolites.txt` 和带有已知错误的基准模型变体。

因此，下文状态采用：

- **A - 原生可执行**：已有专用工具，补充少量编排即可。
- **B - 可执行但需代码沙箱**：可由 `run_python` 完成，仍需 Reviewer 核验。
- **C - 条件执行**：缺少模型、论文、参考代码或稳定实现，输入补齐后才能验收。

## 3. 整理后的任务目录

### A. 模型理解与领域知识

| ID | 从 PDF 提取的任务 | graphMet 适配后的任务 | 状态 | 可直接使用的 graphMet 提示词 | 主要产物 |
|---|---|---|---|---|---|
| DK-01 | 总结 iML1515，并展示完整结构树 | 对会话中的任意 GEM 生成模型卡；默认 ENGRO2。输出模型元数据、规模、区室、目标函数、培养基审计，以及区室 -> subsystem -> reaction 的可折叠摘要。禁止把数千个反应直接塞进一棵不可读的树。 | A | **提示词：**读取我上传的 GEM；若未上传则使用 `models/ENGRO2_annotated.xml`。生成模型卡，包含模型 ID、物种、反应/代谢物/基因/区室数量、目标函数和培养基审计。按“区室 -> subsystem -> 代表性反应”生成可读结构树，不要展开全部反应。区分模型中直接读取的事实与推断，并输出 Markdown 和 JSON。 | 模型卡 Markdown/JSON、结构摘要 |
| DK-02 | 列出以 acetyl-CoA 为底物的全部反应并总结通路 | 先解析 acetyl-CoA 在各区室中的真实 metabolite ID，再筛选化学计量系数小于 0 的反应；同时标注可逆性、subsystem、GPR 与证据来源。 | B | **提示词：**在我上传的 GEM 中识别所有区室里的 acetyl-CoA 代谢物 ID。列出 acetyl-CoA 化学计量系数小于 0 的全部反应，包含反应 ID、名称、方程、区室、可逆性、subsystem、GPR 和系数。随后按通路汇总并解释其作用。不要仅按名称搜索，也不要把产物侧反应误列为底物反应。输出 CSV、摘要和 Notebook。 | CSV 表、通路摘要、Notebook |
| DK-03 | 用代码完成 acetyl-CoA 反应识别 | 与 DK-02 合并为“代码落地版本”；不再把“是否写代码”当成独立科学任务，而是作为可复现性验收项。 | B | **提示词：**使用 graphMet 的代码执行环境完成 acetyl-CoA 反应识别。代码必须从模型对象程序化查找各区室 acetyl-CoA，而不是硬编码反应列表；筛选其作为底物的反应并生成 CSV。请实际运行代码，检查异常和空结果，先报告执行状态与关键结果，再提供可复现 Notebook 和简要方法说明。 | 可运行 Notebook、CSV |
| DK-04 | 总结 NAD(H)/NADP(H) 相关反应和通路 | 按区室区分氧化态/还原态辅因子，输出反应方向、净生成/消耗、subsystem，并避免把可逆反应的单一方向当成生理方向。 | B | **提示词：**分析我上传 GEM 中涉及 NAD+、NADH、NADP+、NADPH 的反应。按区室和 subsystem 分组，列出反应 ID、方程、可逆性及四种辅因子的化学计量系数。分别总结氧化还原辅因子的潜在生成与消耗路径；对于可逆反应，不要把方程书写方向当成唯一生理方向。输出反应表、通路统计和 Notebook。 | 辅因子反应表、通路统计 |
| DK-05 | 解释 biomass objective 的组成、作用及 WT/core 差异 | 对当前模型实际目标反应做化学计量解析；若模型不存在 WT/core 两套 biomass，则明确“不适用”，改为比较默认 biomass 与可选 maintenance/core objective。 | B | **提示词：**读取当前 GEM 的实际 objective，识别 biomass 相关反应并解析其所有组分、系数和区室。解释 biomass objective 在 FBA 中的作用。检查模型是否确实包含 WT 与 core biomass；若没有，请明确说明“不适用”，不要虚构比较，并改为比较默认 biomass 与可用的 maintenance 或其他核心目标。输出组成表和解释报告。 | Biomass 组成表、解释报告 |
| DK-06 | 设计 MVA 路径生产 limonene，选择宿主并分析瓶颈 | 作为证据驱动的通路设计任务：Retrieval 检索基因、酶、EC、反应和宿主证据；Execution 仅在适合的微生物模型上传后进行计算。ENGRO2/Human-GEM 不作为默认 limonene 生产宿主。 | C | **提示词：**基于文献和数据库证据，设计从 acetyl-CoA 经 MVA 路径生产 limonene 的方案。列出每一步的底物、产物、反应、酶、基因、EC、来源物种和异源表达需求；比较至少两个合理微生物宿主，分析前体、ATP/NADPH、毒性和调控瓶颈，并提出增产改造。如果要做模型计算，请先确认我已上传适合的微生物 GEM；不要默认使用 ENGRO2 或 Human-GEM。 | 通路表、宿主比较、证据链 |
| DK-07 | 基于截断 JSON 比较 iML1515 与 yeast850 | 删除“截断 JSON”方案。要求上传两个完整模型，分别程序化提取统计与通路集合后比较；若只收到一个模型，触发 human-in-the-loop。 | C | **提示词：**比较我上传的两个完整 GEM。分别程序化提取模型版本、规模、区室、目标、培养基、subsystem 和反应集合，再比较共享与特有反应/通路及代谢能力。如果只收到一个模型，请暂停并要求补充第二个模型；不要基于截断 JSON 或记忆补全缺失内容。输出差异表、可复现 Notebook 和限制说明。 | 模型差异表、Notebook |
| DK-08 | 不提供模型，仅凭知识比较 iML1515 与 yeast850 | 保留为领域知识问答，但必须与“基于文件的比较”分开标记；定量字段需要文献引用，不能伪装成模型实测结果。 | A | **提示词：**在不加载模型文件的前提下，从领域知识角度比较 iML1515 与 Yeast8，覆盖物种、细胞区室、中央碳代谢、发酵、氨基酸、脂质/甾醇代谢及工程应用。明确标注这是知识型比较而非文件实测；所有版本相关或定量信息附文献来源，并说明可能随模型版本变化。 | 带引用的比较报告 |
| DK-09 | GSM 应用、局限与未来方向 | 由 Retrieval 提供近期文献证据，覆盖菌株设计、通路优化、高值化学品、稳态假设、调控/动力学缺失、多组学与酶约束模型。 | A | **提示词：**以系统生物学和代谢工程视角综述 GSM 的应用、局限和改进方向。覆盖菌株设计、通路优化和高值化学品案例；讨论稳态假设、目标函数、网络缺口、调控与动力学缺失如何影响预测；总结多组学、酶约束、动力学/调控模型和 AI 辅助精化。使用近期可靠文献，并把事实、观点和推断分开。 | 综述式回答、引用列表 |
| DK-10 | 解释 growth coupling、方法、价值、限制与未来方向 | 输出 strong/weak coupling 的操作性定义，比较 FVA、production envelope、OptKnock 等方法；概念结论与具体模型计算结果必须分栏。 | A | **提示词：**解释代谢产物与生长耦联的定义，并给出 strong、weak 和 uncoupled 的可操作判定。比较 FVA、production envelope、OptKnock 等分析方法，说明工程价值、成功案例、模型预测与实验偏差、局限和未来方向。将概念说明与任何具体模型计算结果分开，并为案例和算法结论提供引用。 | 方法比较表、带引用说明 |

### B. 通量预测

| ID | 从 PDF 提取的任务 | graphMet 适配后的任务 | 状态 | 可直接使用的 graphMet 提示词 | 验收重点 |
|---|---|---|---|---|---|
| FP-01 | 对 iML1515 运行 FBA，并列出最高通量前 10 个反应 | 默认对 ENGRO2 运行；先审计培养基与目标反应，必要时请求确认。按绝对通量排序，同时保留正负号，排除或单独标注 exchange/demand/sink。 | A | **提示词：**对我上传的 GEM 运行 FBA；若未上传则使用 `models/ENGRO2_annotated.xml`。先展示默认目标函数与培养基审计；若培养基为空或过度开放，请暂停并让我确认。求解成功后报告 solver status、objective value 和实际约束，并按绝对通量列出前 10 个反应，同时保留正负号、名称和 subsystem。exchange、demand、sink 反应请单独标注。输出 CSV、Notebook 和审计 JSON。 | 求解状态为 optimal；报告 objective、约束和 top-10 |
| FP-02 | 对 iML1515 运行 FVA，找最大 flux range 的 10 个冗余反应 | 默认 ENGRO2，显式给出 `fraction_of_optimum`。将“冗余”改为“可变性最大”，因为大范围不自动等于生物学冗余；另列 blocked、fixed 与 high-variability 反应。 | A | **提示词：**对我上传的 GEM 运行 FVA；若未上传则使用 `models/ENGRO2_annotated.xml`，设置 `fraction_of_optimum=0.9`。先确认目标和培养基。分别列出 blocked、fixed 和 flux range 最大的 10 个反应，报告 minimum、maximum 和 range。不要把“大范围”直接称为“冗余”；请结合 subsystem 与可替代通路解释。输出完整 CSV、Notebook 和约束审计。 | minimum/maximum/range 正确；术语不误导 |

### C. 代谢通路构建

| ID | 从 PDF 提取的任务 | graphMet 适配后的任务 | 状态 | 可直接使用的 graphMet 提示词 | 验收重点 |
|---|---|---|---|---|---|
| PC-01 | 找出 iML1515 合成 limonene 的缺口并建议反应/酶 | 需要上传 iML1515 或其他微生物宿主模型。先判断目标产物、前体和产物交换是否存在，再用 blocked/dead-end 分析、KG 与文献生成候选。候选不得未经审核直接写入模型。 | C | **提示词：**分析我上传的微生物 GEM 是否具备 limonene 合成能力。先确认 limonene、IPP、DMAPP、GPP 及产物交换反应是否存在，再执行 dead-end、blocked reaction 和目标可达性分析。结合 graphMet 知识图谱与文献生成 gap-fill 候选，逐项给出方程、区室、酶、基因、EC、来源、置信度和风险。当前只提出候选，未经我确认不要修改模型。 | 每个候选含方程、区室、酶/基因、来源和置信度 |
| PC-02 | 比较 iML1515 与 yeast850 的 limonene 缺口、宿主优劣和增产策略 | 要求两个完整模型与一致的培养基/目标定义；分别计算前体可达性、理论产率与关键瓶颈，再结合实验文献评价宿主。 | C | **提示词：**比较我上传的两个完整宿主 GEM 在 limonene 生物合成方面的能力。先为两个模型设置一致且可审计的培养基、碳源和产物目标，再分别评估 IPP/DMAPP/GPP 可达性、缺失反应、理论产率和瓶颈。结合实验文献评价宿主优劣与增产策略。将模型计算结果、文献证据和推断分栏；若缺少任一模型或条件，请暂停询问。 | 计算条件一致；模型结论与文献结论分开 |
| PC-03 | 将 MVA-limonene 通路写入 iML1515 并运行 FBA | 在会话副本中完成 metabolite/reaction ID 发现、元素与电荷检查、化学计量检查、目标/交换反应设置和 FBA。写入前需人工确认，输出新 SBML/JSON 及修改清单。 | C | **提示词：**把经过确认的 MVA-limonene 通路加入我上传模型的会话副本。先程序化检查已有 metabolite/reaction ID，避免重复；验证区室、元素、电荷、方向和化学计量，并展示拟新增清单等待我确认。确认后添加产物交换/需求反应，设置目标并运行修改前后 FBA。输出新 SBML/JSON、Notebook、质量平衡报告和逐项变更清单，绝不覆盖原模型。 | 原模型不变；新增反应无重复；前后 FBA 与质量平衡通过 |

### D. 代谢通量优化

| ID | 从 PDF 提取的任务 | graphMet 适配后的任务 | 状态 | 可直接使用的 graphMet 提示词 | 验收重点 |
|---|---|---|---|---|---|
| FO-01 | 在 iML1515 上用 FVSEOF 寻找 succinate 上/下调靶点 | 做成通用 FVSEOF 工作流：输入模型、产物反应、培养基、强制产物水平序列和最低生长约束。每级执行 FVA，计算 minimum 与 maximum 对产物水平的趋势和稳健相关性。 | B/C | **提示词：**在我上传的 GEM 上运行通用 FVSEOF。目标产物反应为 `<product_reaction_id>`；先确认培养基、底物摄取和最低生长约束。生成从基线到最大可达产量的强制产物水平序列，在每一级执行 FVA，并计算各反应 minimum 和 maximum flux 与强制产量的相关性/斜率。过滤 blocked、常量和边界反应，报告前 10 个上调与下调候选、不可行层级、统计值和通路解释。保存 CSV、图和 Notebook。 | 过滤常量/blocked 反应；报告不可行层级、相关系数与多重候选 |
| FO-02 | 对 iML1515_limonene 做单敲除，在保留 10% 最大 biomass 时提高产率 | 使用完整模型上传；先计算 WT 最大 biomass，再固定 biomass 下限为其 10%，批量单基因删除并优化产物。报告产物 flux、葡萄糖归一化 yield 和生长变化。 | B/C | **提示词：**对我上传的产物通路模型执行全基因组 single-gene knockout。先在当前培养基下计算 WT 最大 biomass，再把 biomass 下限固定为 WT 的 10%，逐基因敲除并优化 `<product_reaction_id>`。报告每个候选的状态、biomass、产物 flux、相对 WT 变化和按实际底物摄取归一化的 molar yield。过滤不可行和无效结果，输出 top 候选、CSV 与 Notebook。 | 10% 约束基于同一培养基的 WT；区分 flux 与 yield |
| FO-03 | 在厌氧 iJR904 中建议并验证 succinate 敲除 | 保留为外部模型场景。氧交换 ID 需程序化识别/人工确认；候选来自文献或计算，随后逐个 KO，比较 biomass、succinate flux 与 yield。 | C | **提示词：**使用我上传的 iJR904 模型分析厌氧 succinate 生产。先程序化识别氧、葡萄糖和 succinate 交换反应并让我确认，设置可审计的厌氧培养基。计算 biomass 最优时的 succinate flux，再在至少保留 10% WT biomass 的条件下优化 succinate。结合文献或计算提出敲除基因，逐个验证 biomass、产物 flux 和 yield；不存在的基因/反应必须报错，不能静默跳过。 | 厌氧设置可审计；不存在的基因/反应不能静默跳过 |
| FO-04 | 判断 iML1515_limonene 的生产是否 growth-coupled | 使用 production envelope：在多个 biomass 水平下计算最小/最大产物通量，并区分 strong、weak、uncoupled。不能只检查单个最优解。 | B/C | **提示词：**判断我上传模型中的 `<product_reaction_id>` 是否与生长耦联。先确认培养基、biomass 和产物反应，计算最大生长率；随后在从 0 到最大生长率的多个水平上固定 biomass，并分别求产物最小和最大通量，绘制 production envelope。根据明确阈值判定 strong、weak 或 uncoupled，说明可行域、数值容差与异常状态。不要只依据一个最优解下结论。 | 有可视化 envelope、判定阈值和可行域说明 |

### E. 论文驱动的案例复现

| ID | 从 PDF 提取的任务 | graphMet 适配后的任务 | 状态 | 可直接使用的 graphMet 提示词 | 验收重点 |
|---|---|---|---|---|---|
| CV-01 | 阅读并总结论文 | 通过 PDF 附件解析全文；输出研究问题、模型、约束、算法、关键结果、局限和可复现参数。引用必须定位到论文内容，不能仅复述摘要。 | A | **提示词：**阅读我上传的论文 PDF。按研究问题、实验/计算模型、数据、培养基与约束、算法步骤、关键参数、主要结果、局限和可复现性风险进行结构化总结。引用应定位到正文、表格或图，不要只复述摘要。另输出一份“复现参数清单”，明确论文未报告或含糊的设置。 | 结构化摘要、引用、参数清单 |
| CV-02 | 基于论文计算 E. coli 中 L-lysine 的理论/最大可达产率 | 要求论文与 iML1515。先从论文提取 YT/YA 定义，再明确碳基、摩尔基或质量基归一化方式；用 D-glucose、有氧条件复现。 | C | **提示词：**基于我上传的论文和 iML1515 模型，复现 E. coli 使用 D-glucose、在有氧条件下生产 L-lysine 的 YT 与 YA。先从论文逐条提取 YT/YA 定义、公式、单位、培养基、氧和 ATP maintenance 设置；若定义不明确请暂停询问。程序化识别 glucose、oxygen、lysine 和 biomass 反应，运行计算，并并列报告论文值与模型复现值、差异及原因。输出公式、审计 JSON、CSV 和 Notebook。 | 单位与公式明确；模型值和论文值并列 |
| CV-03 | 计算 E. coli 在有氧/厌氧/微需氧下的 ethanol YT/YA | 将微需氧设定从论文中结构化提取；三种条件共享除氧约束外的培养基。输出每种条件的 biomass、产物 flux、底物摄取和 yield。 | C | **提示词：**基于我上传的论文和 iML1515，计算 E. coli 在有氧、厌氧和微需氧条件下的 ethanol YT 与 YA。先从论文提取微需氧的具体 O2 bound，不允许自行猜测。三种条件除氧约束外使用相同培养基和 glucose 设置。每种条件报告 solver status、biomass、ethanol flux、实际 glucose uptake、yield、公式和单位，并与论文结果比较。输出对比表和 Notebook。 | 条件可比；微需氧不是任意猜值 |
| CV-04 | 在 yeast850 上复现同一 ethanol 条件比较 | 与 CV-03 相同，但要求 Yeast8/850 模型，并额外审计线粒体/胞质区室与模型特有 maintenance/培养基设置。 | C | **提示词：**基于我上传的论文和 Yeast8/yeast850 模型，复现 S. cerevisiae 在有氧、厌氧和微需氧条件下的 ethanol YT 与 YA。提取并严格采用论文的氧设置，审计模型版本、培养基、maintenance、glucose/ethanol 交换反应及胞质/线粒体区室。报告每种条件的求解状态、biomass、产物 flux、底物摄取、yield、单位及与论文值的差异。 | 物种特异设置正确 |
| CV-05 | 在 iCW773 中加入 R-mevalonate 路径并计算 YT | 要求模型与通路表。先去重 internal metabolites，复制模型后添加路径；ATPM 下限设 0、glucose -10、O2 -1000 等论文条件必须作为实验配置记录，而不是隐藏在代码里。 | C | **提示词：**使用我上传的 iCW773 模型和 R-mevalonate 通路表，在会话副本中加入该通路并计算 D-glucose 有氧条件下的 theoretical molar yield。先检查内部代谢物和反应，避免重复，并验证区室、元素、电荷和化学计量。将 ATPM lower bound=0、glucose uptake=-10、O2=-1000 及其他条件记录为显式实验配置。确认新增清单后再修改，输出新模型、变更清单、FBA、yield 公式和 Notebook。 | 路径无重复、条件审计、molar yield 可复算 |
| CV-06 | 仅依据论文实现 iBridge | 先把论文算法转成伪代码、输入/输出与边界条件，再实现最小可验证版本；以小模型做单元测试后才运行大模型。当前 graphMet 无原生 iBridge 工具。 | C | **提示词：**仅依据我上传的论文实现 iBridge。第一步先提取算法定义、数学公式、输入/输出、排除规则、参数和停止条件，形成可审查的伪代码；明确论文未说明之处并暂停等待确认。第二步在 graphMet 代码环境实现最小版本，先用小模型和人工构造案例做单元测试，通过后再应用到上传的目标模型。输出实现、测试、运行日志、候选靶点和局限。 | 算法忠实性测试、失败模式、运行日志 |
| CV-07 | 依据论文和 GitHub 代码实现 iBridge，并应用 excluded metabolites | 将参考代码保存为会话附件，记录版本/commit；比较论文算法与代码差异，解析排除代谢物列表，并输出一致性测试。 | C | **提示词：**基于我上传的 iBridge 论文、参考代码和 `excluded_metabolites.txt` 实现并运行 iBridge。记录参考代码版本或 commit，先比较论文描述与代码行为的差异，再验证排除代谢物的 ID 映射与实际过滤效果。对同一小模型比较 graphMet 实现和参考实现输出，设定数值容差；一致性通过后再分析目标模型，并输出溯源、测试和候选表。 | 来源可追溯；排除列表实际生效 |
| CV-08 | 仅依据论文实现 FVSEOF | 复用 FO-01 的标准接口；先核对论文中 enforced levels、目标函数、FVA 设置和候选排序定义。 | B/C | **提示词：**仅依据我上传的论文复现 FVSEOF。先提取产物强制水平、目标函数、最低生长约束、FVA 参数、候选过滤和排序定义，并标注未报告参数。确认复现配置后，使用 graphMet 标准 FVSEOF 工作流在上传模型上运行；报告不可行层级、反应趋势、候选靶点、论文值对照和可复现 Notebook。 | 复现参数完整；结果可重复 |
| CV-09 | 依据论文和 GitHub 代码实现 FVSEOF | 在 CV-08 基础上记录参考代码版本，并对同一小模型比较 graphMet 与参考实现输出。 | C | **提示词：**基于我上传的 FVSEOF 论文和 GitHub 参考代码复现该算法。记录代码版本或 commit，比较论文与实现中的 enforced levels、FVA、相关性和排序逻辑。先在同一小模型上运行 graphMet 与参考实现，按明确容差比较数值和候选排名；解释差异后再运行目标模型。输出配置、对照结果、CSV、图和 Notebook。 | 数值差异有阈值和解释 |

### F. 模型错误检测

原 PDF 在表 S14 中为同一错误设计了多种提示方式。对 graphMet，最有价值的不是保留重复问法，而是把它们整理成两条可复现的基准轨道。

| ID | 原始提示变体 | graphMet 适配后的任务 | 状态 | 可直接使用的 graphMet 提示词 | 通过标准 |
|---|---|---|---|---|---|
| QC-01 | 文本判断、一般代码、COBRApy 专用代码、文本参考对比、代码参考对比：检测化学计量符号错误 | 从基准模型的会话副本制造一个已知符号错误。先用 `reaction.check_mass_balance()` 扫描非 boundary/biomass 反应，再与参考模型逐反应比较系数、方向、边界和 GPR。文本推理只作解释，不作为唯一检测方法。 | B | **提示词：**比较我上传的 reference model 与 test model，检测化学计量符号或质量/电荷平衡错误。只分析 internal reactions，排除 exchange、demand、sink 和 biomass pseudo-reactions。使用 COBRApy 扫描 mass/charge balance，并逐反应比较 metabolite coefficient、方向、bounds 和 GPR。返回异常反应 ID、名称、参考/测试方程、差异系数、受影响代谢物和证据类型；保存差异 CSV 和 Notebook。不要修改任何输入模型。 | 准确定位被修改反应；无基准模型写入；误报可统计 |
| QC-02 | 一般问题、询问缺失反应、一般代码、糖酵解引导检查、代码参考对比、文本参考对比：检测缺失反应 | 从基准模型副本删除一个已知 internal reaction。执行参考差异、代谢物连通性、blocked reaction 与目标可达性检查；若无参考模型，再使用预定义通路清单与 KG/文献作为较弱证据。 | B | **提示词：**比较我上传的 reference model 与 test model，识别缺失的 internal reactions 或丢失的代谢连通性。程序化比较反应集合、方程、bounds 和 GPR，再检查受影响代谢物、blocked reactions、关键通路完整性和目标可达性。排除 exchange、demand、sink 与 biomass pseudo-reactions。输出缺失反应 ID、名称、参考方程、受影响代谢物、功能后果和证据强度；若没有 reference model，请明确说明结论置信度较低，并使用预定义通路清单、KG 和文献交叉检查。 | 找到缺失反应 ID/名称/方程及受影响代谢物；解释功能后果 |

建议将 S14 的提示变体作为测试维度保留：

1. 无工具、泛化提示；
2. 允许代码、泛化提示；
3. 指定 COBRApy/专用工具；
4. 给出结构化分析步骤；
5. 提供参考模型并做程序化对比。

这样可以测量“提示与工具增强带来的增益”，而不是把五种问法当成五个产品功能。

## 4. 推荐的统一任务模板

每个 graphMet 任务应使用以下输入结构：

```yaml
task_id: FP-01
model:
  source: session_attachment
  path_or_artifact_id: <required>
analysis:
  objective_reaction: <explicit-or-confirm>
  medium_constraints: <explicit-or-confirm>
  organism: <optional-but-recommended>
  solver_tolerance: 1e-9
outputs:
  - summary_markdown
  - result_table_csv
  - reproducible_notebook
  - audit_json
validation:
  require_optimal_status: true
  require_reviewer: true
```

对于模型修改任务，额外要求：

```yaml
modification_policy:
  edit_session_copy_only: true
  require_human_confirmation: true
  validate_mass_charge_balance: true
  compare_before_after: true
  export_change_manifest: true
```

## 5. 推荐执行顺序

### Phase 1：立即可落地的基准

- DK-01、DK-02、DK-04、DK-05；
- FP-01、FP-02；
- QC-01、QC-02；
- CV-01。

这些任务能覆盖模型加载、代码执行、Reviewer、附件解析、Notebook 与结构化 artifact，且大多可用现有 ENGRO2/e_coli_core 完成。

### Phase 2：补齐通用算法封装

- FO-01 FVSEOF；
- FO-02 批量单基因删除；
- FO-04 production envelope/growth coupling；
- 统一产率计算与模型质量报告。

建议把这些算法从临时代码提升为经过测试的 graphMet 工具，并为每个工具增加小模型单元测试。

### Phase 3：准备外部基准资产

- 获取并固定版本的 iML1515、Yeast8、iJR904、iCW773；
- 保存模型来源、下载日期、校验和和许可证信息；
- 准备 limonene、succinate、ethanol、L-lysine 的目标反应与培养基配置；
- 构造只在会话副本中使用的 case1/case2 错误模型。

### Phase 4：论文算法复现

- CV-02 至 CV-09；
- 优先 FVSEOF，再实现 iBridge；
- 每个算法同时保留论文参数、参考代码版本、数值对照和失败案例。

## 6. 统一验收与评分规则

原 PDF 分别给领域任务和代码任务使用 1-5 分 rubric。对 graphMet，建议合并为 100 分制：

| 维度 | 权重 | 5 分档的最高标准 |
|---|---:|---|
| 计算正确性 | 25 | 求解、筛选、排序和单位均正确，关键结果可独立复算 |
| 可复现性 | 20 | 输入版本、约束、随机种子、代码、Notebook 和 artifact 完整 |
| 生物学有效性 | 15 | 物种、区室、方向、GPR、培养基和目标解释正确 |
| 完整性 | 10 | 所有子问题与所需输出均覆盖 |
| 鲁棒性与错误处理 | 10 | 文件缺失、ID 不存在、模型不可行和非最优状态均有明确处理 |
| 证据与可追溯性 | 10 | 模型事实、KG、论文和推断明确区分并可追溯 |
| 清晰与简洁 | 5 | 结构可读，表格优先，不输出不可浏览的原始大对象 |
| 修改安全性 | 5 | 只改会话副本，高风险步骤经确认，输出变更清单 |

最低通过线建议为 80/100，同时设置三个硬门槛：

- 计算任务未报告 solver status：直接不通过；
- 培养基/目标依赖结论却没有记录约束：直接不通过；
- 修改原始基准模型或无变更清单：直接不通过。

## 7. 可直接用于 graphMet 的示例提示词

### 示例 1：FBA

> 对我上传的 GEM 运行 FBA。先识别并展示默认目标函数与培养基审计；如果培养基为空或过度开放，请暂停并让我确认。求解成功后报告 objective value，并按绝对通量列出前 10 个反应，同时保留通量正负号、反应名称和 subsystem。将完整结果保存为 CSV 和可复现 Notebook。

### 示例 2：FVA

> 对我上传的 GEM 运行 FVA，`fraction_of_optimum=0.9`。分别列出 blocked、fixed 和 flux range 最大的 10 个反应。不要把“大范围”直接称为“冗余”；请结合 subsystem 与可替代通路解释，并输出 CSV、Notebook 和约束审计。

### 示例 3：缺失反应检测

> 比较我上传的 reference model 与 test model，只检查 internal reactions。用代码比较反应集合、化学计量系数、bounds 和 GPR，再检查丢失的代谢连通性及 blocked reactions。返回缺失反应的 ID、名称、方程、受影响代谢物和功能后果，并把“程序化事实”与“生物学推断”分开。

### 示例 4：安全的 gap filling

> 分析我上传模型中与目标产物相关的 dead ends 和 blocked reactions，结合 graphMet 知识图谱与文献生成 gap-fill 候选。先只给候选表，包括反应方程、区室、酶/基因、来源、置信度和潜在风险；未经我确认不要写入模型。确认后只修改会话副本，并提供修改前后 FBA、质量/电荷平衡、目标可达性和 SBML 变更清单。

## 8. 与原 PDF 相比最重要的修订

1. 将“把完整 JSON 放进提示词”改成文件/附件驱动，适配 graphMet 的上传、会话和 artifact 机制。
2. 将“输出代码”改成“执行、验证并保存 Notebook”；代码是否能运行比代码块是否完整更重要。
3. 将模糊的“top flux”“redundant reaction”“theoretical yield”改成带排序规则、定义和单位的可验证指标。
4. 将模型写入与 gap filling 变成有人审、可回滚、可审计的流程。
5. 将当前不存在的微生物模型和算法明确标记为条件任务，避免误用 ENGRO2/Human-GEM。
6. 将 S14 的重复提示合并为两条科学任务，并保留不同提示/工具条件作为基准实验维度。
