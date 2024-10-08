# LLMs 信息

这个项目里目前就俩 JSON 文件，持续更新ing：
- `models_info.json`: 包含不同大模型的信息，如品牌、价格（每百万token）和最大上下文长度。
- `companies_info.json`: 包含提供这些模型的公司的信息，包括用于 OpenAI SDK 的 base_url 

你可以通过 requests 的方式，直接拉取这些信息。

目前包括以下几家公司的最新模型，持续更新ing：
- **百川 (Baichuan)**
- **深度求索 (DeepSeek)**
- **月之暗面 (Moonshot)**
- **OpenAI**
- **通义千问 (Qwen)**
- **阶跃星辰 (StepFun)**
- **智谱 (Zhipu)**

（先选择选择这些，是因为它们都支持 OpenAI SDK）
另："max_context_length" 和 "max_output_length" 这俩值可能有问题，有些我给的 0，是因为官网没有找到，我需要进一步手动测试。


## 使用方法

### 获取 JSON 数据

要从提供的URL获取并解析JSON数据，可以使用以下示例代码（colab）：

```python
import requests
import json

def fetch_json_from_url(url):
  response = requests.get(url)
  
  # Check if the request was successful
  if response.status_code == 200:
      # Parse the JSON content
      return json.loads(response.text)
  else:
      print(f"Failed to fetch data. Status code: {response.status_code}")
      return None

models_info = fetch_json_from_url("https://raw.githubusercontent.com/CocoSgt/LLMs_info/main/models_info.json")
companies_info = fetch_json_from_url("https://raw.githubusercontent.com/CocoSgt/LLMs_info/main/companies_info.json")
```

### 示例：基于这个 JSON，封一个请求

以下示例代码展示了如何调用不同的语言模型，获取模型的地址、价格和其他信息，并在项目中使用：
(假定使用 colab，并将 key 以 Key_xxx 的方式存放)
```python
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
from openai import OpenAI
import queue
import time
import datetime
import pytz
from google.colab import userdata

def LLM(messages, model_name, models_info=models_info, companies_info=companies_info, params={}, stream=False):
    # 获取模型信息
    model = models_info.get(model_name)
    if not models_info:
        raise ValueError(f"Model {model_name} not found")

    model_config = companies_info.get(model['brand'])

    if not model_config:
        raise ValueError(f"Configuration for brand {model['brand']} not found")

    client = OpenAI(
        api_key=userdata.get(f"Key_{model['brand']}"),
        base_url=model_config['base_url'],
    )

    api_params = {
        "model": model_name,
        "messages": messages,
        "stream": stream,
    }

    if stream:
        api_params["stream_options"] = {"include_usage": True}

    start_time = time.time()

    try:
        response = client.chat.completions.create(**{**api_params, **params})
        rsp = {
            'model': model_name,
            'content': "",
            'token': {
                'input': 0,
                'output': 0,
                'total': 0
            },
            'time': 0.0  # 初始化耗时字段
        }

        if stream:
            for chunk in response:
                if chunk.choices and chunk.choices[0].delta.content is not None:
                    # print(chunk)
                    chunk_content = str(chunk.choices[0].delta.content)
                    print(chunk_content, end='')
                    rsp['content'] += chunk_content
                if chunk.usage:
                    rsp['token']['input'] = chunk.usage.prompt_tokens
                    rsp['token']['output'] = chunk.usage.completion_tokens
                    rsp['token']['total'] = chunk.usage.total_tokens
        else:
            rsp['content'] = response.choices[0].message.content
            rsp['token']['input'] = response.usage.prompt_tokens
            rsp['token']['output'] = response.usage.completion_tokens
            rsp['token']['total'] = response.usage.total_tokens

        end_time = time.time()
        rsp['time'] = end_time - start_time

        return rsp
    except Exception as e:
        print(e)

def cost(rsp, models_info=models_info):
    rsp['cost'] = {
                'input': 0,
                'output': 0,
                'total': 0,
                'currency': ''
            }
    rsp['cost']['input'] = models_info[rsp['model']]['input_cost'] * rsp['token']['input'] / 1_000_000
    rsp['cost']['output'] = models_info[rsp['model']]['output_cost'] * rsp['token']['output'] / 1_000_000
    rsp['cost']['total'] = rsp['cost']['input'] + rsp['cost']['output']
    rsp['cost']['currency'] = models_info[rsp['model']]['currency']

    print(f"""{rsp['model']}:
返回：{rsp['content']}

token：
输入：{rsp['token']['input']}
输出：{rsp['token']['output']}
合计：{rsp['token']['total']}
耗时：{round(rsp['time'],2)} 秒
输出速度（忽略网络延迟 & 理解输入）：
{round(rsp['token']['total']/rsp['time'], 2)} token/s

千次开销: {format(1000 * rsp['cost']['total'], '.5g')} {rsp['cost']['currency']}""")
    if rsp['cost']['currency'] == "USD":
        print(f"千次折合人民币：{format(1000 * 7.27 * rsp['cost']['total'], '.3g')} CNY")
    print()
    return rsp
```



### 调用请求 & 计算成本
```python
messages = [
    {
        "role": "system",
        "content": "You are a helpful assistant."
    },
    {
        "role": "user",
        "content": "Hi~"
    }
]

rsp = LLM(messages=messages, model_name='Baichuan4', stream=True, params={'max_tokens':1024})
```

打印信息如下
```python
gpt-4o-mini:
返回：Hello! How can I assist you today?
          
token：
输入：19
输出：9
合计：28    
耗时：0.5 秒
输出速度（忽略网络延迟 & 理解输入）：
55.74 token/s

千次开销: 0.00825 USD
千次折合人民币：0.06 CNY
```

并且，rsp 为
```python
{'model': 'gpt-4o-mini',
 'content': 'Hello! How can I assist you today?',
 'token': {'input': 19, 'output': 9, 'total': 28},
 'time': 0.5023195743560791,
 'cost': {'input': 2.8500000000000002e-06,
  'output': 5.399999999999999e-06,
  'total': 8.249999999999999e-06,
  'currency': 'USD'}}
```

也可以像这样批量测
```python
for key in models_info.keys():
  try:
    print(key)
    rsp = LLM(messages=messages, model_name=key, stream=True, params={'max_tokens':1024})
    print(rsp)
    print("\n\n")
  except:
    continue
```

## 其他

如果有任何建议或改进，请随时提交Pull Request或Issue，谢谢支持！


**GitHub 项目地址**: [LLMs_info](https://github.com/CocoSgt/LLMs_info)

**个人公众号**: 赛博禅心

![image](https://github.com/user-attachments/assets/0f699f3c-eae6-4d6a-b609-1a22cb06b4c4)

