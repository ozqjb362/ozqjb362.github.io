---
layout:     post
title:      Python爬虫实战：抓取CoinMarketCap历史TOP100数字货币行情并双轨存储到Excel与MongoDB
date:       2025-09-05
header-img: img/post-bg-desk.jpg
catalog: true
---

> 关键词：Python爬虫、CoinMarketCap历史数据、数字货币、数据采集、Excel、MongoDB

如果你是区块链研究员、量化程序员，或只是想留存一份“价值回忆”的数据爱好者，这条脚本将手把手带你用简单 100+ 行 Python 代码，爬取 CoinMarketCap **历史排行榜前100条数字货币行情记录**，同时写入 Excel 和 MongoDB，双轨留存，随时复盘。看完即可直接用在本地运行，可扩展可移植。  

👉 [立即掌握“0 封禁”抓取高频数据的最新黑科技](https://okxdog.com/)

---

## 1. 项目目标与思路拆解

| 目标 | 细节说明 |
|---|---|
| 抓取范围 | CoinMarketCap 历史页面每日 TOP100 市值排行 |
| 时间粒度 | 2013-04-28 起任意指定日期 |
| 存储形态 | Excel 表格 + MongoDB 集合 |
| 核心字段 | 排名、币种名、符号、流通市值、价格、24h 涨跌幅、供应总量、流通量 |
| 防封措施 | 随机 User-Agent + 间隔时间 + 不并发 |

---

## 2. 环境准备

```bash
pip install requests pyquery openpyxl pymongo fake-useragent
```

> 如果你尚未安装 MongoDB，先在本地 [快速配置](https://docs.mongodb.com/manual/installation)。

---

## 3. 依赖统一导入（代码顶部）

```python
import requests, random, time, re, os
from pyquery import PyQuery as pq
from fake_useragent import UserAgent
from openpyxl import Workbook, load_workbook
import pymongo
```

---

## 4. 核心爬取流程

### 4.1 获取「历史列表页」的所有日期

CoinMarketCap 会把每日快照折叠在 `/historical/{YYYYMMDD}/` 格式下。  
第一步：抓历史列表页，提取日期文字。

```python
def fetch_html(url):
    ua = UserAgent()
    headers = {'User-Agent': ua.random}
    resp = requests.get(url, headers=headers, timeout=20)
    return resp.text if resp.status_code == 200 else None

def extract_date_list(html):
    """返回 YYYYMMDD 列表"""
    doc = pq(html)
    return [re.search(r'/(\d{8})/$', a.attr('href')).group(1)
            for a in doc('.dropdown__list a[href*="historical"]').items()]
```

### 4.2 单日详情页解析

历史页是 100 行表格，DOM 结构稳定，直接用 CSS 选择器定位：

| 所需列索引 | 数据含义 |
|:--|:--|
| 0 | 排名 |
| 1 | 名称 |
| 2 | 符号 |
| 3 | 市值 |
| 4 | 价格 |
| 5 | 24h % |
| ... | ... |

```python
def parse_daily(date_str):
    url = f'https://coinmarketcap.com/zh/historical/{date_str}/'
    html = fetch_html(url)
    doc = pq(html)
    rows = doc('table tbody tr')
    data_list = []
    for tr in rows.items():
        tds = tr('td')
        rank = tds.eq(0).text().strip()
        name = tds.eq(1)('.kMJABh').text()
        symbol = tds.eq(2).text()
        market_cap = tds.eq(3)('span').attr('data-nosnippet') or ''
        price = tds.eq(4)('span').attr('data-nosnippet') or ''
        change_24h = tds.eq(5).text()
        circulating = tds.eq(6)('span').attr('data-nosnippet') or ''
        data_list.append({
            'date': f'{date_str[:4]}-{date_str[4:6]}-{date_str[6:]}',
            'rank': int(rank),
            'name': name,
            'symbol': symbol,
            'market_cap': market_cap,
            'price': price,
            'change_24h': change_24h,
            'circulating': circulating
        })
        if len(data_list) >= 100:
            break
    return data_list
```

---

## 5. 数据双轨存储

### 5.1 写入 Excel（增量追加）

```python
def write_excel(records, path='crypto_history.xlsx'):
    if os.path.exists(path):
        wb = load_workbook(path)
        ws = wb.active
    else:
        wb = Workbook()
        ws = wb.active
        ws.append(['日期', '排名', '名称', '符号', '市值USD', '价格USD',
                   '24h涨跌%', '流通量'])
    for r in records:
        ws.append([r['date'], r['rank'], r['name'], r['symbol'],
                   r['market_cap'], r['price'], r['change_24h'],
                   r['circulating']])
    wb.save(path)
```

### 5.2 写入 MongoDB（去重 upsert）

```python
client = pymongo.MongoClient('mongodb://localhost:27017/')
db = client['crypto']
col = db['top100_history']

def save_mongo(records):
    for r in records:
        col.update_one({'date': r['date'], 'symbol': r['symbol']},
                       {'$set': r}, upsert=True)
```

---

## 6. 主循环：可持续、可中断

```python
TARGET_DATES = ['20240101', '20241231']  # 自行设定范围

for date in TARGET_DATES:
    print(f'>>> 抓取 {date} ...')
    data = parse_daily(date)
    write_excel(data)
    save_mongo(data)
    time.sleep(random.uniform(3, 6))  # 信封式节奏
```

运行后，Excel 表格在本地 `crypto_history.xlsx` 逐行追加，MongoDB 集合也同步 upsert。后续改为定时任务 `cron`，每日凌晨抓前一天数据，可持续一年。

---

## 7. 案例演示：Excel 结果示意

| 日期       | 排名 | 名称     | 符号 | 市值USD        | 价格USD | 24h涨跌% | 流通量         |
|:-----------|:-----|:---------|:-----|:---------------|:--------|:---------|:---------------|
| 2024-01-01 | 1    | Bitcoin  | BTC  | 8.48 × 10¹¹    | 44713   | +5.2     | 19.57M BTC     |
| 2024-01-01 | 2    | Ethereum | ETH  | 2.68 × 10¹¹    | 2289    | +3.7     | 120.50M ETH    |
| ……         | …    | …        | …    | …              | …       | …        | …              |

---

## 8. 常见问题 FAQ

> **Q1**：重跑脚本会重复插入 MongoDB 吗？  
> **A1**：不会。代码使用 `upsert=True`，以日期+币符号为唯一索引，自动去重。  

> **Q2**：访问过于频繁会被封 IP 吗？  
> **A2**：示范代码中每次间隔 3~6 秒并随机切换 UA，已足以规避常见封禁，若仍触发，可在 Header 里再加 `Accept-Language`、`Sec-Fetch-Mode` 等。  

> **Q3**：Excel 一旦打开就不能追加？  
> **A3**：openpyxl 会在保存时覆盖不了已打开文件，建议在脚本外部先关闭，或先写入临时文件再 rename。  

> **Q4**：如何定时抓取？  
> **A4**：Linux/macOS 可写 crontab：`0 3 * * * /usr/bin/python3 fetch_cmc.py`。  

> **Q5**：想用 DateFrame 分析？  
> **A5**：直接 `pip install pandas`，把 `records` 喂给 `pd.DataFrame`，一键绘图。  

> **Q6**：只想保存新增的币怎么办？  
> **A6**：在进入 MongoDB 前对比昨日快照，取差集，逻辑 5 行内搞定。  

---

## 9. 延伸：用脚本做“定投日历”

将历史 365 天价格做归一化，计算每周定投收益，可视化出季度策略盈亏曲线，再对接通知机器人，自动提示“下次买入点”。这一套组合拳，你已拥有全部基础数据与存储格式。

👉 [立即动手创建专属量化脚本模板](https://okxdog.com/)

---

## 10. 注意事项 & 最佳实践

- 不要一次拉取 **上千条历史页**，可先 prototype 10 条语句测试，确认字段稳定。  
- `fake_useragent` 有时离线拉 FailBack，建议本地缓存 UA 列表文件，以免部署失效。  
- CoinMarketCap 频繁改版，解析逻辑每半年复盘一次，确保选择器 still 工作。  
- 若需 **WebSocket 实时行情**，再接入官方 API Pro，与静态历史数据对比校正。

---

至此，一条从“拿网址”到“存数据库”再到“全局复盘”的完整工作流就搭建完成了。愿你把这条脚本当作积木，奠基石是你手里的历史数据，目标则是无限想象的未来。