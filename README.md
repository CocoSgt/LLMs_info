# LLMs 信息

这个项目包含了关于支持 OpenAI SDK 的各种语言模型和提供这些模型的公司的信息的JSON文件。项目还包括了示例代码，用于获取这些信息并在其他项目中使用。

## 支持的模型

目前，这个项目包含以下几家公司的最新一批语言模型：

- **百川 (Baichuan)**
- **深度求索 (DeepSeek)**
- **月之暗面 (Moonshot)**
- **OpenAI**
- **通义千问 (Qwen)**
- **阶跃星辰 (StepFun)**
- **智谱 (Zhipu)**

之所以选择这些模型，是因为它们都支持 OpenAI SDK，能够方便地集成到各类应用中去。

## 内容

- `models_info.json`: 包含不同语言模型的信息，如品牌、费用和最大上下文长度。
- `companies_info.json`: 包含提供这些模型的公司的信息，如基础URL和中文名称。

## 使用方法

### 获取JSON数据

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
companies_info = fetch_json_from_url("https://raw.githubusercontent.com/CocoSgt/LLMs_info/main/companies_info.json")```

### 示例：调用语言模型

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
                if chunk.choices:
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

        # rsp['cost']['input'] = models_info[rsp['model']]['input_cost'] * rsp['token']['input'] / 1_000_000
        # rsp['cost']['output'] = models_info[rsp['model']]['output_cost'] * rsp['token']['output'] / 1_000_000
        # rsp['cost']['total'] = rsp['cost']['input'] + rsp['cost']['output']
        # rsp['cost']['currency'] = models_info[rsp['model']]['currency']

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



调用这个模型 & 计算成本
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

rsp = LLM(messages=messages, model_name='gpt-4o-mini', params={'max_tokens':4096})
rsp = cost(rsp)
```


## 说明

这里的模型信息并不完全，仅包含了支持 OpenAI SDK 的各家公司最新一批模型。我会不断维护和更新这个项目，以确保信息的时效性和准确性。

## 贡献

欢迎大家参与到这个项目中来，提供更多的语言模型信息和改进代码。如果您有任何建议或改进，请随时提交Pull Request或Issue，一起维护和完善这个项目。谢谢支持！

---

**GitHub 项目地址**: [LLMs_info](https://github.com/CocoSgt/LLMs_info)
**我的公众号**: 赛博禅心
