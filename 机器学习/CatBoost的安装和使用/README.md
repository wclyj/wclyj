# CatBoost：类别特征不用先手工编码

CatBoost 属于梯度提升树这一族，2017 年由 Yandex 的研究发表。它和常见的提升树解决同一类监督问题，差别主要在三件事：类别列怎么进树、同一层的分裂是否共享、叶子上的数值怎么估计。

## 类别列

很多表里的列是城市、渠道、商品类目，而不是连续数。若先做成独热向量，列会突然变宽，稀有类目几乎全是 0。CatBoost 把类别换成目标统计量：看这个取值在历史上对应的标签平均水平，再加一个先验，避免某个只出现一次的类目把统计量拉满。

更关键的是统计量的计算顺序。若用「包括当前这一行」的标签去做统计，模型等于偷看了答案，训练误差会虚低。CatBoost 采用有顺序的统计：对当前样本，只用排在它前面的样本估计该类别的目标均值。换一个随机排列再估一次，可以减轻「谁排在前面」带来的偶然性。

它还可以把几个类别列拼成新列，例如「城市 + 设备」。交互不必全部靠人工去叉乘，但组合会变多，深度和迭代轮数仍要限制。

## 对称树

基学习器是对称树（也叫 oblivious tree）：同一层只用同一个特征和同一个阈值，该层所有结点一起分。树的形状比普通 CART 呆板，表达力靠多棵树补上。好处是预测时一层只要比一次，结构规则，不容易长出只照顾极少数样本的细枝。

叶子上的输出也不等于落在该叶子里的标签简单平均。配合前面的顺序原则，叶子值用的是尽量不包含当前样本自身标签的估计，用来压低训练集上的偏移。

## 安装

优先用已有的 Python 环境：

```bash
pip install catboost
```

若走 conda，先配好镜像再 `conda install -T 某个频道 catboost` 即可。装完后在解释器里 `import catboost`，没有报错就说明动态库可用。GPU 轮子和 CPU 轮子不要混装，导入失败时先看是不是缺了对应的运行库。

## 二分类用法草图

下面用一份虚构的流失表：数值列是近 30 天登录次数，类别列是渠道。标签是是否离开。验证集只用来看对数损失，不参与构造训练行的类别统计之外的调参循环时，仍要避免拿测试标签反复试错。

```python
import numpy as np
import pandas as pd
from catboost import CatBoostClassifier as CatModel

frame = pd.read_csv("churn_train.csv")
hold = pd.read_csv("churn_holdout.csv")
feat_cols = ["login_30d", "channel"]
cat_idx = [frame[feat_cols].columns.get_loc("channel")]

clf = CatModel(
    iterations=200,
    depth=4,
    learning_rate=0.08,
    loss_function="Logloss",
    cat_features=cat_idx,
    verbose=False,
)
clf.fit(
    frame[feat_cols],
    frame["left"],
    eval_set=(hold[feat_cols], hold["left"]),
)
score = clf.predict_proba(hold[feat_cols])[:, 1]
```

`cat_features` 传的是列的位置，不是列名字符串，传错位置时类别列会被当成连续值硬切。`iterations` 大而 `learning_rate` 也大，验证损失会先降后升，应提早停止。精度不要只看整体命中率：离开的人若只占一成，全猜「留下」也能到九成。看验证集的对数损失或 ROC 面积更合适。

## 何时不用

样本只有几十行时，类别统计量极不稳定，对称树也学不出稳定叶子。特征全是已经标准化的连续值、没有类别列时，CatBoost 仍能跑，但相对别的提升树优势变小。缺失值可以留给它内部处理，但「缺失本身代表一种业务状态」时，最好显式做成类别，而不是指望默认策略读懂业务。
