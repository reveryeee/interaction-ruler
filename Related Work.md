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
