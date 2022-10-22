---
title: FOFA通用产品收集脚本
date: 2022-10-22 09:50:45
categories: 工具
tags: FOFA
---

## 0x00 前言

本项目是对项目：https://github.com/RaiderZP/cnvd_fofa_gather 进行的修改，更改为使用FOFA官方提供的查询接口。本项目去除了原项目中的代理池功能，通过设置5秒的访问间隔时间来保证查询的成功率。

## 0x01 简介

通过公司名称，在fofa上搜索可能存在通用产品的公司，原理是判断网站标题数目以及独立IP数达到一定条件时将该标题以及公司名称导出；如果想挖掘cnvd证书，可导出注册资金大于5000w的公司到这个脚本中进行通用系统收集。

## 0x02 使用方法

修改脚本第10、11行为你的FOFA账号的邮箱和API KEY

```python
# 填入fofa账号的email和API_KEY
email = 'YOUR_EMAIL'
api_key = 'API_KEY'
```

将公司名称放入`gs.txt`文件中，执行该脚本即可。

```python
python3 new_cnvd_fofa_gather.py
```

## 0x03 项目地址

https://github.com/colind0pe/new_cnvd_fofa_gather

## 0x04 脚本代码

 ```python
 import base64
 import json
 import re
 import time
 import requests
 
 requests.packages.urllib3.disable_warnings()
 
 # 填入fofa账号的email和API_KEY
 email = 'YOUR_EMAIL'
 api_key = 'API_KEY'
 
 # 设置请求头
 headers = {
     'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36'
 }
 
 
 # 调用fofa api的统计聚合接口
 def fofa_search(kjgs):
     fofa_api = 'https://fofa.info/api/v1/search/stats??email={}&key={}&fields=title&qbase64='.format(email,api_key)
     keyword = '"' + kjgs + '"'
     bs64_keyword = str(base64.b64encode(keyword.encode("utf-8")), "utf-8")
     url = fofa_api + bs64_keyword
 
     result = requests.get(url, headers=headers, verify=False, timeout=10)
     time.sleep(5)   # fofa限制请求速率，设置请求间隔为5秒
     json_result = json.loads(result.content)
     ip_count = json_result['distinct']['ip']
     if ip_count < 20:
         print(kjgs + " ---> 独立IP总数：" + str(ip_count))
         with open(r'result.txt', 'a+') as f:
                 f.write(kjgs + " ---> 独立IP总数：" + str(ip_count))
                 f.write('\n')
                 f.write('\n')
                 f.close()
     else:
         # 独立IP总数不为0时获取对应的标题榜首和标题对应数量
         title = json_result['aggs']['title'][0]['name']
         title_count = json_result['aggs']['title'][0]['count']
         print(kjgs + " ---> 独立IP总数：" + str(ip_count) + " ---> 标题榜首：" + title + " ---> 标题对应数：" + str(title_count))
         with open(r'result.txt', 'a+') as f:
                 f.write(kjgs + " ---> 独立IP总数：" + str(ip_count) + " ---> 标题榜首：" + title + " ---> 标题对应数：" + str(title_count))
                 f.write('\n')
                 f.write('\n')
                 f.close()
         
 
 
 if __name__ == '__main__':
     # 打开公司列表，获取公司名称
     print("开始收集--------")
     for f in open('gs.txt', 'rb'):
         gs = str(f, "utf-8")
         gs = gs.strip()
 
         # 去除科技、技术、股份、有限公司等字符
         try:
             if re.search(r'科技', gs):
                 start = re.search(r'科技', gs).span()[0]
                 kj = gs[:start]
 
                 # 去除括号内容
                 if '(' in kj:
                     start = re.search(r'\(', kj).span()[0]
                     end = re.search(r'\)', kj).span()[1]
 
                     kj_last = kj.replace(kj[start:end], '')
 
                     fofa_search(kj_last)
                 else:
                     fofa_search(kj)
 
             elif re.search(r'技术', gs):
                 start = re.search(r'技术', gs).span()[1]
                 kj = gs[:start]
                 if '(' in kj:
                     start = re.search(r'\(', kj).span()[0]
                     end = re.search(r'\)', kj).span()[1]
 
                     kj_last = kj.replace(kj[start:end], '')
 
                     fofa_search(kj_last)
                 else:
                     fofa_search(kj)
 
             elif re.search(r'软件', gs):
                 start = re.search(r'软件', gs).span()[1]
                 kj = gs[:start]
                 if '(' in kj:
                     start = re.search(r'\(', kj).span()[0]
                     end = re.search(r'\)', kj).span()[1]
 
                     kj_last = kj.replace(kj[start:end], '')
 
                     fofa_search(kj_last)
                 else:
                     fofa_search(kj)
 
             elif re.search(r'股份', gs):
                 start = re.search(r'股份', gs).span()[0]
                 kj = gs[:start]
                 if '(' in kj:
                     start = re.search(r'\(', kj).span()[0]
                     end = re.search(r'\)', kj).span()[1]
 
                     kj_last = kj.replace(kj[start:end], '')
 
                     fofa_search(kj_last)
                 else:
                     fofa_search(kj)
 
             elif re.search(r'有限', gs):
                 start = re.search(r'有限', gs).span()[0]
                 kj = gs[:start]
                 if '(' in kj:
                     start = re.search(r'\(', kj).span()[0]
                     end = re.search(r'\)', kj).span()[1]
 
                     kj_last = kj.replace(kj[start:end], '')
 
                     fofa_search(kj_last)
                 else:
                     fofa_search(kj)
 
             else:
                 if '(' in gs:
                     start = re.search(r'\(', gs).span()[0]
                     end = re.search(r'\)', gs).span()[1]
 
                     gs_last = gs.replace(gs[start:end], '')
 
                     fofa_search(gs_last)
                 else:
                     kj = gs
                     fofa_search(kj)
         except Exception as u:
             print('main_err:', u)
 ```
