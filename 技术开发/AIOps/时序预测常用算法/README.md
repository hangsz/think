时间序列是现实中见的最多的一种数据类型，在人工智能领域有专门的一系列算法来处理它，下面说说时间序列预测不得不做的那点小事。

# 传统时间序列的预测套路

## 肉眼观测趋势和周期

一般情况下，趋势和周期都会有。

此时没必要去进行白噪声和平稳性检验，因为都会失败。

## 过滤趋势和周期

如果有周期的话，这一步要成功完成，必须提供一个周期参数。

python的statmodels库，是这样过滤趋势和周期的：

函数：[statsmodels.tsa.seasonal.seasonal_decompose](https://www.statsmodels.org/stable/generated/statsmodels.tsa.seasonal.seasonal_decompose.html)

1. 每个点的趋势值计算：沿着周期的长度直接平均。
2. 每个点的周期值计算：原始时间序列减去趋势，然后间隔周期的长度进行平均。
3. 剩下的residual就是我们想要的。
4. 如果没有周期，可以直接用差分来滤掉趋势，一般用到二次差分（差分的差分）就够了。

## 白噪声检验

函数：[statsmodels.stats.diagnostic.acorr_ljungbox](https://www.statsmodels.org/dev/generated/statsmodels.stats.diagnostic.acorr_ljungbox.html)

检验residual是否是白噪声，如果是则无法预测，不要白费功夫，statmodels中有标准的检测方法。

## 平稳性检验

函数：[statsmodels.tsa.stattools.adfuller](https://www.statsmodels.org/dev/generated/statsmodels.tsa.stattools.adfuller.html)

检验residual是否是平稳时间序列，如果是平稳的，可以用成熟的算法来解决它，statmodels中有标准的检测方法。

**为什么要平稳？**

这个问题不要问，问就是必须平稳。

哈哈，说白了就是因为平稳序列具有比较好的性质，具有时间平移不变性，具体的说法是弱平稳时间序列均值为常数，协方差只与lag有关。其容易建立模型来预测未来，不平稳的不好搞。

## ARMA模型中的p值和q值确定

**p值可以通过看pacf图来确定**

- pacf图中的值为偏自相关系数，**p值指的是自回归模型AR的系数数量**

**自回归模型AR**

x(t) = a1 x(t-1)+a2 x(t-2)

自回归模型的思路是过去几点的值，会影响当前的值。

其中，a1和a2是偏自相关系数，每个系数反映每个lag对未来单独的影响，上式p值为2。

**如何计算自回归模型系数和p？**

statmodels中这样计算计算偏自相关系数

函数：[statsmodels.tsa.stattools.pacf](https://www.statsmodels.org/devel/generated/statsmodels.tsa.stattools.pacf.html)

那么怎么找到p呢，我们可以假设时间序列符合自回归模型，穷举所有可能的p，实际中最多算到40，意味着要求解40次最大似然估计问题。

比如p=1，取出a1；

p=2，取出返回得到的a2……

p=40，取出返回得到的a40

这样就得到了40阶lag的偏自相关系数。

**纯自回归模型AR模型的性质**

- pacf截尾，acf拖尾。这个性质用于判断是否为AR模型以及选定p。

**q可以通过看acf图来确定**

- acf中图中值为自相关系数，q值指的是移动平均模型MA的系数数量

**移动平均模型**

x(t) = b1e(t-1)+b2 e(t-2)

其中，这个系数我们暂且先不管，上式q值为2。

移动平均模型的思路是过去几点上产生的冲击，会对当前产生影响。举个例子，周三周四晚上食堂吃饭人数都增加了，那么周五吃饭人数有理由相信也会增加。

**如何计算移动平均模型系数和q？**

statmodels中这样计算计算自相关系数：

函数：[statsmodels.tsa.stattools.acf](https://www.statsmodels.org/devel/generated/statsmodels.tsa.stattools.acf.html)

**自相关系数为两个量的相关性，所以很容易计算。**比如lag=1相关性，计算x(t)……x(end-1)与x(t+1)……x(end)的相关性就行了，其他lag同理。

**纯MA模型的性质**

- acf截尾，pacf拖尾。这个性质用于判断是否为MA模型以及选定q。

p和q确定之后，怎么确定自回归和移动平均模型系数？还是最大似然估计那一套，有时候你会看到各种AIC BIC准则，其实这就是加了正则项的最小平方差函数，本质是最大后验估计MAP。这么一看，统计和机器学习本质上是一回事。

# 机器学习加成下时间序列的预测

**首先传统的1.1-1.4步骤还是需要的，其可以帮助我们了解数据。**

## GBDT

**需要提取特征**

tsfresh提供了几百个时间序列特征，总有一款适合你。

[tsfresh - tsfresh 0.12.0 documentation](https://tsfresh.readthedocs.io/en/latest/)

## GPR(高斯过程回归)

需要设计核函数

scikit-learn提供了各种核函数，反映趋势的RBF核，反映周期的ExpSineSquared核，组合组合总会适合你。

[API Reference - scikit-learn](https://scikit-learn.org/stable/modules/classes.html)  
  

## LSTM

连特征都不需要了，确定好样本长度，无脑上吧。

[LSTM](https://pytorch.org/docs/stable/generated/torch.nn.LSTM.html)

# 总结

新的牛人发明了简单、自动化的新的算法，淘汰了旧的算法，淘汰了旧的大牛。

然后小白直接上手了新算法，由于便宜，淘汰了新的大牛。