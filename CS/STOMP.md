Simple Text oriented Messaging Protocol

: 문자열로 된 메세지를 안정적이고 간단하게 보내기 위한 규약

기본 설정

- Text기반이지만 바이너리 메세지 전송도 허용
- 기본 인코딩: UTF-8, 하지만 메세지 본문에 대한 대체 인코딩 사양 지원
- Text로 된 메세지를 커스텀하여 사용 가능. (창의적으로 활용 가능)

Frame

```
COMMAND
header:value

Body^@
```

COMMAND → 사전에 정의 된 문자열

header:value → key value 형식 사용

body: 실제 송수신 값 (SEND, MESSAGE, ERROR 시에만 사용 가능. (이외에 COMMAND에서는 사용 ❌))

# Header

### content-length

: 메세지 본문 길이

- 모든 메세지에는 content-length를 포함시켜야 한다.

### content-type

: body 인코딩 타입

frame에 body가 포함되어있는 경우 필수로 넣어주어야 함 (SEND, MESSAGE, ERROR)

넣지 않을 경우 default로 text/UTF-8 로 설정 됨.

### receipt

클라이언트는 CONNECT를 제외한 모든 COMMAND에 receipt header를 임의의 값으로 지정 가능.

이렇게 하면 서버가 클라이언트에게 RECEIPT Frame을 송신.

### 기타

- header가 반복될 시 첫번째 헤더를 사용
    
    ```
    MESSAGE
    foo:World
    foo:Hello
    
    ^@
    
    // foo 값은 World
    ```
    
- 서버는 DoS 공격을 막기 위한 방법으로 아래 항목에 대하여 제한을 걸 수 있음.
    
    - 단일 Frame에 허용되는 header 수
    - header 라인 최대 길이
    - Frame의 최대 크기

그 외 세부 COMMAND에 대한 내용은 아래 링크에 정리되어 있음

[STOMP](http://stomp.github.io/)
