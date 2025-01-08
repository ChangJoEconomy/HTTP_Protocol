# Read Me

## 프로그램 개요

### 전체 구조

본 프로젝트는 Server-Client 구조로 되어 있습니다

- 서버는 HTTP 요청을 처리하며 4가지(GET, POST, HEAD, PUT) 메서드를 지원합니다.
- 클라이언트는 PyQt5 기반의 GUI로 작성하여 HTTP 요청을 직접 작성하여 전송하거나 테스트 케이스를 선택하여 서버를 점검해 볼 수 있습니다.
- 요청과 응답은 TCP 소켓 프로그래밍을 이용하여 통신합니다.

### 언어 및 버전

- Python 3.8 이상으로 테스트 하였습니다.
- 서버 및 클라이언트 모두 Python으로 작성 되었습니다.

### 개발/실행 환경

OS: Window 11 / Linux(Fedora)

IDE: visual studio code

Compiler: python 기본 인터프리터

## 실행방법

### Python 코드 실행 방법

1. 서버 실행 (server디렉토리에서)
    
    ```bash
    python3 ./server.py
    ```
    
2. 클라이언트 실행 (client 디렉토리에서)
    
    ```bash
    python3 ./client.py
    ```
    

### 실행 순서

1. 먼저 [server.py](http://server.py)를 실행하여 서버를 실행합니다.
2. 이후 client.py를 실행하여 클라이언트에서 서버로 요청을 보냅니다.

## 서버측 주요 코드

### 상수 설정

```python
MAX_URI_LENGTH = 2048
MAX_CONTENT_LENGTH = 30 * 1024 * 1024
MAX_DOWNLOAD_SIZE = 50 * 1024 * 1024
```

- MAX_URI_LENGTH: 허용되는 최대 URI 길이를 나타내며 2048 바이트로 설정하였습니다. 웹 표준에서 흔히 쓰이는 제한 크기로 설정하였습니다.
- MAX_CONTENT_LENGTH: 업로드 최대 가능 크기를 30MB로 제한하였습니다.
- MAX_DOWNLOAD_SIZE: 서버에서 클라이언트로 전송할 수 있는 최대 응답 크기를 50MB로 제한하였습니다.

### 함수 ParseRequest

```python
def parseRequest(connectionSocket):
    # request line + header 파싱
    requestData = b""
    while True:
        chunk = connectionSocket.recv(1)
        if not chunk:
            break
        requestData += chunk
        # Header 끝 라인 검사
        if b"\r\n\r\n" in requestData:
            break
    headerDataStr = requestData.decode(errors="replace")
    lines = headerDataStr.split("\r\n")
```

connectionSocket에서 request 데이터를 수신합니다.

1 바이트씩 소켓에서 데이터를 읽어 \r\n\r\n를 만날 때까지 반복합니다.

\r\n\r\n을 통해 헤더 부분을 구분합니다.

```python
headerDataStr = requestData.decode(errors="replace")
lines = headerDataStr.split("\r\n")
# request line 파싱
request_line = lines[0] if len(lines) > 0 else ""
```

받은 데이터를 문자열로 디코딩하며 오류 발생 시 ‘�’ 문자로 대체합니다.

\r\n를 기준으로 request line과 header를 분리합니다.

```python
# 헤더 추출 (line 2 ~ end line)
headers = {}
for line in lines[1:]:
    # 헤더 끝 검사
    if line == "":
        break
    # 헤더는 "Key: Value" 형태, 딕셔너리에 저장
    parts = line.split(":", 1)
    if len(parts) == 2:
        header_key = parts[0].strip()
        header_val = parts[1].strip()
        headers[header_key.lower()] = header_val
```

헤더를 처리할 때에는 사전 형태로 저장합니다. (key:value)

키는 소문자로 변환하여 딕셔너리로 저장합니다.

```python
if method.upper() in {"POST", "PUT"} and "content-length" not in headers: # POST/PUT 요청인데 content-Length가 없음
    return 411, method, path, httpVersion, headers, body
elif contentLength > MAX_CONTENT_LENGTH:  # 서버의 업로드 제한 response 413 Payload Too Large
    return 413, method, path, httpVersion, headers, body
```

Expect: 100-Continue에서 body를 수신하기 전 문제가 있으면 미리 return 합니다.

```python
expectHeader = headers.get('expect', '').lower()
if expectHeader == '100-continue':
    resp_100 = "HTTP/1.1 100 Continue\r\n\r\n"
    connectionSocket.sendall(resp_100.encode())
```

예외처리에서 문제가 없을 경우 바디를 수신 받습니다. Expect: 100-continue가 있을 경우 100 Continue를 전송 후 클라이언트에서 본문을 수신 합니다.

```python
return 200, method, path, httpVersion, headers, body
```

문제가 없다면 200 상수값과 함께 파싱한 정보들을 return 합니다.

### 함수 makeResponse

```python
def makeResponse(status_code, status_msg, extraHeaders=None, bodyData=None):
    statusLine = f"HTTP/1.1 {status_code} {status_msg}\r\n"
```

status_code/msg의 함수 인자를 기반으로 “HTTP/1.1 200 OK” 와 같은 statusLine를 생성합니다.

```python
if extraHeaders:
  for k, v in extraHeaders.items():
    headerPart += f"{k}: {v}\r\n"
```

추가적인 헤더를 딕셔너리를 순회하며 응답 헤더에 추가 합니다.

```python
if bodyData:
  if isinstance(bodyData, str):
      bodyData = bodyData.encode()
  headerPart += f"Content-Length: {len(bodyData)}\r\n"
```

본문 내용이 문자열이면 바이트로 인코딩 합니다. 이후 헤더에 본문의 길이를 Conent-Length로 추가합니다.

### 함수 handleRequest

```python
def handleRequest(connectionSocket, addr):
```

parseRequest 함수를 이용하여 request를 파싱하고 이에 따라 적절한 예외처리 & makeResponse를 생성하여 응답 후 소켓 종료 등을 수행 합니다.

파싱 정보에 따라 method의 적절한 함수를 수행합니다.

- def methodGetResponse(path, headers):
- def methodPostResponse(path, headers, body):
- def methodHeadResponse(path, headers):
- def methodPutResponse(path, headers, body):

## 클라이언트측 주요 코드

### IP 및 Port 설정

```python
serverName = '106.249.182.41'
serverPort = 8080
```

전역변수에서 서버와 port를 설정합니다.

### GUI 구성

```python
self.testSelector = QComboBox(self)
for i in range(1, 14):
  self.testSelector.addItem(f"TEST #{i}")
```

콤보 박스를 생성하여 1~13번의 테스트 케이스를 직접 선택할 수 있습니다.

```python
self.requestTextBox = QTextEdit(self)
self.requestTextBox.setPlaceholderText("Request Message")
```

테스트 케이스를 이용할 수도 있지만 직접 요청을 작성 할 수도 있는 박스를 제공합니다.

### request 전송 및 reponse 수신

```python
request = request.replace("\r\n", "\n")
request = request.replace("\n", "\r\n")
```

pyqt의 텍스트 박스에 텍스트를 저장할 때 \r이 소실 되는 문제가 있어 \n를 \r\n으로 치환하는 작업을 추가 하였습니다.

```python
if "Expect: 100-Continue" in request:
    clientSocket.send((request.split("\r\n\r\n")[0]+"\r\n\r\n").encode())
    response = clientSocket.recv(4096).decode()
    self.responseTextBox.setPlainText(response)
    if "100 Continue" in response:
        clientSocket.send(request.split("\r\n\r\n")[1].encode())
    else:
        clientSocket.close()
        return
else:
    clientSocket.send(request.encode())
```

클라이언트 소켓을 생성하고 서버(8080포트)와 연결 후 전송합니다.

만약 본문에 "Expect: 100-Continue"가 있다면 body를 분리하여 header만 전송합니다. 이후 서버로부터 100 continue 전송이 오면 그때 body를 전송합니다.

만약 100 continue가 아닌 다른 내용이 온다면 화면에 해당 response 표시 후 body를 전송하지 않고 함수를 종료합니다.

## **HTTP 요청/응답 확인 내용**

### 기본 예외처리

- 411 Length Required
    - POST/PUT 요청이면서 request header에 content-Length가 없는 경우
- 413 Payload Too Large
    - content-Length가 MAX_CONTENT_LENGTH를 넘어서는 경우
- 400 Bad Request
    - HTTP 1.1의 경우 header에 host가 필수적이지만 누락된 경우
- 414 URI Too Long
    - URI의 길이가 MAX_URI_LENGTH를 넘어서는 경우
- 405 Method Not Allowed
    - {"GET", "POST", "PUT", "HEAD"} 이렇게 4가지 메서드에 해당하지 않는 경우
- 505 HTTP Version Not Supported
    - 서버는 HTTP 1.1를 사용하는데 버전이 불일치할 경우
- 400 Bad Request
    - content-length와 body 사이즈가 불일치 할 경우

### GET

- 200 OK
    - 가장 기본적인 정상 통신 환경
    - ‘/’ 경로 요청 시 기본인 /index.html를 읽어서 응답합니다.
- 301 Moved Permanently
    - 옛날 리소스를 요청하는 경우 (old_page로 그 상황을 연출함)
- 404 Not Found
    - GET를 하는 경로가 서버에 없는 경우

### POST

- 200 OK
    - 가장 기본적인 정상 통신 환경
- 400 Bad Request
    - content-type이 누락된 경우
- 404 Not Found
    - POST 대상 경로가 존재하지 않을 경우
- 500 Internal Server Error
    - 서버 내부적인 에러가 발생한 경우

### HEAD

GET 요청과 유사하지만 본문을 제외하고 응답합니다.

- 200 OK
    - 가장 기본적인 정상 통신 환경
- 301 Moved Permanently
    - 옛날 리소스를 요청하는 경우 (old_page로 그 상황을 연출함)
- 404 Not Found
    - HEAD를 하는 경로가 서버에 없는 경우

### PUT

- 200 OK
    - 가장 기본적인 정상 통신 환경
- 400 Bad Request
    - content-type이 누락된 경우
- 404 Not Found
    - PUT 대상 경로가 존재하지 않을 경우
- 500 Internal Server Error
    - 서버 내부적인 에러가 발생한 경우

## **실행 화면 및 결과 wireshark 분석**

클라이언트 GUI를 통해 아래와 같이 13가지 상황을 테스트 할 수 있습니다.

- GET 200 OK
    - get를 이용하여 기본 홈페이지를 get 요청 하였고 서버가 이에 따라 200 OK와 함게 index.html를 본문에 담아 전송한것을 wireshark 및 GUI를 통해 알 수 있다.
    
    ![image.png](b4c153c1-0307-4ee2-bbb4-c2e41614e5ae.png)
    
    ![image.png](image.png)
    
    ![image.png](image%201.png)
    
- GET 400 Bad Request
    - HTTP 1.1에서 필수적인 HOST를 누락하였기에 서버에서 400 Bad request를 응답하였다.
    
    ![image.png](8ffddab7-600a-42f1-86eb-122b991bde87.png)
    
    ![image.png](image%202.png)
    
    ![image.png](image%203.png)
    
- GET 505 HTTP Version Not supported
    - 서버는 HTTP 1.1를 사용하는데 request의 버전과 일치하지 않아 505 HTTP Version Not supported를 응답하였다.
    
    ![image.png](e995ad29-03ef-4315-aa0a-7feb89dbdf6b.png)
    
    ![image.png](image%204.png)
    
    ![image.png](image%205.png)
    
- GET 404 Not Found
    - 서버에서 /NotFoundError라는 디렉토리를 찾을 수 없기 때문에 404 NOT Found로 응답 하였다.
    
    ![image.png](163cf28e-df82-4bd2-9fd5-2968da44ec75.png)
    
    ![image.png](image%206.png)
    
    ![image.png](image%207.png)
    
- GET 301 Moved Permanently
    - 클라이언트 측에서 오래된 페이지를 요구 하였기에 서버에서 301 Moved Permanently로 최신 경로를 알려주는 응답을 전송 하였다.
    
    ![image.png](e4b287f9-ae23-4126-b056-5b492d8b1822.png)
    
    ![image.png](image%208.png)
    
    ![image.png](image%209.png)
    
- POST 100 continue & 200 OK (100 ok 도착 확인 후 body 전송 이후 200 OK 응답)
    - 먼저 request에 Expect: 100-Continue가 포함되어 있었기 때문에 서버에서 100 Continue를 통해 받을 준비가 되었음을 알리고 클라이언트에서 body를 전송하여 최종적으로 200 OK 응답이 클라이언트로 전송 되었다.
    
    ![image.png](6a6538f2-97ce-474c-a079-593dc25e5d46.png)
    
    ![image.png](image%2010.png)
    
    ![image.png](image%2011.png)
    
    ![image.png](image%2012.png)
    
- POST 411 Length Required (post인데 content-length 누락)
    - POST method인데 필수적인 content-length를 누락하였기에 서버에서 411 Length Required를 전송하였다.
    - 미리 예외처리를 통해 100-continue를 보내지 않았기 때문에 클라이언트도 body를 전송하지 않았다.
    
    ![image.png](681116da-ef42-4bc1-b6c6-1ec4e1b99a1a.png)
    
    ![image.png](image%2013.png)
    
    ![image.png](image%2014.png)
    
- POST 404 Not Found
    - POST 대상 경로를 서버에서 찾을 수 없었기 때문에 서버에서 404 Not Found로 응답 하였다.
    
    ![image.png](266c64e9-48f3-4711-9553-ded3f46b7fc9.png)
    
    ![image.png](image%2015.png)
    
    ![image.png](image%2016.png)
    
    ![image.png](image%2017.png)
    
- HEAD 200 ok
    - GET과 유사하지만 HEAD 명령어 이기 때문에 서버에서 본문을 제외하고 header부분만 전송 하였다.
    
    ![image.png](8f729136-8d30-4582-bf2f-5073184ff851.png)
    
    ![image.png](image%2018.png)
    
    ![image.png](image%2019.png)
    
- HEAD 404 not found
    - HEAD 대상 경로를 찾을 수 없기 때문에 404 Not Found를 응답 하였다.
    
    ![image.png](efdb37ee-fefa-41ea-aa4f-1ee35228356d.png)
    
    ![image.png](image%2020.png)
    
    ![image.png](image%2021.png)
    
- HEAD 301 Moved Permanently
    - GET 케이스와 동일하게 옛날 경로를 접근하려 하여 서버에서 최신 경로를 알려주는 301 Moved Permanently를 응답 하였다.
    
    ![image.png](0d9a4858-eaac-445a-a83a-f5d74aa2137e.png)
    
    ![image.png](image%2022.png)
    
    ![image.png](image%2023.png)
    
- PUT 200 ok
    - PUT 명령어를 통해 /texts/user_file를 생성하였고 200 OK 응답을 받았다.
    
    ![image.png](850adf89-dcc2-4fb8-8c3a-7531e6ac3699.png)
    
    ![image.png](image%2024.png)
    
    ![image.png](image%2025.png)
    
    ![image.png](image%2026.png)
    
- PUT 404 Not Found
    
    PUT 대상 경로를 찾을 수 없어 404 Not Found로 응답 하였다.
    
    ![image.png](b44a9e21-4b49-4a19-9323-ce1dff8668e3.png)
    
    ![image.png](image%2027.png)
    
    ![image.png](image%2028.png)
    
    ![image.png](image%2029.png)

## 이미지 예시
![b4c153c1-0307-4ee2-bbb4-c2e41614e5ae.png](./b4c153c1-0307-4ee2-bbb4-c2e41614e5ae.png)
![image.png](./image.png)
![image 1.png](./image 1.png)
![8ffddab7-600a-42f1-86eb-122b991bde87.png](./8ffddab7-600a-42f1-86eb-122b991bde87.png)
![image 2.png](./image 2.png)
![image 3.png](./image 3.png)
![e995ad29-03ef-4315-aa0a-7feb89dbdf6b.png](./e995ad29-03ef-4315-aa0a-7feb89dbdf6b.png)
![image 4.png](./image 4.png)
![image 5.png](./image 5.png)
![163cf28e-df82-4bd2-9fd5-2968da44ec75.png](./163cf28e-df82-4bd2-9fd5-2968da44ec75.png)
![image 6.png](./image 6.png)
![image 7.png](./image 7.png)
![e4b287f9-ae23-4126-b056-5b492d8b1822.png](./e4b287f9-ae23-4126-b056-5b492d8b1822.png)
![image 8.png](./image 8.png)
![image 9.png](./image 9.png)
![6a6538f2-97ce-474c-a079-593dc25e5d46.png](./6a6538f2-97ce-474c-a079-593dc25e5d46.png)
![image 10.png](./image 10.png)
![image 11.png](./image 11.png)
![image 12.png](./image 12.png)
![681116da-ef42-4bc1-b6c6-1ec4e1b99a1a.png](./681116da-ef42-4bc1-b6c6-1ec4e1b99a1a.png)
![image 13.png](./image 13.png)
![image 14.png](./image 14.png)
![266c64e9-48f3-4711-9553-ded3f46b7fc9.png](./266c64e9-48f3-4711-9553-ded3f46b7fc9.png)
![image 15.png](./image 15.png)
![image 16.png](./image 16.png)
![image 17.png](./image 17.png)
![8f729136-8d30-4582-bf2f-5073184ff851.png](./8f729136-8d30-4582-bf2f-5073184ff851.png)
![image 18.png](./image 18.png)
![image 19.png](./image 19.png)
![efdb37ee-fefa-41ea-aa4f-1ee35228356d.png](./efdb37ee-fefa-41ea-aa4f-1ee35228356d.png)
![image 20.png](./image 20.png)
![image 21.png](./image 21.png)
![0d9a4858-eaac-445a-a83a-f5d74aa2137e.png](./0d9a4858-eaac-445a-a83a-f5d74aa2137e.png)
![image 22.png](./image 22.png)
![image 23.png](./image 23.png)
![850adf89-dcc2-4fb8-8c3a-7531e6ac3699.png](./850adf89-dcc2-4fb8-8c3a-7531e6ac3699.png)
![image 24.png](./image 24.png)
![image 25.png](./image 25.png)
![image 26.png](./image 26.png)
![b44a9e21-4b49-4a19-9323-ce1dff8668e3.png](./b44a9e21-4b49-4a19-9323-ce1dff8668e3.png)
![image 27.png](./image 27.png)
![image 28.png](./image 28.png)
![image 29.png](./image 29.png)
