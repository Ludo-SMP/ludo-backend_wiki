## 개요

> 이 글은 [아카](https://github.com/june-777)가 작성했습니다.

2차 마일스톤에서 `SSE 기반 알림 기능`을 담당하며 해결했던 트러블 슈팅 사례를 공유하고자 합니다.  
사례는 총 4가지이며, #1 ~ #3은 네트워크에 대한 이슈, #4은 DB 커넥션에 대한 이슈입니다.

<br> <br>

## #1 알림 구독 API 요청 시, 클라이언트 측에서 Pending 상태로 대기

### 📜 현상

클라이언트는 서버에게 SSE 알림 구독을 요청

이 때 클라이언트는 알림 구독 요청에 대한 응답을 받지 못하고, **Pending 상태로 대기 후 Time Out이 발생**

<br>

<img src="https://github.com/user-attachments/assets/c190cc6e-a743-4721-b1e1-041415cf8577" width=700>

<br>

### 🤔 원인

Server Sent Event(이하 SSE)는 **기본 기조가 HTTP 연결 유지 (Persistence Connection)** 입니다. 이벤트가 발생할 때마다 클라이언트에게 이벤트를 서빙해야하기 때문입니다.
하지만 Nginx는 받은 요청을 Upstream 서버로 전달할 때, 기본적으로 **HTTP 1.0으로 프로토콜 버전을 변경하고 전달**합니다. HTTP 1.0은 연결 유지를 지원하지 않기 때문에, SSE 구독 요청에 대한 응답을 받지 못할 수 있습니다.

<br>

<img src="https://github.com/user-attachments/assets/3df7c6ac-e912-4dd7-8122-e875d7997fc3" width=700>

<br> <br>

<img src="https://github.com/user-attachments/assets/778135d1-3754-4297-97ff-b3f92eea4df5" width=700>

>  Ludo 서비스는 리버스 프록시로서 Nginx를 사용하고 있습니다.

<br>

### 🧐 조치 및 해결

Nginx 설정에서 http 버전을 명시하고, Connection 옵션을 설정해주면 됩니다. 하지만 알림 구동 요청에 대한 네트워크 이슈를 완전히 해결하기 위해선 한 가지 조치를 더 취해주어야 하는데요. 해당 내용은 아래에 이어서 다뤄보겠습니다.

<img src="https://github.com/user-attachments/assets/da23748d-7f39-46c7-905f-c9e840f1e74e" width=500>

<br> <br>

## #2 알림 구독 API 요청 시 Pending 상태가 해제되면, 쌓여있던 실시간 알림이 한 번에 전송

### 📜 현상

클라이언트는 알림 구독 요청에 대해 여전히 Pending 상태. 하지만 아래의 현상이 추가됨

Pending 상태가 해제되면, **쌓여있던 실시간 알림이 한 번에 전송**

<br>

<img src="https://github.com/user-attachments/assets/6d59d480-6528-400a-a019-5e61bf3c5cf2" width=700>

<br>

### 🤔 원인

결론부터 말하면, Nginx의 프록시 버퍼가 Server의 알림 관련 응답을 버퍼하기 때문에 발생했던 문제입니다.

**쌓여있던 데이터가 한 번에 전송**되고 있으므로, **버퍼의 유무**를 생각해볼 수 있겠습니다. [Nginx proxy_buffering 공식문서](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering)를 확인해보면, 서버로부터 받은 응답을 메모리에 버퍼해두는 것을 확인할 수 있습니다. 그리고 **응답의 전체가 버퍼**되었을 때, 버퍼한 응답들을 클라이언트에게 전송합니다. 이를 그림으로 표현하면 아래와 같습니다.

<br>

<img src="https://github.com/user-attachments/assets/387285c8-8052-437c-a96b-fa689dfa2053" width=700>

<br><br>

문제는 SSE의 기조가 클라이언트와 연결을 유지한채로, **"끝나지 않는 응답"** 을 보낸다는 것입니다. 
그리고 SSE 연결이 종료될 때 비로소 끝나지 않는 응답이 끝나게 됩니다. 즉, **SSE 연결이 종료될 때까지 Nginx가 SSE 구독 요청에 대한 응답과 실시간 이벤트들을 버퍼링**하고 있어서 발생했던 문제입니다.

>  SSE의 공식적인 HTTP Content-Type: text/event-stream

<br>

### 🧐 조치 및 해결

해결 방법은 간단합니다. Nginx의 프록시 버퍼 기능을 끄면 됩니다. 하지만 해당 설정을 전역적으로 끄게되면, 알림 이외의 다른 API에서 Nginx의 프록시 버퍼 최적화 기능을 활용할 수 없게됩니다. 이 때 **`X-Accel-Buffering`** 을 활용하면 됩니다. Response Header에 `<X-Accel-Buffring, no>`를 추가하면, Nginx에서 해당 헤더를 식별하여 버퍼링 기능을 사용하지 않습니다.

<img src="https://github.com/user-attachments/assets/35f86030-24b9-439e-a356-ef5a2c1d49b7" width=600>


<br> <br>

## #3 INCOMPLETE_CHUNKED_ENCODING 200 (OK)

### 📜 현상

알림 구독 요청 실패 및 실시간 알림이 전송되지 않던 네트워크 이슈는 모두 해결

하지만 1**분 주기로 네트워크 에러와 함께 SSE 구독 요청을 재전송**. 의도했던 SSE 재구독 요청은 30분임

<br>

<img src="https://github.com/user-attachments/assets/4b2be353-1f69-467c-abb9-87026bc1750b" width=700>

<br>

### 🤔 원인

결론부터 말하면 Nginx의 `read timeout 옵션`으로 인해, **Nginx가 TCP 커넥션을 의도하지 않은 시점에 종료**함으로서 발생한 문제입니다.

해당 현상을 이해하려면, SSE 알림 구독 요청의 표준 명세인 `Transfer-Encoding:chunked`를 이해해야 합니다.

<br>

<img src="https://github.com/user-attachments/assets/83b8ba44-2bf2-4656-962f-3a2c4ede2df4" width=250>

<br>

[RFC 2616 공식 문서 Chunked Transfer Coding](https://datatracker.ietf.org/doc/html/rfc2616#section-3.6.1)를 살펴보면, 아래와 같이 정의하고 있습니다.

> The chunked encoding is ended by any chunk whose size is zero, followed by the trailer, which is terminated by an **empty line.**

`Chunked Transfer Coding`의 표준 응답 명세는 아래와 같습니다.

```
Chunked-Body   = *chunk
                 last-chunk
                 trailer
                 CRLF
```

```
chunk          = chunk-size CRLF
```



아래와 같이 예시를 들 수 있겠습니다.

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Trailer: Expires

7CLRF
#1ChunkDataCLRF
9CLRF
#2ChunkDataCLRF
7CLRF
#3ChunkDataCLRF
0CLRF
CLRF	# empty line
```

중요한 것은 **chuck 데이터의 종료 시점을 empty line으로 판단** 한다는 것입니다.

추정해볼 수 있는 것은 **마지막 chunk 데이터가 오지 않았는데 TCP 커넥션이 종료** 되었다면, 에러 메세지의 의미 그대로 `“Incomplete chunked encoding”`를 의미하게 됩니다.

주목해야하는 것은 **예기치 않은 시점에 TCP 커넥션이 종료**되는 현상입니다. 따라서 **Nginx에서 TCP 커넥션을 종료하는 타임아웃 옵션**이 있는지 살펴볼 필요가 있겠습니다.

<br>

[Nginx 공식문서](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout)에서 read timeout 옵션을 확인할 수 있습니다.

> Defines a timeout for reading a response from the proxied server.
>
> The timeout is set only between **two successive read operations.**
>
> If the proxied server does not transmit anything within this time, **the connection is closed.**

주목해야할 부분은 두가지 입니다.

- 해당 옵션은 두 개의 연속적인 읽기 작업 사이의 최대 대기 시간을 의미한다.

- 최대 대기 시간동안 프록시된 서버로부터 아무런 전송이 없을 때, 커넥션이 닫힘

`read operations`는 프록시된 서버로부터의 응답을 성공적으로 수신했음을 의미합니다. Nginx와 프록시된 서버의 관계에선 Nginx가 클라이언트기 때문입니다. 다시말하면 read timeout 옵션은 설정한 최대 대기 시간동안 프록시된 서버로부터 응답이 없으면, TCP 커넥션을 종료시킴을 의미합니다.

<br>

해당 문제가 발생했던 이유를 정리하면 아래와 같습니다.

1. 알림 구독 요청 성공

2. Nginx가 서버로부터 알림 구독 응답을 수신함

3. 1분동안 알림 이벤트가 발생하지 않음. 즉, 1분동안 서버로부터 추가적인 응답을 받지 못함.

4. read timeout 옵션(default 1분) 에 의해 Nginx가 TCP 커넥션을 종료함

5. 클라이언트는 아직 Chuck 데이터의 끝(empty line)이 오지 않았는데, TCP 커넥션이 끊겨버림
6. `INCOMPLETE_CHUNKED_ENCODING` 에러 발생

<br>

<img src="https://github.com/user-attachments/assets/5c11f36d-5681-4cec-b97f-fe82b93320cb" width=700>

<br>

### 🧐 조치 및 해결

proxy_read_timeout 옵션의 대기 시간을 의도한 시간에 맞게 수정하면 됩니다.

<br>

<img src="https://github.com/user-attachments/assets/629e3fc9-6eab-450c-ba89-602051dab406" width=600>

<br> <br>

## #4 DB 커넥션 고갈 상태

### 📜 현상

실시간 알림은 모두 성공하지만, 일정 횟수의 API 요청 후 **DB 커넥션이 부족**한 서버 에러 발생

<br>

<img src="https://github.com/user-attachments/assets/279c91db-00ff-4ef4-aefd-4f7a931c70b1" width=700>

<br>

### 🤔 원인

Ludo 서비스는 JPA를 사용하고 있는데요. **Open Session In View(이하 OSIV)** 로 인해 발생했던 문제입니다.

일반적으로 트랜잭션 시작시점에 DB 커넥션을 점유하고, 트랜잭션 종료시점에 DB 커넥션을 반납합니다. 하지만 OSIV 가 true일 경우, 트랜잭션 종료시점에 DB 커넥션을 반납하지 않고 HTTP 세션이 종료될 때 DB 커넥션을 반납합니다. SSE 는 "연결 유지"입니다. 즉 HTTP 세션이 종료되지 않으므로, DB 커넥션을 계속 점유하고있게 됩니다. 이는 DB 커넥션이 고갈되는 문제를 야기합니다.

<br>

### 🧐 조치 및 해결

OSIV 설정을 해제하면 됩니다.

<br>

<img src="https://github.com/user-attachments/assets/e0622c96-e206-453d-8f06-0988abad5b82" width=400>

<br>

OSIV 설정을 해제할 경우, 트랜잭션 범위를 벗어난 모든 엔티티는 문제 가능성을 내포합니다. 해당 엔티티는 영속화되어있지 않기 때문에, Lazy Loading을 할 경우 no session 오류가 발생합니다. 따라서 서비스 레이어 등에서 엔티티를 반환하는 경우, DTO를 반환하는 등 리팩터링을 고려해야 합니다.

<img src="https://github.com/user-attachments/assets/578ec022-d7d7-4d4f-bfe6-bae89246b268" width=700>

<br>
