---
layout: post
title: Hello Jekyll
author: gosqo
date: 2024-08-08 19:00 # date 에 적힌 정보가 파일명에 적힌 일자보다 우선한다.
---

게시물 작성할 때
---

`_post` 디렉토리에 `yyyy-MM-dd-title-goes-to-here.md[markdown]` 형식으로 파일을 생성하면,   
* 이 `.md`, `.markdown` 파일의 내용에 기반해서 Jekyll 이 HTML 파일을 생성. 웹 사이트에 렌더링.   
* 제목을 한글로 쓰면 렌더링, URI 또한 한글로 잘 적용된다.
* 파일명에 `yyyy-MM-dd` 형식이 빠지면, 포스트로 인식하지 않고 html 생성, 렌더링 되지 않는다.

Front Matters
---

`_post` 아래 Markdown 파일 작성시 최상단에 아래와 같이 해당 파일의 정보를 입력한다.   

\-\-\-   
**layout**: post   
**title**: 제목 # 제목은 사이트 렌더링에 제목으로 사용되고, 
><small>`<html>``<head>` 태그 아래 `<title>`태그의 textContent 로 쓰인다.</small>   

**data**: yyyy-MM-dd hh:mm # 까지만 해도 인식. 필요하다면 `yyyy-MM-dd hh:mm:ss +0900` [UTC+] 의 형식으로   
**author**: 문서 작성자   
**categories**:   
> &emsp;Super Sub Sub

&emsp;해당 value 입력 시, 계층 구조 좌 -> 우 구분은 **공백**

혹은
>
&emsp;\- Super   
&emsp;\- Sub   
&emsp;\- Sub   

&emsp;형태로도 지정. 계층 구조는 위 -> 아래   
`yyyy/MM/dd/super/sub/sub/title` 의 구조로 정리된다

**category**: 단수형으로 카테고리 하나만 설정. 이 경우 공백을 카테고리 명으로 포함.
\-\-\-   

**Front Matters 에 쓰인 정보가 우선적으로** 적용,   
선언하지 않는다면 다음과 같이 적용됨.
* title: markdown 파일명의 일자 이후에 오는 제목을 Title Case (영문) 로 적용한다. 한글도 가능. (파일명의 '-' 가 띄어쓰기로 치환.)
* date: 파일명에 명시된 일자를 적용한다.

livereload
---
프로젝트 루트 폴더 `bundle exec jekyll serve --livereload` 명령 시, 실시간으로 편집한 내용을 반영한다. 
* markdown 파일명 변경, Front Matters 변경 등 실시간으로 반영.
* 프로젝트 전체를 아우르는 설정인 `_config.yml` 의 업데이트는 CLI `ctrl+c` 로 프로세스를 끝내고,   
해당 **명령을 다시 입력**하면 적용된다.

_config.yml
---

프로젝트 전역적 설정.
* **title**: 사이트 자체의 이름이 된다.
* **theme**: 사이트에 적용할 theme.
* **permalink**: URI(path) 구조 정의   
  * 설정이 꽤나 흥미로웠는데,   
  기본값은 `/:categories/:year/:month/:day/:title:output_ext` 의 형식이다.   
  여기서 `:categories`, `:year` 등 순서를 변경 후 프로세스를 실행하면,   
  `_post` 아래 `.md`, `.markdown`파일 기반 생성한 HTML 디렉토리 구조가 이에 따라 변한다.   
  **URI 역시 이 구조를 따르게 된다**.   
  * 마지막 `:output-ext` 는 파일 확장자로, 이 것이 보기 싫다면 다음과 같이 제거가 가능하다.   
    * `permalink: pretty` 를 사용. (URI 마지막에 '/' 가 붙음.)
    * `permalink: /:categories/:year/:month/:day/:title` 를 사용. (URI 마지막이 타이틀만으로 끝. 현재 적용 중.)

---
---
---

# markdown test.

Good to know you ***Jekyll***.   

<p style="color:green;">static web-site</p>

```java
public class Hello {
    private String name;

    public Hello(String name) {
        this.name = name;
    }

    public sayHello() {
        System.out.printf("Hello %s%n", Objects.requireNonNull(this.name));
    }
}
```

we can write down like [**this**](https://www.markdownguide.org/cheat-sheet/#basic-syntax){:target="_blank"}.

---
---
---

20240817 추가   

markdown 이미지 삽입 및 디렉토리
---

`루트_아래/특정_폴더`를 만들어 해당 폴더를 향하도록 path 를 `![image_title](/path/to/image.extension)` 으로 입력해 참조한다.
* jekyll 문서 참조 중 _site 아래 assets 와 혼동할 수 있는데, _site 아래 자원들은 Jekyll 이 동적으로 관리하는 공간이므로, 폴더를 생성하더라도, 리빌드하면 사라진다.   
-> root 폴더 바로 아래 assets 디렉토리를 만든다.

