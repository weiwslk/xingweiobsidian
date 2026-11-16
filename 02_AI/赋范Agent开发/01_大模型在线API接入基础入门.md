
## 基础概念
什么是Token
大模型处理文本的基本单位，1token = 1-2汉字

红楼梦 约73万汉字，大于50-70万token
DeepSeek V3 大于1-2元， GPT-5 大约60+元

API-Key 你的数字通行证
身份识别、权限控制、计费绑定

SDK：开发者的万能遥控器

Key放哪儿？ 环境变量.env
.env
```
apikey = os.getenv("DEEPAPIKEY")
```

## 平台快速接入

OpenRouter：模型界的超界中介，可以直接使用OpenAI、Anthropic、Google、Meta。正规聚合平台，官方直连，价格透明，长期稳定

DeepSeek：AGI普惠，V3.2 1元/百万Token

阿里百炼：Qwen，Qwen3-Max， Qwen-Plus

智普AI：GLM全能助手；GLM-4.7， GLM-4.6V

![[Pasted image 20261108150215.png]]

![[Pasted image 20261108151552.png]]

![[Pasted image 20261108152523.png]]


## OpenAI Chat Compeletion 的基础使用

API的基础使用

```


# 计算token

```

理解消息结构：三角色消息
- system，系统角色，用于设定AI的行为规范、角色定位、回复风格等，用户看不到，但会影响整个对话的基调
- user，用户角色，代表人类的提问或输入
- assistant，助手角色，代表AI的回复，在构造多轮对话时，需要手动添加历史回复。
```
response = client.chat.completions.create(
  model = "deepseek-chat",
  message = [
      {"role":"system", "content":"你是一位专业的Python编程导师，擅长用简洁明了的语言解释复杂概念"},
      {"role":"user","content":"什么是列表推导式"}
  ]
)

print(response.choice[0].message.content)
```

理解response对象
```
print("模型名称：{response.model}")
print("模型id：{response.id}")
print("创建时间：{response.created}")
...
```

核心参数调优：temperature 与max_tokens

temperature：控制模型输出的随机性，取值范围0-2
- 0： 输出最稳定，每次运行结果几乎相同
- 0.7（默认值），平衡了创造性
- 1.5，

max_tokens:控制输出长度
只控制上线，模型可能生成更短的内容，达到上限时截断

```
response = client.chat.completions.create(
  model = "deepseek-chat",
  message = [
      {"role":"system", "content":"你是一位专业的Python编程导师，擅长用简洁明了的语言解释复杂概念"},
      {"role":"user","content":"什么是列表推导式"}
  ],
  max_tokens = 200,
  temperature = 0.7,
  stream = True
)

```

流式输出：stream = True

```
stream = client.chat.completions.create(
  model = "deepseek-chat",
  message = [
      {"role":"system", "content":"你是一位专业的Python编程导师，擅长用简洁明了的语言解释复杂概念"},
      {"role":"user","content":"什么是列表推导式"}
  ],
  max_tokens = 200,
  temperature = 0.7,
  stream = True
)

for chunk in stream:
    delta_content = chunk.choices[0].delta.content
    
    if delta_content:
        print(delta_content, end="", flush=True) # 实时打印

```


![[Pasted image 20261108155243.png]]


## Chat Completions API进阶使用

### 多轮对话
```
convesion_history = [
    []
]

client = OpenAI(
	
)

# 每次提问和回答都加入 history列表
```

上下文长度管理：滑动窗口与摘要压缩
history会越来越长，消耗的token会急剧增加
- 滑动窗口：只取最近几轮的消息

### Function Calling：让AI调用工具
- 定义工具：告诉AI你有哪些函数可以调用，每个函数的参数是什么
- 第一次调用API
- 检查响应
- 执行函数
- 第二次调用API
- AI生成最终回复

```
def get_weather():
	pass

tools = [
	{
		"type":"function",
		"function":{
			"name":"get_weather",
			"description":""
		}
	}
]

response = client.chat.completions.create(
	...
	tools = tools,
)
```

![[Pasted image 20261108160419.png]]

### 多模态输入：让AI看图说话

```

response = client.chat.completions.create(
	...
	message = [
		{
			"role":"user",
			"content":[
				{"type":"text", "text":"这张图片是什么"},
				{
					"type":"image_url",
					"image_url":{"url":image_url}
				}
			]
		}
	],
	max_tokens = 500
)


```


### 多模态图片生成


## 提示词工程：写出高质量Prompt

- 明确角色
- 要求格式化输出
- 少样本学习， Few-Shot learning
- 链式思考
- 异步执行


### Responses API

面向Agentic的API

![[Pasted image 20261108161745.png]]


## 综合实战：LLM Playgrand

