---
title: '올바른 Lombok 사용법'
metaTitle: '만렙 개발자 키우기'
metaDescription: '스프링부트 관련 내용을 정리한 곳입니다.'
tags: ['Spring Boot']
date: '2021-07-29'
---

## 정리 배경

Lombok은 자바 컴파일 시점에서 특정 어노테이션으로 해당 코드를 추가할 수 있는 라이브러리이다.
이는 간편한 코드 작성, 가독성, 그리고 유지 보수에 많은 도움이 된다. 
하지만 편리한 만큼 잘못 사용하기 쉬운 것이 Lombok 이다.

여러 현업의 선배들로부터, 또는 여러 강의로부터 `Lombok` 의 위험성을 간략하게나마 들었었다. 가령, `Setter`는 열어두면 안좋고, 따라서 엔티티에서 `@Data` 어노테이션은 붙이면 안된다는 등..

하지만 위험성에 대해서 직접 경험해보지 못하였기 때문에, 앞서 선배 개발자분들께서 사용하면서 겪었던 문제점에 대한 해결법을 미리 답습하기 위하여 정리를 시작하였다.

---

## @Data 사용 지양

`@Data` 는 `@ToString`, `@EqualsAndHashCode`, `@Getter`, `@Setter`, `@RequiredArgsConstructor`을 모두 포함하는 강력한 어노테이션이다.
강력한 어노테이션인 만큼 그에 따른 부작용도 많다.

@Data 사용으로 인해 발생할 수 있는 문제들을 확인해보자.

### 1. 무분별한 Setter 남용

`Setter`는 그 의도가 분명하지 않고 객체를 언제든지 변경할 수 있는 상태가 되어서 객체의 안전성이 보장받기 힘들다.

불필요한 변경 포인트를 제공하지 않음으로써 안정성을 취할 수 있다.

### 2. @ToString : 양방향 연관관계 시 순환 참조

`Member` 와 `Coupon` 이 1:N 양방향으로 매핑되어 있는 상황을 가정할 수 있다.

이때, `ToString`을 호출하면 무한 순환 참조가 발생한다.

이러한 문제를 해결하기 위해서는 `@ToString(exclude = "coupons")` 처럼 어노테이션을 사용해서 특정 항목을 제외시키는 방법을 사용할 수 있다.

### 3. @EqualsAndHashCode 

`@EqualsAndHashCode`는 상당히 고품질의 `euqals()`와 `hashCode()` 메소드를 만들어준다. 따라서 잘 사용하면 좋지만, 남발하면 심각한 문제가 생긴다.

특히 문제가 되는 점은 `Mutable` 객체에 아무런 파라미터 없이 그냥 사용하는 경우이다.

```java
@EqualsAndHashCode
public static class Order {
    private Long orderId;
    private long orderPrice;
    private long cancelPrice;

    public Order(Long orderId, long orderPrice, long cancelPrice) {
        this.orderId = orderId;
        this.orderPrice = orderPrice;
        this.cancelPrice = cancelPrice;
    }
}

Order order = new Order(1000L, 19800L, 0L);

Set<Order> orders = new HashSet<>();
orders.add(order); // Set에 객체 추가
System.out.println("변경전 : " + orders.contains(order)); // true

order.setCancelPrice(5000L); // cancelPrice 값 변경
System.out.println("변경후 : " + orders.contains(order)); // false
```

위와 같이 동일한 객체여도 필드 값을 변경시키면 hashCode가 변경되면서 찾을 수 없는 값이 되버린다.

핵심은, 어노테이션 자체의 문제라기 보다는 **변경 가능한 필드에 이를 남발함으로써 생기는 문제이다.**

- `Immutable` 클래스를 제외하고는 아무 파라미터 없는 `@EqualsAndHashCode` 사용은 지양한다.


- 항상 `@EqualsAndHashCode(of={“필드명시”})` 형태로 동등성 비교에 필요한 필드를 명시하는 형태로 사용한다.


- 실무에서는 누군가는 이에 대해 실수하기 마련인지라 차라리 사용을 완전히 금지시키고, 꼭 필요한 필드를 지정하는 것이 나을 수도 있다.

---

## 생성자 자동 생성 어노테이션 사용 지양

세 가지 생성자 자동 생성 어노테이션을 떠올릴 수 있다.

### @NoArgsConstructor 접근 권한 최소화

JPA에서는 프록시를 생성을 위해서 기본 생성자를 반드시 하나를 생성해야한다.
이때 접근 권한이 `protected` 이면 된다. 굳이 외부에서 생성을 열어둘 필요가 없다.

예를 들어, 다음과 같이 접근 권한이 `public`인 경우에 문제가 발생할 수 있다.

```java
@Entity
@Table(name = "product")
@Getter
@NoArgsConstructor(access = AccessLevel.PUBLIC) // 테스트를 위해 임시로 Public
public class Product {

    @Id
    private String id;

    private String name;

    @Builder
    public Product(String name) {
        this.id = UUID.randomUUID().toString();
        this.name = name;
    }
}
```

기본 키 생성을 UUID로 가지도록 하였다. 하지만 `public` 생성자를 통해 객체를 생성하면
Id 값은 null 이 된다.

이처럼 기본 생성자를 아무 이유 없이 열어두는 것은 객체 생성 시 안전성을 심각하게 떨어뜨린다.

그렇다고 `private`으로 되어있으면 JPA가 프록시를 만들 때 접근하지 못해 객체를 생성하질 못하게 된다.

따라서, 스펙에서는 기본 생성자 접근을 `protected`로 열어두길 권장하고 있다.

위의 예에서는, 생성자 메소드에서 UUID 생성 코드를 넣어 객체 생성 시 반드시 ID 값을 보장받도록 하고 있다.

이처럼 객체에 대한 생성자를 하나로 두고 `@Builder`를 통해 사용하면 반드시 필요한 값이 있어야 객체가 생성됨을 보장할 수 있어 안전성을 높일 수 있다.

> 이렇게 사소한 객체 생성부터 고민해보는 것이 `클린 코드`, `유지 보수`를 고려한 좋은 코드를 만들 수 있는 습관이 된다고 생각한다.

### @AllArgsConstructor 사용 지양

매우 편리하게 생성자를 만들어주지만, 별 생각없이 작성한 코드가 치명적인 버그를 만들어낼 수도 있다.

다음과 같은 예를 들 수 있다.

```java
@AllArgsConstructor
public static class Person {
    private String firstName;
    private String lastName;
}

// 성은 권, 이름은 현수
Person me = new Person("권", "현수");
```

위 클래스에 대해 자동으로 `firstName`, `lastName` 순서로 인자를 받는 생성자가 만들어진다.
그런데 누군가 lastName이 성인줄 알고 순서를 다음과 같이 바꾼다고 가정해보자

```java
@AllArgsConstructor
public static class Person {
    private String lastName;
    private String firstName;
}
```

이 경우, IDE가 제공해주는 **리팩토링이 전혀 작동하지 않고**, **lombok이 개발자도 인식하지 못하는 사이에 생성자의 파라미터 순서를 필드 선언 순서에 맞춰 변경해버린다.**

게다가 이 두 필드는 동일한 Type 이라서 기존 생성자 호출 코드에서는 인자 순서를 변경하지 않았음에도, 어떠한 오류도 발생하지 않는다.

이렇게 바뀐 코드는 아무런 에러없이 잘 작동하는 듯 보이지만, 실제로는 `현수권` 이라는 매우 외국 물을 먹은 이름으로 변해버린다..

이러한 문제는 `@AllArgsConstructor` 와 `@RequiredArgsConstructor` 에 둘 다 존재한다.
따라서 이 두 lombok 어노테이션은 사용하지 않도록 지향해야 한다.

대신, 생성자를 직접 만들고 필요한 경우에 직접 만든 생성자에 `@Builder` 어노테이션을 붙이는 것을 권장한다.
그럼 파라미터 순서가 아닌 이름으로 값을 설정하기 때문에 리팩토링에 유연하게 대응할 수 있다.

```java
@AllArgsConstructor
public static class Person {
    private String firstName;
    private String lastName;
    
    @Builder
    private Person(String firstName, String lastName){
        this.firstName = firstName;
        this.lastName = lastName;
    }
}

// 필드 순서를 변경해도 한국식 이름이 만들어진다.
Person me = Person.builder().lastName("현수").firstName("권").build();
System.out.println(me);
```

### @RequiredArgsConstructor 지양

의존 관계를 주입하기 위해 사용하는 어노테이션이다.

의존 주입 방식은 총 3가지가 있다.

- 생성자 주입 방식

- 필드 주입 방식

- Setter 주입 방식

이 중에서 공식 레퍼런스에서는 생성자 주입 방식을 권장하고 있다.

왜냐하면 필수적인 의존성을 모두 만족해야만(파라미터로 모두 받아야만) 객체를 생성할 수 있도록 강제할 수 있기 때문이다.

따라서 직접 생성자 주입 방식을 사용하는 것을 지향하자.

---

## 실제 프로젝트 리팩토링 적용

2021년 하반기부터 컴퓨터학부생을 대상으로 서비스 예정인 `강의실 좌석 예약 시스템 구축 프로젝트`를 진행 중인데, 이번에 공부한 `lombok` 사용법을 실제로 적용하여 다음과 같이 리팩토링을 실시하였다.

### 리팩토링 전 코드

```java
@Entity
@NoArgsConstructor // 문제 상황 1. 기본 생성자의 접근 제어자가 불명확함 
@AllArgsConstructor // 문제 상황 2. 부작용이 많은 모든 파라미터를 받는 생성자 자동 생성
@Builder // 문제 상황 3. 클래스 단위의 Builder 패턴 적용
@Getter
public class Board extends BaseTimeEntity {

    @Id
    @GeneratedValue
    @Column(name="board_id")
    private Long id;

    @Enumerated(EnumType.STRING)
    private Category category;

    private String title;

    private String content;

    private String author;

    @ManyToOne
    @JoinColumn(name="member_id")
    private Member member;

    @Builder.Default @OneToMany(mappedBy="board") // 문제 상황 4. 잘못된 @Builder 위치로 인해 추가해야했던 초기화를 위해 불필요한 코드  
    private List<Comment> commentList = new ArrayList<Comment>();

    public void edit(BoardForm boardForm){
        content = changedInfo(content, boardForm.getContent());
    }

    private String changedInfo(String original, String changed){
        return (changed == null || changed.equals("")) ? original : changed;
    }

    public void setMember(Member member){
        if(this.member!=null){
            this.member.getBoardList().remove(this);
        }
        this.member=member;
        member.getBoardList().add(this);
    }

    // 문제 상황 5. 생성자 메소드 대신 생성해주는 메소드 사용
    public static Board createBoard(Member member,BoardForm boardForm){
        Board board =Board.builder()
            .category(boardForm.getCategory())
            .title(boardForm.getTitle())
            .content(boardForm.getContent())
            .author(member.getNickname())
            .build();

        board.setMember(member);

        return board;
    }
}
```

### 리팩토링 후 코드

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED) // 리팩토링 1. protected 접근 제어자로 생성자의 접근 제어
// @AllArgsConstructor  리팩토링 2. 불필요한 생성자 제거  
@Getter
public class Board extends BaseTimeEntity {

    @Id
    @GeneratedValue
    @Column(name="board_id")
    private Long id;

    @Enumerated(EnumType.STRING)
    private Category category;

    private String title;

    private String content;

    private String author;

    @ManyToOne
    @JoinColumn(name="member_id")
    private Member member;

    @OneToMany(mappedBy="board", cascade = CascadeType.REMOVE) // 리팩토링 3. 클래스 단위의 빌더 패턴 제거로 객체 생성 시 자동으로 List 초기화 
    private List<Comment> commentList = new ArrayList<Comment>();

    public void edit(BoardForm boardForm){
        content = changedInfo(content, boardForm.getContent());
    }

    private String changedInfo(String original, String changed){
        return (changed == null || changed.equals("")) ? original : changed;
    }

    public void setMember(Member member){
        if(this.member!=null){
            this.member.getBoardList().remove(this);
        }
        this.member=member;
        member.getBoardList().add(this);
    }

    @Builder // 리팩토링 4. 생성자 메소드 생성 후 본 메서드에 Builder 패턴 적용  
    public Board (Member member, BoardForm boardForm){
        this.author = member.getNickname();
        this.category = boardForm.getCategory();
        this.title = boardForm.getTitle();
        this.content = boardForm.getContent();
        this.setMember(member);
    }
}

```

## 결론

공부한 내용을 실제로 적용하기 위해 다시 보니까, 그 동안 얼마나 아무 생각없이 그저 제공되는 라이브러리 기능을 사용해왔는지 새로 알게 되었다. 역시 아는 만큼 보이는 것 같다..  

물론 모든 상황에 다 들어맞는 것은 아닐 것이다. 각자의 환경과 상황에 알맞게 `lombok`을 사용하는 것이 가장 바람직하다고 생각한다.

`좋은 캡슐화`, `클린 코드`나, `유지 보수하기 좋은 코드`는 거창한 것이 아니라 이렇게 객체를 생성할 때나 메소드를 사용할 때 조금 더 생각해 보는 것으로부터 비롯된다고 생각한다.