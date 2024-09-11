# 세터 사용을 지양하는 이유

이번 포스팅은 특별한 내용이 아니라, 김영한님의 강의의 내용중에 개발시 앞으로 기억해야할 부분이 있어서 기록용으로 작성.

혼자 스프링부트에서 엔티티를 다룰 때, '세터(setter)' 메서드를 통해 객체의 상태를 변경하는 방식으로 코드를 작성할 때가 있었다.  
예를 들어, `setName()`, `setAge()`, `setEmail`와 같은 메서드들이 그 예이다. 이러한 방식은 간단하고 직관적으로 보일 수 있으나, 객체의 상태 변경을 추적하기 어렵게 만들 수 있는 단점이 있다.  
협업을 하거나 나중에 프로젝트 규모가 커지면 어느지점이 변경지점인지 찾기 어려워서 유지보수성이 떨어질 수 있다.

### 세터(setter)를 통한 상태 변경의 문제점

1. 변경 포인트 추적의 어려움  
  객체의 상태가 여러 세터를 통해 언제 어디서 변경되었는지 추적하기 어렵다.   
특히 큰 프로젝트에서는 수많은 서비스에서 하나의 엔티티를 수정할 수 있으며, 이로 인해 디버깅이나 유지보수가 복잡해질 수 있다. (하나의 비즈니스로직으로 캡슐화 되지않으면 setter들이 따로따로 작동하는 느낌이라 유지 보수성면에서 치명!!)

2. 무분별한 상태 변경  
세터 메서드를 공개적으로 제공하면, 객체의 상태를 누구나 변경할 수 있게 된다. 이는 객체의 불변성을 보장하기 어렵게 만들고, 예상치 못한 부작용을 일으킬 수 있다.

3. 객체의 일관성 유지 어려움  
여러 세터를 통해 객체의 상태를 변경할 때, 필요한 모든 상태 변경을 완료하기 전까지 객체가 일시적으로 일관성이 없는 상태에 놓일 수 있다.  
예를 들어, 유저의 나이나 이메일을 변경하는 도중에는 객체의 상태가 실제 비즈니스 규칙을 반영하지 않을 수 있다. set매서드를 조금이라도 빠뜨리면 비즈니스 로직이 깨질 수 있다.

### 메서드를 통한 해결 방안

이러한 문제들을 해결하기 위한 한 가지 접근 방법은 '의미 있는 메서드'를 사용하여 객체의 상태를 변경하는 것이다.   
즉, 객체의 상태를 변경하는 로직을 잘 정의된 메서드 내에 캡슐화하는 것이다. 
예를 들어, `updateMember(age,email)` 메서드를 만들어 유저의 나이, 이메일만 한 번에 업데이트할 수 있게 하는 것이다!!

이 방식의 장점은 다음과 같다.

1. 변경 로직의 명확성    
   객체의 상태 변경 로직이 명확하게 한 곳에 정의되어 있어, 어떤 변경이 이루어지는지 쉽게 이해할 수 있다.
   코드의 재사용성도 올라갈 수 있다.

2. 일관성 유지    
   메서드 내에서 모든 상태 변경을 완료하기 때문에, 객체가 항상 일관된 상태를 유지하게 된다. 또한, 비즈니스 규칙에 따라 상태 변경을 더욱 엄격하게 제어할 수 있다.

3. 캡슐화와 추상화   
   객체의 내부 상태 변경 방식을 캡슐화함으로써, 객체 외부에서는 상태 변경의 복잡성을 몰라도 된다. 이는 코드의 추상화 수준을 높이고, 유지보수성을 향상시킨다.

#### 결론
의미 있는 메서드를 통한 상태 변경은 객체 지향 프로그래밍의 특징을 잘 반영하며, 코드의 가독성, 유지보수성 등을 향상시키는 방법으로 실무에서 많이 사용되는 방법이다.


#### 진짜 결론
웬만하면 setter쓰지말자 