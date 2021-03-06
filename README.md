# 智慧中国杯全国大数据创新应用大赛
本次参加的是三大赛题中的[用户贷款风险预测（算法竞赛）](http://www.pkbigdata.com/common/cmpt/%E7%94%A8%E6%88%B7%E8%B4%B7%E6%AC%BE%E9%A3%8E%E9%99%A9%E9%A2%84%E6%B5%8B_%E7%AB%9E%E8%B5%9B%E4%BF%A1%E6%81%AF.html)



## **队伍简介** ##
队伍名“一战成名”，队名含义很简单，我们几个都是菜鸟级别玩家，希望可以一战成名。萍水相逢的四名队友，分别来着电子科技大学的[冒菜](https://github.com/jqk2017/Finance)、西安电子科技大学的DG、中山大学的熊掌以及中国海洋大学的火箭(本人)，两个计算机专业，一个管理，我是GIS。
最终线上排名第七，遗憾没进答辩。
![](http://i.imgur.com/nVKwO2i.png)

## **任务** ##
融360与平台上的金融机构合作，提供了近7万贷款用户的基本身份信息、消费行为、银行还款等数据信息，需要参赛者以此建立准确的风险控制模型，来预测用户是否会逾期还款。

[赛题：用户贷款风险预测（算法竞赛）](http://www.pkbigdata.com/common/cmpt/%E7%94%A8%E6%88%B7%E8%B4%B7%E6%AC%BE%E9%A3%8E%E9%99%A9%E9%A2%84%E6%B5%8B_%E7%AB%9E%E8%B5%9B%E4%BF%A1%E6%81%AF.html)

**更新数据下载地址：链接：https://pan.baidu.com/s/1smvX7Pj 密码：jvxa**

训练数据 ：https://pan.baidu.com/share/init?shareid=1950975130&uk=2921663747    
密码：pcsh

测试数据：https://pan.baidu.com/share/init?shareid=1950975130&uk=2921663747  
密码：pcsh



**训练数据包括用户的基本属性user_info.txt、银行流水记录bank_detail.txt、用户浏览行为browse_history.txt、信用卡账单记录bill_detail.txt、放款时间loan_time.txt，以及这些顾客是否发生逾期行为的记录overdue.txt。**

##**评分标准**
**评分标准：**

![](http://i.imgur.com/LaQryMG.jpg)
![](http://i.imgur.com/YZrSRFk.png)

##**解决方案概述**##
本题很多关键属性被脱敏处理，比如时间戳和所有金额的值，这个对我们进行特征构造带来很多的影响，损失了很多业务信息。不过对于参赛者都是公平的，因而我们构造了大量的统计特征，根据模型及线上反馈最佳特征大多来自用户浏览行为**browse_history**和**bill_detail**，此外发现**放款时间**也是个强力特征，详细见代码部分。这里只放了我个人的代码，队友的特征工程很多类似的，也有一些独特之处，这里说几个思路：bill_detail表的特征按放款时间分为放款前放款后分别统计（还可以尝试多划分几个时间窗再统计）、基于熵的分箱处理（特征离散化，熊掌整理了思路见：最优分箱.docx）、排序特征、组合特征等，有兴趣可以自己去实现。模型方面，我本人主要玩了xgboost和lightgbm，队友也基本上是xgboost、RandomForest，在玩Stacking融合的时候还上了ExtraTreesClassifier和Logistic Regression。

##**模型设计与模型融合**##
- **单模型：**  
 还是玩的大杀器xgboost,简单粗暴，然后进行了一些调参工作。个人单模型成绩最佳到0.443+，Top35,队友也基本玩的xgb,组队后大家做了特征融合，单模型能到0.455+，线上Top20。
- **mic加权融合：**  
后期单模型到极限了，也可能我们特征水平有限，已经很难通过特征工程和调参提升成绩了。于是开始玩融合,参考了“不得直视本王”的解决方案，对不同的模型结果计算mic值对比相关性，然后根据线上以及线下的评分进行加权融合，记得那天在群里就模型简单加权融合还是完善特征工程或是优化验证集等等讨论很久，最终决定还是尝试一下mic，结果当天提升到0.46+，直接Top10了。在最后几天的尝试中，我们分工完善特征、组合特征训练模型，在分箱处理后单模型竟然上了0.465，可惜临近结束没有太多的验证机会了，此外Rank融合也没有再验证，理论上是可以优化的。
- **Stacking模型:**  
再玩mic加权融合的同时，我们总结了成绩提升的原因，就是模型多样化。不同的模型结果（不同特征集或者不同的样本集或者不同的模型）融合才能得到好的结果，可以有效避免过拟合。抱着学习的态度，我们开始尝试stacking融合，边学边做，这个轮子是队友找来的，我们对它进行了一些修改，做成了我们自己的stacking轮子。  
主要思路是：根据最佳单模型的xgb特征评分排名将特征集分量两个子集（1、3、5....和2、4、6...）作为基模型的训练集。基模型包括ET、RF、Xgboost,3个基模型分别训练这2个特征子集，保证相同的cv和seed。这样训练完成得到验证集的预测值拼接成新的训练集，一共可以得到6个新特征（2组特征子集，3个模型），最后第二层基于这些新特征训练ET模型，这里可以加上一部分你的原始特征，也可以组合新的特征，这些都值得尝试。Stacking的过程是非常耗时的，因为需要跑多个模型以及多次cv，我16G内存，i7 cpu跑一次完整的需要七八个小时。至于Stacking带来的提升，本次比赛并没有带来火箭，
stacking的思路可以参考下面这张图，如果难以理解建议多查资料或者直接请教别人，然后看代码实现，我一开始也是觉得懂了，但是用起来发现细节还是不清楚。
![](https://i.imgur.com/FvedFX0.jpg)
**相关博文：**  
https://blog.csdn.net/wstcjf/article/details/77989963    
http://mlwave.com/kaggle-ensembling-guide/  
http://blog.csdn.net/a358463121/article/details/53054686#t18

## **个人总结** ##
参加这个比赛是快放寒假的时候，想寒假找点事做做，当时就DC有三个比赛吧，交通赛数据太大玩不动，教育赛觉得没意思，于是乎玩了金融赛，Kaggle的也关注了一下（难度系数高，后面放弃了）。抱着学习的态度从DC群老段子的开源代码玩起，一步步慢慢提升，年前玩到Top40，然后过年荒废了十来天，回校的时候成绩已经70开外，于是开始新一波努力。在这个过程中认识了现在的队友，边交流边提升，可以说如果我们没有组队，我们四个人都到不了Top10，因此我再次觉得打比赛还是要团队协作，这样可以互相佐证思路，实现更多的想法，完成更多的思路，学习到更多的东西。而且当你有队友以后，你会变得更加投入，每个人都有责任感。在群里争论，产生分歧，最后大家统一想法，完成提交。基本每天早上开始各自实现思路，晚上八九点开始讨论融合，十二点前完成提交，有时候因为取得突破激动的很晚才能睡着，也有最后一天最后一搏失败后的互相安慰，这种体会真的很棒。总之非常感激本次比赛的各位队友们，让我真正体会到比赛的乐趣。


有点扯远了...另外，建议，多看以往的大神总结，wepon大神的github吐血推荐！！！https://github.com/wepe  ，本次比赛主要参考了他们去年微额借款用户人品预测大赛冠军解决方案以及拍拍贷风险控制大赛铜奖解决方案，干货多多！

此外还有：金老师的知乎专栏：https://zhuanlan.zhihu.com/jlbookworm  ，收录了各种大神解决方案和开源代码。

然后就是kaggle比赛分享，新手教程等等。

最后，祝大家比赛玩得开心！















