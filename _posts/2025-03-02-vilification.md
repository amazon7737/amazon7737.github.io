---
layout: post
title: 트러블슈팅 open ai api 통신 문제 디버깅 과정
tags: ["study"]
date: 2025-03-02T00:00:00+09:00
key: 2025-03-02 post
---

> open ai api를 통해서 원하는 프롬프트 응답 기능을 배포하던 도중 발생한 문제 원인을 찾기위한 디버깅 과정에 대해서 정리하였습니다.

## 배포 서버 환경

Oracle Cloud EC2 instance
OCPU 개수 : 1
네트워크 대역폭(Gbps):0.48
메모리(GB): 1
메모리 스왑 4GB

Canonical Ubuntu 22.04
Caddy Web Server

open ai api를 활용한 요청과 응답 구조를 위해 배포 환경을 아래와 같이 구성하였다.

## 시스템 구조

<img src = "../images/2025-03-02-openaiapi-통신해결과정1.png" width="60%" height="60%">

Caddy Web Server를 통하여 HTTPS 로 요청을 가능하게하여 Reverse Proxy를 받아 Spring AI 서버가 요청을 받고 open ai api 에게 제목과 게시글을 전달하여 욕설을 검사하고 boolean 값을 반환받도록 하였다.

프롬프트를 작성하는게 제일 중요한 부분이었는데 gpt에게 gpt에게 줄 프롬프트를 작성해달라고 물어보니 아주 단호하고 날카롭게 프롬프트를 작성해주었다.

실제로 잘 작동하였고 오차 없이 정확하게 문맥을 검사하여 욕설을 감지하였다.

## Local 환경과 Production 환경은 어떤 차이가 있을까?

문제 상황 : Local 환경에서 api 테스트를 진행하였을때는 문제없이 잘 작동하였다. open ai api까지 요청, 응답이 잘 이루어졌고 문제없이 boolean을 반환 받았다. 하지만, Production 환경으로 올려 api 테스트를 시도하자 에러가 발생하여 정상적인 응답을 받지 못하고 500 상태 번호가 반환되었다.

#### 에러 내용

```
2025-02-25T15:08:47.152Z  WARN 1 --- [class-review-vilification] [nio-8080-exec-8] o.springframework.ai.retry.RetryUtils    : Retry error. Retry count:1

org.springframework.web.client.ResourceAccessException: I/O error on POST request for "https://api.openai.com/v1/chat/completions": null
	at org.springframework.web.client.DefaultRestClient$DefaultRequestBodyUriSpec.createResourceAccessException(DefaultRestClient.java:588) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.client.DefaultRestClient$DefaultRequestBodyUriSpec.exchangeInternal(DefaultRestClient.java:502) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.client.DefaultRestClient$DefaultRequestBodyUriSpec.retrieve(DefaultRestClient.java:458) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.ai.openai.api.OpenAiApi.chatCompletionEntity(OpenAiApi.java:256) ~[spring-ai-openai-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.openai.OpenAiChatModel.lambda$internalCall$1(OpenAiChatModel.java:274) ~[spring-ai-openai-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.retry.support.RetryTemplate.doExecute(RetryTemplate.java:357) ~[spring-retry-2.0.11.jar!/:na]
	at org.springframework.retry.support.RetryTemplate.execute(RetryTemplate.java:230) ~[spring-retry-2.0.11.jar!/:na]
	at org.springframework.ai.openai.OpenAiChatModel.lambda$internalCall$3(OpenAiChatModel.java:274) ~[spring-ai-openai-1.0.0-M6.jar!/:1.0.0-M6]
	at io.micrometer.observation.Observation.observe(Observation.java:565) ~[micrometer-observation-1.13.11.jar!/:1.13.11]
	at org.springframework.ai.openai.OpenAiChatModel.internalCall(OpenAiChatModel.java:271) ~[spring-ai-openai-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.openai.OpenAiChatModel.call(OpenAiChatModel.java:255) ~[spring-ai-openai-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.DefaultChatClient$DefaultChatClientRequestSpec$1.aroundCall(DefaultChatClient.java:680) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.advisor.DefaultAroundAdvisorChain.lambda$nextAroundCall$1(DefaultAroundAdvisorChain.java:98) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at io.micrometer.observation.Observation.observe(Observation.java:565) ~[micrometer-observation-1.13.11.jar!/:1.13.11]
	at org.springframework.ai.chat.client.advisor.DefaultAroundAdvisorChain.nextAroundCall(DefaultAroundAdvisorChain.java:98) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.advisor.SimpleLoggerAdvisor.aroundCall(SimpleLoggerAdvisor.java:99) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.advisor.DefaultAroundAdvisorChain.lambda$nextAroundCall$1(DefaultAroundAdvisorChain.java:98) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at io.micrometer.observation.Observation.observe(Observation.java:565) ~[micrometer-observation-1.13.11.jar!/:1.13.11]
	at org.springframework.ai.chat.client.advisor.DefaultAroundAdvisorChain.nextAroundCall(DefaultAroundAdvisorChain.java:98) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.DefaultChatClient$DefaultCallResponseSpec.doGetChatResponse(DefaultChatClient.java:493) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.DefaultChatClient$DefaultCallResponseSpec.lambda$doGetObservableChatResponse$1(DefaultChatClient.java:482) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at io.micrometer.observation.Observation.observe(Observation.java:565) ~[micrometer-observation-1.13.11.jar!/:1.13.11]
	at org.springframework.ai.chat.client.DefaultChatClient$DefaultCallResponseSpec.doGetObservableChatResponse(DefaultChatClient.java:482) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.DefaultChatClient$DefaultCallResponseSpec.doGetChatResponse(DefaultChatClient.java:466) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at org.springframework.ai.chat.client.DefaultChatClient$DefaultCallResponseSpec.content(DefaultChatClient.java:516) ~[spring-ai-core-1.0.0-M6.jar!/:1.0.0-M6]
	at com.example.classreviewvilification.ChatClientProvider.request(ChatClientProvider.java:26) ~[!/:0.0.1-SNAPSHOT]
	at com.example.classreviewvilification.api.PromptRequestEndPoint.request(PromptRequestEndPoint.java:22) ~[!/:0.0.1-SNAPSHOT]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:568) ~[na:na]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:255) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:188) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:118) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:926) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:831) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1089) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:979) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1014) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:914) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:590) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:885) ~[spring-webmvc-6.1.17.jar!/:6.1.17]
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:658) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:195) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:51) ~[tomcat-embed-websocket-10.1.36.jar!/:na]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:167) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:90) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:483) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:115) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:93) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:344) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:397) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:63) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:905) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1743) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:52) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1190) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63) ~[tomcat-embed-core-10.1.36.jar!/:na]
	at java.base/java.lang.Thread.run(Thread.java:833) ~[na:na]
Caused by: java.net.ConnectException: null
	at java.net.http/jdk.internal.net.http.HttpClientImpl.send(HttpClientImpl.java:573) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.HttpClientFacade.send(HttpClientFacade.java:123) ~[java.net.http:na]
	at org.springframework.http.client.JdkClientHttpRequest.executeInternal(JdkClientHttpRequest.java:103) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.http.client.AbstractStreamingClientHttpRequest.executeInternal(AbstractStreamingClientHttpRequest.java:70) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.http.client.AbstractClientHttpRequest.execute(AbstractClientHttpRequest.java:66) ~[spring-web-6.1.17.jar!/:6.1.17]
	at org.springframework.web.client.DefaultRestClient$DefaultRequestBodyUriSpec.exchangeInternal(DefaultRestClient.java:496) ~[spring-web-6.1.17.jar!/:6.1.17]
	... 75 common frames omitted
Caused by: java.net.ConnectException: null
	at java.net.http/jdk.internal.net.http.common.Utils.toConnectException(Utils.java:1047) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.PlainHttpConnection.connectAsync(PlainHttpConnection.java:198) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.AsyncSSLConnection.connectAsync(AsyncSSLConnection.java:56) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.Http2Connection.createAsync(Http2Connection.java:378) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.Http2ClientImpl.getConnectionFor(Http2ClientImpl.java:128) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.ExchangeImpl.get(ExchangeImpl.java:93) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.Exchange.establishExchange(Exchange.java:343) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.Exchange.responseAsyncImpl0(Exchange.java:475) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.Exchange.responseAsyncImpl(Exchange.java:380) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.Exchange.responseAsync(Exchange.java:372) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.MultiExchange.responseAsyncImpl(MultiExchange.java:408) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.MultiExchange.lambda$responseAsyncImpl$7(MultiExchange.java:449) ~[java.net.http:na]
	at java.base/java.util.concurrent.CompletableFuture.uniHandle(CompletableFuture.java:934) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture.uniHandleStage(CompletableFuture.java:950) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture.handle(CompletableFuture.java:2340) ~[na:na]
	at java.net.http/jdk.internal.net.http.MultiExchange.responseAsyncImpl(MultiExchange.java:439) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.MultiExchange.lambda$responseAsync0$2(MultiExchange.java:341) ~[java.net.http:na]
	at java.base/java.util.concurrent.CompletableFuture$UniCompose.tryFire(CompletableFuture.java:1150) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:510) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java:1773) ~[na:na]
	at java.net.http/jdk.internal.net.http.HttpClientImpl$DelegatingExecutor.execute(HttpClientImpl.java:157) ~[java.net.http:na]
	at java.base/java.util.concurrent.CompletableFuture.completeAsync(CompletableFuture.java:2673) ~[na:na]
	at java.net.http/jdk.internal.net.http.MultiExchange.responseAsync(MultiExchange.java:294) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.HttpClientImpl.sendAsync(HttpClientImpl.java:654) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.HttpClientImpl.send(HttpClientImpl.java:552) ~[java.net.http:na]
	... 80 common frames omitted
Caused by: java.nio.channels.UnresolvedAddressException: null
	at java.base/sun.nio.ch.Net.checkAddress(Net.java:149) ~[na:na]
	at java.base/sun.nio.ch.Net.checkAddress(Net.java:157) ~[na:na]
	at java.base/sun.nio.ch.SocketChannelImpl.checkRemote(SocketChannelImpl.java:816) ~[na:na]
	at java.base/sun.nio.ch.SocketChannelImpl.connect(SocketChannelImpl.java:839) ~[na:na]
	at java.net.http/jdk.internal.net.http.PlainHttpConnection.lambda$connectAsync$0(PlainHttpConnection.java:183) ~[java.net.http:na]
	at java.base/java.security.AccessController.doPrivileged(AccessController.java:569) ~[na:na]
	at java.net.http/jdk.internal.net.http.PlainHttpConnection.connectAsync(PlainHttpConnection.java:185) ~[java.net.http:na]
	... 103 common frames omitted

```

#### 원인을 향한 여정

원인이 무엇인지 파악하기 힘들었다. Exception Message는 null로 반환되었고, 비슷한 POST 요청의 에러 로그들을 구글링해보니 대부분 timeout 에러를 반환받은 분들이 많았다.

분명 에러메시지가 나와야하는데 왜 나오지 못한 것일까?

** 인텔리제이에서 순서대로 객체 코드들 스샷올리기 **


Docker Container를 사용하여 환경의 이질감을 가지고 있지 않다고 판단하였다. 하지만 에러는 이미 발생하였고, Local 환경과의 차이가 분명히 존재한다고 생각하였고 차근차근히 에러 원인을 찾기위해 에러를 쫓아가 보았다.

#### Tracing

** 코드와 설명 쭉쭉 **

#### DefaultRestClient 에러 발생

```
at org.springframework.web.client.DefaultRestClient$DefaultRequestBodyUriSpec.exchangeInternal(DefaultRestClient.java:502) ~[spring-web-6.1.17.jar!/:6.1.17]
```


```java
try {
    uri = initUri();
    HttpHeaders headers = initHeaders();
    ClientHttpRequest clientRequest = createRequest(uri);
    clientRequest.getHeaders().addAll(headers);
    ClientRequestObservationContext observationContext = new ClientRequestObservationContext(clientRequest);
    observationContext.setUriTemplate(this.uriTemplate);
    observation = ClientHttpObservationDocumentation.HTTP_CLIENT_EXCHANGES.observation(observationConvention,
    DEFAULT_OBSERVATION_CONVENTION, () -> observationContext, observationRegistry).start();
    if (this.body != null) {
        this.body.writeTo(clientRequest);
    }
    if (this.httpRequestConsumer != null) {
        this.httpRequestConsumer.accept(clientRequest);
    }
    clientResponse = clientRequest.execute();
    observationContext.setResponse(clientResponse);
    ConvertibleClientHttpResponse convertibleWrapper = new DefaultConvertibleClientHttpResponse(clientResponse);

      return exchangeFunction.exchange(clientRequest, convertibleWrapper);
    } catch (IOException ex) {
      ResourceAccessException resourceAccessException = createResourceAccessException(uri, this.httpMethod, ex);
      if (observation != null) {
        observation.error(resourceAccessException);
      }
      throw resourceAccessException;
    }
```

1. 요청할 URI 와 HTTP 헤더 초기화
2. 클라이언트 요청 객체를 생성, 헤더 추가
3. 생성된 요청 객체 기반으로 관찰(context) 객체 생성, URI 템플릿 설정
4. HTTP 클라이언트 요청 및 응답 메트릭 수집, 로깅 시작
5. 요청 본문이 존재한다면, 클라이언트 요청에 작성
6. 추가적인 요청 consumer 가 존재한다면 consumer를 적용

위 과정을 진행 도중 IOException이 발생했다.



```
	at org.springframework.ai.openai.api.OpenAiApi.chatCompletionEntity(OpenAiApi.java:256) ~[spring-ai-openai-1.0.0-M6.jar!/:1.0.0-M6]

```


OpenAiApi 의 chatCompletionEntity

```java
public ResponseEntity<ChatCompletion> chatCompletionEntity(ChatCompletionRequest chatRequest,
MultiValueMap<String, String> additionalHttpHeader) {

Assert.notNull(chatRequest, "The request body can not be null.");
Assert.isTrue(!chatRequest.stream(), "Request must set the stream property to false.");
Assert.notNull(additionalHttpHeader, "The additional HTTP headers can not be null.");

    return this.restClient.post()
      .uri(this.completionsPath)
      .headers(headers -> headers.addAll(additionalHttpHeader))
      .body(chatRequest)
      .retrieve()
      .toEntity(ChatCompletion.class);
}
```

1. ChatCompletion 타입의 응답을 ResponseEntity 로 반환
2. ChatRequest, additionalHttpHeader 가 null이 아님을 검증
3. 요청의 stream 속성이 false 여야 한다는 조건을 확인
- restClient를 사용하여 지정된 URI(completionPath) 로 POST 요청을 보냄
- 추가 HTTP 헤더를 요청에 포함시키고, chatRequest를 본문으로 설정
- 응답을 ChatCompletion 타입의 엔티티로 변환하여 반환


#### OpenAiChatModel

```java
ResponseEntity<ChatCompletion> completionEntity = this.retryTemplate
					.execute(ctx -> this.openAiApi.chatCompletionEntity(request, getAdditionalHttpHeaders(prompt)));

```

chatCompletionEntity가 완전한 응답을 반환하기 전에 에러가 발생하였다. 이후 하위의 에러들은 Net.java와 같은 RestTemplate로 HTTP 요청을 진행하는 객체들이었고, 요청과 응답이 정상적으로 진행되지 않았다는 것은 파악되었다.

하지만 요청에 문제가 있을지, 응답에 문제가 있을지에 대해서 확인을 위해 요청이 올바르게 진행되었는지 확인을 진행하였다.

open ai api에서 발급받은 고유 app key를 제대로 입력하지 않은 상태에서 요청을 진행하니

```
org.springframework.ai.retry.NonTransientAiException: 401 - {
    "error": {
        "message": "Incorrect API key provided: sk-proj-********************************************************************************************************************************************************KRsA. You can find your API key at https://platform.openai.com/account/api-keys.",
        "type": "invalid_request_error",
        "param": null,
        "code": "invalid_api_key"
    }
  }
```

401 에러를 반환하였고, 요청이 정상적으로 Open ai api에 도달하고 있음을 확인하였다.

응답이 정상적으로 반환되지 않고 있다는 것을 확인하였고, 이때부터 원인을 찾는데 시간이 많이 걸렸다.


```
"https://api.openai.com/v1/chat/completions": null
```

해당 예외 메시지를 반환하는 객체를 참조해보았다.

```java
catch (IOException ex) {
    ResourceAccessException resourceAccessException = createResourceAccessException(uri, this.httpMethod, ex);
    if (observation != null) {
        observation.error(resourceAccessException);
    }
        throw resourceAccessException;
    }
```

#### createResourceAccessException 메서드

```java
  StringBuilder msg = new StringBuilder("I/O error on ");
  msg.append(method.name());
  msg.append(" request for \"");
  String urlString = url.toString();
  int idx = urlString.indexOf('?');
  if (idx != -1) {
    msg.append(urlString, 0, idx);
  }
  else {
    msg.append(urlString);
  }
  msg.append("\": ");
  msg.append(ex.getMessage());
  return new ResourceAccessException(msg.toString(), ex);
```

- "I/O error on " 라는 기본 메시지에 요청 메서드 이름을 추가
- 쿼리 파라미터(?)를 제외한 URL을 포함
- 마지막 부분에 원래 예외 메시지를 덧붙여 전체 오류 메시지 완성


message가 null로 생성되었고, 에러 메시지를 받지 못한 것이다.

더이상 응답 상황을 알 방법이 없었고, 배포 환경에 대해서 고민하게 되었다.

요청과 응답이 진행된 것을 보았을때, Caddy WebServer 의 프록싱까지 정상적으로 진행된 것으로 판단하였고, 인스턴스 자체의 배포환경에 대해서 고민하게 되었다.

Chat GPT API 의 지역은 한국으로 설정되어있었다.
하지만 Oracle Cloud 의 지역 계정은 Osaka로 설정되어 있었고, 한국 계정으로 변경하여 다시 배포를 진행하였다.

```
2025-02-27T11:44:05.192Z  INFO 1 --- [class-review-vilification] [nio-8080-exec-3] c.e.c.api.PromptRequestEndPoint          : ==== 요청 글 ===== {"postTitle":"좋은 강의","postContent":"아주좋고 자유로운 좋은 강의입니다. "} 
2025-02-27T11:44:05.787Z  INFO 1 --- [class-review-vilification] [nio-8080-exec-3] c.e.c.ChatClientProvider                 : ====== 결과 ===== ChatResponse [metadata={ id: chatcmpl-B5Wb7rh6Wx8AXedYmMDBInXZLLPhN, usage: DefaultUsage{promptTokens=203, completionTokens=2, totalTokens=205}, rateLimit: { @type: org.springframework.ai.openai.metadata.OpenAiRateLimit, requestsLimit: 10000, requestsRemaining: 9999, requestsReset: PT1M4S, tokensLimit: 200000; tokensRemaining: 199792; tokensReset: PT0.062S } }, generations=[Generation[assistantMessage=AssistantMessage [messageType=ASSISTANT, toolCalls=[], textContent=false, metadata={refusal=, finishReason=STOP, index=0, id=chatcmpl-B5Wb7rh6Wx8AXedYmMDBInXZLLPhN, role=ASSISTANT, messageType=ASSISTANT}], chatGenerationMetadata=DefaultChatGenerationMetadata[finishReason='STOP', filters=0, metadata=0]]]] 
```

변경한 인스턴스에서는 정상적으로 진행되는 것을 확인하였고, 인스턴스의 지역 문제가 open ai api의 연동에 문제를 일으켰다는 것으로 해결되었다.

간단한 문제였기도 하였지만, 에러의 원인을 찾기위해서 차근차근 예외 내역을 찾아들어가며 사용되는 클래스들의 동작을 따라가보는 과정에서 HTTP RestTemplate 의 동작 순서에 대해서 이해해보는 경험을 하게 되었다.



