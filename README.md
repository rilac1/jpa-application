# JPA 활용 1편

- Embedded 내장 타입

### 모든 연관관계는 지연로딩으로 설정!

- 즉시로딩( EAGER )은 예측이 어렵고, 어떤 SQL이 실행될지 추적하기 어렵다.
- 특히 JPQL을 실행할 때 N+1 문제가 자주 발생한다.
- 실무에서 모든 연관관계는 지연로딩( LAZY )으로 설정해야 한다.
- @XToOne(`OneToOne`, `ManyToOne`) 관계는 기본이 즉시로딩이므로 직접 지연로딩으로 설정해야 한다.

```java
@ManyToOne(fetch = FetchType.LAZY)
```

### FindAll

리스트로 조회할 때에는 `em.createQuery` 를 사용해야 함.

### Service

- 레포지토리를 호출하는 메소드에는 반드시 `@Transactional` 어노테이션을 붙여야 함.
- `@Transactional` 은 디폴트로 읽기/쓰기가 모두 가능하다.
- 쓰기가 필요없는 메소드는 (readOnly = true)를 옵션을 넣는다.
  - 또는 클래스에 `@Transactional(readOnly = true)` 을 달고,
  - 쓰기가 필요한 메소드에 `@Transactinoal` 을 붙여준다.

## 웹 계층

- 엔티티를 바로 화면에 내보내는 객체(파라미터)로 사용하지 말자.
  - 화면 종속적인 기능 때문에 엔티티가 지저분해진다.
  - 유지보수가 어려워진다.
  - 엔티티는 핵심 비즈니스 로직에만 의존성이 있게만 설계해야 한다.
- API를 만들 때에는 **절대** 엔티티를 반환하면 안된다.
  - API에 비밀번호 같은 정보가 노출된다.
  - 엔티티에 로직을 추가하면 API 스펙이 변경된다.
- 폼 객체나 DTO를 만들어라 (@NotEmpty 사용 등등...)
  - DTO: getter, setter만 있는 데이터 전송 계층

## 중요한 내용

### 준 영속 엔티티

#### 준영속 엔티티란?

- 영속성 컨텍스트가 더 이상 관리하지 않는 엔티티
- 그냥 만든 객체이지만 JPA에 존재하는 식별자를 세팅한 엔티티

#### 준영속 엔티티를 수정하는 방법

- **변경 감지 기능**

  - 영속성 컨텍스트에서 엔티티를 다시 조회한 후에 데이터를 수정하는 방법
  - Service 계층의 Transactional 안에 구현

  1. 트랜잭션 안에서 엔티티를 다시 조회하고
  2. 조회한 엔티티 값 변경.
  3. 트랜잭션 커밋 시점에 변경 감지(Dirty Checking)이 동작해서
  4. 데이터베이스에 UPDATE SQL 실행

- `merge` 기능

  - 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용하는 기능
  - 모든 필드를 전부 바꿔치기 한 후 반환함.
    - 만약 기존에 채워져 있던 필드라도 새로운 엔티티의 필드가 비어있다면 null로 update됨.
    - 그래서 권장하지 않음. (변경 감지 기능을 사용하자.)
  - Repository의 Entity Manager를 사용하여 구현 (`em.merge()`)

### 컨트롤러에서 어설프게 엔티티를 생성하지 마라

- 수정과 같은 로직이 필요하다면
  - 컨트롤러에서 엔티티를 만들지 말고,
  - 변경할 데이터만 파라미터로 서비스에 넘겨라.
  - 필요한 데이터가 많다면 별도의 dto를 만들어서 넘겨라.

### Setter를 쓰지 마라

- 객체를 생성할 때는
  - **Builder 패턴**이나 **정적 팩토리 메서드**를 사용하자.
  - '생성자는 객체 메모리 할당' 이라는 기능에만 집중하자. (간단할 때는 그냥 값 넣고 만들어도 됨)
- 값을 변경할 때는
  - setter 대신 로직을 구체적으로 나타낼 수 있는 이름을 가진 메서드를 따로 정의해라.
  - ex) chage

### 엔티티 값을 변경할 때 Merge를 사용하지 마라

- 변경 감지 기능을 사용하자
- merge는 모든 필드 값을 바꿔치기 하기 때문에 위험함

## 기타

- `cascade = CascadeType.ALL`

  - 넣으면 알아서 연관관계를 persist() 해줌

  - 다른데서도 접근하는 테이블이라면 쓰지말자.

- 참고

  - 엔티티가 비즈니스 로직을 가지고 객체 지향의 특성을 적극 활용하는 것을 **도메인 모델 패턴** 이라 한다. (ORM에 적합)

  - 반대로 엔티티에 비즈니스 로직을 비워두고 서비스 계층에서 모든 비즈니스 로직을 처리하는 것을 **트랜젝션 스크립트 패턴** 이라 한다. (SQL에 적합)

- 컨트롤러에서는 식별자만 서비스로 넘긴다.

  - 객체를 만들지 않는다.
  - 핵심 비즈니스 로직을 영속성 컨텍스트로 관리할 수 있다.

## 테스트

### 예외가 터저야 하는 테스트

```java
@Test(expected = IllegalStateException.class)
```

### 테스트 코드에는 별도의 DB를 두자.

`test` - `resources` - `application.yml`

테스트에 있는 application.yml 파일은 기본적으로 인메모리로 돌아가기 때문에 별도의 설정정보는 넣지 않아도 된다. (logging을 제외하고 전부 비어두자)

# JPA 활용 2편

### 권장 순서

1. 엔티티 조회 방식으로 우선 접근
   1. 페치조인으로 쿼리 수를 최적화
   2. 컬렉션의 경우
      1. 페이징이 필요O: Batch 옵션으로 최적화
      2. 페이징이 필요X: 페치 조인 사용
2. 엔티티 조회 방식으로 해결이 안되면 DTO 조회 방식 사용
3. DTO 조회 방식으로 해결이 안되면 NativeSQL or 스프링 JdbcTemplate

> 대부분의 경우는 1번 엔티티 조회 방식 선에서 해결 가능

### DTO를 반환할 때 아래 두 가지 방법 중 하나를 선택해라.

- 엔티티 조회 방식으로 조회하고, 엔티티를 DTO로 변환하여 반환한다.
  - 가장 정석적인 방법.
- JPA에서 DTO로 바꿔 조회한다.
  - 필요한 부분만 select하기 때문에 성능이 살짝 올라감.
  - 하지만 DTO가 레포지토리에 들어가야 하기 때문에 레포지토리가 API 스펙에 의존성이 생김.
    - 그래서 레포지토리 내에 별도의 디렉토리를 만들어서 새로운 레포지토리에서 관리함.
    - 레포지토리는 순수한 엔티티를 조회하는 용도로만 사용한다. (fetch join 정도만 사용)

### 페이징과 한계 돌파 (중요!!)

#### 페치 조인의 한계

- 페치 조인의 한계 중 가장 치명적인 것은 컬렉션을 페치조인할 경우 페이징이 불가능 하다는 것이다.
- 이를 해결하기 위해 페치 조인 대신 `batch fetch size` 옵션을 걸어준다.

#### Batch 옵션으로 조회

- 적용방법 

  - 글로벌하게 적용 (`appication.yml`)

    ```yaml
    spring:
      jpa:
    		properties:
    			default_batch_fetch_size: 100
    ```

  - 각자 적용

    `@BatchSize(size = 1000)`

  > 일반적으로 글로벌에만 적용한다.

#### 페치 조인과의 차이점

- 컬렉션을 페치 조인 할 경우 쿼리 한 방에 값을 가져올 수 있지만 **값이 뻥튀기 된다.**
- batch 옵션을 걸면 쿼리가 몇 번 더 나가지만 중복없는 데이터를 가져온다.
- 무엇보다 batch 옵션으로 조회하면 **페이징이 가능하다.**

#### 결론

- ToOne 관계는 페치 조인으로 쿼리수를 줄인다.
- 컬렉션(ToMany)은 지연로딩으로 조회한다. (페이징을 위해)
- 이후 성능 최적회가 필요하다면 배치 옵션을 건다.
  - 개수는 100~1000개 정도가 적당

### 기타

- (`@RequestBody` `@Valid` Member member): Json으로 온 Body를 Member에 매핑해서 넣어줌.

- API를 만들 때는 Entity를 외부에 노출하지 말고, 별도의 DTO를 만들어라.

  - DTO도 그대로 내보내지 말고, Json 스펙을 유연하게 갖기 위해 Result로 감싸서 내보내라.

- `@NotEmpty` 는 컨트롤러에서 검사해라.

- Redis/로컬 메모리 등의 캐시를 사용할 때 **엔티티는 절대 캐시하면 안된다.**
  - 무조건 DTO를 변환해서 DTO를 캐시에 담아야 함.

### OSIV

- Open Session In View: 하이버네이트
- Open EntityManagerin In View: JPA
