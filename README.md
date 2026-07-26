# interaction-ruler
A counterfactual ruler for measuring whether motion forecasters actually learn multi-agent interaction — not just accuracy.

# interaction-ruler

Do trajectory prediction models actually learn *which neighbor matters*,
or do they just get accurate by extrapolating their own motion?

Standard metrics (minADE/minFDE) can't tell the difference — a model that
ignores every neighbor can still score well. This repo provides a
**counterfactual ruler** that measures interaction directly: delete a
neighbor from the model's input and see whether the prediction changes
*more* for neighbors that actually matter.

## The one-line finding

Interaction ability is a property of **architecture / inductive bias**, not
of optimization. In a single training run, accuracy keeps improving while
interaction selectivity saturates within the first few epochs — and even the
current SOTA sits far below a perfectly-selective oracle.

## How the ruler works

1. **Relevance** — for each neighbor, a model-free score for how much it
   *should* influence the focal agent (based on time-headway + lateral
   corridor, not "will it collide").
2. **Perturbation** — remove that neighbor from the observation window only;
   the ground-truth future is never touched.
3. **Score** — *distance-matched AUC*: among neighbor pairs at similar
   distance, how often is the model more sensitive to the relevant one?
   - `0.500` = blind (constant-velocity baseline)
   - `1.000` = perfectly selective (oracle)

Both anchors are verified, so any model's score is placed on a calibrated
scale.

## Status

- [x] Ruler defined and calibrated at both ends
- [x] Passes a 6-scene hand-labeled sanity exam
- [x] QCNet adapter validated (minADE₁ 1.610 vs official 1.69)
- [ ] Second and third model families
- [ ] Long-tail correlation (does interaction-AUC predict tail performance?)

## Repo layout

- `PLAN.md` — the single source of truth for *what we're proving*. Read this first.
- `LOG.md` — every experiment: hypothesis → result → did the claim change?
- `src/` — the ruler and model adapters
- `notebooks/` — execution only; no reasoning lives here

## Data

Argoverse 2 (official validation split, re-preprocessed without filtering).
The train/val/test split is **frozen**; the test set is touched exactly once.

## Research Question_260726
现有的代表性SOTA轨迹预测模型面对受控的Agent级反事实干预时，能否对真正相关Agent产生方向正确、强度合理且稳定的响应，同时对无关Agent保持不变？

## Research Hypothesis_260726
| 假设                   | 含义                                  | 对应实验                          |
| -------------------- | ----------------------------------- | ----------------------------- |
| **H1：因果选择性**         | 模型应当对关键Agent的干预比对无关Agent的干预更敏感      | 分别扰动关键Agent和匹配的无关Agent，比较预测变化 |
| **H2：方向正确性**         | 关键Agent变化后，目标轨迹应朝交通逻辑允许的方向变化        | 前车减速、车辆切入、交叉冲突等干预             |
| **H3：剂量—响应单调性**      | 干预越强，目标Agent的避让、减速或改道响应应逐步增强        | 设置多个连续干预等级                    |
| **H4：预测精度与因果鲁棒性不等价** | ADE/FDE排名靠前的模型，不一定在因果响应上最好          | 比较传统精度排名和因果指标排名               |
| **H5：模型解释未必忠实**      | Attention、Shapley或内部重要性与实际干预效应可能不一致 | 比较内部归因和外部反事实响应                |
| **H6：因果响应可能训练不稳定**   | 同一架构换随机种子后，可能依赖不同Agent              | 多随机种子重复训练或测试                  |

## Experiment Design_260726
Level 1 受控因果场景:
我们在此借鉴SpurGen的思路，但面向轨迹预测场景来构造。首先我们选择3类清晰交互：
1. 跟驰减速：前车逐渐增强制动；目标车应逐渐减速或增大间距。
2. 侧方切入：切入车辆逐渐缩短横向距离或提前切入时间；目标车应逐渐减速、避让或改变模式概率。
3. 交叉/合流冲突：对方车辆逐渐缩小到达冲突点的时间差；目标车的让行概率应逐渐提高。
同时加入明确无关Agent，例如远离冲突区域、运动方向发散的车辆。 通过这一层实验，我们应该可以知道：谁被干预，干预强度，合理的干预响应方向，哪些Agent明确无关。

Level 2 真实Argoverse 2高置信场景:
从Argoverse 2筛选：
1. 明确跟驰；
2. 明确切入；
3. 明确合流；
4. 明确交叉冲突。
对真实场景进行物理约束下的轨迹编辑，再测试不同模型。

在这里，我们不声称拥有完整反事实GT，主要评价：
1. 响应方向；
2. 单调性；
3. 因果与无关干预的区分；
4. 跨模型和跨种子稳定性。
绝对响应幅度是否正确，主要应在受控场景中评价；真实AV2里更适合评价相对合理性。

## Intervention Protocal_260726
对关键Agent设置连续等级：
            α∈{0, 0.25, 0.5, 0.75, 1.0}
例如前车制动：α=0：原始轨迹；α=0.25：轻微减速；α=0.5：中等减速；α=0.75：明显制动；α=1.0：强制动。

然后观察目标预测：纵向位移是否减少；速度是否下降；与前车距离是否增大；换道或避让模态概率是否提高；多模态分布是否整体转移。

无关Agent也采用相近幅度的速度、位置或删除干预，防止关键Agent的扰动只是“更大”。

以上叫做匹配干预：因果组和无关组的扰动规模尽量一致。

## Indicator_260726
1. 因果选择差距：比较关键Agent和无关Agent干预造成的预测变化，如下：
                     CSG=Rcausal​−Rnoncausal​

其中 R 是目标预测变化量，越大说明模型越能区分真正相关因素与无关因素。它相当于把CADET的CSI和CRI放到一个相对比较框架中。CADET已经说明，仅看稳定性可能通过过度mask获得，因此真实响应和无关稳定性必须一起测。

2. 方向正确率：例如前车制动后，目标预测是否更保守：
              DRA=总干预次数/方向正确的干预次数
这个类似于CADET中的CRI，但要针对跟驰、切入、交叉等场景分别定义正确方向。

3. 单调性得分：干预强度从弱到强时，模型响应是否保持有序：
              α1​<α2​⇒R(α1​)≤R(α2​)
可使用：有序对正确比例；Spearman相关系数；单调性违例次数。

4. 响应幅度指标
检查模型是否：反应不足；反应合理；反应过度。
受控场景中可以用参考轨迹或简化动力学oracle评价；真实Argoverse 2中先报告归一化响应曲线，不急着宣称绝对校准。

5. 归因—响应一致性
比较Attention/Shapley重要性和真实干预造成的预测变化
可以使用Spearman相关系数或Top-k重合率。

6. 跨种子稳定性
同一个场景、同一种干预，在不同随机种子模型上比较：响应方向一致率；响应曲线方差；Agent重要性排名的一致性。传统ADE、FDE、MR仍然保留，用来验证：传统精度与因果响应质量是否存在脱节。





## 2. Research Question and Research Hypotheses_260726

### 2.1 Core Research Question

> **Can modern motion forecasting models respond selectively, directionally correctly, monotonically, and consistently to controlled interventions on interaction-relevant agents, while remaining stable under matched interventions on irrelevant agents?**

中文表述：

> **现有轨迹预测模型面对受控的 Agent 级反事实干预时，能否对真正与目标车辆交互相关的 Agent 产生有选择性、方向正确、强度有序且稳定的预测响应，同时对匹配的无关 Agent 保持稳定？**

本项目的重点不是重新训练一个预测精度更高的模型，而是构建一套统一的外部评测协议，用于检验不同轨迹预测模型是否真正学到了合理的 Agent 交互规律。

本项目需要区分两种实验语境：

1. 在受控或半合成场景中，关键 Agent 与无关 Agent 的身份由实验构造确定；
2. 在 Argoverse 2 真实场景中，只能识别具有明确交通冲突关系的高置信交互 Agent，不能将其表述为具有绝对真实因果标签的 Agent。

因此，本项目更严谨的研究对象是：

> **The causal response behavior exhibited by motion forecasting models under controlled interventions.**

即：

> **模型在受控干预下表现出的因果响应行为，而不是从真实轨迹数据中恢复完整的真实因果图。**

---

### 2.2 Minimal Notation

设：

- $m$：被测试的轨迹预测模型；
- $s$：一个交通场景；
- $c$：高置信关键 Agent；
- $n$：与关键 Agent 匹配的无关 Agent；
- $j$：任意被干预 Agent；
- $\alpha$：干预强度；
- $\hat{Y}_m^{(s,0)}$：模型在原始场景中的预测；
- $\hat{Y}_m^{(s,j,\alpha)}$：对 Agent $j$ 施加强度为 $\alpha$ 的干预后，模型产生的预测；
- $D(\cdot,\cdot)$：两个预测结果之间的距离函数。

模型对一次干预的总体响应大小定义为：

```math
R_m(s,j,\alpha)
=
D\left(
\hat{Y}_m^{(s,j,\alpha)},
\hat{Y}_m^{(s,0)}
\right)
```

其中，$R_m(s,j,\alpha)$ 只表示模型预测发生了多大变化，并不代表该变化方向一定正确。

为了评价响应方向和剂量—响应关系，进一步定义场景相关的语义响应量：

```math
z_m(s,j,\alpha)
```

统一约定：

> $z_m$ 越大，表示模型在当前场景下表现出的合理减速、避让、让行、延迟进入冲突区域或其他安全响应越强。

例如：

- 在跟驰场景中，$z_m$ 可以表示预测减速程度或跟车间距增加量；
- 在交叉冲突场景中，$z_m$ 可以表示进入冲突区域的延迟；
- 在切入场景中，$z_m$ 可以表示减速、避让或换道模态概率的综合变化。

需要注意：

- $R_m$ 衡量的是“模型变化了多少”；
- $z_m$ 衡量的是“模型是否朝交通语义合理的方向变化”；
- 一个模型可能具有较大的 $R_m$，但其 $z_m$ 变化方向仍然错误。

---

### 2.3 Research Hypothesis Overview

| ID | Research Hypothesis | Core Question | Main Reference Basis | Main Gap Addressed | Priority |
|---|---|---|---|---|---|
| **H1** | **Causal Selectivity** | 模型是否对关键 Agent 比对匹配无关 Agent 更敏感？ | CausalAgents；CADET 的 CSI 和 CRI | 缺少同场景、同类型、同强度的关键 Agent 与无关 Agent 成对比较 | Core |
| **H2** | **Directional Correctness** | 关键 Agent 被干预后，模型是否朝合理方向响应？ | CADET 的 CRI；CounterScene 的冲突语义干预 | 仅测预测变化大小，无法判断变化方向是否正确 | Core |
| **H3** | **Dose–Response Monotonicity** | 干预越强，模型的合理响应是否总体越强？ | CounterScene 的渐进式安全裕度压缩；CaDeT 的多次干预 | 现有工作多为单次二元干预，缺少有序响应曲线评价 | Core |
| **H4** | **Accuracy–Causality Decoupling** | ADE/FDE 更好的模型，因果响应是否也一定更好？ | CADET；CausalAgents；CaDeT；Beyond Patterns | 预测精度仍被大量工作用作因果或鲁棒能力的主要证据 | Core |
| **H5** | **Attribution–Response Misalignment** | Attention、Shapley 或消融归因是否忠实对应外部干预响应？ | Super Agents and Confounders；CADET | 模型“依赖谁”不等于模型“应该依赖谁” | Secondary |
| **H6** | **Cross-Seed Response Instability** | 同一架构不同随机种子是否学到相似的交互响应规律？ | Super Agents and Confounders | 已有工作主要研究归因不稳定，尚未系统评价响应曲线稳定性 | Secondary |

---

### 2.4 H1: Causal Selectivity

#### Hypothesis

> 在匹配的干预条件下，模型对关键 Agent 的干预响应应显著大于对无关 Agent 的干预响应。

形式化表示为：

```math
\mathbb{E}_{s,\alpha}
\left[
R_m(s,c,\alpha)
\right]
>
\mathbb{E}_{s,\alpha}
\left[
R_m(s,n,\alpha)
\right]
```

其中：

- $c$ 是高置信关键 Agent；
- $n$ 是与 $c$ 在类别、距离、历史长度和扰动幅度等方面尽量匹配的无关 Agent。

#### Reference Basis

CausalAgents 提出了基于人工标注的 Causal Agents 与 Non-Causal Agents 对轨迹预测模型进行鲁棒性测试的思路。

其核心观点是：

> 模型不应因为删除一个与目标行为无关的 Agent 而产生明显预测变化。

CADET 进一步把这一问题拆分为两个方面：

- CSI：无关因素改变时，规划结果应保持稳定；
- CRI：真正相关因素改变时，规划结果应产生合理响应。

#### Limitation Addressed

已有工作通常分别评价：

- 模型对无关 Agent 是否稳定；
- 模型对关键 Agent 是否响应。

但是，两组干预未必在以下方面具有可比性：

- 干预幅度；
- Agent 类型；
- Agent 与目标车辆的距离；
- 历史轨迹长度；
- 干预方式；
- 轨迹修改时间窗。

例如，若关键 Agent 被移动了 $5$ 米，而无关 Agent 只被移动了 $0.2$ 米，那么模型对关键 Agent 反应更大，并不能证明模型真正识别了交互关系。

结果可能只是因为关键 Agent 的输入变化幅度更大。

因此，本项目采用匹配干预，将关键 Agent 和无关 Agent 放在尽量一致的干预条件下比较。

#### Important Interpretation

H1 成立只能说明模型具有较好的交互选择性，即：

> 模型对可能影响目标车辆的 Agent 更敏感，而对无关 Agent 相对稳定。

H1 不能单独证明模型响应正确。

模型可能对关键 Agent 产生非常强烈但方向错误的反应。例如：

```text
前车增强制动
→ 模型预测目标车辆加速
```

虽然模型的预测变化很大，但这种响应在交通语义上是错误的。

因此，H1 必须与 H2 联合评价。

---

### 2.5 H2: Directional Correctness

#### Hypothesis

> 当关键 Agent 的行为朝更高冲突或更高风险方向变化时，模型对目标 Agent 的预测应朝交通语义合理的方向变化。

由于已经约定 $z_m$ 越大表示合理安全响应越强，因此 H2 可以表示为：

```math
z_m(s,c,\alpha)
-
z_m(s,c,0)
>
\epsilon_{\mathrm{dir}}
```

其中：

- $\alpha > 0$ 表示风险增强型干预；
- $\epsilon_{\mathrm{dir}}$ 是用于过滤数值噪声的容忍阈值。

#### Examples

##### Car-Following

```text
前车制动增强
→ 目标车预测速度降低
→ 或预测跟车间距增大
```

##### Intersection Conflict

```text
对方车辆更早到达冲突点
→ 目标车预测更晚进入冲突区域
```

##### Lateral Cut-In

```text
旁车切入趋势增强
→ 目标车减速、避让或换道模态概率提高
```

##### Merge and Yield

```text
合流车辆更早进入目标车道
→ 目标车让行、减速或增大合流间隔的概率提高
```

#### Reference Basis

CADET 的 CRI 已经提出：

> 当真正相关的 query 被修改后，规划结果的变化方向应与预期安全响应方向一致。

CounterScene 强调，反事实修改应沿具体交通冲突机制进行，例如：

- 压缩空间安全裕度；
- 缩短双方到达冲突区域的时间差；
- 增强关键 Agent 进入冲突区域的趋势。

这比无结构的随机轨迹扰动更具有交通语义。

#### Limitation Addressed

CausalAgents 和 CaDeT 等工作主要评价：

- 删除或扰动 Agent 后，预测误差变化了多少；
- 模型在扰动下是否保持稳定。

但是：

> 预测发生明显变化，不等于预测变化方向正确。

例如，前车增强制动后，目标预测发生巨大变化，但目标车辆反而被预测为加速，那么这种响应仍然是不合理的。

CADET 虽然已经检查响应方向，但其 CRI 主要是二元方向判断，而且研究对象主要是规划器。

本项目需要将方向判断扩展到：

- 多模态轨迹预测；
- 多种交通交互类型；
- 多种可能的合理响应方式；
- 轨迹坐标变化与模态概率变化。

#### Important Interpretation

不能简单规定所有风险增强型干预都必须让目标车辆减速。

在不同场景中，合理响应可能包括：

- 减速；
- 换道；
- 横向避让；
- 延迟进入冲突区域；
- 提高某个安全模态的概率；
- 在安全条件允许时提前通过。

因此，H2 需要针对不同场景定义对应的合理响应集合，而不是只检查单一的纵向位移变化。

---

### 2.6 H3: Dose–Response Monotonicity

#### Hypothesis

> 当关键 Agent 的干预强度逐渐增强时，模型表现出的合理响应总体上也应逐渐增强，而不应无规律跳变或频繁翻转。

设干预强度满足：

```math
0
=
\alpha_0
<
\alpha_1
<
\cdots
<
\alpha_K
```

理想情况下，应满足：

```math
z_m(s,c,\alpha_{k+1})
\geq
z_m(s,c,\alpha_k)
-
\epsilon_{\mathrm{mono}}
```

其中，$\epsilon_{\mathrm{mono}}$ 用于容忍轻微数值波动。

#### Reference Basis

CounterScene 通过逐渐压缩关键 Agent 与 ego 之间的空间和时间安全裕度，生成不同风险强度的反事实场景。

这一设计为本项目提供了一个重要启发：

> 干预不应只有“原场景”和“删除后场景”两个离散状态，而应具有连续且可解释的风险强度。

CaDeT 在潜在空间中进行多次干预，用于评价模型对虚假表示变化的鲁棒性。

#### Limitation Addressed

现有工作大量采用如下二元测试：

```text
原始场景
vs.
一个删除或扰动后的场景
```

这种测试无法发现模型是否存在：

- 轻微干预就产生极端反应；
- 中等干预反而比弱干预反应更小；
- 响应方向随着干预强度反复翻转；
- 只有达到极端风险时才突然响应；
- 不同模型具有完全不同的响应阈值；
- 响应曲线在局部区域出现明显不连续。

CaDeT 虽然进行了多次干预，但这些干预主要用于潜在表示鲁棒性分析，不一定具有明确的“风险由弱到强”的有序交通语义。

#### Important Interpretation

H3 要求的是语义响应总体单调，而不是每一个预测坐标都必须严格单调。

例如，随着切入风险增强，模型可能先选择：

```text
保持车道并减速
```

随后切换为：

```text
主动换道避让
```

此时轨迹坐标可能出现非线性变化，但总体冲突规避程度应增强。

因此，我们需要为不同交互场景定义一个统一方向的语义响应量 $z_m$，而不是直接要求所有预测坐标逐点单调。

---

### 2.7 H4: Accuracy–Causality Decoupling

#### Hypothesis

> 在 ADE、FDE 或 MR 上表现更好的预测模型，不一定在因果选择性、方向正确性和剂量—响应单调性上表现更好。

定义传统预测精度排名与因果响应排名之间的相关性：

```math
\rho_{\mathrm{rank}}
=
\mathrm{Spearman}
\left(
\mathrm{Rank}_{\mathrm{accuracy}},
\mathrm{Rank}_{\mathrm{causal}}
\right)
```

H4 预测：

> 两套排名之间可能出现明显差异，包括较低的排名相关性以及具体模型之间的排名翻转。

本项目不应提前规定 $\rho_{\mathrm{rank}}$ 必须低于某个固定值，而应由实验结果判断差异程度。

#### Reference Basis

CADET 发现，TCM 改变了 SparseDrive 对虚假 query 的依赖，并改变了 CSI，但 open-loop L2 几乎没有变化。

这一结果说明：

> 位移误差不能充分反映模型依赖的是合理交通因素，还是虚假线索。

CausalAgents 说明，一个模型在常规预测指标上表现正常，并不代表它对非因果 Agent 具有稳定性。

CaDeT 和 Beyond Patterns 尝试通过因果表示、反事实学习或 backdoor adjustment 提高轨迹预测性能，但仍主要通过以下指标证明效果：

- ADE；
- FDE；
- RMSE；
- 泛化性能；
- 扰动鲁棒性。

这些指标尚不能独立证明模型对 Agent 干预具有正确且有序的响应。

#### Limitation Addressed

目前缺少一套统一协议，将不同轨迹预测范式放在相同干预条件下比较，例如：

- QCNet；
- HPNet；
- MTR；
- HiVT；
- Forecast-MAE；
- 具有因果模块的轨迹预测模型。

因此，尚不清楚：

> 传统预测排行榜上的强模型，是否也是更懂 Agent 交互的模型。

#### Important Interpretation

H4 不是预设高精度模型一定具有较差的因果响应。

它检验的是：

> 传统预测精度是否足以作为模型因果交互能力的代理指标。

即使传统精度与因果指标存在一定正相关，只要出现稳定的排名翻转或典型失败案例，仍然可以说明新增因果评测指标具有独立价值。

例如：

```text
Model A:
- ADE 更低
- 但对无关 Agent 非常敏感
- 对前车制动的响应不单调

Model B:
- ADE 略高
- 但因果选择性和方向正确性更好
```

这类结果就可以说明：

> 传统精度和因果响应能力不是完全等价的评价维度。

---

### 2.8 H5: Attribution–Response Misalignment

#### Hypothesis

> 模型内部或事后归因方法所识别的重要 Agent，不一定对应受控外部干预下实际影响最大且响应方向正确的 Agent。

设：

- $A_{m,j}$：模型对 Agent $j$ 的归因分数；
- $\bar{R}_{m,j}$：对 Agent $j$ 进行多个强度干预后的平均响应大小。

平均响应大小定义为：

```math
\bar{R}_{m,j}
=
\frac{1}{K}
\sum_{k=1}^{K}
R_m(s,j,\alpha_k)
```

H5 预测，在部分场景中：

```math
\mathrm{Rank}
\left(
A_{m,j}
\right)
\ne
\mathrm{Rank}
\left(
\bar{R}_{m,j}
\right)
```

#### Reference Basis

Super Agents and Confounders 使用 Shapley attribution 研究不同周围 Agent 对预测性能的贡献，并发现：

- 部分 Agent 会降低预测性能；
- 人工标注的 Causal Agents 不一定稳定获得较高正向归因；
- 不同训练运行可能形成不同的 Agent 依赖机制。

CADET 也表明，influence-only 方法只能识别：

> 移除哪个 Agent 会明显改变输出。

但它无法区分这种影响是否具有物理和交通合理性。

#### Limitation Addressed

Attention、Shapley 和 query ablation 主要衡量：

> 模型实际上使用了谁。

它们不能单独回答：

> 模型是否应该依赖这个 Agent，以及它对该 Agent 的响应方向是否合理。

此外，不同模型的原生 Attention 结构并不相同，因此 Attention 权重未必可以直接进行跨架构比较。

#### Important Interpretation

H5 评价的是解释忠实度，而不是直接证明真实因果性。

即使归因与外部响应高度一致，也可能出现：

```text
模型稳定依赖了错误 Agent
→ 归因结果正确识别了这种错误依赖
→ 外部干预也确认模型确实依赖该 Agent
→ 但这种依赖在交通语义上仍然不合理
```

因此，H5 必须与 H1 和 H2 联合分析。

H5 更准确地回答的是：

> 模型的解释结果是否忠实描述了模型自身行为？

而不是：

> 该解释是否等于真实世界因果关系？

---

### 2.9 H6: Cross-Seed Response Instability

#### Hypothesis

> 同一模型架构使用不同随机种子训练后，可能学习出不同的关键 Agent 依赖、响应方向、响应阈值和剂量—响应曲线。

设同一架构的两个随机种子版本为：

```math
m^{(u)},\quad m^{(v)}
```

H6 预测，在部分场景中：

```math
R_{m^{(u)}}(s,c,\alpha)
\not\approx
R_{m^{(v)}}(s,c,\alpha)
```

更严重的情况是，不同随机种子可能对相同干预产生相反方向的响应：

```math
\mathrm{sign}
\left(
z_{m^{(u)}}(s,c,\alpha)
-
z_{m^{(u)}}(s,c,0)
\right)
\ne
\mathrm{sign}
\left(
z_{m^{(v)}}(s,c,\alpha)
-
z_{m^{(v)}}(s,c,0)
\right)
```

#### Reference Basis

Super Agents and Confounders 发现，相同架构在多次训练后，Shapley Agent 归因和决策机制的一致性较低。

这说明模型未必能够在不同训练运行中稳定捕获相同的 Agent 交互关系。

#### Limitation Addressed

Super Agents and Confounders 主要比较：

- 不同随机种子认为哪些 Agent 重要；
- 不同训练运行的 Shapley 归因是否一致。

但是，它没有系统评价：

- 相同干预下响应方向是否一致；
- 响应强度是否一致；
- 响应阈值是否一致；
- 整条剂量—响应曲线是否一致。

#### Important Interpretation

H6 不要求不同随机种子输出完全相同的预测轨迹。

更加合理的要求是：

- 对关键 Agent 的响应方向基本一致；
- 对干预强度的排序基本一致；
- 不同种子不应频繁出现严重相反的交互判断；
- 不应出现一个 seed 预测减速，而另一个 seed 预测加速的极端冲突。

H6 需要重新训练模型，因此计算成本明显高于 H1–H4。

在 Benchmark v0 中，H6 属于次级研究假设。

---

### 2.10 Hypothesis Priority

第一阶段的核心研究主线为：

```math
\boxed{H1 + H2 + H3 + H4}
```

四项核心假设分别回答：

| Hypothesis | Main Question |
|---|---|
| **H1** | 模型是否知道应该对谁敏感？ |
| **H2** | 模型是否知道应该怎样响应？ |
| **H3** | 模型响应是否会随着风险强度有序变化？ |
| **H4** | 传统预测精度是否能够代表上述能力？ |

H5 和 H6 属于增强分析：

| Hypothesis | Additional Value |
|---|---|
| **H5** | 检查模型解释是否忠实对应实际行为 |
| **H6** | 检查模型学到的交互规律是否具有可重复性 |

因此，最小可行版本不需要等待 H5 和 H6 全部完成后才开始实验。

第一轮实验应优先验证：

> **关键 Agent 与无关 Agent 的匹配干预，能否在一个模型、一个交互类型和少量场景中产生可测量的选择性、方向性和单调性差异。**

## 3. Two-Layer Experimental System

### 3.1 Why a Two-Layer Experimental System Is Necessary

真实轨迹数据通常只提供实际发生过的历史轨迹与未来轨迹，但不会提供完整的反事实答案。

例如，Argoverse 2 不会直接告诉我们：

- 如果某辆前车不存在，目标车辆原本会怎样运动；
- 如果前车的制动强度增加，目标车辆应该减速多少；
- 如果切入车辆更早进入目标车道，目标车辆应该怎样调整；
- 哪个周围 Agent 是真实因果 Agent；
- 多个 Agent 同时变化时是否存在联合影响；
- 模型的响应幅度究竟是合理、过弱还是过强。

因此，仅依赖真实数据，很难获得完整的 causal ground truth 和 counterfactual ground truth。

另一方面，完全人工构造的场景虽然因果关系明确，但也可能存在以下问题：

- 交通行为过于简单；
- 轨迹分布与真实数据差异较大；
- 被测模型可能把场景视为 out-of-distribution 输入；
- 简化规则未必能够代表真实驾驶行为；
- 在合成数据上得到的结论未必能迁移到真实交通场景。

为同时解决这两类问题，本项目采用两个互补的实验层次：

```math
\text{Controlled Layer}
+
\text{Real-World Layer}
```

其中：

- Controlled Layer 用于验证评测方法本身是否有效；
- Real-World Layer 用于验证评测发现是否具有现实意义。

---

### 3.2 Overview of the Two Experimental Layers

| Layer | Main Purpose | Scenario Source | Main Reference Basis | Main Limitation Addressed |
|---|---|---|---|---|
| **Layer A: Controlled or Semi-Synthetic Benchmark** | 在因果结构和干预含义已知的条件下，验证指标是否能正确识别合理与不合理响应 | 基于真实或真实风格地图构造跟驰、切入、交叉和合流场景 | CADET 的 SpurGen；CounterScene 的时空冲突干预 | 真实数据缺少完整反事实 GT，无法精确判断方向、幅度和单调性 |
| **Layer B: Real AV2 High-Confidence Scenarios** | 检查受控实验中发现的问题是否也存在于真实复杂交通场景 | 从 Argoverse 2 中筛选高置信交互场景 | CausalAgents 的真实场景标注思路；CaDeT 的真实数据实验；CADET 的真实规划器审计 | 合成场景可能过于简单或偏离真实数据分布 |

两层实验的基本分工为：

```math
\text{Layer A validates the ruler}
```

```math
\text{Layer B validates practical relevance}
```

即：

> Layer A 用来验证“尺子是否能够正确测量”。

> Layer B 用来验证“这把尺子在真实场景中是否能发现有意义的问题”。

---

### 3.3 Layer A: Controlled or Semi-Synthetic Benchmark

#### 3.3.1 Main Purpose

Layer A 的核心目标不是模拟所有真实交通复杂性，而是建立一组因果结构明确、干预变量明确、预期响应方向明确的测试场景。

在这一层中，我们应当明确知道：

- 哪个 Agent 是关键 Agent；
- 哪个 Agent 是无关 Agent；
- 被修改的交通变量是什么；
- 干预强度是多少；
- 合理响应方向是什么；
- 在条件允许时，合理响应幅度大致是多少。

Layer A 主要用于评价：

- H1：Causal Selectivity；
- H2：Directional Correctness；
- H3：Dose–Response Monotonicity；
- Response Magnitude Calibration。

---

#### 3.3.2 Reference Basis

CADET 使用 SpurGen 构造具有已知身份的 causal agents、spurious agents 和 benign agents，从而可以直接计算检测精度、召回率和因果响应指标。

这一设计说明：

> 当真实数据缺少完整因果标签时，可以先在因果结构已知的受控环境中验证评测框架。

CounterScene 则通过修改 Agent 的空间安全裕度和到达冲突区域的时间关系，生成具有明确交通语义的反事实场景。

这一设计说明：

> 反事实干预不应只是随机移动轨迹，而应沿具体交通冲突机制变化。

本项目将两类思路结合：

- 借鉴 SpurGen 的明确因果结构；
- 借鉴 CounterScene 的交通语义干预；
- 将其适配到 motion forecasting，而不是端到端规划。

---

#### 3.3.3 Controlled Scenario Types

Benchmark v0 建议首先采用以下三类核心交互场景：

1. Car-Following Braking；
2. Lateral Cut-In；
3. Intersection or Merge Conflict。

后续版本可以再加入：

- Pedestrian Crossing；
- Lane Change Negotiation；
- Multi-Agent Bottleneck；
- Indirect Interaction Chain。

---

#### 3.3.4 Car-Following Braking

关键 Agent 为目标车辆前方的同车道车辆。

干预变量可以包括：

- 前车历史速度；
- 前车历史加速度；
- 前车与目标车辆的距离变化率；
- 前车制动开始时间。

风险增强型干预可以表示为：

```math
a_c^{(\alpha)}(t)
=
a_c(t)
-
\alpha \Delta a(t)
```

其中：

- $a_c(t)$ 为关键 Agent 的原始历史加速度；
- $\Delta a(t)$ 为附加制动趋势；
- $\alpha$ 为干预强度；
- $\alpha$ 越大，表示前车制动越强。

合理的目标预测响应包括：

- 预测速度下降；
- 预测纵向进度减小；
- 预测跟车间距增大；
- 更保守轨迹模态的概率提高。

---

#### 3.3.5 Lateral Cut-In

关键 Agent 为目标车辆侧前方或侧方、具有进入目标车道趋势的车辆。

干预变量可以包括：

- 横向位置；
- 横向速度；
- 朝向角；
- 到达目标车道边界的预计时间；
- 切入开始时间。

风险增强型干预可以表示为：

```math
y_c^{(\alpha)}(t)
=
y_c(t)
+
\alpha \Delta y(t)
```

其中，$\Delta y(t)$ 应使关键 Agent 的历史轨迹呈现更明显的切入趋势。

合理的目标预测响应包括：

- 预测减速；
- 预测横向避让；
- 换道模态概率提高；
- 保持安全间隔的轨迹模态概率提高。

---

#### 3.3.6 Intersection or Merge Conflict

关键 Agent 与目标车辆的潜在路径在交叉区域、合流区域或冲突区域相交。

干预变量可以包括：

- 到达冲突点的预计时间；
- 进入冲突区域的速度；
- 与目标车辆的到达时间差；
- 进入主路或目标车道的趋势。

设原始到达冲突点的时间差为：

```math
\Delta T_{\mathrm{conflict}}
```

风险增强型干预可以表示为：

```math
\Delta T_{\mathrm{conflict}}^{(\alpha)}
=
(1-\alpha)
\Delta T_{\mathrm{conflict}}
```

随着 $\alpha$ 增大，双方到达冲突点的时间差逐渐缩小。

合理的目标预测响应包括：

- 推迟进入冲突区域；
- 让行概率提高；
- 预测速度下降；
- 横向避让或调整通行顺序；
- 冲突暴露程度降低。

---

### 3.4 Layer B: Real AV2 High-Confidence Scenarios

#### 3.4.1 Main Purpose

Layer B 从 Argoverse 2 中筛选真实复杂交通场景，并对这些场景中的关键 Agent 进行受控历史轨迹干预。

这一层不负责提供完整的反事实真值，而是评价：

- 模型是否对高置信交互 Agent 产生合理响应；
- 模型是否对匹配无关 Agent 保持相对稳定；
- 响应是否随干预强度总体单调；
- 不同模型之间是否存在稳定差异；
- 传统 ADE/FDE 排名是否与因果响应排名一致。

---

#### 3.4.2 Reference Basis

CausalAgents 说明，因果 Agent 评测不能只停留在人工合成场景，还需要回到真实运动预测数据。

CaDeT 和 Beyond Patterns 等工作也在 AV2、WOMD 等真实数据集上检验其鲁棒性或因果建模效果。

CADET 则同时使用 SpurGen 和真实 nuScenes 场景，说明受控验证和真实审计应当互补。

---

#### 3.4.3 High-Confidence Scenario Selection

真实 AV2 场景不能全部被视为具有明确的因果交互关系。

建议优先筛选：

- 明确跟驰场景；
- 明确切入场景；
- 明确交叉冲突场景；
- 明确合流与让行场景。

关键 Agent 的候选筛选条件可以包括：

- 与目标 Agent 位于同一车道或拓扑相连车道；
- 运动方向存在汇聚趋势；
- 预计路径在未来时间范围内存在冲突；
- 到达潜在冲突点的时间差较小；
- 历史轨迹中已经出现减速、切入、跟随或让行迹象；
- Agent 具有完整且连续的历史观测；
- 地图匹配结果可靠。

无关 Agent 的候选条件可以包括：

- 运动方向与目标 Agent 明显发散；
- 不在目标 Agent 的可达冲突区域内；
- 不会在预测时域内进入目标路径；
- 与目标 Agent 具有足够大的时间或空间隔离；
- 具有完整历史轨迹，能够进行匹配干预。

---

#### 3.4.4 Manual Verification

自动筛选完成后，Benchmark v0 应对初始样本进行可视化人工复核。

人工复核主要确认：

- 关键 Agent 是否真的具有明确交互可能；
- 无关 Agent 是否确实不会在预测时域内产生明显作用；
- 地图拓扑是否识别正确；
- 原始轨迹是否存在缺帧或跳点；
- 干预后轨迹是否仍然符合物理和地图约束。

真实层中的 Agent 标签应表述为：

> high-confidence interaction-relevant agents

而不是：

> ground-truth causal agents

---

### 3.5 Claims and Limitations of Each Layer

| Experimental Layer | Claims We Can Make | Claims We Should Avoid |
|---|---|---|
| **Layer A** | 在已知因果结构和参考响应下，模型是否具有正确的选择性、方向性、单调性和幅度 | 模型已经在所有真实交通场景中理解真实因果关系 |
| **Layer B** | 在高置信真实交互场景中，模型是否表现出合理的相对响应 | 我们获得了真实世界完整反事实 GT；我们精确知道目标车辆应该变化多少 |
| **Combined Evidence** | 某类模型失败模式既能在受控场景中验证，也能在真实场景中观察到 | Benchmark 已经完整恢复真实交通因果图 |

---

### 3.6 Recommended Scope for Benchmark v0

第一版建议控制在：

- 3 类核心交互场景；
- 每类 100 至 300 个高质量场景；
- 每个场景 1 个关键 Agent；
- 每个场景 1 个匹配无关 Agent；
- 5 个干预等级；
- 2 至 3 个代表性预测模型。

第一阶段优先验证：

```math
H1 + H2 + H3 + H4
```

暂时不要求完成：

- 大规模多 Agent 联合干预；
- 完整间接因果链；
- 闭环世界模型；
- 完整 CounterScene 式场景生成；
- 所有主流轨迹预测模型；
- 复杂的 test-time 修复方法。

---

## 4. Matched Intervention Protocol

### 4.1 General Intervention Form

对场景 $s$ 中的 Agent $j$ 施加强度为 $\alpha$ 的干预：

```math
X_{s,j}^{(\alpha)}
=
X_s
+
\alpha \Delta X_{s,j}
```

其中：

- $X_s$ 为原始场景输入；
- $\Delta X_{s,j}$ 为对 Agent $j$ 的结构化修改；
- $\alpha$ 为干预强度；
- $\alpha=0$ 表示原始场景；
- $\alpha=1$ 表示预先设定的最大可行干预。

Benchmark v0 建议使用：

```math
\alpha
\in
\{0,\ 0.25,\ 0.5,\ 0.75,\ 1.0\}
```

对应：

| Intervention Level | Meaning |
|---|---|
| $\alpha=0$ | 原始场景 |
| $\alpha=0.25$ | 弱干预 |
| $\alpha=0.5$ | 中等干预 |
| $\alpha=0.75$ | 较强干预 |
| $\alpha=1.0$ | 最大物理可行干预 |

---

### 4.2 Intervention Must Modify Observable History

轨迹预测模型只能看到历史轨迹，因此干预必须作用在模型可见的观测历史上。

设历史观测时间范围为：

```math
t
\in
[-L,0]
```

干预可以修改最后若干历史帧中的：

- 位置；
- 速度；
- 加速度；
- 横向速度；
- 朝向角；
- 到达冲突区域的趋势。

不应直接修改模型不可见的未来 ground truth，并将该修改称为模型输入干预。

---

### 4.3 Smooth Temporal Intervention

干预不能只修改最后一个历史时间点，否则可能产生明显轨迹跳变。

建议使用平滑权重函数：

```math
w(t)
\in
[0,1]
```

使干预在历史末端逐渐增强：

```math
X_{s,j}^{(\alpha)}(t)
=
X_{s,j}(t)
+
\alpha w(t)\Delta X_{s,j}(t)
```

其中：

- 历史较早时间点的 $w(t)$ 较小；
- 越接近当前时刻，$w(t)$ 越大；
- 修改后的速度、加速度和位置应保持连续。

---

### 4.4 Why Matched Interventions Are Necessary

H1 要比较模型对关键 Agent 和无关 Agent 的敏感性。

如果两组干预的大小、类型或位置完全不同，那么结果可能被干预本身的差异混淆。

例如：

```text
关键 Agent：
位置改变 5 米

无关 Agent：
位置改变 0.2 米
```

模型对关键 Agent 的响应更大，可能只是因为其输入变化更大。

因此，本项目要求：

```math
\left\|
X_{s,c}^{(\alpha)}
-
X_s
\right\|
\approx
\left\|
X_{s,n}^{(\alpha)}
-
X_s
\right\|
```

其中：

- $c$ 为关键 Agent；
- $n$ 为匹配无关 Agent。

---

### 4.5 Matching Variables

关键 Agent 与无关 Agent 应尽量在以下方面匹配：

- Agent 类型；
- 原始速度区间；
- 与目标 Agent 的距离区间；
- 历史轨迹长度；
- 轨迹观测完整性；
- 干预变量类型；
- 干预时间窗口；
- 位置变化范数；
- 速度变化范数；
- 加速度变化范数；
- 地图区域类型；
- 道路类型；
- 可见性或检测质量。

优先采用：

> 同一场景内的 matched irrelevant agent。

同场景匹配可以最大程度控制：

- 地图；
- 天气；
- 时间；
- 交通密度；
- 目标 Agent 状态；
- 模型输入上下文。

如果同一场景中不存在合适的无关 Agent，可以在相似场景中进行 nearest-neighbor matching。

---

### 4.6 Paired Intervention Set

对每个场景构造以下输入集合：

```math
\mathcal{I}_s
=
\left\{
X_s,\
X_{s,c}^{(\alpha)},\
X_{s,n}^{(\alpha)}
\right\}
```

其中：

- $X_s$ 为原始场景；
- $X_{s,c}^{(\alpha)}$ 为关键 Agent 干预；
- $X_{s,n}^{(\alpha)}$ 为匹配无关 Agent 干预。

两种干预应使用：

- 相同的 $\alpha$；
- 相同的变量类型；
- 相同或近似的修改范数；
- 相同的历史修改窗口；
- 相同的平滑方式。

示例：

```text
Original:
- Lead vehicle speed unchanged
- Irrelevant side vehicle speed unchanged

Relevant-agent intervention:
- Lead vehicle historical speed reduced by 2 m/s

Matched irrelevant-agent intervention:
- Irrelevant side vehicle historical speed reduced by 2 m/s
```

---

### 4.7 Intervention Families

| Intervention Family | Modified Variable | Typical Scenario | Purpose |
|---|---|---|---|
| **Acceleration Intervention** | 历史速度或加速度趋势 | 跟驰 | 测试模型对前车制动的响应 |
| **Lateral-Motion Intervention** | 横向位置、速度或朝向趋势 | 切入、合流 | 测试模型对切入趋势的响应 |
| **Conflict-Time Intervention** | 到达冲突点的预计时间差 | 交叉、合流 | 测试模型对冲突时间压缩的响应 |
| **Removal Intervention** | 删除或遮蔽 Agent 历史 | 通用 | 与 CausalAgents、CADET 直接对照 |
| **Trajectory Replacement** | 使用另一条物理可行历史替换原历史 | 强反事实测试 | 测试更明显的行为变化 |
| **Placebo Intervention** | 修改理论上不应影响目标的无关 Agent | 通用 | 检查模型的虚假敏感性 |

Removal Intervention 应作为基线保留，但不应成为唯一干预方式。

原因包括：

- 删除是离散干预；
- 删除可能产生明显分布偏移；
- 删除无法构造剂量—响应曲线；
- 删除不能区分模型是反应不足还是反应过度。

---

### 4.8 Physical Validity Constraints

所有干预后的 Agent 历史轨迹都应满足基本物理约束。

速度约束：

```math
v_{\min}
\leq
v_j^{(\alpha)}(t)
\leq
v_{\max}
```

加速度约束：

```math
a_{\min}
\leq
a_j^{(\alpha)}(t)
\leq
a_{\max}
```

加加速度约束：

```math
\left|
a_j^{(\alpha)}(t+1)
-
a_j^{(\alpha)}(t)
\right|
\leq
J_{\max}
```

同时要求：

- 轨迹位置连续；
- 速度变化平滑；
- 朝向与运动方向基本一致；
- 车辆不能无理由穿越不可行驶区域；
- Agent 身份和类别保持不变；
- 不产生明显瞬移；
- 不产生与地图拓扑冲突的历史；
- 修改后轨迹不能明显脱离数据集合理分布。

---

### 4.9 Map Validity Constraints

车辆轨迹应尽量保持在：

- 可行驶区域；
- 合理车道附近；
- 与车辆朝向一致的道路方向；
- 可解释的切入或合流路径中。

对于切入干预，可以允许 Agent 向相邻车道边界靠近，但不能使其直接穿越：

- 道路隔离带；
- 不可行驶区域；
- 方向相反的道路；
- 建筑物或地图外区域。

---

### 4.10 Distribution-Shift Checks

为了避免指标只是在测模型对异常输入的敏感性，应检查干预是否造成明显 OOD。

可以比较干预前后的：

- 速度分布；
- 加速度分布；
- 横向速度分布；
- 轨迹曲率；
- 与车道中心线的距离；
- 历史轨迹平滑度；
- 最近邻距离。

如果干预样本明显偏离原始数据分布，应：

- 降低最大干预强度；
- 删除该样本；
- 或将其单独标记为 stress-test 场景。

---

### 4.11 Number of Model Evaluations

设每个场景具有：

- 1 次原始预测；
- $K$ 个关键 Agent 干预等级；
- $K$ 个无关 Agent 干预等级。

则每个场景所需模型调用次数为：

```math
N_{\mathrm{calls\ per\ scene}}
=
1
+
2K
```

当非零干预等级数量为 $K=4$ 时：

```math
N_{\mathrm{calls\ per\ scene}}
=
1
+
2\times4
=
9
```

即：

- 1 次原始预测；
- 4 次关键 Agent 干预预测；
- 4 次无关 Agent 干预预测。

---

### 4.12 Intervention Protocol for Benchmark v0

Benchmark v0 建议首先固定：

- 单 Agent 干预；
- 5 个干预等级；
- 关键 Agent 与无关 Agent 成对匹配；
- 修改最后若干历史帧；
- 保证动力学与地图可行性；
- 删除干预作为基线；
- 连续历史轨迹干预作为主要实验。

多 Agent 联合干预暂时作为后续扩展：

```math
X_{s,\{i,j\}}^{(\alpha,\beta)}
```

用于研究：

- 联合效应；
- 协同效应；
- 抵消效应；
- 间接因果链。

---

## 5. Evaluation Metric System

### 5.1 Metric Design Principle

本项目不能只评价模型是否稳定。

一个完全忽略所有周围 Agent 的模型可能具有很高的无关干预稳定性，但它也不会对真正危险的 Agent 作出响应。

因此，指标必须同时评价：

```math
\text{Correct invariance to irrelevant agents}
```

以及：

```math
\text{Correct response to relevant agents}
```

完整指标体系需要回答六个不同问题：

1. 模型是否对正确的 Agent 更敏感？
2. 模型的响应方向是否正确？
3. 干预越强时，响应是否总体越强？
4. 响应幅度是否合理？
5. 模型归因是否忠实对应其实际行为？
6. 不同随机种子是否学到稳定的响应规律？

---

### 5.2 Metric Overview

| Metric | Abbreviation | Main Question | Corresponding Hypothesis |
|---|---|---|---|
| Causal Selectivity Gap | CSG / nCSG | 模型是否对关键 Agent 比对无关 Agent 更敏感？ | H1 |
| Directional Response Accuracy | DRA | 模型是否朝合理方向响应？ | H2 |
| Monotonic Response Score | MRS | 响应是否随干预强度总体单调？ | H3 |
| Response Magnitude Calibration | RMC | 模型是否反应不足或反应过度？ | H2、H3 |
| Attribution–Response Consistency | ARC | 模型归因是否忠实对应实际干预响应？ | H5 |
| Cross-Seed Response Stability | CSRS | 不同随机种子的响应规律是否稳定？ | H6 |

此外，传统指标仍然保留：

- minADE；
- minFDE；
- MR；
- brier-minFDE 或 mAP，在模型支持时。

---

### 5.3 Prediction Response Magnitude

在前文中，模型对干预的总体响应大小定义为：

```math
R_m(s,j,\alpha)
=
D\left(
\hat{Y}_m^{(s,j,\alpha)},
\hat{Y}_m^{(s,0)}
\right)
```

其中，$D$ 可以根据模型输出形式选择。

对于单轨迹输出，可以使用：

```math
D_{\mathrm{traj}}
=
\frac{1}{T}
\sum_{t=1}^{T}
\left\|
\hat{y}^{(\alpha)}_t
-
\hat{y}^{(0)}_t
\right\|_2
```

对于多模态输出，$D$ 应同时考虑：

- 各模态轨迹变化；
- 模态概率变化；
- 最可能模态变化；
- 预测分布整体变化。

需要强调：

> $R_m$ 只衡量模型改变了多少，不判断改变是否正确。

---

### 5.4 Metric 1: Causal Selectivity Gap

关键 Agent 干预响应为：

```math
R_{m,c}(s,\alpha)
=
R_m(s,c,\alpha)
```

匹配无关 Agent 干预响应为：

```math
R_{m,n}(s,\alpha)
=
R_m(s,n,\alpha)
```

Causal Selectivity Gap 定义为：

```math
\mathrm{CSG}_m
=
\mathbb{E}_{s,\alpha>0}
\left[
R_{m,c}(s,\alpha)
-
R_{m,n}(s,\alpha)
\right]
```

由于不同模型的输出尺度可能不同，建议同时报告归一化版本：

```math
\mathrm{nCSG}_m
=
\mathbb{E}_{s,\alpha>0}
\left[
\frac{
R_{m,c}(s,\alpha)
-
R_{m,n}(s,\alpha)
}{
R_{m,c}(s,\alpha)
+
R_{m,n}(s,\alpha)
+
\epsilon
}
\right]
```

解释：

- $\mathrm{nCSG}>0$：模型对关键 Agent 更敏感；
- $\mathrm{nCSG}\approx0$：模型对两类 Agent 的响应接近；
- $\mathrm{nCSG}<0$：模型对无关 Agent 反而更加敏感。

CSG 和 nCSG 只能评价选择性，不能评价方向是否正确。

因此，它们必须与 DRA 联合报告。

---

### 5.5 Metric 2: Directional Response Accuracy

前文定义了场景相关的语义响应量：

```math
z_m(s,j,\alpha)
```

统一规定：

> $z_m$ 越大，表示模型表现出的合理安全响应越强。

对于风险增强型关键 Agent 干预，方向正确条件为：

```math
z_m(s,c,\alpha)
-
z_m(s,c,0)
>
\epsilon_{\mathrm{dir}}
```

定义方向正确指示量：

```math
C_{\mathrm{dir}}(s,\alpha)
=
\mathbf{1}
\left[
z_m(s,c,\alpha)
-
z_m(s,c,0)
>
\epsilon_{\mathrm{dir}}
\right]
```

Directional Response Accuracy 定义为：

```math
\mathrm{DRA}_m
=
\frac{1}{N}
\sum_{s,\alpha>0}
C_{\mathrm{dir}}(s,\alpha)
```

解释：

- DRA 高：模型大多数情况下朝合理方向响应；
- DRA 低：模型经常无响应或朝错误方向响应。

---

### 5.6 Scenario-Specific Semantic Response

不同场景不能使用完全相同的 $z_m$。

#### Car-Following

可以使用：

- 预测纵向进度减少量；
- 预测速度降低量；
- 跟车间距增加量。

例如：

```math
z_m^{\mathrm{follow}}
=
d_{\mathrm{gap}}^{(\alpha)}
-
d_{\mathrm{gap}}^{(0)}
```

#### Intersection Conflict

可以使用目标 Agent 进入冲突区域的延迟：

```math
z_m^{\mathrm{intersection}}
=
T_{\mathrm{enter}}^{(\alpha)}
-
T_{\mathrm{enter}}^{(0)}
```

#### Lateral Cut-In

可以使用：

- 减速响应；
- 横向避让响应；
- 安全换道模态概率变化。

对于允许多种安全策略的场景，可以构造复合响应：

```math
z_m
=
w_1 z_{\mathrm{brake}}
+
w_2 z_{\mathrm{avoid}}
+
w_3 z_{\mathrm{mode}}
```

其中，权重应在正式实验前固定，不能根据结果临时调整。

---

### 5.7 Metric 3: Monotonic Response Score

设干预强度为：

```math
0
=
\alpha_0
<
\alpha_1
<
\cdots
<
\alpha_K
```

对于任意相邻强度，如果更强干预没有导致明显更弱的安全响应，则视为满足局部单调性：

```math
z_m(s,c,\alpha_{k+1})
\geq
z_m(s,c,\alpha_k)
-
\epsilon_{\mathrm{mono}}
```

定义局部单调指示量：

```math
C_{\mathrm{mono}}(s,k)
=
\mathbf{1}
\left[
z_m(s,c,\alpha_{k+1})
\geq
z_m(s,c,\alpha_k)
-
\epsilon_{\mathrm{mono}}
\right]
```

Monotonic Response Score 定义为：

```math
\mathrm{MRS}_m
=
\frac{1}{NK}
\sum_s
\sum_{k=0}^{K-1}
C_{\mathrm{mono}}(s,k)
```

解释：

- MRS 接近 1：响应曲线大部分保持有序；
- MRS 较低：模型经常出现反向变化或无规律跳变。

同时建议报告每个场景中 $\alpha$ 与 $z_m$ 的 Spearman correlation。

由于 GitHub 数学公式不需要写出函数宏，记为：

```math
\rho_{\mathrm{dose}}(s)
=
\rho_S
\left(
\{\alpha_k\},
\{z_m(s,c,\alpha_k)\}
\right)
```

其中，$\rho_S$ 表示 Spearman rank correlation。

---

### 5.8 Metric 4: Response Magnitude Calibration

DRA 只能判断方向正确与否。

例如，两种模型都对前车制动作出减速响应：

```text
Model A:
减速 0.2 m/s

Model B:
减速 8.0 m/s
```

两者方向都正确，但一个可能反应不足，另一个可能反应过度。

因此，在具有可信参考响应的 Layer A 中，需要评价响应幅度。

设参考响应曲线为：

```math
z_{\mathrm{ref}}(s,\alpha)
```

模型响应曲线为：

```math
z_m(s,\alpha)
```

定义归一化曲线误差：

```math
E_{\mathrm{curve}}
=
\frac{
\sum_{k=1}^{K}
w_k
\left|
z_m(s,\alpha_k)
-
z_{\mathrm{ref}}(s,\alpha_k)
\right|
}{
\sum_{k=1}^{K}
w_k
\left(
z_{\mathrm{ref,max}}
-
z_{\mathrm{ref,min}}
\right)
+
\epsilon
}
```

Response Magnitude Calibration 定义为：

```math
\mathrm{RMC}_m
=
\max
\left(
0,\
1-E_{\mathrm{curve}}
\right)
```

解释：

- RMC 高：模型响应幅度接近参考响应；
- DRA 高但 RMC 低：方向正确，但反应不足或过度；
- MRS 高但 RMC 低：响应有序，但幅度不合理。

RMC 主要用于 Layer A。

在真实 AV2 层中，由于没有完整反事实 GT，不应声称获得绝对幅度校准。

真实层可以报告：

- 响应曲线斜率；
- 响应曲线面积；
- 极端反应比例；
- 迟钝响应比例；
- 不同模型之间的相对响应幅度。

---

### 5.9 Metric 5: Attribution–Response Consistency

设模型对 Agent $j$ 的归因分数为：

```math
A_{m,j}
```

归因方法可以包括：

- Shapley value；
- Agent occlusion；
- query ablation；
- gradient-based attribution；
- 原生 Attention，作为辅助分析。

Agent $j$ 的平均外部干预响应定义为：

```math
\bar{R}_{m,j}
=
\frac{1}{K}
\sum_{k=1}^{K}
R_m(s,j,\alpha_k)
```

Attribution–Response Consistency 定义为：

```math
\mathrm{ARC}_m
=
\rho_S
\left(
\{A_{m,j}\},
\{\bar{R}_{m,j}\}
\right)
```

其中，$\rho_S$ 表示 Spearman rank correlation。

同时报告 Top-$k$ 重合率：

```math
\mathrm{TopKOverlap}
=
\frac{
\left|
\mathrm{TopK}(A)
\cap
\mathrm{TopK}(\bar{R})
\right|
}{
k
}
```

解释：

- ARC 高：归因结果较忠实地反映模型实际依赖；
- ARC 低：模型解释与真实功能响应不一致。

需要注意：

> ARC 评价的是解释忠实度，而不是交通因果正确性。

一个模型可能稳定依赖错误 Agent，且归因方法准确识别了这种错误依赖。

因此，ARC 必须与 nCSG 和 DRA 联合分析。

---

### 5.10 Metric 6: Cross-Seed Response Stability

设同一架构训练得到 $S$ 个随机种子版本：

```math
m^{(1)},\
m^{(2)},\
\ldots,\
m^{(S)}
```

每个 seed 在同一场景中产生一条响应曲线：

```math
\mathbf{z}_{m^{(u)}}
=
\left[
z_{m^{(u)}}(\alpha_0),
z_{m^{(u)}}(\alpha_1),
\ldots,
z_{m^{(u)}}(\alpha_K)
\right]
```

Cross-Seed Response Stability 可以定义为所有 seed 对之间响应曲线相关性的平均：

```math
\mathrm{CSRS}
=
\frac{2}{S(S-1)}
\sum_{u<v}
\rho_S
\left(
\mathbf{z}_{m^{(u)}},
\mathbf{z}_{m^{(v)}}
\right)
```

同时报告方向一致率。

设：

```math
\Delta z_{m^{(u)}}(s,\alpha)
=
z_{m^{(u)}}(s,c,\alpha)
-
z_{m^{(u)}}(s,c,0)
```

若所有 seed 的响应方向一致，则：

```math
\mathrm{sign}
\left(
\Delta z_{m^{(1)}}
\right)
=
\cdots
=
\mathrm{sign}
\left(
\Delta z_{m^{(S)}}
\right)
```

方向一致率可以统计为：

```math
\mathrm{SeedDirectionAgreement}
=
\frac{
\text{方向一致的场景与干预数量}
}{
\text{全部场景与干预数量}
}
```

还应报告每个干预强度下的跨 seed 标准差：

```math
\sigma_{\mathrm{seed}}(\alpha)
=
\mathrm{Std}
\left(
z_{m^{(1)}}(\alpha),
\ldots,
z_{m^{(S)}}(\alpha)
\right)
```

解释：

- CSRS 高：不同训练运行学到相似的响应结构；
- CSRS 低：模型的交互机制对随机初始化敏感；
- 方向一致率低：不同 seed 可能产生相反交通判断。

---

### 5.11 Traditional Accuracy Metrics

传统预测指标仍然需要报告：

- minADE；
- minFDE；
- MR；
- mAP 或 brier-minFDE，在模型和数据集支持时。

这些指标用于回答 H4：

> 传统预测精度与因果响应能力是否一致？

分别建立：

- Accuracy Ranking；
- Causal Response Ranking。

两套排名之间的 Spearman rank correlation 记为：

```math
\rho_{\mathrm{rank}}
=
\rho_S
\left(
\mathrm{Rank}_{\mathrm{accuracy}},
\mathrm{Rank}_{\mathrm{causal}}
\right)
```

需要分析：

- 排名相关性；
- 模型排名翻转；
- 高精度但低因果响应模型；
- 精度略低但因果响应更稳定的模型。

---

### 5.12 Do Not Use a Single Composite Score Initially

Benchmark v0 不建议一开始就把六项指标压缩成一个固定加权总分。

例如，不建议直接定义：

```math
S_{\mathrm{total}}
=
w_1\mathrm{nCSG}
+
w_2\mathrm{DRA}
+
w_3\mathrm{MRS}
+
w_4\mathrm{RMC}
+
w_5\mathrm{ARC}
+
w_6\mathrm{CSRS}
```

原因是：

- 权重选择具有主观性；
- 高稳定性可能掩盖低因果响应；
- 高方向正确率可能掩盖严重过度反应；
- 平均分可能掩盖少数严重失败场景；
- 不同指标对应不同性质的模型能力。

第一版应优先报告完整的 causal response profile。

---

### 5.13 Recommended Result Table

| Model | minADE | minFDE | MR | nCSG | DRA | MRS | RMC | ARC | CSRS |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Model A |  |  |  |  |  |  |  |  |  |
| Model B |  |  |  |  |  |  |  |  |  |
| Model C |  |  |  |  |  |  |  |  |  |

同时应分别报告：

- Controlled Layer 结果；
- Real AV2 Layer 结果；
- 不同交互类型结果；
- 不同干预强度结果；
- 关键 Agent 与无关 Agent 响应曲线；
- 典型成功案例；
- 典型失败案例。

---

### 5.14 Minimal Metric Set for Benchmark v0

第一版最小可行指标为：

```math
\mathrm{nCSG}
+
\mathrm{DRA}
+
\mathrm{MRS}
```

并保留：

```math
\mathrm{minADE}
+
\mathrm{minFDE}
+
\mathrm{MR}
```

因此，Benchmark v0 首先回答：

1. 模型是否对关键 Agent 比对无关 Agent 更敏感？
2. 模型是否朝合理方向响应？
3. 模型响应是否随干预强度总体单调？
4. 传统精度排名与因果响应排名是否一致？

RMC 首先只在 Controlled Layer 实现。

ARC 和 CSRS 在主流程稳定后加入。

---

### 5.15 Minimal Pilot Experiment

第一轮 pilot experiment 可以只包含：

1. 选择 10 至 20 个明确跟驰场景；
2. 选择一个已经能够正常运行的预测模型；
3. 确定每个场景的前车关键 Agent；
4. 选择一个匹配无关 Agent；
5. 对前车施加 5 级制动干预；
6. 对无关 Agent 施加相同幅度的速度干预；
7. 绘制两组响应曲线；
8. 计算 nCSG、DRA 和 MRS；
9. 检查干预历史是否物理可行；
10. 检查模型响应是否具有可解释差异。

这一 pilot 的目标不是立即获得论文结果，而是验证：

> **关键 Agent 与无关 Agent 的匹配连续干预，是否能够产生稳定、可测量、可比较的模型响应。**
