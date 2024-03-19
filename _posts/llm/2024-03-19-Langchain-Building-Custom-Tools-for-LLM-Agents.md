---
title: "Building Custom Tools for LLM Agents: LLM Agent를 위한 커스텀 툴 제작하기"
date: 2024-03-18 12:28
categories:
  - ML
  - LLM
tags:
  - ML
  - LLM
  - LangChain
---
> 본 글은 Pinecone의 [Building Custom Tools for LLM Agents | Pinecone](https://www.pinecone.io/learn/series/langchain/langchain-tools/) 를 참고하여 작성하였습니다. 대부분의 코드가 현재 작동하지 않는 옛날 코드라 코드부분을 주로 최신화 하였습니다.

[Agent](https://python.langchain.com/docs/modules/agents/) 는 LLM에서 가장 파워풀하고 매력적인 접근 방식 중 하나이고 이러한 관심이 다양한 AI의 유즈케이스를 만들어냈습니다. Agent가 여러 툴에 접근할 수 있도록 하면서 무한한 가능성을 부여받았습니다. 툴을 통해 LLM은 검색, 계산, 코드 실행, 그리고 그 이상을 할 수 있죠

LangChain에선 잘 만들어진 툴도 제공을 하지만 실무에선 그 이상의 요구사항이 너무나 많습니다. 즉 우리 만의 커스텀한 툴이 필요하다는 말입니다.

이번 챕터에서는 커스텀 툴을 어떻게 만들고 랭체인에서 사용할 수 있을지 알아볼 것입니다. 간단한 예시부터 좀 더 복잡한 예시까지 다루며 어떻게 ML모델에게 더 많은 능력을 부여할 수 있을지 알아봅시다.

## LangChain에서 설명하는 Tool

랭체인에서는 Tool을 아래 사항들을 포함하는 인터페이스로 에이전트가 실세계와 상호작용할 수 있도록 하는 목적이라고 설명하고 있습니다.

1. The name of the tool (이름)
2. A description of what the tool is (설명)
3. JSON schema of what the inputs to the tool are (Input JSON Schema)
4. The function to call (실행할 Function)
5. Whether the result of a tool should be returned directly to the user (결과 유무)

- 특히 1~3번이 LLM이 Tool이 이해하는데 큰 도움이 됨
- 간단한 Input일수록 LLM이 더 쉽게 사용함, 대부분 하나의 String Input을 사용


## Defining Custom Tools

> https://python.langchain.com/docs/modules/agents/tools/custom_tools 발췌

- Agent에서 Custom Tool은 필요함
- Tool을 만드는 방법은 여러가지고 3가지 컴포넌트(이름, 설명, 인수 스키마)가 주로 필요
- 여기선 예시로 2가지 Tool를 만들어봄
	1. LangChain만 답변을 하는 검색(search) Tool
	2. 2가지 Int 인자를 곱하는 곱셈(multiply) Tool

```python
# 일반적으로 아래 모듈 import 필요
from langchain.pydantic_v1 import BaseModel, Field
from langchain.tools import BaseTool, StructuredTool, tool
```

### 2.1 @tool decorator
- 가장 간단한 방법으로 Function에 데코레이터를 추가함
	- `name`: 함수 이름으로 지정
	- `description`: 함수의 docstring이 활용되기 때문에 반드시 지정
	- `args_schema`: 함수의 parameter를 활용

```python
@tool
def search(query: str) -> str:
    """Look up things online."""
    return "LangChain"
```
```
search
search(query: str) -> str - Look up things online.
{'query': {'title': 'Query', 'type': 'string'}}
```

```python
@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```
```
multiply
multiply(a: int, b: int) -> int - Multiply two numbers.
{'a': {'title': 'A', 'type': 'integer'}, 'b': {'title': 'B', 'type': 'integer'}}
```

- Pydantic BaseModel를 활용해 스키마를 설정할 수 있음

```python
class SearchInput(BaseModel):
    query: str = Field(description="should be a search query")


@tool("search-tool", args_schema=SearchInput, return_direct=True)
def search(query: str) -> str:
    """Look up things online."""
    return "LangChain"
```

### 2.2 Subclass BaseTool

- BaseTool를 상속받아 서브클래스로 만들 수 있음
- 가장 최대한의 제어를 할 수 있지만 공수가 더 많이 듬
- 써보니 비동기 처리나 여러면에서 유리함

```python
from typing import Optional, Type

from langchain.callbacks.manager import (
    AsyncCallbackManagerForToolRun,
    CallbackManagerForToolRun,
)


class SearchInput(BaseModel):
    query: str = Field(description="should be a search query")


class CustomSearchTool(BaseTool):
    name = "custom_search"
    description = "useful for when you need to answer questions about current events"
    args_schema: Type[BaseModel] = SearchInput

    def _run(
        self, query: str, run_manager: Optional[CallbackManagerForToolRun] = None
    ) -> str:
        """Use the tool."""
        return "LangChain"

    async def _arun(
        self, query: str, run_manager: Optional[AsyncCallbackManagerForToolRun] = None
    ) -> str:
        """Use the tool asynchronously."""
        raise NotImplementedError("custom_search does not support async")

```

```python

class CalculatorInput(BaseModel):
    a: int = Field(description="first number")
    b: int = Field(description="second number")


class CustomCalculatorTool(BaseTool):
    name = "Calculator"
    description = "useful for when you need to answer questions about math"
    args_schema: Type[BaseModel] = CalculatorInput
    return_direct: bool = True

    def _run(
        self, a: int, b: int, run_manager: Optional[CallbackManagerForToolRun] = None
    ) -> str:
        """Use the tool."""
        return a * b

    async def _arun(
        self,
        a: int,
        b: int,
        run_manager: Optional[AsyncCallbackManagerForToolRun] = None,
    ) -> str:
        """Use the tool asynchronously."""
        raise NotImplementedError("Calculator does not support async")
```

```python
multiply = CustomCalculatorTool()
print(multiply.name)
print(multiply.description)
print(multiply.args)
print(multiply.return_direct)

# Calculator 
# useful for when you need to answer questions about math 
# {'a': {'title': 'A', 'description': 'first number', 'type': 'integer'}, 'b': {'title': 'B', 'description': 'second number', 'type': 'integer'}} True
```

### 2.3 StructuredTool dataclass

- 위 2가지를 섞은 방식인 `StructuredTool` 를 사용하여 만들 수 있음
- 간단하게 만들 수도 있고, BaseTool처럼 스키마 지정까지 복잡하게도 할 수 있음
- 다만 간단하게 쓸거면 @tool 데코레이터가 확실히 좋고, 굳이 BaseTool를 대체할만큼 간단한지도 의문

```python
def search_function(query: str):
    return "LangChain"


search = StructuredTool.from_function(
    func=search_function,
    name="Search",
    description="useful for when you need to answer questions about current events",
    # coroutine= ... <- you can specify an async method if desired as well
)
```

```python
class CalculatorInput(BaseModel):
    a: int = Field(description="first number")
    b: int = Field(description="second number")


def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b


calculator = StructuredTool.from_function(
    func=multiply,
    name="Calculator",
    description="multiply numbers",
    args_schema=CalculatorInput,
    return_direct=True,
    # coroutine= ... <- you can specify an async method if desired as well
)
```

### 2.4 Handling Tool Errors

- Tool에서 발생한 특수한 에러를 `ToolException` 으로 처리할 수 있음
- raise하면 agent는 작동을 멈출 수도 있고, `handle_tool_error` 를 어떻게 설정했느냐에 따라 agent가 그 다음을 처리할 수도 있음

```python
from langchain_core.tools import ToolException


def search_tool1(s: str):
    raise ToolException("The search tool1 is not available.")
    
search = StructuredTool.from_function(  
func=search_tool1,  
name="Search_tool1",  
description="A bad tool",  
)  
  
search.run("test")
```

> 위 예시로 실행하면 바로 Exception이 발생하면서 에러 종료

```python
def _handle_error(error: ToolException) -> str:
    return (
        "The following errors occurred during tool execution:"
        + error.args[0]
        + "Please try another tool."
    )


search = StructuredTool.from_function(
    func=search_tool1,
    name="Search_tool1",
    description="A bad tool",
    handle_tool_error=_handle_error,
)

search.run("test")
```

> 위 예시처럼 handle_error가 있다면 에러 메시지가 반환될 뿐 정상 종료


## How to use it

- Tool은 Agent와 같이 사용해야한다. LangChain에서는 여러가 Agent Type을 제공하지만 최근 OpenAI Functions Agent는 tool로 바뀌면서 deprecated되었음
- 즉  `OpenAI Tools Agent` 를 사용하면 됨

```python
from langchain.tools import BaseTool
from typing import Union


class CircumferenceToolInput(BaseModel):
    # radius: Union[int, float] = Field(description="the radius of a circle") -> int형 인자로 잘못 넘김
    radius: float = Field(description="the radius of a circle")


class CircumferenceTool(BaseTool):
    name = "CircumferenceCalculator"  # 'Circumference calculator' does not match '^[a-zA-Z0-9_-]{1,64}$' -
    description = "use this tool when you need to calculate a circumference using the radius of a circle"
    args_schema: Type[BaseModel] = CircumferenceToolInput
    # return_direct: bool = True #  Tools that have `return_direct=True` are not allowed in multi-action agents (type=value_error)

    def _run(
        self,
        radius: float,
        run_manager: Optional[CallbackManagerForToolRun] = None,
    ):
        """Use the tool."""
        """
        Add run_manager: Optional[CallbackManagerForToolRun] = None
        to child implementations to enable tracing,
        """
        print(f"Result of Tool: {radius} -> {float(radius) * 2.0 * pi}")
        return float(radius) * 2.0 * pi

    def _arun(
        self,
        radius: float,
        run_manager: Optional[AsyncCallbackManagerForToolRun] = None,
    ):
        raise NotImplementedError("This tool does not support async")
```

```python
from langchain import hub
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_openai import ChatOpenAI

# Get the prompt to use - you can modify this!
prompt = hub.pull("hwchase17/openai-tools-agent")
tools = [CircumferenceTool()]

agent = create_openai_tools_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
agent_executor.invoke({"input": "what is LangChain?"})
```

```python
agent_executor.invoke(
    {
        "input": "can you calculate the circumference of a circle that has a radius of 7.81mm"
    }
)
```

```python
> Entering new AgentExecutor chain... Invoking: `CircumferenceCalculator` with `{'radius': 7.81}` Result of Tool: 7 -> 43.982297150257104 43.982297150257104The circumference of a circle with a radius of 7.81 mm is approximately 43.98 mm. > Finished chain.

{'input': 'can you calculate the circumference of a circle that has a radius of 7.81mm', 'output': 'The circumference of a circle with a radius of 7.81 mm is approximately 43.98 mm.'}
```

```python
> Entering new AgentExecutor chain... Invoking: `CircumferenceCalculator` with `{'radius': 7.81}` Result of Tool: 7.81 -> 49.071677249072565 49.071677249072565The circumference of a circle with a radius of 7.81 mm is approximately 49.07 mm. > Finished chain.

{'input': 'can you calculate the circumference of a circle that has a radius of 7.81mm', 'output': 'The circumference of a circle with a radius of 7.81 mm is approximately 49.07 mm.'}
```

- 매우 잘 사용하고 있지만, 초반에 input schema에서 radius를 [int|float]형으로 잡았더니 int로 변환해서 넣는 오류가 계속 발생함 -> 결국 float 하나로 통일해야 문제가 해결됨
- 웬만하면 input type을 하나로 통일하는 것이 알아듣기 쉬운 듯
- 다만 이렇게만 보면 내부적으로 어떻게 동작하는지 알기가 어려워 Trace를 살펴보니 LangSmith로 제공함

```python
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = LANGCHAIN_API_KEY
```

> Trace 화면

![](https://i.imgur.com/uYlzfIe.png)

- 이렇게 보니 어떤 방식으로 Call 오고가고, 전체적인 Token도 파악하기가 쉬웠음
- 아쉬운건 결국 유료 서비스라 LangChain이 어디서 수익모델을 잡는지 파악하게 된 계기
- Kubernetes로 Self-Hosted Server도 제공하고 있지만 Enterprise Plan을 위한 용도
- 오픈소스로 비슷한 툴이 있을까 찾아보니 두 개가 대표적
	- [langfuse/langfuse: 🪢 Open source LLM engineering platform. Observability, metrics, evals, prompt management 🍊YC W23 🤖 SDKs + integrations for Typescript, Python, OpenAI, Langchain, LlamaIndex, Litellm (github.com)](https://github.com/langfuse/langfuse)
	- [Arize-ai/phoenix: AI Observability & Evaluation (github.com)](https://github.com/Arize-ai/phoenix)
	- Reddit 글을 보니 자체 임베딩이 있다면 phoenix가 좋고, 그 외는 MIT 라이센스로 완전한 오픈소스인 langfuse를 추천함


## More Advanced Tool Usage

- [2303.17580.pdf (arxiv.org)](https://arxiv.org/pdf/2303.17580.pdf) 를 참고하여 로컬 모델의 인퍼런스를 활용하는 방법을 소개
- ChatGPT 3.5는 이미지를 볼 수 없기 때문에 로컬에서 이미지를 한번 파싱한 정보를 제공하여 이미지 정보를 알 수 있게함

```python
import torch
from transformers import BlipProcessor, BlipForConditionalGeneration

# 사용할 이미지 캡션 모델
hf_model = "Salesforce/blip-image-captioning-large"
```

- 아무래도 맥에서 그냥하면 느리기 때문에 MPS를 활용

```python
# Using MPS Backend
# ref: https://pytorch.org/docs/stable/notes/mps.html

if not torch.backends.mps.is_available():
    if not torch.backends.mps.is_built():
        print(
            "MPS not available because the current PyTorch install was not "
            "built with MPS enabled."
        )
    else:
        print(
            "MPS not available because the current MacOS version is not 12.3+ "
            "and/or you do not have an MPS-enabled device on this machine."
        )
else:
    mps_device = torch.device("mps")
```
```python
processor = BlipProcessor.from_pretrained(hf_model)
model = BlipForConditionalGeneration.from_pretrained(hf_model).to(mps_device)
```

```python
import requests
from PIL import Image

img_url = "https://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80"
image = Image.open(requests.get(img_url, stream=True).raw).convert("RGB")
image
```

![](https://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80)

 > 위처럼 이미지를 캡셔닝할 수 있는 환경을 준비한다


```python
class ImageCaptionToolInput(BaseModel):
    url: str = Field(description="Image URL")


class ImageCaptionTool(BaseTool):
    name = "ImageCaptioner"
    description = """use this tool when given the URL of an image that you'd like to be 
    described. It will return a simple caption describing the image."""
    args_schema: Type[BaseModel] = ImageCaptionToolInput

    def _run(self, url: str):
        image = Image.open(requests.get(img_url, stream=True).raw).convert("RGB")
        inputs = processor(image, return_tensors="pt").to(mps_device)
        out = model.generate(**inputs, max_new_tokens=20)
        caption = processor.decode(out[0], skip_special_tokens=True)
        return caption

    def _arun(self, query: str):
        raise NotImplementedError("This tool does not support async")
```

> Image Caption 툴 작성

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

new_sys_msg = "You are very powerful assistant, but bad at captioning images."

use_tool_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", new_sys_msg),
        ("user", "{input}"),
        MessagesPlaceholder(variable_name="agent_scratchpad"),
    ]
)

tools = [ImageCaptionTool()]

image_cap_agent = A.create_openai_tools_agent(llm, tools, use_tool_prompt)
image_cap_agent_executor = A.AgentExecutor(
    agent=image_cap_agent, tools=tools, verbose=True
)
```

> 이미지 캡션 기능을 추가한 Agent 생성

```python
img_url = "https://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80"

image_cap_agent_executor.invoke({"input": f"what is in this image?\n{img_url}"})
```

```python
> Entering new AgentExecutor chain... Invoking: `ImageCaptioner` with `{'url': '[https://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80'}`](https://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80%27}`) there is a monkey that is sitting in a treeThe image contains a monkey sitting in a tree. > Finished chain.

{'input': 'what is in this image?\[nhttps://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80](nhttps://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80)', 'output': 'The image contains a monkey sitting in a tree.'}
```

> 생성 결과: 원숭이를 잘 탐지함

더 나아가 한글로 질의

```python
image_cap_agent_executor.invoke({"input": f"이미지에 있는 걸 모두 말해줘\n{img_url}"})

{'input': '이미지에 있는 걸 모두 말해줘\[nhttps://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80](nhttps://images.unsplash.com/photo-1616128417859-3a984dd35f02?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2372&q=80)', 'output': '이 이미지에는 나무에 앉아 있는 원숭이가 있습니다.'}
```

> `이 이미지에는 나무에 앉아 있는 원숭이가 있습니다.` 라는 로컬 머신을 번역한 대답을 생성

- 기존의 LLM이 가지고 있는 한계를 여러 특화 모델을 결합하여 생성할 수 있다는 것에 의의가 있지만 결국 여러번 핑퐁을 하면서 많은 토큰을 사용할 수 밖에 없음
- 목적이 명확하다면 처음부터 같이 보내주는게 나은 방법이고 이러한 방법은 결국 여러가지 태스크를 동시에 처리하기 위한 범용성이 필요한 경우에 유용
- 여기선 OpenAI의 프로토콜이 지배적으로 작용하지만 오픈소스에서도 어떻게 풀 수 있을지 고민이 필요해보임

## Reference
- [Building Custom Tools for LLM Agents | Pinecone](https://www.pinecone.io/learn/series/langchain/langchain-tools/)
- [Tools | 🦜️🔗 Langchain](https://python.langchain.com/docs/modules/agents/tools/)
- https://docs.smith.langchain.com/cookbook/tracing-examples/nesting-tools 
- [Function calling - OpenAI API](https://platform.openai.com/docs/guides/function-calling)
- [Open-source LLMs as LangChain Agents (huggingface.co)](https://huggingface.co/blog/open-source-llms-as-agents)
