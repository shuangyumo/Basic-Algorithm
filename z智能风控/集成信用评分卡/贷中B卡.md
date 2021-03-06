## **风控业务背景**

随着新客获客成本越来越高，贷中客户管理越来越重要，包括额度管理（提降额度）、利率调整、提单意愿预测、流失倾向预测、营销响应预测等。

行为评分卡（Behavior Scoring）是一种根据客户在账户使用期间所产生的各种行为，动态预测客户风险的评分模型。其像是对客户过去一段时间的动态表现录像，然后与其在未来时间的一些状态照片对比。本文以信用卡和小额信贷分期产品为例，介绍行为评分卡（B卡）的基本知识。

> 目录
> Part 1. 信贷场景
> Part 2. 样本准备
> Part 3. 目标定义
> Part 4. 可用数据
> Part 5. 建模分析
> Part 6. 总结
> 致谢
> 版权声明
> 参考资料

## **Part 1. 信贷场景**

信贷生命周期管理大致可分为贷前（准入审核、额度授信、支用审批等）、贷中（额度管理、流失预测、营销响应等）、贷后（催收：还款率预测、账龄滚动、失联预测）三个阶段。

![img](https://pic3.zhimg.com/80/v2-49c7ab3ca970bb3f64e20d1c59f9206a_1440w.jpg)图 1 - 信用卡和小额信贷分期产品

如图1所示，我们以信用卡和小额信贷分期产品为例，分别介绍两者的特点：

**1. 信用卡**

信用卡在审批下卡前的阶段称为贷前，机构（银行、信用卡公司）会对客户风险综合评估，给予一个初始信用额度（如8000元）。下卡并激活后，进入贷中阶段，期间客户可在信用额度范围内进行透支消费，每两个账单日之间的消费流水账单将在后一个账单日（例如每月8号）通知客户。

账单日至最晚还款日（例如每月26号）前，客户可以随时还款，期间免息。还款方式一般支持一次性还清和分期还款。分期还款将产生利息收入，因此对于机构而言，自然是希望客户分期，默认推荐项也就是这个（为提高转化率，UI设计时肯定在右手边 ）。一旦客户逾期，那就进入贷后催收阶段。

~~~
解释一下免息期这个时间段
账单日：银行通知你在还款日需要还多少钱
还款日：应还欠款时间
容期日：还款日往后推一个时间段 比如还款日往后推3天

假设当前月是6月9号，账单日提醒你需要还1000元，那么这1000元是哪个时间段的消费呢？
账单日：6月8号
还款日：6月28号

上个月5月
账单日：5月8号
还款日:5月28号

这1000元的消费时间段是在5月8日(假设账单日的消费不计入当期还款)至6月7号。这1000元你可以在6月8号至6月28号之间任意一天进行还款。

~~~



**2. 小额信贷分期**

在贷前阶段，小额信贷分期产品所产生的每笔支用订单都需审批，通过后才放款到客户手中。放款后至结清的这段时间称为贷中。订单具有金额、期限、利率等属性，其约定了出借人和借款人之间的契约。与信用卡分期还款类似，小额信贷分期产品在每个还款日也必须偿还相应的本金和利息。

在客户发起支用申请订单后，将会生成一张还款计划表，如图2所示。显然，该还款方式为**等额本息**，即：在还款期内，每月偿还同等数额的贷款(包括本金和利息)。按此还款计划，借款本金共¥6000，年利率（APR）为18%。客户选择还款期限为6期，共产生6个还款日，每期偿还贷款¥1,053.15 (包括本金和利息)。

在额度允许范围内，客户可申请支用多笔订单。机构在审批新的一笔订单时，通常会参考历史订单的还款行为。如果客户发生逾期行为，并仍未出催，那么在其发起下一笔支用申请时，势必会被拒绝。该决策主要基于2个原因考虑：

1. **及时止损**：老订单的损失还未挽回，新订单大概率会引起进一步的损失，不能继续送钱。
2. **借新还旧：**已经发生逾期，说明客户的现金流存在问题。很可能是在借新钱还旧债。

![img](https://pic2.zhimg.com/80/v2-f5e7a88462a7074762e17074d94a3ba9_1440w.jpg)图 2 - 小额信贷分期产品-客户还款计划表

如图3所示，为了满足不同阶段风控需要，我们通常利用A卡（申请评分卡）、B卡（行为评分卡）、C卡（催收评分卡）对客户进行风险评分。

**贷前风控**：申请评分（**A**pplication Scoring），主要目的是为了识别获客阶段用户的逾期风险。一般应用于准入、额度授信、风险定价、支用审批，需要实时T+0部署到线上。例如，一些网贷产品考虑到用户体验，额度授信很多都设计在几分钟内计算出额。

**贷中风控**：行为评分（**B**ehavior Scoring），根据借款人放贷后的行为表现，预测未来逾期风险。在贷前阶段，我们对借款人的履约行为掌握相对较少，而且是静态的，引入B卡的目的是为了动态监控放款后的风险变化。B卡一般不需要实时上线，离线T+1计算即可。

**贷后风控**：催收评分（**C**ollection Scoring），在借款人当前还款状态为逾期的情况下，预测未来出催（还款）的可能性。这有助于催收员根据不同的逾期程度，采取不同的催收措施。例如，轻微逾期采用短信、电话提醒（以及相应的话术），重度逾期采用律师函等手段。C卡同样不需要实时上线，离线T+1计算即可。

![img](https://pic3.zhimg.com/80/v2-0815ca071d855af4e4e606a9f0882c0a_1440w.jpg)图 3 - 小额信贷分期产品A卡、B卡、C卡的作用范围

## **Part 2. 样本准备**

至此，本文粗略介绍了2种信贷业务和风控框架，主要帮助大家建立基本概念。统计学习建模离不开合适的样本。接下来，我们开始介绍B卡的样本准备，其主要适用于符合以下特点的产品：

1. **还款周期长**：如果周期短（如7天、1个月），用户风险可能变化较小，以及新增贷中客户信息有限，B卡则与A卡没有太大区别；反之，则可以引入还款履约行为，从而更为准确识别客户逾期风险。
2. **循环授信类**：在贷前阶段，我们无法很好识别客户风险，因此给予一个初始额度。随着下卡/放款后，客户与我们产生更多的交互，暴露出借款、还款等行为，我们就可以客户的初始额度进行提降。当风险升高时，降低额度以减少损失；反之，提额。

另外，B卡建模样本主要基于老客。主要原因在于老客具有足够长的申贷、还款记录。针对新/老客的口径定义，每家都有不同的说法。例如，我们可以定义：

> 新客：无历史在途/结清订单
> 老客：有至少1笔结清订单。

## **Part 3. 目标定义**

风控建模时，我们常把样本标签分为**GBIX**四类，其中：G = Good（好人，标记为0），B = Bad（坏人，标记为1），I = Indeterminate （不定，未进入表现期），X = Exclusion(排斥，异常样本)。

对于银行稳定的客群和长期限产品，一般可能会利用滚动率（roll rate）和vintage分析来定义好坏。但对于小额信贷分期产品，在实践中通常都结合产品期限，沿用常用指标。例如，对于6期产品，表现期可能就拍定MOB3。对于12期产品，表现期可能就拍定MOB6。

由于是客户级（customer level）风险模型，因此当用户在多个产品多笔订单时，我们需要综合多笔订单的逾期信息给用户打标。所提取的特征也是在用户级。

![img](https://pic3.zhimg.com/80/v2-1a269c6f739b4c466dd944ecb4d1c51e_1440w.jpg)图 4 - B卡建模目标

## **Part 4. 可用数据**

在平常生活中，我们把钱借给别人后，一般也都会旁敲侧击打听对方的行为，包括但不限于：

1. **消费行为**：也就是资金用途，即是否按借钱原因所述，真正专款专用。如果对方借到手后是用于不良嗜好（如黄赌毒、奢侈品等），那么我们就有理由相信还钱的可能性不高。因此，可以从信用卡账单、借记卡流水、电商数据等提取消费行为。
2. **负债压力**：除了欠这边之外，其他地方是否还有（多头）负债。用户共债压力越大，还款可能性就越低。通常情况下，一般哪家催的急，客户就选择还哪一家。
3. **收入能力**：是否有其他收入？如果没有收入，还款能力将直接受影响。收入数据包括流动资产（工资、公积金）、固定资产等。
4. **履约历史**：之前是否就有欠钱不还的黑历史，还是能按时履约。因此，需要记录历史的还款行为，比如提前还款，说明用户目前手头资金充足，重视诚信记录；习惯性逾期，则说明用户手头紧张，或者不够重视诚信记录。
5. **活跃行为**：借钱之后是否还能联系到？如果从此失联，消失在你的视野里，那么大概率就是不会还钱。在网贷产品中，我们可以观察用户登陆App的行为。

这些行为数据能让我们对借款人的贷后行为基本的判断，确定对方是否能正常履约。基本的特征工程可参考《[时间滑窗统计特征体系](https://zhuanlan.zhihu.com/p/85440355)》。

![img](https://pic3.zhimg.com/80/v2-a71ab48f42daf0cef1d99b3b0b1c76fe_1440w.jpg)图 5 - 可用数据维度

## **Part 5. 建模分析**

在经过样本准备、目标定义、特征工程、模型设计等环节后，建模流程可参考《[风控模型建模流程标准化](https://zhuanlan.zhihu.com/p/90251922)》。根据经验，B卡的模型区分度（KS）一般都能达到0.5以上。在A卡中，我们很难得到区分度如此高的模型，那么这是什么原因呢？

行为评分卡用到的一个重要特征是借款人的历史逾期行为。假设总客群中好客户为 ![[公式]](https://www.zhihu.com/equation?tex=g_t+%3D+8000)，坏客户为 ![[公式]](https://www.zhihu.com/equation?tex=b_t+%3D+200) 。根据历史是否逾期这个维度，我们将客群细分为2个：

- 客群1：好客户 ![[公式]](https://www.zhihu.com/equation?tex=g_1+%3D+830) 个，坏客户 ![[公式]](https://www.zhihu.com/equation?tex=b_1+%3D+160) 个，总数 ![[公式]](https://www.zhihu.com/equation?tex=n_1+%3D+g_1+%2B+b_1+%3D+990) .
- 客群2：好客户 ![[公式]](https://www.zhihu.com/equation?tex=g_2+%3D+7170) 个，坏客户 ![[公式]](https://www.zhihu.com/equation?tex=b_2+%3D+40) 个，总数 ![[公式]](https://www.zhihu.com/equation?tex=n_2+%3D+g_2+%2B+b_2+%3D+7210) .

在《[区分度评估指标(KS)深入理解应用](https://zhuanlan.zhihu.com/p/79934510)》中，我们介绍过KS曲线的绘制方法。如图6所示，我们只以分群变量进行排序。在子客群中，我们随机评分，其实也不需要在乎排序性。此时，在取得最大KS间隔的位置，好人/坏人累积比例分别为：

![[公式]](https://www.zhihu.com/equation?tex=good%5C_dist+%3D+%5Cfrac%7Bg_1%7D%7Bg_1%2B+g_2%7D+%3D+%5Cfrac%7B830%7D%7B830%2B7170%7D+%3D+0.104%5C%5C+bad%5C_dist+%3D+%5Cfrac%7Bb_1%7D%7Bb_1%2B+b_2%7D+%3D+%5Cfrac%7B160%7D%7B160%2B40%7D+%3D+0.8+%5Ctag%7B1%7D)

因此，我们容易计算：

![[公式]](https://www.zhihu.com/equation?tex=ks+%3D+max%28bad%5C_dist+-+good%5C_dist%29+%3D+0.8+-+0.104+%3D+0.696+%5Ctag%7B2%7D)

![img](https://pic1.zhimg.com/80/v2-1188e04d2302e50db00cb187b1f43b6c_1440w.jpg)图 6 - KS曲线图

因此，逾期类变量在行为评分卡中具有重要的贡献。在分群建模时，我们就可以评估：

> 模型最终效果 = 分群变量贡献 + 分群局部拟合贡献

## **Part 6. 总结**

本文系统介绍了贷中行为评分卡的制作过程。随着新客获客成本越来越高，老客关系管理日益重要。无论是交叉销售，还是挽回老客户所做的营销，我们都需要结合行为风险分数来做。因此，掌握B卡建模流程及应用场景，尤为重要。





>相关问题：
>
>1.客户级模型在进行多笔订单处理的时候，是将订单按放款对齐切分观察期表现期吗？
>
>**答**：
>
>1）样本在订单维度：每个订单就是一个观察点，但选择所有订单建模，表现期和观察期都跟着相应订单走，表现期内y定义为相应订单未来n天出现逾期。（有n笔订单就有n个样本）
>2）样本在客户维度：每个订单就是一个观察点，需要从中选择一个合适的观察点进行建模，表现期内y定义为未来n天任意一笔订单出现逾期。（有m个用户就有m个样本，m < n）
>
>对于方案1）:
>好处：多笔订单相当于对客户在不同时间点进行观察，特征x信息更全。
>坏处：由于各订单都是绑定在一个客户上，那么多个样本之间实际不独立（不满足建模样本的独立同分布假设）。这些订单样本的x不同，但y可能相同。
>
>对于方案2）：
>好处：在y定义上打通了各产品线订单，对好坏定义更贴近于人维度。
>坏处：牺牲了一些样本，会使得样本大量减少。

> 问题:
>
> \1. B卡的作用是不是用来决定下次该用户申贷是的利率和额度等呢
>
> \2. 客户级风险模型是什么意思呢？
>
> \3. “但对于小额信贷分期产品，在实践中通常都结合产品期限，沿用常用指标。”为什么要结合产品期限呢？
>
> 答：
>
> ​	1. 对于有信贷历史的客户而言，根据B卡分对卡账户提降额度。如果客户的信用良好，后期费率也会得以降低。
>
> 2. 客户级是相对于产品级而言的。比如某金融机构有3个信贷产品，产品级是指根据各自样本独立建模，客户级是打通各产品的样本所构建的模型。