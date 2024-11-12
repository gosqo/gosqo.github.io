---
layout: post
title: SSL 적용한 Nginx, React 앱으로 프록시 패스 시, 브라우저 - React 앱 웹소켓 형성 실패 시
author: gosqo
categories: React
---

## 목차
1. [사례의 배경](#사례의-배경)
1. [문제 발생](#문제-발생)
1. [원인 추적](#원인-추적)
    1. [웹 소켓 연결 실패의 원인](#웹-소켓-연결-실패의-원인)
    1. [웹 소켓 연결 정보](#웹-소켓-연결-정보)
    1. [웹 소켓 연결 정보 설정](#웹-소켓-연결-정보-설정)
1. [마치며](#마치며)
1. [번외](#번외)

개발환경에서 react 앱을 프론트 서버로 사용하려는 시도 중, 브라우저 - react 앱 간의 웹소켓 형성 실패 사례와 해결방법을 남깁니다.

<br />

## 사례의 배경

서버로 향하는 요청을 받는 리버스 프록시 서버 nginx에 SSL을 적용해 모든 요청을 HTTPS 프로토콜로 처리고 있습니다.  
(포트 80 에 들어오는 요청은 포트 443 으로 리다이렉트)

react 앱을 프론트 서버로 둘 것이기에, `/` 아래의 요청은 리액트 앱이 동작하는 포트 3000 으로 프록시 패스합니다.

```conf
// nginx.conf

// ...
upstream react-server {
    server 127.0.0.1:3000;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /path/to/cert.crt;
    ssl_certificate_key /path/to/priv.key;

    location / {
        proxy_pass http://react-server; // 내부적으로는 http 프로토콜로 패스.

        proxy_http_version 1.1;
        proxy_set_header Host $host;

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "keep-alive";
    }
}

    // ...

server {
    listen 80;
    server_name localhost;

    location "/" {
        return 301 https://$host$request_uri;
    }
}

```

nginx.conf 변경 사항 적용,  
포트 3000 에 react 앱 구동,
이후, 브라우저를 통해 `http[s]://localhost/` 로 요청을 보냅니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

<br />

## 문제 발생

브라우저 뷰에 리액트 앱의 응답이 잘 나타납니다.  
하지만 에디터 수정 시, 수정 사항을 곧바로 반영하지는 못합니다.  
브라우저 콘솔에는 다음의 에러 메시지가 출력됩니다.

```
WebSocket connection to 'wss://localhost:3000/ws' failed: Error in connection establishment: net::ERR_SSL_PROTOCOL_ERROR        WebSocketClient.js:13
```

에러가 발생한 구간인 `WebSocketClient.js:13` 부근의 코드를 살펴보면 다음과 같습니다.

```javascript
// WebSocketClient.js

function WebSocketClient(url) {
  _classCallCheck(this, WebSocketClient);
  this.client = new WebSocket(url);
  this.client.onerror = function (error) {
    log.error(error);
  };
}
```

`create-react-app(cra)` 으로 생성한 프로젝트에는 `webpack`, `Babel` 등이 사전 설정되어 있고 감춰져 있습니다.  
사전 설정에 포함된 `webpack-dev-server` 가 개발 중 발생하는 코드의 변경 사항을 실시간으로 브라우저에 업데이트 해주는데요.

이 같은 실시간 업데이트는 브라우저와 앱(개발하고 있는 프로그램) 사이 웹 소켓 연결을 통해 가능해집니다.  
위에서 확인한 에러 메시지는 해당 역할을 하는 브라우저(클라이언트)-앱(서버) 간의 소켓 연결이 실패했다고 안내합니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

<br />

## 원인 추적

디버그 툴을 통해 인자로 받는 `url`을 살펴보면 `wss://localhost:3000/ws` 를 받고 있음을 확인할 수 있습니다.

문제를 해결하기 위해서는

- 이 `url` 의 연결이 왜 실패하는 지를 알아야 하고,
- `url` 이 어떻게 형성되는 지와 어떻게 설정할 수 있는 지를 알아야 합니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

<br />

### 웹 소켓 연결 실패의 원인

해당 url 연결이 실패하는 이유는 프로토콜이 달라서 였습니다.

브라우저는 https 프로토콜로 리버스 프록시와 통신을 했기에, 서버와의 웹 소켓 통신을 위해 wss 프로토콜을 사용하는 것이 기본값인 것으로 보입니다.

> wss 는 https 와 같이 secure 가 더해진 웹 소켓 프로토콜로,   
> http 에는 ws 프로토콜이 대응됩니다.

실제 3000 포트가 통신하는 프로토콜은 HTTP 입니다.

```conf
    location / {
        proxy_pass http://react-server; // 내부적으로는 http 프로토콜로 패스.

        // ...
    }
```

실제 리액트 앱은 `http://localhost:3000` 에서 동작하고 있기 때문에 ws 프로토콜로 요청하면 웹 소켓 연결이 될 것이라 예상됩니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

<br />

### 웹 소켓 연결 정보

웹 소켓 연결 대상인 `url`을 형성은 다음과 같이 이뤄집니다.

조사와 시행착오를 통해 기본적으로 해당 `url`은  
브라우저의 window.location 기반 `protocol`, `hostname`과 포트 3000 과 pathname `/ws`을 고정적으로 사용하고 있음을 알 수 있었습니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

<br />

### 웹 소켓 연결 정보 설정

이러한 설정이 `webpack.config.js` 에서 가능하다는 것을 알았습니다.

`cra` 로 생성한 앱의 경우 해당 설정은 감춰져 있는 것으로 알고 있습니다. 해당 설정을 덮어쓰기 위해서는 `create-react-app configuration override (craco)`를 사용하면 됩니다.

> `craco` 는 `cra` 로 생성한 앱의 설정을 덮어쓰는 라이브러리입니다.

다음의 명령을 통해 `craco` 의존성을 추가합니다.
```
$ npm install @craco/craco
```

package.json 내부 scripts 를 다음과 같이 수정합니다.

```json
// package.json

// ...
"scripts": {
"start": "craco start", // react-scripts -> craco
"build": "craco build",
"test": "craco test",
"eject": "craco eject"
},
// ...
```

프로젝트 루트에 `craco.config.js` 파일을 추가해 다음과 같이 작성해 기존의 설정을 덮어씁니다.

```javascript
// craco.config.js

module.exports = {
  devServer: {
    client: {
      webSocketURL: {
        protocol: "ws", // 기본값: "ws" // 단독 사용 시 설정 무시됨. hostname 등 다른 정보와 혼합하면 인식.
        hostname: "localhost", // 기본값?: window.location.hostname
        port: 3000, // 기본값: "3000"
        pathname: "/ws", // 기본값: "/ws"
      },
    },
  },
};
```

위와 같이 설정을 추가후 다시 리액트 앱을 실행하면  
웹 소켓 연결 요청은 wss 프로토콜이 아닌 ws 프로토콜을 사용.  
url `ws://localhost:3000/ws`로 요청을 보내고, 성공적으로 웹 소켓 연결이 됩니다.

> 웹 소켓 연결에 사용하는 `url` 은 `protocol://hostname:port/pathname` 의 형식으로 지정합니다. (포트가 0 이면 프로토콜에 따라 80, 443 으로 가는 것으로 보입니다.)

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

<br />

## 마치며

리액트, node 라이브러리를 처음 접하다보니 시행착오가 많았는데, 결과적으로는 네트워크를 더 잘 이해해야겠다는 생각도 듭니다.

<br />

---

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

<br />

## 번외 (약간 다른 접근)

해당 글을 작성하면서 프로토콜과 포트에 대해 곱씹어보니, 아래와 같은 방법을 떠올릴 수도 있었습니다.

`craco.config.js` 에서 `devServer.client.webSocketURL.하위 정보`를 다음과 같이 설정,  
nginx.conf 에 아래 코드를 추가 하면 리버스 프록시를 통해서도 웹 소켓 연결이 가능함을 알 수 있었습니다.

- wss://localhost:443/ws -> 아래 코드 입력으로 리버스 프록시 통해 연결 가능
- wss://localhost/ws (port: 0) -> wss 프로토콜로 443 포트 직행 위와 같이 성공

- ws://localhost:80/ws -> 301 redirect 응답으로 연결 실패.

```conf
# nginx.conf

# ...
upstream react-server {
    server 127.0.0.1:3000;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /path/to/cert.crt;
    ssl_certificate_key /path/to/priv.key;

    location /ws {                          # 추가
        proxy_pass http://react-server/ws;  # "/ws" 요청 리액트 앱에서 처리

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# ...

server {
    listen 80;
    server_name localhost;

    location "/" {
        return 301 https://$host$request_uri;
    }
}

```

저는 설정 편의와 직관성이 중요하다고 생각해서 본문에서 다룬 `ws://localhost:3000/ws` 로 설정하는 것을 선택했습니다.

처음 접하는 환경의 문제 해결에 초점을 두어 글을 작성했습니다.  
모자란 부분이 있을텐데요. 해당 부분을 알려주시거나 힌트를 주시면 공부해서 다듬어보겠습니다.

감사합니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>
