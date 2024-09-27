---
layout: post
title: visibilitychange Event 는 페이지를 떠나는 중에 Promise 내용을 확인할 수 있을까요
author: gosqo
categories: Web-App
---

프로젝트 클라이언트 코드에는 `visibilitychange Event.visibilityState === 'hidden'`(이하 `visibility hidden`)을 트리거로 동작하는 `Fetch API`요청이 있습니다. 해당 기능을 적용하면서 알게된 내용을 기록합니다.

## 배경

페이지에서 일어난 자원 상태 수정 요청을 모아뒀다 페이지를 떠날 때 서버로 일괄 요청하는 의도를 기획했습니다.
목적은 페이지를 떠날 때 확정된 자원 상태를 반영하기 위함입니다. 
* 페이지 안에서 자원 수정 이벤트는 여러번 발생할 수 있지만, 
* 페이지를 떠나는 시점의 자원 상태가 확정된 수정 요청이라고 여겼습니다.
    * 만약 수정 이벤트 마다 서버로 요청을 보내면, 서버 자원 활용 측면에서 비효율적이라고 생각했습니다.

기존에 `beforeunload Event`를 적용해 작업했지만, 모바일 브라우저에서는 인식하지 못하는 문제가 발생했습니다.
해당 문제를 해결하기 위해 다른 이벤트를 찾아 봤고, 대안으로 선택한 이벤트는 `visibility hidden` 입니다.

`visibilitychange` 는 다음의 상황에서 발생합니다.
> 사용자가 새로운 페이지로 이동하거나, 탭을 바꾸거나, 탭을 닫거나, 브라우저를 닫거나 최소화하거나, 모바일 기기에서는 다른 앱으로 전환하는 경우에는 visibilityState가 hidden으로 바뀌고 이 이벤트가 발생합니다.
> <div dir=rtl><a href="https://developer.mozilla.org/ko/docs/Web/API/Document/visibilitychange_event#사용_일람" target="_blank">mdn web docs</a></div>

`visibility hidden`을 선택한 이유는 데스크탑, 모바일 환경에서 안정적으로 인식하기 때문입니다.

## 도입하면서

해당 이벤트 핸들러로, 새로고침 시, 서버에 비동기 요청을 보내는 `Fetch API` 가 정상적으로 작동하지 않는 문제가 발생했습니다.
(페이지를 켜둔 채, 다른 탭으로 이동하는 경우(`visibility hidden` 경우 중 하나)에는 잘 작동합니다.)

새로고침의 경우 아래와 같은 에러가 발생하며 요청을 보내지 못합니다.

```error
Fetch API cannot load <url> due to access control checks.
```

원인은 새로고침 시, 페이지를 벗어나며 수명이 다한 것으로 간주, Fetch API 에서는 이 요청을 접근 제어 대상으로 보는것 같습니다.

에러 메세지 관련 검색 중, 제가 의도한 방식을 위해 이전에 고안된 `Navigator.sendBeacon() API` 문서를 보다 아래 내용을 읽게 됐습니다.

> keepalive   
>> 요청이 페이지 수명보다 오래 지속되는 걸 허용합니다. keepalive 플래그는 Navigator.sendBeacon() API의 대체제입니다.   
> <div dir=rtl><a href="https://developer.mozilla.org/ko/docs/Web/API/Window/fetch#keepalive" target="_blank">mdn web docs</a></div>

따라서 fetch option 에 해당 속성을 추가하면 문제가 해결됩니다.

```javascript
let options = {
    method: "POST",
    headers: {
        "Authorization": localStorage.getItem("access_token"),
    },
    // visibilitychange 이벤트 핸들러 visibilityState === "hidden"상태에서
    // Fetch API 호출, 새로고침, 탭 종료 등 페이지 수명이 다하는 경우 요청이 지속되도록 허용
    keepalive: "true"
};

```

이제 새로고침, 페이지 종료(브라우저 탭 닫기)에도 요청이 서버로 잘 가게 됩니다.

## 본 문제

하지만 또 문제가 있었습니다.   
클라이언트 인증 정보가 만료가 된 경우, 재인증, 기존 요청 재시도가 이뤄지지 않는 문제입니다.

서버 자원 상태 수정 요청의 경우 서버에서 발급한 `JWT`를 요청 헤더에 실어 사용자 인증, 인가 확인 후 처리합니다.
기본적으로 인증이 필요한 요청은 

* `JWT`가 만료된 경우에 대비, 
    * 401 응답에 대응해 refresh 요청, 
    * 기존 요청 재시도 

로직을 담은 메서드로 요청을 보내는데요.   

우연찮게 액세스 토큰이 만료된 시점에 페이지를 떠나며 요청을 보내게 됐습니다.
* 서버 로그에는 최초 요청(만료된 토큰을 헤더에 담은) 기록이 남았지만,
* refresh 요청과 기존 요청 재시도의 기록은 남지 않았습니다.   
* 자원 상태 역시 변하지 않았습니다.

해당 문제를 해결하기 위해 이벤트 핸들러에 `JWT` refresh 로직을 담기도 해봤지만 비동기식(동시다발적)으로 요청이 수행되기에 **갱신된 토큰을 헤더에 담지 못했습니다**. 해당 방법은 효과가 없었습니다.

사실 위 문장에 언급한 **갱신된 토큰**이란 페이지를 떠난 이후에 받을 수 없습니다. 
왜냐면 페이지를 떠나며 수행된 `fetch`는 `await` 혹은 `then()` 이후의 동작을 보장하지 않기 때문입니다.

다시 말해, 서버 응답 데이터를 사용해 조치를 취할 수는 없었습니다.

`keepalive: "true"` 속성 추가 시, 요청이 지속되어 `fetch` 응답 `Promise` 객체가 이행되면 이것을 사용할 수 있을 거라 생각했지만 **페이지 수명이 다해 갈 때, 짧은 시간 내에 요청을 보내는 것을 허용**할 뿐, `await`, `then()` 이후의 후속 조치를 할 수 있는 것은 아니었습니다.

여기서 이 글 제목의 물음에 답하자면, `visibility hidden`의 경우,   
**`await` 을 만나는 즉시, 이후 코드를 읽지 않았습니다.**

> 위 사실을 알기 위해 해당 이벤트 핸들러에 브라우저 개발자 도구 브레이크 포인트를 걸어봤습니다.   
Chrome 브라우저는 `visibility hidden` 상태에서 브레이크 포인트가 활성화되지 않았고,   
Safari 개발자 도구에서는 브레이크 포인트 **활성이 가능했습니다.**   
>
> ![safari breaks after visibility hidden](/assets/img/2024-09-27-visibilitychange-Event-페이지를-떠나는-중에-Promise-내용을-확인할-수-있을까요/safari-breaks-after-visibility-hidden.png)
> 
> Safari 개발자 도구를 통해 `await` 이후의 코드를 읽지 않음을 알 수 있었습니다.


때문에 요청 결과인 `await`으로 받은 응답을 사용하는 후속 작업이 불가했습니다.

## 본 문제 해결 방안

후속 작업이 불가해서 예외가 발생하고, 이 사실을 알고 있다면,   
먼저 예외를 처리 해주는 방법을 선택했습니다.

사전 예외 처리를 위해 고려할 사항은 다음과 같습니다.
* 액세스 토큰의 만료는 발급 후 30 분 입니다.
* 게시물, 댓글을 보는 데에 5 분 정도의 시간이 소요된다고 가정합니다.

사전 예외 처리 구현 방향은 다음과 같습니다.
* 해당 페이지 로드할 때, 액세스 토큰의 만료시점을 확인합니다.
    * JWT 는 base64 인코딩 되어있습니다. 클라이언트 측에서도 자바스크립트로 이것을 디코딩해 토큰 만료시점을 확인합니다.
* 만료 시간이 5 분 이내로 남았다면 리프레시 요청을 보냅니다.
* 5 분 이상 남았다면, 남은 만료시간 - 5 분 후에 리프레시 요청을 보냅니다.
* 더 오래 페이지에 머무를 상황을 대비해, 25 분의 간격으로 리프레시 요청을 보냅니다.

구현한 코드는 다음과 같습니다.   
페이지 로드 시점에 다음의 메서드 호출을 추가하는 것으로 작업을 마무리했습니다.

```javascript
static forceReissueAccessToken() {
    const accessToken = localStorage.getItem("access_token");
    const parsed = this.parseJwt(accessToken);
    const expiry = parsed.exp * 1000; //javascript epochTime 단위는 milliseconds
    const now = Date.now();
    const toExpiry = expiry - now;
    const fiveMinutes = 1000 * 60 * 5; // 5 mins
    const updateTimeout = toExpiry - fiveMinutes; // minus 5 mins.
    const intervalTimeout = 1000 * 60 * 25; // 25 mins

    if (toExpiry < fiveMinutes) { // 30 mins left
        Fetcher.refreshBeforeAuthRequiredRequest();

        setInterval(() => {
            Fetcher.refreshBeforeAuthRequiredRequest();
        }, intervalTimeout)
        return ;
    }

    setTimeout(() => {
        Fetcher.refreshBeforeAuthRequiredRequest();
    }, updateTimeout)

    setInterval(() => {
        Fetcher.refreshBeforeAuthRequiredRequest();
    }, intervalTimeout)

}

```

## visibilitychange 사용은

데스크탑, 모바일 환경에서 안정적으로 세션의 종료를 감지하며 페이지 생명주기 마지막에 요청을 보내는 데에 적합합니다.
* 페이지에 머무르는 동안 수집한 정보 전송 목적에 유용합니다.
* 하지만 이 요청이 예외 처리가 필요하다면,
    * 사전에 예외 발생 요인을 제거하는 작업이 필요하겠습니다.

* 혹은 해당 이벤트가 아닌 사용자 상호작용 이벤트, 
    * 혹은 특정 시간 간격으로 서버에 요청을 전송하는 것이 나은 선택이 될 수도 있다고 생각합니다.

## 마치며,

출발점은 확정된 요청만을 서버로 한번만 보내어 서버 자원을 아끼자는 취지였습니다. 이 기획에 따라 처음 보는 문제가 꼬리에 꼬리를 물어 발생해 기획을 바꿀까 싶기도 했지만, 주어진 상황에서 어떻게든 해결한다는 마음가짐으로 작업에 임해 기획 의도대로 수행하는 결과를 만들 수 있었습니다.

어떻게 보면 새로고침이란 특수한 이벤트에 대응하기 위해 위 과정을 거쳤다고 볼 수도 있습니다. 하지만 새로고침뿐 아니라 페이지 이탈 시에도 적용되는 점을 생각하면, 자원 상태 변경 요청을 보다 안정적으로 처리해 데이터 일관성을 유지할 수 있습니다.   
과정을 거치면서 웹 페이지 생명주기에 대한 이해를 더하고, visibilitychange 이벤트에 적합한 요청과 사전 예외 처리를 경험할 수 있었습니다.

이후에 비슷한 상황이 생긴다면, 이번 경험을 떠올리며 보다 효과적으로 처리할 수 있도록 고민해봐야겠습니다.

> 아래는 참고 자료 입니다.
> 
> ***
> 
> ![page being unloaded events and browsers](/assets/img/2024-09-27-visibilitychange-Event-페이지를-떠나는-중에-Promise-내용을-확인할-수-있을까요/page-being-unloaded-events-and-browsers.png)
> [Don't lose user and app state, use Page Visibility: igvita.com](https://www.igvita.com/2015/11/20/dont-lose-user-and-app-state-use-page-visibility/)
> 
> ***
>
> ![page life cycle](https://developer.chrome.com/docs/web-platform/page-lifecycle-api/image/page-lifecycle-api-state.svg)
> [페이지 수명 주기 API](https://developer.chrome.com/docs/web-platform/page-lifecycle-api?hl=ko#developer-recommendations-for-each-state)
> 
> ***
