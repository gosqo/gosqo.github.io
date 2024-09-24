---
layout: post
title: Page, Slice 인터페이스 타입으로 응답한 JSON 역직렬화 (@SpringBootTest, TestRestTemplate, Page<T>, Slice<T>)
author: gosqo
categories: SpringBoot Test
---

개인 프로젝트 개선 작업과 테스트 중 마주친 JSON 객체 역직렬화에 대해 기록을 남깁니다.   
학습하는 단계에서 문제를 해결하며 기록한 글이라 이해와 설명이 부족한 부분이 있습니다.

[작업 내용](https://github.com/gosqo/manidues/commit/cd796fb918f00887cc048150e6cb3b9272c20e86)

## 배경

여러 개체에 쓰기 작업과 해당 개체와 관련된 개체의 조회 작업에 동시성 문제가 생겨 End-to-End 테스트를 진행하고 있었습니다.
`@SpringBootTest`를 통해 앱 전체 컨텍스트를 사용하는 테스트 중, `Slice` 객체를 응답 본문에 포함하는 엔드포인트에 `TestRestTemplate.exchange()` 로 요청을 보냈습니다.

## 문제

`TestRestTemplate.exchange()` 를 통해 얻은 응답 본문을 읽지 못하는 문제가 발생했습니다.  
오류 메시지는 다음과 같았습니다.

```text
org.springframework.http.converter.HttpMessageConversionException: Type definition error: [simple type, class org.springframework.data.domain.Pageable]

Caused by: com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
Cannot construct instance of `org.springframework.data.domain.Pageable`
(no Creators, like default constructor, exist):
abstract types either need to be mapped to concrete types, have custom deserializer, or contain additional type information
```

추상 타입은 구체 타입으로 매핑되어야 하고, 커스텀 역직렬화나 추가적인 타입 정보가 필요하다고 합니다.

## 원인

오류 메시지 기반 검색을 통해, 원인은 `TestRestTemplate.exchange()` 를 통해 얻은 JSON 데이터를 자바 객체로 매핑하는 Jackson 라이브러리가 인터페이스 타입인 `Slice` 객체를 생성할 수 없기 때문임을 알았습니다.

컨트롤러에서 `SliceImpl` 타입으로 반환하면 잘 찾을 수 있지 않을까 했었는데,
`SliceImpl` 인스턴스를 생성할 때, `Pageable` 객체가 필요합니다. 이 `Pageable` 역시 인터페이스이기 때문에 Jackson 이 대신 생성할 수 없었습니다.

## 해결 과정

* [stack overflow question: Spring Boot Page Deserialization - PageImpl No constructor](https://stackoverflow.com/questions/52490399/spring-boot-page-deserialization-pageimpl-no-constructor)
* [Baeldung: Consuming Page Entity Response From RestTemplate](https://www.baeldung.com/resttemplate-page-entity-response)

두 참고 자료를 통해 추상 타입으로 응답 받은 JSON 데이터를 자박 객체에 매핑하는 방법을 알게 됐습니다.

해당 문제만 확인하기 위한 테스트 코드를 아래와 같이 작성했습니다.

```java
@Test
void getCommentsSlice() {
    Long targetBoardId = boards.get(0).getId();
    final String requestUri = String.format("/api/v1/board/%d/comments?page-number=1", targetBoardId);

    RequestEntity<String> request = buildGetRequestEntity(requestUri);

    ResponseEntity<CustomSliceImpl<CommentGetResponse>> response = template.exchange(
            request,
            new ParameterizedTypeReference<>() {}
    );

    assertThat(response.getBody()).isNotNull();
    assertThat(response.getBody().getSize()).isEqualTo(CommentService.NORMAL_COMMENTS_SLICE_SIZE);
}

```

<!-- <br /> -->


매핑을 위한 커스텀 구체 클래스는 `Slice<T>` 객체를 응답 본문으로 삼고 있기 때문에 약간의 조정을 통해 아래와 같이 `CustomSliceImpl` 클래스를 작성했습니다.   

커스텀 구체 클래스의 생성자는 

* 기본 생성자(NoArgs), 
* 상속을 통한 상위 객체 생성자

를 기본으로 두었는데, 매핑이 적용된 이후 아래 두 생성자를 제거해도 잘 동작이 되고 있습니다.


<!-- <br /> -->

```java
public class CustomSliceImpl<T> extends SliceImpl<T> {

    // JSON 에서 자바 내장 객체, 기본 자료형이 아닌 객체는 JsonNode 로 매개변수를 받고 있었습니다.
    @JsonCreator(mode = JsonCreator.Mode.PROPERTIES)
    public CustomSliceImpl(
            @JsonProperty("content") List<T> content
            , @JsonProperty("pageable") JsonNode pageable
            , @JsonProperty("first") boolean first
            , @JsonProperty("last") boolean last
            , @JsonProperty("size") int size
            , @JsonProperty("number") int number
            , @JsonProperty("sort") JsonNode sort
            , @JsonProperty("numberOfElements") int numberOfElements
            , @JsonProperty("empty") boolean empty
    ) {
        super(content, PageRequest.of(number, size), !last);
    }

   public CustomSliceImpl(List<T> content, Pageable pageable, boolean hasNext) {
       super(content, pageable, hasNext);
   }

   public CustomSliceImpl() {
       super(new ArrayList<>());
   }
}
       

```

커스텀 구체 클래스 생성자 매개변수는 다음을 참고했습니다.

![slice-fields](/assets/img/2024-09-23-Page-Slice-인터페이스-타입으로-응답한-JSON-역직렬화/slice-fields.png)

위와 같이 커스텀 구체 클래스 생성자에 `@JsonCreator`, `@JsonProperty` 애너테이션 적용 후에도 매핑이 되지 않아   
아래와 같이 매핑 모듈을 등록했습니다. (사용할 곳에서 아래 메서드 호출)

```java
public class ObjectMapperUtility {

    public static void addCustomSliceImplToObjectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();
        SimpleModule module = new SimpleModule();
        module.addAbstractTypeMapping(Slice.class, CustomSliceImpl.class); // 추상화된 인터페이스를 구체 클래스로 매핑함을 지시.
        objectMapper.registerModule(module);
    }
}

```

위와 같은 과정을 통해 `@SpringBootTest` 에서 `TestRestTemplate` 으로 `Slice<T>` 객체를 반환하는 경우 응답 본문을 자바 객체로 매핑할 수 있었습니다.   
`Page<T>` 또한 같은 과정으로 적용할 수 있어 보입니다.

## 의문

정확한 이유가 파악이 되고 있지 않습니다만,   
커스텀 구체 클래스 생성자 코드 설명글에 `한 번 적용된 이후 아래 두 생성자를 제거해도 잘 동작이 되고 있습니다.' 라는 대목이있는데요.   

희한하게, 처음엔 커스텀 구체 클래스 매핑 적용이 안되던 것이 `ObjectMapperUtility.addCustomSliceImplToObjectMapper()` 코드를 통해 한번 적용되고 나니, `ObjectMapperUtility` 코드를 제거해도 커스텀 구체 클래스 인식이 잘 되고 있습니다.

어떤 차이인지 확인하기 위해 생성자 제거, objectMapper 모듈 추가 코드 제거 등 여러 경우를 조합해 시도해봤지만 아직은 잘 모르겠네요. 정확한 원인 파악이 된다면 수정해서 추가 해보겠습니다.





