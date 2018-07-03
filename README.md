# -*- coding: utf-8 -*-
#%% 准备工作
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt


#%% 读取数据
df = []
f = open('D:/NLP/text_label.txt', encoding = 'utf-8')
while True:
    line = f.readline()
    if not line:
        break
    fields = line.strip().split()
    if len(fields[0]) > 8:
        score = 0
        date0 = fields[0].split(':')[1]
        time0 = fields[1]
          
    else:
        cate,date0 = fields[2].split(':')
        if cate == '__POS__]':
            score = 1
        else:
            if cate == '__NEG__]':
                score = -1
            else:
                score = 0
        time0 = fields[3]
           
    record = [date0,time0,score]
    df.append(record)
df = pd.DataFrame(df)
df.columns = ['Date','Time','Score']     


#%% 整理数据结构
#### 生成一天中的各秒并合并
def sec_to_time(seconds):
    m, s = divmod(seconds, 60)
    h, m = divmod(m, 60)
    return ("%02d:%02d:%02d" % (h, m, s))
Time = [sec_to_time(x)  for x in range(24*60*60)]
Date = ['2018/06/13','2018/06/14','2018/06/15','2018/06/16','2018/06/17','2018/06/18','2018/06/19','2018/06/20']
sec = []
for d in Date:
    for t in Time:
        sec.append([d,t])
sec = pd.DataFrame(sec)
sec.columns = ['Date','Time']
df2 = pd.merge(df, sec, how = 'right', on = ['Date','Time'])
df2['Count'] = 1-np.isnan(df2['Score'])

Score = list(df2['Score'])
for i in range(len(Score)):
    if np.isnan(Score[i]):
        Score[i] = 0 
df2['Score'] = Score

df2 = df2.sort_values(by = ['Date','Time'])


#### 生成每10秒的标签
Time_10sec = []
for i in df2.index:
    Time_10sec.append(df2.loc[i,'Time'][:7]+'0')
df2['Time_10sec'] = Time_10sec


#### 生成每分钟的标签
Time_min = []
for i in df2.index:
    Time_min.append(df2.loc[i,'Time'][:5])
df2['Time_min'] = Time_min

Time_10min = []
for i in df2.index:
    Time_10min.append(df2.loc[i,'Time'][:4]+'0')
df2['Time_10min'] = Time_10min


#### 调整各列顺序
df2['Count2'] = df2['Count']
df2['Score2'] = df2['Score']
del df2['Count']
del df2['Score']
df2.columns = ['Date', 'Time', 'Time_10sec', 'Time_min','Time_10min', 'Count','Score']


#%% 尝试画图
for d in Date:    
    #### 筛选出高频时段
    dft = df2[df2['Date'] == d]
    datet = d.replace('/','-')
    df_10min = dft.groupby('Time_10min').sum()['Count']
    temp = []
    for k in range(len(df_10min)):
        if df_10min[k] > 500:
            temp.append(df_10min.index[k])
    if not temp:
        continue
    temp.sort()
    dft = dft[dft['Time_10min'] >= temp[0]]
    dft = dft[dft['Time_10min'] <= temp[-1]]
    
    
    #### 聚合数据
    summ = dft.groupby('Time_min').sum()
    fname = 'Summary on '+datet
    summ.to_csv('D:/NLP/Result1/'+fname+'.csv')
    df_count = summ['Count']
    df_score = summ['Score']
    
    
    #### 寻找峰值
    mn = df_count.mean()
    pos = []
    value = []
    pos0 = []
    value0 = []
    for i in range(len(df_count)):
        if df_count[i] > 3*mn:
            pos0.append(i)
            value0.append(df_count[i])
        else:
            if pos0:
                ind = value0.index(max(value0))
                pos.append(pos0[ind])
                value.append(value0[ind])
                pos0 = []
                value0 = []
                
    text = [df_count.index[i] for i in pos]
    
    
    #### 绘制图形
    pname = 'Plot Count'+fname[7:] 
    plt.figure(figsize = (40,15))
    plt.plot(df_count.index, df_count)
    plt.xticks([])
    plt.tick_params(labelsize=25)
    plt.xlabel('Time',{'size':25})
    plt.ylabel('Count',{'size':25})
    plt.title('Change of Count',{'size':40})
    for i in range(len(pos)):
        plt.text(pos[i], value[i], text[i], fontsize = 20)
    plt.savefig('D:/NLP/Plot1/'+pname+'.png')
    plt.close()
