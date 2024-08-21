# 지연로딩과 프록시

`@ManyToOne(fetch = FetchType.LAZY)`를 사용하는 이유는 연관된 엔티티를 지연 로딩하기 위함이다. 
이는 연관된 엔티티를 필요할 때만 로딩하여 메모리 사용을 최적화하고, 불필요한 데이터 로딩을 방지하기 위한 목적이다.

반대로, 즉시 로딩(EAGER)을 사용하면 데이터 조회 시 연관된 모든 엔티티를 한 번에 불러오기 때문에, 필요한 데이터보다 더 많은 데이터를 미리 로딩하게 되어 메모리를 더 많이 소비할 수 있다. 또한, 연관된 엔티티가 많을 경우, 한 번의 조회로 많은 양의 데이터를 불러오는 것이 성능에 영향을 미칠 수 있다.

때문에 스프링부트에서는 `@ManyToOne(fetch = LAZY)` 와 같이 지연로딩기능을 애노테이션으로써 구현할 수 있게 지원한다.  
참고로 fetch의 디폴트 값은 `@xxToOne`에서는 EAGER, `@xxToMany` 에서는 LAZY이다.  

## 그런데 스프링부트는 어떻게 지연로딩을 구현할까?
![img.png](img.png)  
지연 로딩의 원리는 프록시와 리플렉션에 있다. 
JPA 구현체(예: Hibernate)는 리플렉션을 사용하여 클래스 필드와 데이터베이스 컬럼을 매핑한다. 
지연 로딩이 설정된 필드에 대해서는 프록시 객체가 생성되어, 실제 데이터가 필요할 때까지 데이터베이스 접근을 지연시킨다


### 리플렉션과 프록시의 개념이 낯설다면 위의 설명이 와닿지 않을 수 있기때문에 간단하게 두 개념을 짚고 넘어가보자  

## 리플렉션
리플렉션은 간단하게 프로그램이 프로그램이 실행 중에 동적으로 클래스의 구조를 검사하는 것이라고 보면 된다! 모든 클래스와 메서드가 컴파일 타임에 고정되는 일반적인 방식과 다르다고 볼 수 있다.

## 프록시 객체
프록시객체는 실제 객체를 대신하여 사용되는 가상의 객체라고 생각하면 쉽다. 처음에는 실제로 객체를 생성하지는 않고 가상의 객체를 생성해두고 필요할시 실제 객체를 생성하거나 데이터베이스에서 불러오게된다.
즉 이 객체는 처음부터 실제 객체를 생성하지 않고, 필요할 때 실제 객체를 생성하거나 호출하는 역할을 한다.
예를 들어, 메뉴판에 적힌 메뉴가 프록시 객체라면, 실제 음식은 실제 객체라고 할 수 있다. 
메뉴판(프록시 객체)은 음식(실제 객체)을 대신하여 손님에게 보여지며, 손님이 주문할 때 비로소 실제 음식(실제 객체)이 준비되는 것과 같은 원리로 이해하면 된다.

다시 지연로딩으로 돌아와 보자면 스프링에서 도메인을 설계시 N+1 문제는 다음과 같은 경우 발생할 수있다.

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    
    @OneToMany(fetch = FetchType.EAGER, mappedBy = "user")
    private List<Order> orders;
    
}
```
위 코드를 보면 하나의 User 엔티티가 여러 Order 엔티티와 연관되어 있다. 이때 Order 엔티티가 fetch = FetchType.EAGER 으로 매핑되어있다.  
때문에 User 엔티티를 조회하면 연관되어있는 order 리스트가 즉시로드되게된다.  


아래와 같이 지연로딩으로 해당 문제를 해결할 수 있다.  
```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "user")
    private List<Order> orders;
    
}

```
fetch = FetchType.LAZY 설정은 User 객체를 가져올 때 orders 리스트를 
즉시 로드하지 않고, 
실제로 orders 리스트가 필요할 때까지 지연시키는 역할을 한다.

## 간단하게 내부 ORM 프레임워크 구현을 살펴보자!


스프링에서는 Hibernate와 같은 ORM 프레임워크는 프록시 객체를 사용하여 지연 로딩을 구현한다. 
아래는 Hibernate에서 프록시 객체를 사용한 지연 로딩 동작 예시이다.  
```java
    User user = entityManager.find(User.class, userId); // User 객체는 로드됨
    List<Order> orders = user.getOrders(); // 이 시점에 프록시 객체가 작동하여 Order 객체들을 로드!!!
```
처음 User 객체를 로드할 때, orders 필드는 프록시 객체로 설정된다.
user.getOrders() 메서드를 호출할 때 프록시가 데이터베이스와 상호작용하여 실제 Order 객체들을 가져오게된다.

