---
layout: post
title: "Socket Exception" 
author: Gunju Ko
cover:  "/assets/instacode.png"
categories: [java]
---

# Java SocketException

### SocketTimeoutException

이름에서 예상할 수 있듯이, SocketTimeoutException은 소켓을 통해 데이터를 읽거나 소켓을 연결할 때 시간초과가 나면 발생한다.  그럼 SocketTimeoutException이 발생할 수 있는 상황에서 대해서 좀 더 자세히 알아보자

#### ServerSocket.accept

우선 ServerSocket.accept() 메소드에서도 SocketTimeoutException이 발생할 수 있다. 소켓 서버를 만들기위해서는 ServerSocket를 사용한다.

- new ServerSocket(port) : port로 들어오는 클라이언트 요청을 감시하는 작업이 시작된다.
- ServerSocket.accept() : accept() 메소드에서는 클라이언트의 Socket이 연결될 때까지 계속 기다린다. 마침내 클라이언트 요청이 들어요면 클라이언트와의 통신을 위해 (현재 쓰이고 있지 않은 포트에 대한) Socket를 리턴한다. 리턴되는 Socket의 포트 번호는 ServerSocket의 포트 번호와 다르다. 그래야 ServerSocket이 다른 클라리언트가 연결하는지 감시할 수 있기 때문이다.

보통 accept() 메소드는 클라이언트 요청이 들어올때까지 무한정 기다리지만, 특정 시간동안 요청이 들어오지 않으면 예외가 발생하도록 할 수 있다. 설정하는 방법은 ServerSocket.setSoTimeout(int timeout) 메소드를 통해서 설정할 수 있다. 그러면 해당 시간동안만 accept() 메소드가 block된다. 그리고 timeout 시간동안 클라이언트 요청이 들어오지 않으면 SocketTimeoutException이 발생한다. 

``` java
serverSocket = new ServerSocket(port);
serverSocket.setSoTimeout(timeout);

// timeout동안 클라리언트 요청이 들어오지 않는 경우 SocketTimeoutException 발생
serverSocket.accept();
```

#### Socket.connect 메소드 호출

Socket을 특정 서버에 연결할 때 connect 메소드를 호출한다. 이 때도 timeout을 지정할 수 있다. 그러면 해당 시간동안 서버와 연결이 되지 않으면, SocketTimeoutException이 발생한다. connect 메소드는 서버와 연결이 완료되거나 혹은 에러가 발생할 때까지 block된다. 

#### Socket.read 메소드 호출

Socket.getInputStream() 메소드를 통해 Socket으로부터 입력스트림을 가져올 수 있다. InputStream의 read() 메소드를 통해 소켓으로 들어오는 입력 데이터를 읽을 수 있다. 

Socket.setSoTimeout은 InputStream의 read() 메소드의 타임아웃을 설정한다. setSoTimeout 메소드를 통해 타임아웃 값을 설정하면 read() 메소드는 해당 시간동안만 block된다. timeout이 지나면 SocketTimeoutException이 발생한다. 

``` java
Socket socket = new Socket();

// connectTimeout안에 Connection을 맺지 못하면 SocketTimeoutException이 발생
socket.connect(new InetSocketAddress(InetAddress.getByName(host), port), connectTimeout);
socket.setSoTimeout(readTimeout);

InputStream inputStream = socket.getInputStream();
// readTimeout 동안 block되며, readTimeout이 지나면 SocketTimeoutException이 발생한다
inputStream.read()
```

### SocketException

SocketException은 Socket을 생성하거나 접근하려고 할 때 문제가 있으면 발생한다. SocketException은 IOException을 상속하고 있다. 참고로 SocketTimeoutException은 InterruptedIOException을 상속하고 있다. (InterruptedIOException은 IOException을 상속하고 있다)

Exception

  ㄴ IOException

​    ㄴ SocketException

Exception

  ㄴ IOException

​    ㄴ InterruptedIOException

​      ㄴ SocketTimeoutException

SocketException은 보다 일반적인 커넥션 문제를 나타낸다. (반면에 SocketTimeoutException은 타임아웃과 관련된 문제가  생겼을때만 발생한다) 

예를 들어 클라이언트가 소켓을 통해 특정 서버로 연결을 시도했지만, 해당 서버로 연결이 실패한 경우는 ConnectionException이 발생한다. (ConnectionException은 SocketException의 하위 클래스이다.)

혹은 소켓을 통해 데이터를 읽거나 쓰는 도중에 소켓 연결이 끊키는 경우에도 SocketException이 발생한다. 이 때 에러메시지는 "Connection Reset"이다. ("Connection reset" 에러 메시지는 클라이언트 혹은 서버가 커넥션을 close() 메소드를 통해 정상적으로 닫지 않은 경우에 발생한다) 혹은 연결이 닫힌 소켓을 이용해서 데이터를 읽거나 쓰려는 경우에도 SocketException이 발생한다. 이 때 에러 메시지는 "Socket is closed"이다. 이렇게 SocketException은 에러메시지를 통해 예외가 발생한 상황에 대한 정보를 제공한다. 

### 참고

- [Java Exception Handling](https://airbrake.io/blog/java-exception-handling/sockettimeoutexception)
- [Java Exception Handling](https://airbrake.io/blog/java-exception-handling/java-socketexception)



