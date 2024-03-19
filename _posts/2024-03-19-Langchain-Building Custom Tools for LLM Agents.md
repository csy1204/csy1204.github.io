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
- Tool을 만









[2303.17580.pdf (arxiv.org)](https://arxiv.org/pdf/2303.17580.pdf)


