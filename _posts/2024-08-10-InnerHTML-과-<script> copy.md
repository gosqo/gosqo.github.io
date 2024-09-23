---
layout: post
title: Element.innerHTML 과 <script></script>
author: gosqo
published: false
---

Spring Boot, JavaScript 기반

[개인 프로젝트](https://flyin-heron.duckdns.org)에 게시물 본문 중 url 형식의 문자열이 있다면, 하이퍼 링크로 변경하는 기능을 추가했다.   

기능에 대해서 자세히 하자면,
1. 서버로 부터 뷰 템플릿(`.html` 파일)을 가져온다. ()
2. 템플릿이 호출하는 `.js` 파일들이 로드된다.
3. 템플릿에 필요한 데이터를 가져오기 위해, `.js` 에 작성해둔 메서드, 함수들이 필요한 데이터를 담당하는 서버 API 에 요청을 보낸다.
4. 성공 응답을 받은 객체의 값을 템플릿 적재적소에 위치시킨다.   
  - 이때, 게시물, 댓글 본문은 링크 형식을 가진 문자열을 하이퍼링크로 변환해 Element.innerHTML 을 사용.
  - 이외의 경우, Element.textContent 를 통해 데이터를 위치.

기능 내부에 Element.innerHTML 을 사용했다. 이후 기능이 잘 동작하는지 테스트 하다가, 스크립트가 들어가면 위험할 수 있겠는데? 라는 생각이 들었다.   
그래서 직접 `<script></script>` 태그를 넣어서 확인해보니, 우려와 달리 스크립트는 실행되지 않았다.

우선적으로 innerHTML 로 입력한 `<script></script>` 태그의 실행이 되지 않는 이유로 보이는 브라우저의 동작 순서를 알아봐야겠고,   
브라우저 스토리지에는 회원 식별을 위한 최소한의 정보가 존재하니, 이것에 대한 보호를 위해 XSS(Cross site scripting) 공격에 대해서 알아보자.
