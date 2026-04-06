## 算法0: 基于规则的异常检测

|   |   |   |
|---|---|---|
|observation|规则|anomaly_score|
|x|if x<target 1 then elif x<target 2 then|{1, 2, 3} 一般、中等、重度|

## 算法1: 指标加权移动平均算法 EWMA

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1700796906703-737e08fb-98b6-4860-92c4-16b45a54975f.png)

## 算法2: 高斯过程回归 GPR

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1700796906335-7b242c28-d74f-413b-b3f1-9521dfac687e.png)

**模型原理**

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1700796906820-3202491f-1bb8-495c-afd7-5430973aad8e.png)

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1700796906969-ac57e6c3-de4a-48e1-b3d4-627cc1a43590.png)

**特征工程**

- 提取时间特征就可以了，用于核函数距离计算

**模型训练**

- 最大似然估计，计算核函数的参数
- [sklearn.gaussian_process.GaussianProcessRegressor](https://scikit-learn.org/stable/modules/generated/sklearn.gaussian_process.GaussianProcessRegressor.html#sklearn.gaussian_process.GaussianProcessRegressor), fit函数

**模型推理**

- (4)和(5)公式
- [sklearn.gaussian_process.GaussianProcessRegressor](https://scikit-learn.org/stable/modules/generated/sklearn.gaussian_process.GaussianProcessRegressor.html#sklearn.gaussian_process.GaussianProcessRegressor), predict函数

## 算法3: 梯度提升树 GBDT

**模型原理**

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1700796907038-f2a93573-5033-4464-b616-885f5fd752d8.png)

**特征工程**

- 通过tsfresh提取关注的时间位置、统计特征，[https://tsfresh.readthedocs.io/en/latest/](https://tsfresh.readthedocs.io/en/latest/)

**模型训练**

- [Python API Reference — xgboost 2.0.0 documentation](https://xgboost.readthedocs.io/en/stable/python/python_api.html#module-xgboost.training), xgboost.train函数

**模型推理**

- [Python API Reference — xgboost 2.0.0 documentation](https://xgboost.readthedocs.io/en/stable/python/python_api.html#module-xgboost.training), booster.predict函数

## 算法4: 隔离森林

## 算法5: DBSCAN

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1700796947988-3556a727-8de5-4189-88a9-903aa119be8b.png)

**模型原理**

![](https://cdn.nlark.com/yuque/0/2023/png/12409702/1700796948174-2d8b5b2c-eaa3-4f09-94b4-bb0fd5fd996d.png)

**模型训练**

- [sklearn.cluster.DBSCAN](https://scikit-learn.org/stable/modules/generated/sklearn.cluster.DBSCAN.html), fit_predict函数
- min_samples = total/5
- eps: success_rate,5%(绝对值). duration, 10%(相对于边基线值)

## 异常度计算

|   |   |   |   |
|---|---|---|---|
|观测|基线|异常度|说明|
|x|y|(x-y)/ (exp(10, len(y)-1)={1, 2, 3(>=3)}|绝对偏离|
|x|y|10(x-y)/ y= {1, 2,3(>=3)}|相对偏离|

## 异常压缩

目前

## 异常排序