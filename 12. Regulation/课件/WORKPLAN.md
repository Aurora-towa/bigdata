# 第 12 讲 PBL 工作计划：过拟合的对抗：正则化与验证策略

主题：如何让模型更稳健、更可信？

状态：已获教师审批。审批意见：本讲形成独立案例；Packet D 加入 bootstrap 置信区间；作业要求学生提交一页 validation protocol。

## 1. 学习目标

课程目标：让学生从“模型在训练/内部验证中看起来很好”这个诱惑性判断出发，学会用正则化、验证拆分、交叉验证、嵌套交叉验证、校准和外部验证边界来判断一个医学预测模型是否更稳健、更可信。

可观察学习目标：

1. 能解释正则化是在损失函数中加入复杂度惩罚，以降低过拟合风险。
2. 能比较 L1、L2 和 Elastic Net 对系数、特征数量、稳定性和解释性的不同影响。
3. 能为小样本高维医学数据设计 train/validation/test、k-fold CV、nested CV 与 locked test 的组合方案。
4. 能用均值、标准差或置信区间报告交叉验证结果，而不是只报告一个最佳分数。
5. 能同时报告 AUC、sensitivity、specificity、Brier/calibration 等证据，并说明各自的局限。
6. 能说明为什么独立外部验证比内部随机划分更接近真实部署风险，并能识别“代理外部验证”不能替代真实外部队列。

## 2. 主题模型

主题类型：model improvement + validation strategy。

关键概念：

- 过拟合与模型复杂度
- 正则化惩罚项与超参数 C/lambda
- L1 regularization 与稀疏系数
- L2 regularization 与权重收缩
- Elastic Net 与相关特征场景
- train/validation/test split 与 locked test
- k-fold cross-validation 与 nested cross-validation
- 外部验证、校准、sensitivity/specificity 与报告不确定性

常见误解：

- L1 选出的特征一定是真实生物标志物。
- 测试集分数高就说明模型可以临床使用。
- 交叉验证平均分最高的方案必然最可信。
- validation set 和 test set 可以反复调参共用。
- 外部验证只是“再随机划分一次”。

失败模式：

- 在测试集上反复选择超参数，导致测试集泄漏。
- 只报告平均 AUC，不报告波动、敏感性、特异性和校准。
- 高相关特征下把 L1 的稀疏结果过度解释为因果或机制证据。
- 小样本中随机划分偶然性大，模型排序不稳定。
- 用同源数据的 stress split 冒充真正的跨中心外部验证。

## 3. 数据集选择

候选数据集：

- 首选：sklearn Breast Cancer Wisconsin Diagnostic。569 个样本，30 个连续特征，二分类；scikit-learn 本地内置，原始来源为 UCI Breast Cancer Wisconsin Diagnostic。
- 备选：sklearn diabetes。442 个样本，10 个特征，回归任务；适合 L1/L2，但不如本讲的 sensitivity/specificity 与医学分类报告要求贴合。
- 备选：本地课程数据。当前未发现专门用于第 12 讲的本地 CSV 数据。

选定数据集：sklearn Breast Cancer Wisconsin Diagnostic。

来源与许可说明：

- scikit-learn 文档说明该数据集为 2 类、569 个样本、30 个正实数特征，来源指向 UCI ML Repository。
- UCI 页面列出 DOI `10.24432/C5DW2B`，许可证为 CC BY 4.0。
- 课程中会注明该数据集是教学基准，不是临床部署数据。

数据概要：

- 样本数：569
- 特征数：30
- 目标变量：malignant cancer = 1, benign = 0
- 类别分布：malignant 212, benign 357；阳性率约 37.3%
- 缺失值：0
- 适配理由：样本量中等、特征相关性强、分类指标可解释，能制造“L1 更稀疏但未必在敏感性/校准上最好”的 PBL 冲突。

## 4. 临时主结果图

已生成临时主图：`tmp/lecture12_regularization/planned-main-result-figure.png`

图中计划展示：

- 不同 C 值下 L1/L2/Elastic Net 的 5-fold CV AUC 均值与标准差。
- 不同 C 值下非零系数数量变化，凸显 L1 稀疏化。
- locked test 上 sensitivity、specificity、Brier score 的对照。
- L1 与 L2 的 calibration curve。

为什么重要：这张图把“性能最高”“特征最少”“校准更好”“验证更可信”拆成不同证据，让学生看到正则化不是单一答案，而是需要按医学任务权衡。

初步结果：

- L1 C=1：非零系数 14/30，test AUC 0.996，sensitivity 0.929，specificity 0.986，Brier 0.024。
- L2 C=1：非零系数 30/30，test AUC 0.995，sensitivity 0.952，specificity 0.986，Brier 0.021。
- Elastic Net C=1：非零系数 25/30，test AUC 0.997，sensitivity 0.952，specificity 0.986，Brier 0.021。
- Nested CV 外层 5 折均值：AUC 0.992 +/- 0.006，balanced accuracy 0.959 +/- 0.008，malignant sensitivity 0.935 +/- 0.025，benign specificity 0.982 +/- 0.012。

## 5. PBL 决策问题与驱动问题

学生决策问题：

一个模型团队想把乳腺细胞核特征的 logistic regression 模型写入报告。他们发现 L1 可以把特征压缩到更少，并且 AUC 很高；另一个同学认为 L2 或 Elastic Net 更稳健。你会批准哪一种建模和验证方案进入报告？还需要补充哪些证据？

驱动问题：

在乳腺肿块 FNA 细胞核特征的良恶性分类任务中，面对样本量有限、特征相关且模型可能过拟合的风险，应如何选择 L1/L2/Elastic Net 与验证策略，才能用可观察的系数、CV 波动、locked test、sensitivity/specificity、校准和外部验证边界证明模型更稳健；如果缺少这些证据，哪些结论不能成立？

## 6. Packet A-D 证据弧

Packet A：原始困境与诱惑性主张

- 释放内容：数据概要、类别分布、30 个细胞核特征、模型团队的初始说法：“L1 让模型只用 14 个特征且 AUC 很高，所以更可信。”
- 隐藏内容：CV 波动、locked test、校准、nested CV、stress/external validation。
- 学生判断：是否接受这个报告结论？缺什么证据？

Packet B：L1/L2/Elastic Net 第一轮审计

- 释放内容：系数数量、平均系数幅度、locked test AUC、accuracy、balanced accuracy、sensitivity、specificity、Brier/log loss。
- 冲突点：L1 更稀疏，但 malignant sensitivity 低于 L2/Elastic Net；“变量少”不等于“医学上更安全”。

Packet C：超参数路径与嵌套交叉验证

- 释放内容：C 路径图、CV mean +/- SD、nonzero coefficient path、nested CV 外层折结果。
- 冲突点：调参必须在内层完成，外层估计泛化；只报告最优均值会遮蔽不确定性。

Packet D：最终报告与外部验证边界

- 释放内容：locked test 指标、校准曲线、代理 stress validation 表、医学模型报告 checklist。
- 冲突点：同源数据的 stress split 只能提示分布变化风险，不能替代真实跨中心/跨时间外部验证。

## 7. 拟交付物与输出目录

输出目录：`class2026/materials/12. Regulation/`

计划文件：

- `pbl-regularization-validation-student-worksheet.docx`
- `pbl-regularization-validation-teaching-plan-flipped-classroom.docx`
- `pbl-regularization-validation-evidence-pack.docx`
- `pbl-regularization-validation-design-rationale.docx`
- `package-manifest.json`
- `assets/pbl-regularization-validation/`
- `notebook/pbl-regularization-validation-optional-lab.ipynb`

本讲形成独立案例。仅参考上一讲 `class2026/materials/11. Over-fitting/` 的视觉风格，不复制其教学内容和证据逻辑。

## 8. 验证与报告计划

数据拆分：

- 初始教学演示：stratified train/validation/test = 60%/20%/20%。
- 超参数选择：k-fold CV 比较 L1/L2/Elastic Net 与 C 路径。
- 偏倚控制：nested CV 用于展示“调参后的性能估计”。
- 最终评估：locked test 只用于最终报告。
- 外部验证：本数据没有真实独立外部队列；课程将设计“代理 stress validation”作为风险演示，并明确它不能替代真实外部验证。

报告指标：

- AUC、accuracy、balanced accuracy
- sensitivity for malignant、specificity for benign
- Brier score、log loss、calibration curve
- CV mean +/- SD；Packet D 中加入 locked test bootstrap 95% CI
- 系数数量、系数幅度、正则化路径
- 数据划分方式、调参流程和测试集锁定说明

## 9. 课堂练习与作业

课堂练习：

- 对 logistic regression 分别加入 L1 和 L2 正则化。
- 观察系数变化、非零特征数量变化、测试集性能变化、sensitivity/specificity 与校准变化。
- 要求学生解释：如果 L1 更稀疏但 sensitivity 稍低，医学报告应如何写边界？

作业：

- 为一个小样本高维数据设计合理验证方案，提交一页 validation protocol。
- 必须说明训练/验证/测试拆分、是否使用 k-fold 或 nested CV、如何报告不确定性、是否需要外部验证、以及不能做出的临床/因果结论。

## 10. 限制与边界

- 该包仅用于教学，不提供临床诊断、治疗或部署建议。
- UCI/sklearn 数据是经典基准，不能代表当前真实医院、多中心、设备或人群分布。
- L1 选中特征不等于发现生物机制或因果标志物。
- 高 AUC 不代表校准好、阈值合适或临床净获益充分。
- 没有真实外部验证前，不应声称模型可泛化到新医院、新设备或新患者群体。

## 11. 已确认的教师判断

1. 文件名采用 `regularization-validation`。
2. 第 12 讲形成独立案例。
3. Packet D 加入 bootstrap 置信区间。
4. 作业要求学生交一页 validation protocol。
