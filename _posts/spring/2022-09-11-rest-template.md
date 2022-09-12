---
title: RestTemplate 사용법과 커스텀 설정
author: taeyeong
categories: [Spring]
tags: [Spring, RestTeamplate]
date: 2022-09-11 20:09:20 +0900
math: true
mermaid: ture
---

## RestTemplate 동작 원리

![](/assets/images/2022-09-11-rest-template-01.png)

Application에서 `RestTemplate`을 선언하고 URI와 HTTP 메서드, Header, Body 등을 설정한다.

외부 API로 요청을 보내게 되면 `RestTemplate`에서 `HttpMessageConverter`를 통해 `RequestEntity`를 요청 메시지로 변환한다.

`RestTemplate`은 변환된 요청 메시지를 `ClientHttpRequestFactory`를 통해 `ClientHttpRequest`로 가져온 후 외부 API로 요청을 보낸다.

외부에서 요청에 대한 응답을 받으면 `RestTemplate`은 `ResponseErrorHandler`로 오류를 확인하고, 오류가 있다면 `ClientHttpResponse`에서 응답 데이터를 처리한다.

받은 응답 데이터가 정상적이라면 다시 한 번 `HttpMessageConverter`를 거쳐 자바 객체로 변환해서 애플리케이션으로 반환한다.

## RestTemplate의 대표적인 메서드

|메서드|HTTP형태|설명|
|-|-|-|
|getForObject|GET|GET 형식으로 요청한 결과를 객체로 변환|
|getForEntity|GET|GET 형식으로 요청한 결과를 ResponseEntity 형식으로 반환|
|postForLocation|POST|POST 형식으로 요청한 결과를 헤더에 저장된 URI로 반환|
|postForObject|POST|POST 형식으로 요청한 결과를 객체로 반환|
|postForEntity|POST|POST 형식으로 요청한 결과를 ResponseEntity 형식으로 반환|
|delete|DELETE|DELETE 형식으로 요청|
|put|PUT|PUT 형식으로 요청|
|patchForObject|PATCH|PATCH 형식으로 요청한 결과를 객체로 반환|
|optionsForAllow|OPTIONS|해당 URI에서 지원하는 HTTP 메서드를 조회|
|exchange|any|HTTP 헤더를 임의로 추가할 수 있고, 어떤 메서드 형식에서도 사용할 수 있음|
|execute|any|요청과 응답에 대한 콜백을 수정|

## RestTemplate 사용하기

### getForEntity

```java
@Service
public class TestService {

    public UserDto getUser() {
        URI uri = UriComponentsBuilder
                .fromUriString("http://localhost:8000")
                .path("/api/v1/user")
                .encode()
                .build()
                .toUri();

        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<UserDto> responseEntity = restTemplate.getForEntity(uri, UserDto.class);

        return responseEntity.getBody();
    }
}
```

아래와 같이 query parameter를 추가할 수도 있다.

```java
URI uri = UriComponentsBuilder
        .fromUriString("http://localhost:8000")
        .path("/api/v1/user")
        .queryParam("name", "taeyeong")
        .queryParam("age", 20)
        .encode()
        .build()
        .toUri();
```

<br/>

`expand()` 메서드를 사용해서 `path`에 변수를 넣을 수 있다.

```java
URI uri = UriComponentsBuilder
        .fromUriString("http://localhost:8000")
        .path("/api/v1/user/{userId}")
        .encode()
        .build()
        .expand(10)
        .toUri();
```

값을 여러 개 넣어야 하는 경우는 콤마(,)로 구분해서 나열한다.

```java
URI uri = UriComponentsBuilder
        .fromUriString("http://localhost:8000")
        .path("/api/v1/user/{name}/{age}")
        .encode()
        .build()
        .expand("taeyeong", 20)
        .toUri();
```

### PostForEntity

```java
@Service
public class TestService {
    public UserResponse postUser() {
        URI uri = UriComponentsBuilder
                .fromUriString("http://localhost:8000")
                .path("/api/v1/user")
                .encode()
                .build()
                .toUri();

        UserRequest userRequest = new UserRequest();
        userRequest.setName("taeyeong");
        userRequest.setAge(20);

        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<UserResponse> responseEntity = restTemplate.postForEntity(uri, userRequest, UserResponse.class);

        return responseEntity.getBody();
    }
}
```

### exchange

```java
@Service
public class TestService {

    public UserResponse exchangeUser() {
        URI uri = UriComponentsBuilder
                .fromUriString("http://localhost:8000")
                .path("/api/v1/user")
                .encode()
                .build()
                .toUri();

        UserRequest userRequest = new UserRequest();
        userRequest.setName("taeyeong");
        userRequest.setAge(20);

        RequestEntity<UserRequest> requestEntity = RequestEntity
                .post(uri)
                .header("my-header", "hi")
                .body(userRequest);

        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<UserResponse> responseEntity = restTemplate.exchange(requestEntity, UserResponse.class);

        return responseEntity.getBody();
    }
}
```

## RestTemplate 커스텀 설정

`RestTemplate`은 `HTTPClient`를 추상화하고 있다. 

`RestTemplate`은 기본적으로 커넥션 풀을 지원하지 않는다. 이 기능을 지원하지 않으면 매번 호출할 때마다 포트를 열어 커넥션을 생성하게 되는데, `TIME_WAIT` 상태가 된 소켓을 다시 사용하려고 접근한다면 재사용하지 못하게 된다.

이를 방지하기 위해 커넥션 풀 기능을 활성화해야 하는데, 대표적으로 아파치에서 제공하는 `HttpClient`로 대체해서 사용하는 방법이 있다.

- httpclient 의존성 추가

```gradle
implementation 'org.apache.httpcomponents:httpclient:4.5.13'
```

---

- 커스텀 RestTemplate 객체 생성 메서드 작성하기

`RestTemplate`의 생성자에는 `ClientHttpRequestFactory`를 매개변수로 받는 생성자가 존재한다.

```java
public RestTemplate(ClientHttpRequestFactory requestFactory) {
    this();
    this.setRequestFactory(requestFactory);
}
```

`ClientHttpRequestFactory`는 함수형 인터페이스로, 대표적인 구현체로는 `SimpleClientHttpRequestFactory`와 `HttpComponentsClientHttpRequestFactory`가 있다.

별도의 구현체를 설정해서 전달하지 않으면 `SimpleClientHttpRequestFactory`를 사용한다.

<br/>

`HttpComponentsClientHttpRequestFactory` 객체를 생성해서 사용하면 `RestTemplate`의 `Timeout`을 설정할 수 있다.

추가로 커넥션 풀을 설정하려면 `HttpClient`를 생성해 설정해주어야 하는데, `HttpClient`를 생성하는 방법에는 다음 두 가지가 있다.

- `HttpClientBuilder.create()`
- `HttpClients.custom()`

```java
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();

    // HttpClientBuilder.create()
    CloseableHttpClient client = HttpClientBuilder.create()
            .setMaxConnTotal(500)
            .setMaxConnPerRoute(500)
            .build();

    // HttpClients.custom()
    CloseableHttpClient httpClient = HttpClients.custom()
            .setMaxConnTotal(500)
            .setMaxConnPerRoute(500)
            .build();

    factory.setHttpClient(httpClient);
    factory.setConnectTimeout(3000);
    factory.setReadTimeout(5000);

    RestTemplate restTemplate = new RestTemplate(factory);

    return restTemplate;
}
```

생성한 `HttpClient`는 `factory`의 `setHttpClient()` 메서드를 통해 인자로 전달해서 설정할 수 있다.

이렇게 설정된 `factory` 객체를 `RestTemplate`을 초기화 하는 과정에 인자로 전달하면 된다.

## Reference

- 스프링 부트 핵심 가이드 - 장정우