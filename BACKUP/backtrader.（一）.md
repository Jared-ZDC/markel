---
title: backtrader （一）
date: 2024-12-19 08:28:25
tags: 程序人生
categories: 程序人生
toc: true
---
当前组合版本： numpy==1.23.5   backtrader==1.9.78.123 matplotlib==5.6

但是始终在调用plot的时候出现错误：
```bash
AttributeError: 'Plot_OldSync' object has no attribute 'mpyplot'

```
尝试过不同的版本，都解决不了这个问题，目前尚不知道什么原因，暂时不使用画图，或者后续有空了用pyecharts自己画；

<!-- more -->


```python
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

from backtest.tool import *


# Create a Stratey
class TestStrategy(bt.Strategy):

    def log(self, txt, dt=None):
        ''' Logging function for this strategy'''
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # Keep a reference to the "close" line in the data[0] dataseries
        self.dataclose = self.datas[0].close

    def next(self):
        # Simply log the closing price of the series from the reference
        self.log('Close, %.2f' % self.dataclose[0])


if __name__ == '__main__':
    # Create a cerebro entity
    cerebro = bt.Cerebro()

    # Add a strategy
    cerebro.addstrategy(TestStrategy)

    df = load_data_from_csv("000001.SZ")

    #加载数据
    data = dataframe_to_datafeeds(df)

    # Add the Data Feed to Cerebro
    cerebro.adddata(data)

    # Set our desired cash start
    cerebro.broker.setcash(100000.0)

    # Print out the starting conditions
    print('Starting Portfolio Value: %.2f' % cerebro.broker.getvalue())

    # Run over everything
    cerebro.run()

    # Print out the final result
    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())


```



---

补充一下工具函数：
```python
import os
from pathlib import Path

import backtrader.feeds as btfeeds
import backtrader as bt
import tushare as ts
import pandas as pd
from datetime import datetime


class PandasDataExtend(bt.feeds.PandasData):
    # 只定义需要的列
    lines = ('open', 'high', 'low', 'close', 'vol')

    # 设置列映射
    params = (
        ('datetime', None),
        ('open', 'open'),
        ('high', 'high'),
        ('low', 'low'),
        ('close', 'close'),
        ('volume', 'vol'),  # Tushare使用'vol'表示成交量
    )


def dataframe_to_datafeeds(df: pd.DataFrame, start_date: str = "20100101", end_date: str = "20250101"):
    return bt.feeds.PandasData(dataname=df, fromdate=pd.to_datetime(start_date), todate=pd.to_datetime("20240101"))

def load_data_from_csv(ts_code: str, start_date: str = "20100101", end_date: str = "20250101",
                       adj: str = "hfq") -> pd.DataFrame:
    """
        获取指定股票在指定日期范围内的日线行情数据。

        参数:
        - ts_code: 股票代码
        - start_date: 开始日期，格式 YYYYMMDD
        - end_date: 结束日期，格式 YYYYMMDD

        返回:
        - 包含日线行情数据的Pandas DataFrame
        """
    # 获取当前文件的绝对路径
    project_root = Path(__file__).resolve().parents[1]
    # 使用项目路径加载数据文件
    data_path = project_root / 'data'

    df = pd.read_csv(f"{data_path}/{ts_code}_{start_date}_{end_date}_{adj}.csv")
    if len(df) == 0:
        raise ValueError("csv数据找不到！！！")
    df['trade_date'] = pd.to_datetime(df['trade_date'])
    df.set_index('trade_date', inplace=True)
    df.sort_index(inplace=True)  # 确保数据按日期排序
    df.fillna(0.0, inplace=True)

    return df

```