---
layout: single
title: "제네릭을 이용한 엔티티 간접 참조"
categories: project
tag: [주차의상상은현실이된다, 제네릭, JPA, DDD]
typora-root-url: ../
toc: true
author_profile: false
---

“주차의 상상은 현실이된다” 프로젝트를 진행하면서 객체를 직접 참조하는 방법에서 간접 참조하도록 리팩토링한 과정을 기록하고자 한다.

## 상황

우선 서비스에서 사용하고 있는 Parking 엔티티를 보면 다음과 같다.

```java
public class Parking extends AuditingEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Embedded
    private BaseInformation baseInformation;

    @Embedded
    private Location location;

    @Embedded
    private Space space;

    @Embedded
    private FreeOperatingTime freeOperatingTime;

    @Embedded
    private OperatingTime operatingTime;

    @Embedded
    private FeePolicy feePolicy;
}
```

- 주차장을 나타내기 위해 많은 값을 포함하고 있다.
- 또한 필드에서 `@Embedded`로 가지고 있는 필드는 모두 값 객체이다.

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Review {

  private static final int MAX_CONTENTS_SIZE = 3;
  
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "parking_id", nullable = false)
  private Parking parking;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "reviewer_id", nullable = false)
  private Member reviewer;

  @Convert(converter = ContentConverter.class)
  private List<Content> contents;
    
}
```

주차장을 참조하고 있는 리뷰의 경우 테스트에서 주차장 엔티티를 생성하기란 굉장히 번거롭다.

또한, 객체 참조의 경우 **결합도가 높다**.  결합도가 높다는 것은 다른 객체의 기능에 의존할 가능성을 높이는 것이다.

예를 들어, 배송지 정보를 변경하면서 동시에 해당 배송지 정보를 회원의 주소로 변경하는 기능이 있다고 가정하면

```java
public class Order {
	private Orderer orderer;
	
	public void shipTo(ShippingInfo newShippingInfo,
		boolean useNewShippingAddrAsMemberAddr) {
		verifyNotShipped();
		setShippiongInfo(newShippingInfo);
		if (useNewShippingAddrAsMemberAddr) {
			orderer.getMember().changeAddress(newShippingInfo.getAddress());
		}
	}
}
```

위 예시를 보면 주문(Order) 엔티티에서 참조하고 있는 주문자(Orderer) 엔티티를 같이 수정하고 있다.  
트랜잭션 범위를 생각하면 트랜잭션 범위는 작을 수록 좋다.   
하지만, 위의 예시처럼 강결합하고 있다면 여러 객체의 변경이 일어나고 이로 인해 여러 테이블을 수정해서 잠금 대상이 많아지며  
결국에는 성능을 떨어뜨리는 문제를 야기할 수 있다.

이런 이유들을 바탕으로 객체 참조에서 간접 참조로 수정하기로 결정했다.

------

## Id 표현 방식

간접 참조로 바꾸면서 참조 되는 엔티티의 id를 어떻게 변경할 것인가에 대해 생각을 해보았다.

**기본 값으로 나타내기**

```java
private Long reviewerId;
```

장점

- 불필요한 어노테이션이 안붙는다.

단점

- 어떤 것과 연관관계인지 바로 파악하기 힘들다.

**엔티티당 하나씩 값 객체로 나타내기**

```java
@Embeddable
public class MemberId {
	
	private Long id;
}
```

장점

- 어떤 것과 연관 관계인지 파악하기 쉽다.

단점

- 엔티티마다 값 객체를 만들어줘야한다.

**제네릭을 사용한 값 객체로 나타내기**

```java
@Embeddable
public class Association<T> {

    private Long id;
}
```

장점

- 어떤 것과 연관관계가 있는지 파악하기 쉽다.
- 엔티티마다 값 객체를 만들어주지 않아도 된다.

단점을 계속 보완해 가면서 제네릭을 사용한 방법을 찾았다.

------

## 코드 적용

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Review {

  private static final int MAX_CONTENTS_SIZE = 3;
  
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  
  @Embedded
  @AttributeOverride(name = "id", column = @Column(name = "parking_id"))
  private Association<Parking> parkingId;
  
  @Embedded
  @AttributeOverride(name = "id", column = @Column(name = "reviewer_id"))
  private Association<Member> reviewerId;
  
  @Convert(converter = ContentConverter.class)
  private List<Content> contents;
  
}
```

- `Association` 이라는 타입으로 다른 엔티티를 참조하고 있음을 알수 있다.
- 제네릭을 이용함으로서 어떤 엔티티를 참조하는지 알기 쉽다.
  - 만약`reviewerId`를 기본 값으로 표현했다면 생성하는 시점에 무슨 엔티티의 ID인지 헷갈릴 수 있다.

추가로, `@Embedded` 와 `@AttributeOverride`가 중복되고 보기 싫어 질 수 있는데 이는 Converter를 사용하면 된다.

Converter는 JPA에서 자바 객체와 데이터베이스간에 변환을 도와준다. 

```java
@Converter
public class AssociationConverter implements AttributeConverter<Association, Long> {

    @Override
    public Long convertToDatabaseColumn(Association attribute) {
        return attribute.getId();
    }

    @Override
    public Association convertToEntityAttribute(Long dbData) {
        return Association.from(dbData);
    }
}

```

- `convertToDatabaseColumn` 으로 데이터베이스에 저장될 값을 정의하면 된다.
- `convertToEntityAttribute` 으로 데이터베이스 값을 자바 객체로 불러올 때를 정의하면 된다.

이후, 아래 처럼 Converter를 적용해주면 된다.

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Review {

  private static final int MAX_CONTENTS_SIZE = 3;
  
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  
  @Convert(converter = AssociationConverter.class)
  private Association<Parking> parkingId;
  
  @Convert(converter = AssociationConverter.class)
  private Association<Member> reviewerId;
  
  @Convert(converter = ContentConverter.class)
  private List<Content> contents;
  
}
```

- `@Converter(autoApply = true)` 를 통해서 Converter에 글로벌로 설정 할 수 있는 데, 이는 팀에서 정하면 된다. 

---

## 결론

객체 참조 보다는 간접 참조를 사용하는게 결합도를 낮추고 유연한 설계를 할 수 있는거 같다.
