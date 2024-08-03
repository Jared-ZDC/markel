---
title: python In 机器学习(一)
date: 2024-08-03 08:25:19
tags: python 机器学习
categories: python 机器学习
toc: true
---
# LinearRegression

这里简单记录一下最近学习的算法代码

<!-- more -->

## 多元线性回归算法

这里主要通过sklearn获取数据进行实验

```python
from sklearn import *
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error
import torch

if __name__ == '__main__':
    # 加载糖尿病数据集
    data = datasets.load_diabetes()
    data_X = data.data
    data_Y = data.target
    # 拆分训练集以及预测数据
    X_train, X_test, Y_train, Y_test = train_test_split(data_X, data_Y, test_size=0.2)

    # 创建一个多元线性回归算法对象
    lr = LinearRegression()

    # 使用训练集训练模型
    lr.fit(X_train, Y_train)

    # 使用测试集进行预测
    Y_pred = lr.predict(X_test)
    Y_pred_train = lr.predict(X_train)
    # 打印模型的均方差
    print(f"pred loss : {mean_squared_error(Y_test, Y_pred)}")

    print(f"Y_pred_train loss : {mean_squared_error(Y_train, Y_pred_train)}")
    pass

```



## 分类

这里是分类的应用：

* logistics回归 ： OVR，one vs other，将多分类当成多个二分类
* softmax：一次得到多个分类的概率

```python
from sklearn import datasets
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

if __name__ == '__main__':
    # 加载数据
    iris = datasets.load_iris()

    X = iris.data
    Y = iris.target

    print(f"Y = {Y}")

    # 切分数据
    X_train, X_test, y_train, y_test = train_test_split(X, Y, test_size=0.2)

    # 创建模型, 根据数据决定是用二分类还是多分类
    # 使用logistics回归还是softmax回归 ，取决于multiclass参数
    # ovr ： 1对其它
    # multinomial ： softmax
    #  "multi_class": [
    #             StrOptions({"auto", "ovr", "multinomial"}),
    #             Hidden(StrOptions({"deprecated"})),
    #         ],
    lr = LogisticRegression(max_iter=1000000)

    # 使用训练集训练模型
    lr.fit(X_train, y_train)

    # 对测试集进行预测
    y_pred = lr.predict(X_test)

    # 打印模型的准确率
    print(f"accuracy = {accuracy_score(y_test, y_pred)}")

```

