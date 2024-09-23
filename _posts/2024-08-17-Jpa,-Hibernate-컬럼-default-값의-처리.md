---
layout: post
title: JPA, Hibernate 컬럼 default 값의 처리
author: gosqo
categories: JPA
---

JPA, Hibernate 가 데이터베이스 테이블 컬럼 default 값을 처리하는 방식이 궁금해 직접 적용, 정리 해봅니다.   
오늘의 목표인 `Comment`개체의 `likeCount`필드.

```java
@Entity
@Getter
...
public class Comment {
    ...

    @ColumnDefault(value = "0")
    private Long likeCount;

    ...
}
```

`@ColumnDefault(value = "0")`을 통해, Hibernate DDL 에 `default 0` 설정이 됨을 알 수 있습니다.

```sql
create table comment (
    ...
    like_count bigint default 0,
    ...
    primary key (id)
)
```

insert 쿼리가 나갈 때, 해당 컬럼 값으로 null 이 할당됩니다.

```sql
insert
    into
        comment
        (board_id, content, like_count, member_id, register_date, update_date, id)
    values
        (?, ?, ?, ?, ?, ?, default)
```

```
TRACE org.hibernate.orm.jdbc.bind:   35 - binding parameter (3:BIGINT) <- [null]
```

해당 컬럼은 null 값을 가지면 안되기 때문에 `@Column(nullable = false)` 애너테이션을 추가.  

이유는: 해당 컬럼 관련 로직 작성 시, null 이 아닌 값이 들어있음을 상정하고 작성하는 로직일 것이므로, null 이 될 가능성 자체를 제외하는 것이 효과적이라 생각했기 때문입니다.

```java
@Entity
@Getter
...
public class Comment {
    ...

    @ColumnDefault(value = "0")
    @Column(nullable = false) // 추가
    private Long likeCount;

    ...
}
```

`nullable = false` 컬럼에 `null` 이 들어갈 수 없기에 에러 발생.  

`ColumnDefault` 를 통해 컬럼에 `default 값`을 설정하지만, 이것은 데이터베이스만이 알고 있는 것으로 보입니다. Hibernate 이 쿼리를 만들 때는 `default 값`을 할당하지 않는 것을 알 수 있습니다.

```
DEBUG org.hibernate.SQL:  135 -
    insert
    into
        comment
        (board_id, content, like_count, member_id, register_date, update_date, id)
    values
        (?, ?, ?, ?, ?, ?, default)
...
TRACE org.hibernate.orm.jdbc.bind:   35 - binding parameter (3:BIGINT) <- [null] // likeCount
...

WARN  o.h.e.jdbc.spi.SqlExceptionHelper:  145 - SQL Error: 23502, SQLState: 23502
ERROR o.h.e.jdbc.spi.SqlExceptionHelper:  150 - NULL not allowed for column "BOARD_ID"; SQL statement:
insert into comment (board_id,content,like_count,member_id,register_date,update_date,id) values (?,?,?,?,?,?,default) [23502-224]

```

> 추측이지만, Hibernate 이 이처럼 동작하는 이유는 @ColumnDefault 애너테이션의 역할은 데이터베이스에게 **해당 컬럼이 default 값을 가짐을 알리는 것** 그 자체이기 때문으로 보입니다.

자칫 default 설정이 곧, 레코드 삽입 시 해당 컬럼이 무조건적으로 가져야할 값을 할당하는 개념으로 여길 수도 있습니다.  
하지만, 경우에 따라 해당 컬럼에 `null` 을 할당 할 수도, `default 가 아닌 다른 값`을 할당할 수 있기 때문에, `default == 언제나 해당 값` 이라고 여기긴 어려울 것입니다. insert 구문에 `default` 예약어가 있는 만큼, **조건부 일정한 값의 삽입을 위한 장치**라고 볼 수 있겠습니다.

물론 Hibernate 은 `default == 언제나 해당 값` 을 위한 장치를 갖고 있기도 합니다. (@Generated, @DynamicInsert)

본론으로 돌아와, 저의 경우에 필요한 것은 `default == 언제나 해당 값`을 만드는 장치 입니다. 해당 목표를 이루기 위해 다음과 같은 방법을 찾을 수 있었습니다.

<table style="font-size: .8rem; table-layout: fixed;">
  <tr style="text-align: center;">
    <th width="8%">방식</th>
    <th colspan="2">Application 레벨에서 할당</th>
    <th colspan="2">Database 가 값을 할당하도록 위임</th>
  </tr>
  <tr>
    <th>방법</th>
    <td>
      생성자 + 필드 초기화<br />
      (@Builder.Default)
    </td>
    <td>
      @PrePersist<br />
      메서드에서 필드 초기화
    </td>
    <td>
      필드에<br />
      Hibernate @Generated<br />
      애너테이션 적용
    </td>
    <td>
      엔티티 클래스에<br />
      @DynamicInsert<br />
      애너테이션 적용
    </td>
  </tr>
  <tr style="text-align: center;">
    <th>할당 시점</th>
    <td>
      인스턴스 생성 시점
    </td>
    <td>
      인스턴스 생성 후,<br />
      <b>영속화 직전</b>
    </td>
    <td colspan="2">
      Database 에 insert 시점
    </td>
  </tr>
  <tr>
    <th>작동 원리</th>
    <td>
      Builder 패턴으로 인스턴스 생성 시,<br />
      해당 값을 명시하지 않는다면 필드에 초기화된 값을 기본값으로 가지게 세팅
    </td>
    <td>
      EntityManager 가 영속성 컨텍스트에서 insert 지연 쿼리를 날리기 전,<br />
      엔티티 컬럼에 값 할당
    </td>
    <td>
      Hibernate 에게 데이터베이스가 값을 생성하거나 변경할 수 있음을 알려줌.<br /><br />
      설정에 따라 데이터베이스에 의해 insert, update 시,<br />
      Hibernate 이 이를 인식하고 엔티티 상태를 동기화
    </td>
    <td>
      레코드 insert 시,<br />
      엔티티 필드 중 null 인 필드를 제외한 insert query 생성.<br /><br />
      not null 컬럼에 null 값이 들어가면,<br />
      해당 컬럼을 insert query 에 포함하지 않음.<br />
      해당 컬럼 값은 <b>데이터베이스가 기본값을 적용</b>하도록 함.
    </td>
  </tr>
  <tr>
    <th>비고</th>
    <td colspan="2">
      Application 수준에서 값을 가지므로, 영속성 컨텍스트와 데이터베이스 간극을 신경쓰지 않아도 됨.<br /><br /><hr /><br />
      인스턴스 생성의 방식을 통일하거나 신경써야 함. 현재 프로젝트에서는 빌더로 통일 중이지만,<br /><br />빌더, 생성자 초기화, setter 등 인스턴스 생성 및 필드 초기화 방법이 세부적으로 갈릴 수 있음. 인스턴스 생성 및 필드 값 할당 방식에 제약을 구현하는 코드가 추가될 수 있을 것으로 예상.
    </td>
    <td>
      애네테이션 <b>event: EnumType[]</b> 속성: 데이터베이스에 의해 값이 생성될 이벤트를 특정함.<br /><br />
      • <b>INSERT</b>: insert query 실행 이후, 생성된 값을 select query 를 실행, 해당 값을 다시 읽어와 영속성 컨텍스트와 DB 를 동기화 함.<br />
      • <b>UPDATE</b>: update query 실행 이후, 이하 동일.
    </td>
    <td>
      - 실행 환경에서 동적으로 쿼리를 생성하기에 이에 따른 비용이 존재.<br />
      <small>(기본적으로 준비된 정적 쿼리는 모든 컬럼에 대한 insert, update query)</small><br /><br />
      - 테이블의 컬럼의 많고, null 로 할당 된 값이 많을 경우 성능 향상을 기대.
    </td>
  </tr>
  <tr>
    <th>장점</th>
    <td>
      - 필드 초기화가 명확해 알아보기 쉬움.<br /><br />
      - 생성과 동시에 초기화 되므로 변경하지만 않으면 일관된 값 할당 가능.
    </td>
    <td>
      영속화 되기 직전에 값이 할당 되므로, <span style="display: inline-block;">인스턴스 생성 ~ 영속화</span> 사이에 값이 변경 되더라도 <b>영속화 시 일관된 값을 할당</b> 가능.
    </td>
    <td colspan="2">
      Application 수준에서 관리하는 코드 라인 수가 적어진다.
    </td>
  </tr>
  <tr>
    <th>단점</th>
    <td>
      - 필드 초기화가 어떻게 되는지 생성자 관련 메서드를 뜯어 봐야 함.<br /><br />
      - 인스턴스 생성 ~ 영속화까지 값이 변할 수 있는 요소에 대비해야함.
    </td>
    <td>
      - 필드에 초기화 명시에 비해, @PrePersist 메서드 까지 봐야 초기화 값을 알 수 있음
    </td>
    <td>
      - 데이터베이스가 생성한 값을 읽어오는 추가적 쿼리 자동 실행.
    </td>
    <td>
      - 영속성 컨텍스트, 데이터베이스 동기화 문제 발생 여지 존재.
    </td>
  </tr>
</table>

이렇게 다양한 선택 지 중 어떤 방법을 선택할 것인가?

- 선택 기준

  - 안정감: 코드 작성 후 해당 기능에 대해 신경을 덜 쓸 수 있나?
  - 명시성: 시간이 지나 코드를 봤을 때, 쉽게 알 수 있나?
  - 데이터베이스와 영속성 컨텍스트간의 동기화: 신경을 덜 쓸 수 있나?

  -> 결국, 다시 봐도 잘 알 수 있고, 얼마나 신경을 덜 쓸 수 있는지?

* 생성자 + 필드 초기화

  - @Builder.Default 애너테이션을 빌려 builder 패턴 이용 시 필드에 명시된 값으로 초기화 된다는 점을 인식하고 있어야 된다.
  - @AllArgsConstructor 로 생성 시, 해당 필드 값 지정에 대해 인지하고 있어야 된다.
  - 인스턴스 생성 후, 혹시 해당 필드 값을 변경할 장치가 있다면?

  -> 결국 신경써야된다.

* @PrePersist

  - JPA, Hibernate 사용하는 입장에서 애너테이션이 가지는 의미를 잊기 어렵고, 명확함.
  - 인스턴스 생성 후, 결국 영속화 되기 전 해당 필드 값을 할당하기 때문에 비교적 안정적
  - 값을 할당하고 쿼리를 보내기 때문에 데이터베이스와 영속성 컨텍스트간의 동기화를 신경쓰지 않아도 된다.

  **-> 신경을 가장 덜 쓰고, 직관적이다.**

* Hibernate @Generated, @DynamicInsert

  - 영속성 컨텍스트와 데이터베이스 사이의 동기화를 신경 써야한다.

  -> 신경 써야 된다.

분명 프로젝트 규모가 커지거나, 모종의 이유로 지금은 생각지 못한 @PrePersist 의 단점을 마주할 수 있겠지만,  
현재까지 조사하고 적용해본 바, 선택 기준에 가장 높은 점수로 부합하는 방법입니다.  
무엇보다 **영속성 컨텍스트와 데이터베이스간의 동기화 문제가 적은 것**이 가장 매력적인 선택 요인으로 다가왔습니다.

최종 선택을 적용한 코드.

```java
@Entity
@Getter
...
public class Comment {
    private static final Long DEFAULT_LIKE_COUNT_VALUE = 0L; // 상수 추가

    ...

    @ColumnDefault(value = "0")
    @Column(nullable = false)
    private Long likeCount;

    @PrePersist
    private void prePersist() {                    // 애너테이션, 메서드 추가
        this.likeCount = DEFAULT_LIKE_COUNT_VALUE;
    }

    ...
}
```

## 테스트: @PrePersist 에서 값을 할당

`entityManager.persist(entity)` 이전

![before_persist](/assets/img/2024-08-17-Jpa,-Hibernate-컬럼-default-값의-처리/PrePersist_before_persist.png)

`entityManager.persist(entity)` 이후, Hibernate 이 만드는 id, TimeStamp, @PrePersist 에서 할당한 값이 들어간 결과.  
데이터베이스에 쿼리를 날리기 전, 값들이 초기화 된것을 볼 수 있습니다.
자연스럽게, insert query 에 해당 값들을 바인딩합니다.

![right_after_persist](/assets/img/2024-08-17-Jpa,-Hibernate-컬럼-default-값의-처리/PrePersist_right_after_persist.png)
![right_after_persist](/assets/img/2024-08-17-Jpa,-Hibernate-컬럼-default-값의-처리/PrePersist_right_after_persist_console.png)

영속성 컨텍스트에 저장된 개체

![stored_entity](/assets/img/2024-08-17-Jpa,-Hibernate-컬럼-default-값의-처리/PrePersist_stored_entity.png)

영속성 컨텍스트와 데이터베이스의 동기화를 위해 작성한 `entityManager.refresh(entity)`코드를 실행해봅니다.(select query 가 나갑니다.)  
`refresh` 이후, 이전과 객체의 필드 값 모두 같은 것을 알 수 있습니다.  
Application 수준에서 모든 컬럼을 초기화해 데이터베이스로 쿼리를 날렸기 때문에, 서로 같은 값을 가지고 있는 것을 알 수 있습니다.

![stored_entity](/assets/img/2024-08-17-Jpa,-Hibernate-컬럼-default-값의-처리/PrePersist_after_refresh.png)
![stored_entity_console](/assets/img/2024-08-17-Jpa,-Hibernate-컬럼-default-값의-처리/PrePersist_after_refresh_console.png)


```java
  assertThat(stored.getLikeCount()).isEqualTo(0L); // 알고자 하는 부분
```

![test_passed](/assets/img/2024-08-17-Jpa,-Hibernate-컬럼-default-값의-처리/PrePersist_test_pass.png)

저장된 객체의 likeCount 필드 값이 0 으로 저장된 것을 확인합니다.
