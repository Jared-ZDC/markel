---
title: backtrader 简单双均线策略
date: 2024-12-20 03:21:08
tags: 程序人生
categories: 程序人生
toc: true
---
策略的主要行为：
1） 初始化5日均线以及20日均线，sma5， sma20
2)    sma5上穿sma20买入
3） sma5 下穿sma20卖出

策略绩效：
```bash
Starting Portfolio Value: 100000.00
Final Portfolio Value: 99796.00
```
<!-- more -->

```python
# 简单用双均线，测试一下
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)

import time

from backtest.tool import *
import backtrader as bt

# Create a Stratey
class TestStrategy(bt.Strategy):
    params = (
        ('period_low', 5),
        ('period_high', 20),
    )

    def log(self, txt, dt=None):
        ''' Logging function for this strategy'''
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.sma_low = bt.indicators.SimpleMovingAverage(self.datas[0].close, period=self.params.period_low)
        self.sma_high = bt.indicators.SimpleMovingAverage(self.datas[0].close, period=self.params.period_high)
        self.sell_sig = self.sma_high > self.sma_low
        self.buy_sig = self.sma_high < self.sma_low
        self.last_sig = False #上一次信号是买或者是卖， False表示上一次是卖
    def next(self):
        if self.sell_sig[0] and self.last_sig :
            self.sell()
            self.last_sig = False

        if self.buy_sig[0] and not self.last_sig:
            self.buy()
            self.last_sig = True



if __name__ == '__main__':
    # Create a cerebro entity
    cerebro = bt.Cerebro()

    # Add a strategy
    cerebro.addstrategy(TestStrategy)

    df = load_data_from_csv("000001.SZ")
    # 加载数据
    data = bt.feeds.PandasData(dataname=df, fromdate=pd.to_datetime("20100101", format='%Y%m%d'),
                               todate=pd.to_datetime("20241201", format='%Y%m%d'))

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