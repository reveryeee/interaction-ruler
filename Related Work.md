## 1. Tolstaya et al., "Identifying Driver Interactions via Conditional Behavior Prediction," ICRA 2021 (Waymo) 

## 2. Saadatnejad et al., "Are socially-aware trajectory prediction models really socially-aware?" Transportation Research Part C, 2022

## 3.  "Surprise Potential as a Measure of Interactivity in Driving Scenarios," 2025 — arXiv 2502.05677

## 4. "Super Agents and Confounders: Influence of surrounding agents on vehicle trajectory prediction," 2026 — arXiv 2604.03463

## 5. Schumann & Hagenus et al., "Realistic Adversarial Attacks... via Future State Perturbation," TU Delft, 2025 — arXiv 2505.06134
Schumann 与 Hagenus 等人（2025） 在其研究中，直击了传统对抗攻击在自动驾驶等闭环控制系统中极易失效的学术痛点。他们指出，传统的对抗攻击（如 PGD）多属于单帧、开环攻击，而现实中的智能体通常采用模型预测控制（MPC）或深度强化学习（DRL）等闭环机制，其固有的“滚动优化”与“反馈纠偏”功能能够轻易稀释并消除孤立的瞬时干扰。为了攻克这一安全韧性，作者创新性地提出了未来状态扰动（Future State Perturbation, FSP）攻击框架。

该方法的核心思想是将攻击建模为时间序列上的协同优化问题。FSP 不再盲目追求单步动作偏差的最大化，而是通过时间反向传播（BPTT）算法，精准计算出连续时间步内的一组微小扰动序列。这些扰动在时序上高度相干，且严格受限于极小的物理感知边界。当闭环控制器吸入这些微扰后，其对“未来状态”的预测会产生方向一致的系统性漂移。这种漂移在控制器的滚动时间窗内通过“蝴蝶效应”不断累积，最终导致控制器在做出看似合理的局部决策时，不知不觉地越过物理安全边界。

在 CARLA 等高保真仿真环境中的实验表明，相比于碰撞率仅有约 10% 的传统单帧攻击，FSP 在相同的微小扰动预算下，能将闭环控制系统的碰撞率大幅提升至 90% 左右，并能诱发执行器产生剧烈的动作抖动与物理失控。此外，该攻击还表现出极强的黑盒可转移性与时序隐蔽性，普通的安全检测器极难将其与正常的物理噪声区分开来。该工作打破了学术界“闭环控制天然具备抗扰性”的固有认知，为后续开发基于时序一致性检验的主动安全防御机制奠定了重要的理论基石。

## 6.  Chen et al., "Human Trajectory Prediction via Counterfactual Analysis," ICCV 2021 — arXiv 2107.14202
在人类轨迹预测领域，Chen 等人（2021） 针对传统深度学习模型过度拟合场景上下文、导致虚假空间偏见（Spatial Bias）这一学术痛点展开了研究。他们指出，基于关联性概率建模的传统方法无法区分行人的真实运动意图与场景先验。为了解决这一问题，作者首次将因果推断理论引入轨迹预测任务，构建了结构因果模型（SCM），并创新性地提出了基于反事实分析的反事实网络（CF-Net）框架。该方法的核心思想在于利用反事实干预，在推理阶段剥离场景特征对预测结果的直达效应。

具体而言，CF-Net 巧妙地将总效应（Total Effect）分解为自然直达效应（Natural Direct Effect, NDE）与总间接效应（Total Indirect Effect, TIE）。在推理过程中，模型通过双分支结构并行运行两次前向传播：事实分支输入真实的轨迹与场景特征，而反事实分支则将历史轨迹输入强制置为基准状态（即静止状态），从而捕获仅凭场景特征产生的“盲目预测”偏见。最终，模型通过简单的“因果减法”将这两者相减，动态地从输出端过滤掉环境带来的虚假空间偏见，从而实现“纯净”的、无偏的轨迹因果效应预测。

在 SDD 和 ETH/UCY 等经典数据集上的实验结果表明，CF-Net 作为一种即插即用的模块，能够显著提升主流骨干网络（如 Social-GAN 和 PECNet）的预测精度。更重要的是，该方法在跨场景的分布外（OOD）泛化测试中展现出极强的鲁棒性，克服了传统模型在陌生场景下性能骤降的缺陷。该工作打破了传统黑盒关联预测的局限性，从因果解耦的角度为具有高泛化性、可解释性的人类轨迹预测研究开辟了新的技术路径。

## 7. QCNet — Zhou et al., "Query-Centric Trajectory Prediction," CVPR 2023 — arXiv/CVF (Attention在scence上。不要想"Attention对象"，而是要想“什么应该成为一个合理的state”)
History → Scene Encoder (agent encoder + lane encoder) → Query → Trajectory （State（z）应该表示："当前交通场景是什么样子（Current Scene））

现有基于图神经网络的轨迹预测方法（如 LaneGCN、HiVT、MTR）主要依赖于目标车辆（target-specific）的场景编码或图消息传递机制，在多智能体预测过程中存在场景重复编码、计算冗余以及可扩展性受限等问题。QCNet通过提出以查询（Query）为中心的场景表示（Query-centric Representation），将场景编码与目标预测过程解耦，首先构建可共享的全局场景表示，再由不同目标车辆通过Query主动检索与自身预测任务最相关的上下文信息，从而实现高效的信息交互、多目标共享计算以及更优的轨迹预测性能。然而，QCNet本质上仍属于判别式轨迹预测模型，其学习目标仍是建立当前场景到未来轨迹的直接映射，并未显式建模交通场景在潜空间中的动态演化过程（Latent Dynamics）。

## 8. HPNet — Tang et al., "Dynamic Trajectory Forecasting with Historical Prediction Attention," CVPR 2024 — arXiv 2404.06351 (Attention在history prediction上,每一段prediction的attention weight是不一样的)
History → Prediction → Prediction History → Attention → New Prediction （State（z）应该表示："当前预测演化到了什么阶段（Prediction State）"。 假如我们要走RSSM(注意RSSM自己本身是不带memory的，要像HPNet这样的话那就一定要加其他组件)或者其他world model范式，那么照猫画虎可以架构成为History → Latent state → Latent state History → Attention → New Latent state。 但其实这只是及格线，具体挖掘1. Memory里面到底存什么，2. 能把“为什么过去某一个latent应该比另一个latent更重要”才是好的）

失败case1: 突然Cut-in。模型还是预测直行。 因为：Prediction History一直告诉它直行，于是Memory反而成为惯性。
失败case2: 突然Emergency Brake。Prediction更新来不及。因为Memory里面全都是高速巡航。
失败case3: 多人交互。例如五辆车同时互相让行。Prediction History其实已经不足以表达复杂交互.


QCNet和HPNet都只追求了纯预测能力，但是不知道是不是真的理解了概念、因果关系和推理过程。 这就是world model要主攻的方向，即学习一个能够预测未来的 Latent State甚至能够把State Semantics破译出来让我们能看得懂 （可观测场景
位置、速度、朝向、周围车辆、地图 (observation) → Encoder → latent state sₜ → Dynamics → future latent state sₜ₊₁ → Trajectory Decoder → 未来轨迹。  即： 模型根据可观测交通场景推断内部潜状态，再用动力学模型把它向未来rollout，最后从未来潜状态解码出轨迹。   但前提是假如world model范式在ADE/FDE等指标上无法超过这些SOTA，那么我们要在其他部分做出贡献，比如多步rollout更稳定，对扰动和反事实更合理，少量数据下更有效，Latent可解释、可诊断等等）
<img width="396" height="555" alt="image" src="https://github.com/user-attachments/assets/0f0e6696-b697-499a-8d45-05fc237868b0" />


## 9.  Forecast-MAE — Cheng, Mei, Liu, ICCV 2023 ()


## 10. MTR

## 11. HiVT

## 12. CausalAgents：A Robustness Benchmark for Motion Forecasting using Causal Relationships
CausalAgents最重要的贡献，是把“模型是否真正忽略无关车辆”变成了一项可测量的实验。它证明了标准minADE很低的模型，仍可能因为删除无关Agent而大幅改变预测，因此“预测准确”与“预测可靠”是两件不同的事。

但这把尺子只测了因果能力的一半：对无关Agent是否保持稳定。它没有进一步测试删除关键Agent后，模型是否产生合理方向的变化，也没有衡量某个Agent影响目标车的强弱、正负方向和持续时间。标签也是AV中心的二元人工标签，基本不处理间接因果链，更不是一个自动发现Agent因果关系的方法。

它的删除实验本身也有局限。一次删除大量Agent可能让场景分布与训练数据差别很大，因此模型变化既可能来自虚假相关性，也可能部分来自普通的输入缺失敏感性。它采用的轨迹集合IoU又忽略了时间、速度和模式概率，所以无法区分“路径相同但到达时间不同”等重要情况。

因此，我们的升级方向很清楚：在“删除非因果Agent后的稳定性”之外，增加关键Agent删除、速度与位置扰动，检查预测变化的方向、强度和单调性；进一步评估间接交互链、多Agent联合一致性以及地图先验与动态交互冲突。这样就能从一把单纯测鲁棒性的尺子，扩展为一套诊断模型是否真正学到谁影响谁、影响多强、通过什么路径影响的框架。

## 13. CRiTIC：Curb Your Attention——Causal Attention Gating for Robust Trajectory Prediction
CRiTIC的出发点是，不再让轨迹预测网络自己在一个黑盒Attention里同时完成“判断谁重要”和“预测未来”两件事，而是把它拆成两个模块。第一个 Causal Discovery Network先根据过去一段时间内的多车运动和地图信息，输出一张有方向的关系图。第二个轨迹预测网络再利用这张图，通过 Causal Attention Gating压低非因果Agent的信息，让Attention主要读取被判断为关键的Agent。因此，CRiTIC相较CausalAgents向前走了一步：CausalAgents依靠人来标注谁相关，CRiTIC尝试由模型自动推断Agent之间的关系，并把关系强度直接用于控制预测网络。

CRiTIC相比CausalAgents向前走了一步：它不再依赖人工标签来决定谁重要，而是通过Causal Discovery Network自动生成一张稀疏、有方向的Agent关系图，再用这张图控制Attention的信息传递。实验表明，这种设计可以显著降低模型对非因果Agent删除的敏感度，同时基本保持原有预测精度，并改善跨域泛化。

但它还不能证明自己真正发现了物理因果。Causal Discovery Network学到的是“哪些Agent的信息有助于预测和重建”，本质上更接近Granger式预测因果；共同原因、地图信号和数据集共现关系仍可能被误判为Agent之间的直接影响。更关键的是，模型越少读取其他Agent，删除Agent后自然越稳定，因此它的鲁棒性提升有多少来自正确因果识别、有多少只是稀疏化，目前仍没有被完全分开验证。

对我们的造尺子工作而言，CRiTIC是一个非常合适的被测对象。我们不仅可以测试它删除无关Agent后是否稳定，还可以检查它输出的关系强度是否与真实干预效果一致：高分Agent被删除或减速后，目标预测是否变化更大；低分Agent被扰动后，预测是否基本不变；影响方向是否合理；间接影响是否能够通过多跳关系被识别。

所以，CausalAgents提供了“人工标签＋删除测试”，CRiTIC提供了“自动关系发现＋因果Gate”，而我们的机会是建立一套更严格的外部评价体系，判断这些模型输出的所谓因果强度，是否真的能够预测反事实干预下的变化。换句话说，CRiTIC尝试开发因果交互模型，而我们负责验证它到底有多因果。
