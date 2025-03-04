---
layout: post
title: 욕설 감지 서버 개선기
tags: ["study"]
date: 2025-03-03T00:00:00+09:00
key: 2025-03-03 post
---

> 작년 해커톤때 제작하였던 flask의 korcen 욕설 감지 서버를 spring ai와 chat gpt api로 개선하게 된 과정에 대해서 정리한 글입니다.

## 개선이 필요한 사항


- 변형적 욕설에 대한 필터가 개선 필요


## 기존 욕설 감지는 어떻게 처리하였나?

수강후기 글 제목과 내용 텍스트를 배열로 관리하여 두 글 덩어리 마다 예측값을 측정하여
45% 이상으로 욕설 가능성을 포함하였을때, 욕설 감지로 처리한다.

정상적인 문장- <br/>
<img src = "https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250227211849.png" width="100%" height="100%">

욕설감지사례- <br/>
<img src = "https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250227211902.png" width="100%" height="100%">

오해받은 욕설사례- <br/>
<img src = "https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250227212405.png" width="100%" height="100%">

korcen 이라는 한국어 모델에게 text들을 input하여 예측 퍼센티지를 반환받는다.

### 문제점

- 유사 욕설, 변형 욕설들의 예측 퍼센티지를 감지 수준 미만으로 인지하였을 경우, 욕설을 감지하지 못함.

- 문장마다 띄어쓰기로 인한 욕설로 오해하는 경우, 욕설로 잘못 감지함.

위 두 문제는 문맥을 이해하지 못하는 AI 모델이라는 같은 문제를 가지고 있었다.

따라서 문맥을 이해하고 판단할 수 있는 AI 모델이 필요하였는데, 현재 잘 나와있는 생성형 AI 모델과 채팅 API 를 통해서 욕설을 판단하기로 하였다.

## 개선 내용

- Chat GPT 4o-mini 경량형 모델을 open ai api로 연동
    - chat/completions 엔드포인트로 통신하여 욕설이 포함된 문장이면 true. 욕설이 포함되지 않은 문장이면 false로 반환하도록 프롬프트 작성

- Spring AI Framework의 open ai 연동 객체를 활용


### spring ai 도입

<img src = "https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250304172434.png" width="100%" height="auto"/>

spring ai 프레임워크를 도입하여 프롬프트들도 모두 객체로 관리할 수 있도록 구성하였고, open ai api 와 통신 요청 객체까지 사용하여 편하게 프롬프트 통신을 통해서 원하는 응답을 가져올 수 있다.

### 코드 구성
#### Vilification

```java
public class Vilification {

    private static String text = "너는 수강후기 욕설감지 챗봇이다. 아래의 규칙을 반드시 따르라:\n" +
            "1. 입력된 문장에 욕설이 포함되어 있으면 반드시 \"true\"로 응답하고, 욕설이 포함되어 있지 않으면 반드시 \"false\"로 응답한다.\n" +
            "2. 응답은 오직 \"true\" 또는 \"false\"만 출력해야 하며, 추가 설명이나 부가 정보를 포함해서는 안 된다.\n" +
            "3. 욕설 판별 시, 특수문자가 중간에 섞여 있거나 외국어와 한국어가 혼합되어 사용되는 영리한 욕설 문장도 고려하여 정확하게 판단한다.\n" +
            "4. 입력된 문장의 모든 문맥을 분석하여 욕설 여부를 신중하게 결정한다.";


    public static String systemMessage() {
        return text;
    }
}
```

GPT에게 직접 욕설 감지를 위한 프롬프트를 작성하도록 명령하여 받은 사전 학습 메시지를 관리하는 클래스를 구성한다.

#### ChatClientProvider

```java
    public String request(String userMessage) {

        ChatResponse response = ChatClient.create(chatModel).prompt()
                .advisors(new SimpleLoggerAdvisor())
                .options(OpenAiChatOptions.builder().model("gpt-4o-mini").build())
                .system(Vilification.systemMessage())
                .user(userMessage)
                .call()
                .chatResponse();

        log.info("====== 결과 ===== {} " , response);

        return response.getResult().getOutput().getText();

    }
```

Spring ai 에서 제공하는 ChatClient 의 동작을 가지고 있는 클래스를 생성한다.

**ChatClient 속성**
- advisors : 대화 진행 상태, 로그 기록 어드바이저 추가(대화의 흐름 추적)
- options : 설정 옵션 지정, 모델 선택
- system : 사전 학습 메시지
- user : 사용자 요청 메시지


반환받은 응답 모습 <br/>
<img src = "https://raw.githubusercontent.com/amazon7737/amazon7737.github.io/refs/heads/main/images/Pasted%20image%2020250304172211.png" width = "100%" height="100%" />

### 개선결과

- 문장의 욕설 가능성을 학습된 텍스트로 반영된 판별이 아닌 문맥을 이해한 욕설 가능성을 탐지하도록 구성
