
"""
Created on Wed Aug 12 10:45:40 2020

@author: yao71
"""
import talib as ta
import tushare as ts
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import datetime
import random

ts.set_token('你的token')
pro = ts.pro_api()
#获取股票列表
stocklist = pro.stock_basic(exchange='', list_status='L', fields='ts_code,symbol,name')
#抽样模块
sample = []
while len(sample) < 100:
    rdm = random.randint(0,len(stocklist)-1)
    if rdm not in sample:
        df = ts.pro_bar(ts_code=stocklist.loc[rdm,'ts_code'], adj='qfq', start_date='20180812', end_date='20200812')
        if len(df) >400:
            sample.append(stocklist.loc[rdm,'ts_code'])
            print('Sampling...',len(sample)/100*100,'%')
#计算模块

df_record = pd.DataFrame()
m = 0
for z in sample:
    df = ts.pro_bar(ts_code=z, adj='qfq', start_date='20170812', end_date='20200812')
    df = df.drop('vol',axis=1)
    df = df.drop('amount',axis=1)
    df = df.drop('pre_close',axis=1)
    df = df.sort_index(axis=0,ascending=False)
    df = df.reset_index(drop = True)
    for i in range(0,len(df)):
        df.loc[i,'trade_date'] = datetime.datetime.strptime(df.trade_date[i],'%Y%m%d')
    #计算MACD
    close = [float(x) for x in df['close']]
    df['MACD'],df['MACDsignal'],df['MACDhist'] = ta.MACD(np.array(close),
                                fastperiod=12, slowperiod=26, signalperiod=9)   
    df = df.drop('MACD',axis=1)
    df = df.drop('MACDsignal',axis=1)
    #计算KDJ
    df['9max'] = df['high'].rolling(window=9,min_periods=9).max()
    df['9min'] = df['low'].rolling(window=9,min_periods=9).min()
    for i in range(9,len(df)):
        if i == 9:
            df.loc[i,'RSV'] = (df.loc[i,'close'] - df.loc[i,'9min']) / (df.loc[i,'9max'] - df.loc[i,'9min']) * 100
            df.loc[i,'K'] = 50
            df.loc[i,'D'] = 50
            df.loc[i,'J'] = df.loc[i,'K'] * 3 - df.loc[i,'D'] * 2
        else:
            df.loc[i,'RSV'] = (df.loc[i,'close'] - df.loc[i,'9min']) / (df.loc[i,'9max'] - df.loc[i,'9min']) * 100
            df.loc[i,'K'] = df.loc[i-1,'K'] * 2/3 + df.loc[i,'RSV'] * 1/3
            df.loc[i,'D'] = df.loc[i-1,'D'] *2/3 + df.loc[i,'K'] * 1/3
            df.loc[i,'J'] = df.loc[i,'K'] * 3 - df.loc[i,'D'] * 2
    df = df.drop(range(0,33))
    df = df.reset_index(drop=True)
    df.loc[0,'status'] = 'Unhold'
    df.loc[0,'money'] = 100
    i = 1
    while i < len(df):
        if df.MACDhist[i] > df.MACDhist[i-1]:
            if df.J[i] > df.K[i] and df.J[i] > df.D[i] and df.J[i-1] < df.K[i-1] and df.J[i-1] < df.D[i-1] and i != len(df) -1:
                df.loc[i,'status'] = 'Buy'
                start_price = df.close[i]
                df.loc[i,'money'] = df.money[i-1]
                j = i + 1
                while df.J[j] >= df.K[j] and j != len(df)-1:
                    df.loc[j,'status'] = 'Hold'
                    df.loc[j,'money'] = df.money[j-1] * (1 + df.pct_chg[j] / 100)
                    j += 1
                df.loc[j,'status'] = 'Sell'
                df.loc[j,'money'] = df.money[j-1] * (1 + df.pct_chg[j] / 100)
                i = j + 1
            else:
                if df.status[i-1] == 'Sell':
                    df.loc[i,'status'] = 'Unhold'
                else:
                    df.loc[i,'status'] = df.status[i-1]
                df.loc[i,'money'] = df.money[i-1]
                i += 1
        else:
            if df.loc[i-1,'status'] == 'Sell':
                df.loc[i,'status'] = 'Unhold'
            else:
                df.loc[i,'status'] = df.status[i-1]
            df.loc[i,'money'] = df.money[i-1]
            i +=1
    print('finished',z, m/len(sample)*100,'%')
    total_growth = (df.loc[len(df)-1,'close'] - df.loc[0,'close']) / df.loc[0,'close']
    strategy_growth = (df.loc[len(df)-1,'money'] - df.loc[0,'money']) / df.loc[0,'money']
    df_record.loc[m,'stock'] = z
    df_record.loc[m,'stg_growth'] = strategy_growth
    df_record.loc[m,'total growth'] = total_growth
    df_record.loc[m,'Override'] = 'Yes' if strategy_growth > total_growth else 'No'
    df_record.loc[m,'Com_Pct'] = (strategy_growth - total_growth) / abs(total_growth)
    
    m += 1

m = 0
for i in range(0,len(df_record)):
    if df_record.loc[i,'Override'] == 'Yes':
        m +=1
print('在',len(df_record),'支股票中，该策略的胜率为',m/len(df_record))
df_record['stg_growth'].mean()
print('你的100元变成了',100 * (1+ df_record['stg_growth'].mean()),'元 ^_^')

#TO DO:加入波动率对比,在胜的里面平均胜多少，败的里面平均亏多少？函数化？最大最小值？最大回撤？在牛市和熊市中分别的表现？
    #抄底优化？T-检验显著性？

                    
                    





