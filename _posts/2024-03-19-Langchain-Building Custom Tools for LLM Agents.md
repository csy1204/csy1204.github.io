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

### 2.3 StructuredTool dataclass


### 2.4 Handling Tool Errors









> 

[2303.17580.pdf (arxiv.org)](https://arxiv.org/pdf/2303.17580.pdf)


