# Function Calling of LLM

## 定义

- 函数调用是一种将 LLM 与外部工具连接起来的能力，可实现有效的工具使用以及与外部 API 的交互。

- GPT-4 和 GPT-3.5 等微调过的LLM可以检测何时需要调用函数，然后输出包含参数的 JSON 来调用函数。

- 函数调用是构建由 LLM 驱动的聊天机器人或代理的一项重要能力，它通过将自然语言转换为 API 调用来检索 LLM 的上下文或与外部工具进行交互。

## 工作流程

假定我们提供了一个根据位置获取天气的函数并，当我们尝试与它交互时会自动决定是否调用函数，如果需要调用函数则返回参数及参数值。

然后我们根据返回的结果调用函数并将函数的返回值再次提供给LLM，最后由LLM生成最终结果。
![Function Calling](https://cdn.openai.com/API/docs/images/function-calling-diagram-steps.png)

### 下面是ChatGPT官方的一个例子演示如何工作的
```python
import json
from openai import OpenAI
# 创建工具
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "获取指定城市的天气",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": {"type": "string", "description": "城市名称"},
                }
            },
            "required": [
                "location"
            ],
            "additionalProperties": False
        },
        "strict": True
    }
}]

def get_weather(location: str):
    return "晴天"

#创建LLM client
client = OpenAI(
)

# 用户提问
user_query = "北京今天天气如何？"
messages = [{"role": "user", "content": user_query}]
completion = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=tools,
)
# 模型决定调用哪个函数，并返回函数内容和参数
tool_call = completion.choices[0].message.tool_calls[0]
args = json.loads(tool_call.function.arguments)

# 我们手动执行函数
result = get_weather(args["location"])

# 将函数调用信息和结果添加到消息列表中
messages.append(completion.choices[0].message)  # append model's function call message
# 将执行结果添加到消息列表中
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,
    "content": str(result)
})
# 将结果纳入其输出
completion_2 = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=tools,
)
print(completion_2.choices[0].message.content)
```


## 代码示例

### 1. 今天穿什么？

在这个例子中，我定义了一个穿衣代理，并提供以下两个函数：

1. `search_temperature(location)`: 根据位置查询气温。

2. `search_clothes(temperature)`: 根据气温查询穿什么。

当我询问穿衣代理“在大连我今天应该穿什么？”时，代理将执行以下操作：

1. 需要调用函数 `search_temperature`，传入位置参数“大连”。

2. 执行函数并返回温度为2。

3. 进一步调用 `search_clothes` 函数，将上述函数的输出作为参数。

4. 执行函数并返回“毛呢大衣”作为穿衣建议。

5. 提供最终答案。


``` python
# 创建LLM
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini")

# 定义LLM可以使用的函数（工具）
from langchain_core.tools import tool
@tool
def search_temperature(location: str):
    """Search temperature according to the location.
    Args:
       location: The location to search for temperature.
    """
    if location == "大连":
        return 2
    return 21
@tool
def search_clothes(temperature: int):
    """Search for clothes to wear according to the temperature.
    Args:
         temperature: The temperature to search for clothes.
    """
    if (temperature < 10):
        return f"毛呢大衣."
    return f"T恤."
tools = [search_temperature, search_clothes]

# 创建Agent
from langchain_core.messages import HumanMessage
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import MemorySaver
prompt="""
You are given a number of tools as functions, you must use these tools and then give a final response.
"""
checkpointer = MemorySaver()
agent_executor = create_react_agent(llm, tools, prompt=prompt, checkpointer=checkpointer)

# 运行Agent
config = {"configurable": {"thread_id": 1}}
for chunk in agent_executor.stream(
        {"messages": [HumanMessage(content="在大连我今天应该穿什么?")]}, config
):
    print(chunk)
    print("----")

# 运行结果
{'agent': {'messages': [AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_jWNKJFDYBKQzD9LCwLeynyWs', 'function': {'arguments': '{"location":"大连"}', 'name': 'search_temperature'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 15, 'prompt_tokens': 123, 'total_tokens': 138, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini', 'system_fingerprint': 'fp_b705f0c291', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-b7ef0ef6-406f-4e7e-bb95-de83a389dc50-0', tool_calls=[{'name': 'search_temperature', 'args': {'location': '大连'}, 'id': 'call_jWNKJFDYBKQzD9LCwLeynyWs', 'type': 'tool_call'}], usage_metadata={'input_tokens': 123, 'output_tokens': 15, 'total_tokens': 138, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}}
----
{'tools': {'messages': [ToolMessage(content='2', name='search_temperature', id='76d66230-f559-4c2f-a22b-1ed904646e65', tool_call_id='call_jWNKJFDYBKQzD9LCwLeynyWs')]}}
----
{'agent': {'messages': [AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_MM8rzxGLRyZzy4VHiqH3aOxs', 'function': {'arguments': '{"temperature":2}', 'name': 'search_clothes'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 15, 'prompt_tokens': 147, 'total_tokens': 162, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini', 'system_fingerprint': 'fp_b705f0c291', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-bb2d69e7-015a-4fdd-b739-dd03ed7644f7-0', tool_calls=[{'name': 'search_clothes', 'args': {'temperature': 2}, 'id': 'call_MM8rzxGLRyZzy4VHiqH3aOxs', 'type': 'tool_call'}], usage_metadata={'input_tokens': 147, 'output_tokens': 15, 'total_tokens': 162, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}}
----
{'tools': {'messages': [ToolMessage(content='毛呢大衣.', name='search_clothes', id='38d3bfdb-d2e3-4628-9fb7-4d8a8643b530', tool_call_id='call_MM8rzxGLRyZzy4VHiqH3aOxs')]}}
----
{'agent': {'messages': [AIMessage(content='\n当前大连的温度为2℃，您应该穿毛呢大衣。', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 322, 'total_tokens': 336, 'completion_tokens_details': None, 'prompt_tokens_details': None}, 'model_name': 'THUDM/glm-4-9b-chat', 'system_fingerprint': '', 'finish_reason': 'stop', 'logprobs': None}, id='run-c3537d17-f5ed-49ec-9962-c743d2550fa8-0', usage_metadata={'input_tokens': 322, 'output_tokens': 14, 'total_tokens': 336, 'input_token_details': {}, 'output_token_details': {}})]}}
----
```
### 2. 文件管家

在这个例子中，使用了langchain提供的文件管理工具库，可以基于一个工作目录进行文件操作。

当我让代理将一句话写入文件时，代理会创建文件并将内容写入其中。（注：也可以让代理删除文件）

```python


# 创建LLM
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini")

# 定义LLM可以使用的函数（工具）
from langchain_community.agent_toolkits import FileManagementToolkit
from tempfile import TemporaryDirectory
import os
current_directory = os.getcwd()
working_directory = TemporaryDirectory()
toolkit = FileManagementToolkit(
    root_dir=str(current_directory+"/temp")
)  # If you don't provide a root_dir, operations will default to the current working directory
tools = toolkit.get_tools()

# 创建Agent
from langchain_core.messages import HumanMessage
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import MemorySaver
prompt="""
You are given a number of tools as functions, you must use these tools and then give a final response in chinese.
"""
checkpointer = MemorySaver()
agent_executor = create_react_agent(llm, tools, prompt=prompt, checkpointer=checkpointer)

# 运行Agent
config = {"configurable": {"thread_id": 15}}
for chunk in agent_executor.stream(
        {"messages": [HumanMessage(content="""
        把下面这段话写入文件a.txt中：
        '今天天气真好！！'
        """)]}, config
):
    print(chunk)
    print("----")


# 运行结果
{'agent': {'messages': [AIMessage(content='', additional_kwargs={'tool_calls': [{'id': '019537329bbc89a972f73bce8ba69ce3', 'function': {'arguments': '{"file_path": "a.txt", "text": "今天天气真好！！"}', 'name': 'write_file'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 20, 'prompt_tokens': 1019, 'total_tokens': 1039, 'completion_tokens_details': None, 'prompt_tokens_details': None}, 'model_name': 'THUDM/glm-4-9b-chat', 'system_fingerprint': '', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-c4b1c131-fae5-49d5-96a1-64ae0802f1fa-0', tool_calls=[{'name': 'write_file', 'args': {'file_path': 'a.txt', 'text': '今天天气真好！！'}, 'id': '019537329bbc89a972f73bce8ba69ce3', 'type': 'tool_call'}], usage_metadata={'input_tokens': 1019, 'output_tokens': 20, 'total_tokens': 1039, 'input_token_details': {}, 'output_token_details': {}})]}}
----
{'tools': {'messages': [ToolMessage(content='File written successfully to a.txt.', name='write_file', id='13d9ece7-651d-481a-8ca4-76595916b6bc', tool_call_id='019537329bbc89a972f73bce8ba69ce3')]}}
----
{'agent': {'messages': [AIMessage(content='\n已经将"今天天气真好！！"这段话写入到文件a.txt中。', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 18, 'prompt_tokens': 1030, 'total_tokens': 1048, 'completion_tokens_details': None, 'prompt_tokens_details': None}, 'model_name': 'THUDM/glm-4-9b-chat', 'system_fingerprint': '', 'finish_reason': 'stop', 'logprobs': None}, id='run-4a57e0da-f83d-4263-a35f-07d2b2b9e957-0', usage_metadata={'input_tokens': 1030, 'output_tokens': 18, 'total_tokens': 1048, 'input_token_details': {}, 'output_token_details': {}})]}}
----
```

## Function calling和Tool Calling的区别

Function Calling是早期ChatGPT的概念，在调用API的时候提供参数functions，但是在一次对话中只能调用一个函数，现在已经过时。
```python
completion = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": user_query}],
    functions=functions,
    function_call="auto"
)
```
Tool Calling是Function calling的基础上进行增强，可以调用多个函数，同时不仅限于function，参数由functions改成了tools，不过官方文档的标题还是叫做Function calling。
```python
completion = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": user_query}],
    tools=tools,
    tool_choice="auto",
)
```

> 参考 [Anthropic的官方文档标题Tool use (function calling)](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)，个人理解得他们现在在各个大模型中叫法不同，但是本质上是一样的。
>
> 参考 [From Function Calls to Tool Calls: A Comprehensive Guide](https://www.tasking.ai/blog/from-function-calls-to-tool-calls-a-comprehensive-guide)
> <b>Function Calls</b>
>
> Using function_calls, developers could directly interact with the AI to perform functions, but <b>this approach had limitations, especially in handling parallel function calls. For instance, within a single chat completion request, it is not feasible to query multiple stock prices simultaneously, as only one tool function can be called at a time</b>. This method is now marked as deprecated on OpenAI's official website.
> <b>Tool Calls</b>
>
> With the introduction of tool_calls, OpenAI provided a more robust framework enabling more complex integrations and interactions.
> - Parallel Function Calling: This feature <b>allows the model to execute multiple function calls simultaneously</b>. For instance, the model can query the weather in three different locations at once. Each function call is managed independently within the tool_calls array, and responses are handled in parallel, reducing round trips and wait times.
> - Integration Flexibility: The structure of <b>tool_calls is designed to support a wide variety of tools, not limited to functions</b>. This design allows for future expansions and integrations of different tool types, enhancing the model's adaptability and application scope.
