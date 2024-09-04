---
layout: post
title: JPA Hibernate Transaction과 영속성 컨텍스트
author: gosqo
categories: JPA 영속성-컨텍스트
comment_issue_id: 1
---

> 문서 업데이트: 2024-09-03

## 목차

1. [들어가며](#들어가며)
2. [문제 상황](#문제-상황)
3. [원인 파악](#원인-파악)
4. [해결](#해결)
5. [또다른 방안 (@Transactional 을 사용하면 어떨까요?)](#또다른-방안-transactional-을-사용하면-어떨까요)
   1. [Spring 공식 문서 test transaction (class TestTransaction)](#spring-공식-문서-test-transaction-class-testtransaction)
   2. [TestTransaction 객체를 활용한 트랜잭션 제어](#testtransaction-객체를-활용한-트랜잭션-제어)
   3. [`@SpringBootTest`에서 직접 트랜잭션을 제어할 때의 문제점](#springboottest에서-직접-트랜잭션을-제어할-때의-문제점)
6. [어떤 방식이 `@SpringBootTest` 통합 테스트에 적합할지 고르자면](#어떤-방식이-springboottest-통합-테스트에-적합할지-고르자면)
7. [기본적인 트랜잭션과 영속성 컨텍스트 유지 범위](#기본적인-트랜잭션과-영속성-컨텍스트-유지-범위)
8. [마치며](#마치며)
9. [번외](#번외)

## 들어가며

프로젝트에 기능 추가 후, `@SpringBootTest`를 통해 통합 테스트를 진행하다 transaction,
영속성 컨텍스트에 대한 이해 부족으로 발생한 문제를 해결하며 영속성 컨텍스트가 어떻게, 어떤
조건으로 유지되는지 알아본 것을 기록으로 남깁니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 문제 상황

* `Comment`와 `CommentLike` 개체가 존재합니다.
* `CommnetLike`에서 `Comment`를 참조합니다.
    * 관계는 단방향 `@ManyToOne` 으로 매핑 됩니다.
    * `CommentLike` 이 최초 저장할 때, (`@PrePersist`)
        * `CommentLike`이 참조하는 `comment.likeCount` 필드의 값을 하나 올립니다.

* 테스트에서 사용할 데이터(개체)를 생성하고, `JpaRepository`를 상속한 `...Repository`를 통해 생성한 데이터를 각각 저장합니다.

`CommentLike`의 @PrePersist 는 다음과 같습니다.

```java

@Entity
@Getter
@Builder
@Table(
        uniqueConstraints = @UniqueConstraint(
                name = "member_id_comment_id_unique"
                , columnNames = {"member_id", "comment_id"}
        )
)
@NoArgsConstructor
@AllArgsConstructor
public class CommentLike extends IdentityBaseEntity {

    // ...

    @Override
    @PrePersist
    public void prePersist() {
        this.status = EntityStatus.ACTIVE;
        this.comment.addLikeCount();
    }

    // ...
}
```

`Comment` 에서 `likeCount`를 다루는 메서드는 다음과 같습니다.

* 영속화 전에 `likeCount` 컬럼 값을 디폴트 값인 0 으로 초기화합니다.
* `CommentLike` 추가, 삭제에 따라 `addLikeCount()`, `subtractLikeCount()`를 사용합니다.

```java

@Entity
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Comment {
    private static final Long DEFAULT_LIKE_COUNT_VALUE = 0L;

    // ...

    @ColumnDefault(value = "0")
    @Column(nullable = false)
    private Long likeCount;

    // ...

    @PrePersist
    private void prePersist() {
        this.likeCount = DEFAULT_LIKE_COUNT_VALUE;
    }

    public void addLikeCount() {
        this.likeCount++;
    }

    public void subtractLikeCount() {
        this.likeCount--;
    }
}
```

<details>
    <summary>번외: likeCount 반정규화</summary>
    <pre>
댓글 좋아요 수를 나타내는 likeCount 를 Comment 엔티티에 필드로 삼은 이유는 다음과 같습니다.

`Comment.likeCount`는 comment_like 테이블에 아래 쿼리를 통해 구할 수 있습니다.
        
    SELECT COUNT(*) 
    FROM comment_like cl 
    WHERE cl.comment_id = ? 
    AND cl.status = 'ACTIVE';
        
하지만 여러 댓글을 한번에 조회할 경우, 댓글 수 만큼 count 쿼리가 나가는 것은
데이터베이스 자원 관리에 비효율적이라 생각했기 때문에 별도의 컬럼으로 관리하고 있습니다.</pre>
</details>

테스트에서 데이터 생성 후 저장은 다음과 같이 진행했습니다.

```java

@Override
@Transactional
void initData() {
    member = memberRepository.save(buildMember());
    boards = boardRepository.saveAll(buildBoards());
    comments = commentRepository.saveAll(buildComments());
    commentLikes = commentLikeRepository.saveAll(buildCommentLikes());
}
```

countLike 필드값이 증가한 것을 확인하기 위한 테스트 코드입니다.

```java

@Test
void with_existing_CommentLike_Comment_likeCount_is_1L() {
    Comment comment = commentRepository.findById(commentIdHasCommentLike).orElseThrow();
    assertThat(comment.getLikeCount()).isEqualTo(1L);
}
```

```
org.opentest4j.AssertionFailedError: 
expected: 1L
 but was: 0L
```

기대와는 다르게 `comment.getLikeCount` 는 `0L` 인것을 확인했습니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 원인 파악

테스트 코드를 디버깅 해보면 테스트 클래스 필드에 선언한(애플리케이션에서 다루는 개체) `comments` 중 타겟 `comment` 는
`likeCount = 1` 을 가지고 있음을 알 수 있습니다.

![field comments has 1 likeCount](/assets/img/2024-09-02-JPA-Hibernate-Transaction-과-영속성-컨텍스트/field-comments-has-1-likeCount.png)

하지만 (데이터베이스에서 가져온 개체)`commentRepository.findById(Long)`을 통해 조회한 개체는 `likeCount = 0` 인 것을 확인했습니다.

![comment that repository returns has 0 likeCount](/assets/img/2024-09-02-JPA-Hibernate-Transaction-과-영속성-컨텍스트/comment-that-repository-returns-has-0-likeCount.png)

이런 차이가 발생하는 원인은 다음과 같이 예상해볼 수 있었습니다.

* `commentLikeRepository.saveAll()`동작에는 `CommentLike`가 저장되며 `comment.addLikeCount()`메서드가 정상 작동.
* 이 때, 애플리케이션에서 다루는 객체(`comments`)에 변경 사항이 적용.
* 하지만 직전 `commentRepository.saveAll()`트랜잭션이 종료될 때 영속성 컨텍스트가 초기화 되어,   
  변경 대상인 개체들(`comments`)이 영속성 컨텍스트에 미포함.
* 때문에 데이터베이스에서 조회한 `comment`에는 이 변경 사항이 적용되지 않았습니다.

위와 같은 예상에 따라, 애플리케이션의 변경 사항이 데이터베이스에 적용되지 않은 결정적인 이유는
상태가 변한 개체가 영속성 컨텍스트에서 관리되지 않고 있기 때문인 것으로 보입니다.

원인을 직접 눈으로 보기 위해 엔티티 매니저를 필드에 등록합니다. 영속성 컨텍스트에 `변화를 감지할 코멘트`를 가지고 있는지 확인해봅니다.

```java

@PersistenceContext
private EntityManager em;

// ...

void initData() {
    member = memberRepository.save(buildMember());
    boards = boardRepository.saveAll(buildBoards());
    comments = commentRepository.saveAll(buildComments());
    commentLikes = commentLikeRepository.saveAll(buildCommentLikes());

    log.info("EntityManger contains the Comment that id = ? {}", em.contains(comments.get(0)));
}
```

로그는 다음과 같이 출력되며, 변경 감지 대상이 영속성 컨텍스트에 존재하지 않음을 알았습니다.

```text
EntityManger contains Comment of which id = 1? false
```

그렇다면 왜 존재하지 않는 지 로그를 따라가봅니다. 아래 로그는 `initData()` 메서드 내부에 `member = memberRepository.save(buildMember());` 실행 후의 로그입니다.
하나의 멤버를 저장하는 이 메서드를 관리하는 Transaction은 `save()` 메서드 하나만을 담당하고 있습니다.

* 해당 메서드가 실행될 때 트랜잭션 시작,
* 해당 메서드를 실행.
* 해당 메서드 실행이 끝나면 트랜잭션 커밋.
* 이후 트랜잭션이 필요하면 트랜잭션 시작. (이 때 영속성 컨텍스트 초기화)

```text
DEBUG o.h.e.t.internal.TransactionImpl:   81 - begin
DEBUG org.hibernate.SQL:  135 - 
    insert 
    into
        member
        (email, nickname, password, register_date, role, id) 
    values
        (?, ?, ?, ?, ?, default)
DEBUG o.h.e.t.internal.TransactionImpl:   98 - committing
DEBUG o.h.e.t.internal.TransactionImpl:   53 - On TransactionImpl creation, JpaCompliance#isJpaTransactionComplianceEnabled == false

DEBUG o.h.e.t.internal.TransactionImpl:   81 - begin // 새 영속성 컨텍스트 생성
```

마찬가지로 `Comment` 를 저장한 후 트랜잭션은 커밋되고, 영속성 컨텍스트에는 이전에 저장 완료한 개체들은 더 이상 존재하지 않게 됩니다.
때문에 `CommentLike` 을 저장하며 참조하는 `Comment` 개체의 필드 값을 수정했지만, 
Hibernate 은 **영속성 컨텍스트에 존재하지 않는 개체의 변경 사항을 인식할 수 없음**을 알 수 있습니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 해결

따라서 문제를 해결하는 방법 중 하나는 다음과 같습니다.

* 애플리케이션에서 변경한 개체를 영속성 컨텍스트에 할당해 데이터베이스에 변경 사항을 적용.

이렇게 되면, 어떤 개체의 저장이 다른 개체에 영향을 주는지 알고 직접 변경 사항을 적용해줘야되는 되는 단점이 있습니다.
하지만 다른 의존성이나 기능, 추가적인 컨텍스트 파악 없이 기존에 사용하는 객체로 문제를 간결하게 해결할 수 있습니다.

```java
void initData() {
    member = memberRepository.save(buildMember());
    boards = boardRepository.saveAll(buildBoards());
    comments = commentRepository.saveAll(buildComments());
    commentLikes = commentLikeRepository.saveAll(buildCommentLikes());

    // 변경 사항을 영속성 컨텍스트에 전달 및 데이터베이스에 적용.
    commentRepository.saveAll(comments);
}
```

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 또다른 방안 (@Transactional 을 사용하면 어떨까요?)

> 결과적으로는 @SpringBootTest 통합 테스트에서는 [적합하지 않다고 판단했습니다](#springboottest에서-직접-트랜잭션을-제어할-때의-문제점).

그럼 영속화 작업의 단위인 Transaction 을 사용하면 되지 않을까 싶었습니다.

* 테스트 메서드에 `@Transactional` 애너테이션을 적용.
* 결과적으로 `@DataJpaTest` 에서와 같이 테스트와 `@BeforeEach` 를 하나의 트랜잭션으로 묶어 실행하기 때문에, 중간에 `transaction.commit()` 이 없어
  `...repository`를 통한 조회 시 저장된 개체가 없음을 알 수 있었습니다.
* 때문에 다른 방법을 찾다, 새로운 트랜잭션을 만들어 독립적으로 트랜잭션 운용이 가능한
  `@Transactional(propagation = Propagation.REQUIRES_NEW)`를 initData() 메서드에 적용 해봤지만 별다른 효과가 없었습니다. (이유는 아래에)

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

### Spring 공식 문서 test transaction (class TestTransaction)

[Test-managed Transactions](https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/tx.html#testcontext-tx-test-managed-transactions)
을 살펴보면

> Note that `@Transactional` is not supported on test lifecycle methods —
> for example, methods annotated with JUnit Jupiter’s `@BeforeAll`, `@BeforeEach`, etc.
> Furthermore, tests that are annotated with `@Transactional`
> but have the `propagation` attribute set to `NOT_SUPPORTED` or `NEVER` are not run within a transaction.

> Method-level lifecycle methods
> — for example, methods annotated with JUnit Jupiter’s @BeforeEach or @AfterEach —
> are run within a test-managed transaction.

라는 안내와

![table: Transactional attribute support](/assets/img/2024-09-02-JPA-Hibernate-Transaction-과-영속성-컨텍스트/Transactional-attribute-support.png)

위와 같은 표를 볼 수 있었습니다.

* 스프링에서 제공하는 `@Transactional` 애너테이션은 JUnit Jupiter 에서 제공하는 
  테스트 생명주기 메서드(`@BeforeAll`, `@BeforeEach` 등 )에 직접적으로 지원되지 않을 알 수 있습니다.   
* 반면 테스트 메서드에 `@Transactional`이 적용된 경우, 메서드 레벨 생명주기 메서드인 `@BeforeEach`, `@AfterEach`는 
  테스트 트랜잭션에 포함되어 실행됩니다.
* 이 때 `@Transactional` 의 `propagation` 속성은 `[NOT_SUPPORTED, NEVER]`만 지원되며,   
  이 설정의 경우, 트랜잭션 내에서 실행하지 않는다는 것을 알수 있었습니다.

표 아래에 나오는 `TestTransaction.flagForCommit()` 을 활용하면 트랜잭션 안에서 원하는 시점에 커밋할 수 있을 것으로 예상했습니다.

해당 메서드의 주석을 살펴보면, 해당 static method 는 트랜잭션이 끝나는 시점에,
현재의 테스트(에 의해 관리되는) 트랜잭션이 커밋되어야 함을 나타낸다고 합니다.

추가적으로 테스트 트랜잭션의 시작과 종료를 지시하는 메서드입니다.
각각,

* start: end 가 호출되었거나, 이전에 시작된 트랜잭션이 없을 때 호출.
* end: 호출 시점에 `TransactionContext.flaggedForRollback` 필드 값에 따라 즉시 커밋, 혹은 롤백을 강제한다고 합니다.

```java
/**
 * Flag the current test-managed transaction for <em>commit</em>.
 * <p>Invoking this method will <em>not</em> end the current transaction.
 * Rather, the value of this flag will be used to determine whether
 * the current test-managed transaction should be rolled back or committed
 * once it is {@linkplain #end ended}.
 * @throws IllegalStateException if no transaction is active for the current test
 * @see #isActive()
 * @see #isFlaggedForRollback()
 * @see #start()
 * @see #end()
 */
public static void flagForCommit() {
    setFlaggedForRollback(false);
}

/**
 * Start a new test-managed transaction.
 * <p>Only call this method if {@link #end} has been called or if no
 * transaction has been previously started.
 * @throws IllegalStateException if the transaction context could not be
 * retrieved or if a transaction is already active for the current test
 * @see #isActive()
 * @see #end()
 */
public static void start() {
    requireCurrentTransactionContext().startTransaction();
}

/**
 * Immediately force a <em>commit</em> or <em>rollback</em> of the
 * current test-managed transaction, according to the
 * {@linkplain #isFlaggedForRollback rollback flag}.
 * @throws IllegalStateException if the transaction context could not be
 * retrieved or if a transaction is not active for the current test
 * @see #isActive()
 * @see #start()
 */
public static void end() {
    requireCurrentTransactionContext().endTransaction();
}

```

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

### TestTransaction 객체를 활용한 트랜잭션 제어

위에서 알게된 내용을 바탕으로
아래와 같이 테스트 데이터를 초기화할 때, 하나의 영속성 컨텍스트에 개체들을 모아, 적절한 시점에 변경 사항까지 적용된 상태로 커밋할 수 있었습니다.

```java
// @BeforeEach 가 호출하는 데이터 초기화 메서드
void initData() {
    // 하나의 트랜잭션 시작. 하나의 영속성 컨텍스트에 아래 개체들이 존재하게 됩니다.
    member = memberRepository.save(buildMember());
    boards = boardRepository.saveAll(buildBoards());
    comments = commentRepository.saveAll(buildComments());
    commentLikes = commentLikeRepository.saveAll(buildCommentLikes());
    log.info("EntityManger contains Comment of which id = 1? {}", em.contains(comments.get(0)));

    TestTransaction.flagForCommit(); // 트랜잭션이 끝날 때 커밋해야함을 표시.
    TestTransaction.end(); // 트랜잭션을 종료. -> 커밋. 영속성 컨텍스트 있는 개체들에 대한, 지연 쿼리 데이터베이스로 발송. 
    TestTransaction.start(); // 이후 테스트에 사용될 트랜잭션을 시작.
}
```

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

### `@SpringBootTest`에서 직접 트랜잭션을 제어할 때의 문제점

이후, 아래와 같이 통합 테스트에서 서비스 레이어 메서드 호출이 포함된 테스트를 진행할 때, 서비스 메서드 호출이 끝난 시점에 커밋이 발생하지 않는 문제가 있었습니다.
테스트 자체가 하나의 큰 트랜잭션으로 묶여있기 때문에 서비스 메서드의 동작이 완료된 시점에 커밋이 일어나지 않는 것이 원인이었습니다.

```java
@Test
void callServiceMethod() {
    Comment comment = commentRepository.findById(targetCommentId).orElseThrow();

    assertThat(comment.getLikeCount()).isEqualTo(0L);

    // 서비스 레이어 메서드 호출: comment 를 참조하는 CommentLike 개체 저장.

    Comment foundComment = commentRepository.findById(targetCommentId).orElseThrow();

    assertThat(comment.getLikeCount()).isEqualTo(1L); // assertion failed. expected: 1L, but actual: 0L;
}
```

해당 문제를 해결하기 위해서는 아래와 같이 서비스 메서드 호출이 끝난 시점에 직접적으로 커밋을 해줘야 됐습니다.

```java

@Test
void callServiceMethod() {
    Comment comment = commentRepository.findById(targetCommentId).orElseThrow();

    assertThat(comment.getLikeCount()).isEqualTo(0L);

    // 서비스 레이어 메서드 호출: comment 를 참조하는 CommentLike 개체 저장.

    // 트랜잭션 제어 추가.
    TestTransaction.flagForCommit();
    TestTransaction.end();
    TestTransaction.start();

    Comment foundComment = commentRepository.findById(targetCommentId).orElseThrow();

    assertThat(comment.getLikeCount()).isEqualTo(1L); // assertion succeeded.
}
```

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 어떤 방식이 `@SpringBootTest` 통합 테스트에 적합할지 고르자면

두 가지 방식을 비교했을 때, 첫번째의 방식을 채택하기로 했습니다.

* (데이터 초기화 시, 영속성 컨텍스트에 의해 관리되지 않는 개체의 변경 사항을 직접 처리)

애플리케이션 전체 컨텍스트를 적용한 테스트의 목적은 실제 앱이 작동하는 방식대로(앱 내부의 동작에 관여하지 않고) 테스트하는 것이기 때문에,
내부 구조를 알고 트랜잭션을 관리하는 것은 목적에 부합하지 않는다고 생각했기 때문입니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 기본적인 트랜잭션과 영속성 컨텍스트 유지 범위

기본적으로 스프링에 의해 트랜잭션이 제어될 때, `JpaRepository의 각 메서드` 마다 트랜잭션을 생성하고 종료함을 알 수 있었습니다.
이 트랜잭션 마다 영속성 컨텍스트가 생성되고 종료됩니다.

> 이것은 해당 포스트 [원인 파악](#원인-파악)의 `initData()` 메서드 디버그를 통해 알 수 있습니다.

하지만 웹 기반 요청은 다릅니다.

* 기본적으로 웹 애플리케이션에서 하나의 HTTP 요청을 처리할 때 하나의 스레드에서 처리.
* 이 하나의 스레드는 요청을 처리하는 동안 하나의 영속성 컨텍스트를 유지. (Request Scope)
* 요청마다 생성된 영속성 컨텍스트를 유지하며. `@Transactional`이 없어도, 매 `JpaRepository` 메서드 호출 마다 커밋 롤백 하지만 영속성 컨텍스트가 유지됨.

덕분에 요청 처리 중 영속성 컨텍스트에는 조회 등을 통한 개체들이 쌓입니다. 요청 처리를 수행한 후,
커밋 혹은 롤백 할 때 영속성 컨텍스트에서 관리되는 개체들의 변경 사항을 데이터베이스에 적용할 수 있게됩니다.

그렇지만 서비스 레이어 메서드에 `@Transactional`을 적용하기로 했습니다.   
여러 테이블의 상태를 조회하고, 읽기, 쓰기, 수정의 작업이 포함되는 메서드이기 때문에

* 하나의 영속성 컨텍스트를 열고 닫음을 명확히 하고,(작업 단위 구분)
* 과정에서 문제가 생겼을 때 롤백을 통해 데이터 무결성을 유지

할 수 있기 때문입니다.

서비스 레이어 메서드에 `@Transactional`을 적용하면, 해당 메서드의 호출 시 트랜잭션이 생성되고,
해당 메서드 종료 시점(JpaRepository 메서드를 마지막으로 사용하는 단계)에 커밋, 트랜잭션 종료하게 됩니다.
서비스 레이어 메서드 return 문 직전에 "asdf"를 로깅해 보면 아래와 같이 출력 됩니다.

```text
// 서비스 레이어 @Transactional 메서드 호출 시점
    DEBUG o.h.e.t.internal.TransactionImpl:   81 - begin 

    ... 서비스 메서드 쿼리 ...

// 해당 메서드 내부, 마지막 JpaRepository 메서드 사용이 끝난 시점
    DEBUG o.h.e.t.internal.TransactionImpl:   98 - committing
    DEBUG org.hibernate.SQL:  135 - 
        update
            ... comment 업데이트 쿼리 ... 

// 메서드 return 문 직전에 찍은 로그
    INFO  c.v.m.service.CommentLikeService:   76 - asdf // commit 시점 확인을 위한 return 이전 로깅.
```

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 마치며

이 포스트를 작성하기 전에 궁금했던 점이    
`@DataJpaTest` 컨텍스트를 사용한 경우, 문제없이 통과하는 케이스였는데, `@SpringBootTest` 컨텍스트에서는 실패하는 원인이었습니다.

둘의 컨텍스트가 다르기 때문에 조회하는 대상이 다르게 됩니다.
`@DataJpaTest`의 경우 `@Transactional`을 포함하고 있어, **모든 작업이 하나의 영속성 컨텍스트**에서 이뤄집니다.
덕분에 JpaRepository 를 통해 조회하더라도 영속성 컨텍스트에서 관리되고 있는 개체의 변화가 반영된 결과를 즉각적으로 확인할 수 있었습니다.

반면 `@SpringBootTest`의 경우, `@Transactional`을 포함하지 않습니다. 테스트 메서드 혹은 클래스에 `@Transactional` 을 적용하지 않는다면,
매 JpaRepository 메서드 사용 마다 트랜잭션, 영속성 컨텍스트가 생성되고 종료됩니다.
조회 대상이 실제 데이터베이스인 것입니다.

따라서 프로덕션, 테스트 코드를 작성할 때는, 트랜잭션이 유지되는 범위와 데이터베이스와 영속성 컨텍스트의 동기화 여부를 염두에 둬야겠습니다.

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>

## 번외

추가로 `@Transactional`이 적용된 서비스 계층의 코드는 다음과 같습니다.

* `memberId`, `commentId` 조합으로 `CommentLike`이 존재하는지 확인.
    * 존재한다면,
        * (stored)개체의 상태가 `ACTIVE`가 아닌 경우,
            * 상태를 `ACTIVE`로 바꾸고, 참조하는 `comment.likeCount`를 증가.
        * 개체의 상태가 `ACTIVE`인 경우,
            * 해당 개체를 `AtomicReference` 에 지정.
    * 아니라면,
        * 새로운 개체를 만들어 데이터베이스에 저장 및 `AtomicReference`에 지정.
* `AtomicReference`에 지정된 개체를 반환.

```java

@Transactional
public CommentLike registerCommentLike(Long memberId, Long commentId) {
    Member member = memberRepository.findById(memberId).orElseThrow(
            () -> new NoSuchElementException("존재하지 않는 회원의 댓글 좋아요 요청.")
    );
    Comment comment = commentRepository.findById(commentId).orElseThrow(
            () -> new NoSuchElementException("존재하지 않는 댓글에 좋아요 요청.")
    );
    AtomicReference<CommentLike> commentLike = new AtomicReference<>();

    commentLikeRepository.findByMemberIdAndCommentId(memberId, comment.getId())
            .ifPresentOrElse(
                    (stored) -> {
                        if (stored.getStatus() != EntityStatus.ACTIVE) {
                            stored.activate();
                            comment.addLikeCount();
                        }
                        commentLike.set(stored);
                    },
                    () -> commentLike.set(commentLikeRepository.save(CommentLike.builder()
                            .member(member)
                            .comment(comment)
                            .build()))
            );

    return commentLike.get();
}
```

<div dir="rtl">
  <a href="#목차">목차로 돌아가기</a>
</div>
