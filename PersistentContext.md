# 영속성 컨텍스트

영속성 컨텍스트란 JPA가 엔티티를 추적하고 관리하는 메모리상의 영역이다.

이 환경에서 엔티티는 영속 상태로 관리된다.

예를들어 영속성 컨텍스트에서 Member 엔티티를 저장하면, 해당 엔티티는 영속성 컨텍스트 내에 존재하게 된다.
다시말해 JPA가 그 엔티티를 '기억'하고 있게 된다!(메모리에서 관리)

트랜잭션이 커밋되면 JPA가 영속성 컨텍스트에 있는 엔티티를 데이터베이스에 반영한다.  

따라서 Member 엔티티를 영속성 컨텍스트에 저장하면, 해당 엔티티는 트랜잭션이 커밋되는 시점에 데이터베이스에 삽입된다.
```java
@Repository
@RequiredArgsConstructor
public class MemberRepository {

    @PersistenceContext
    private final EntityManager em;

    public void save(Member member) {
        em.persist(member);
    }

}
```

위 코드에서 MemberRepository의 save 메서드를 호출하여 Member 엔티티를 저장하면,   
해당 Member 엔티티는 영속성 컨텍스트에 영속 상태로 관리되고, 트랜잭션이 커밋될 때 영속성 컨텍스트에 있는 모든 변경 사항이 데이터베이스에 반영된다.


코드상에서 트랜잭션이 커밋되는 경우, 즉 트랜잭션이 끝나는 경우는 아래 코드에서 확인이 가능하다.

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    @Transactional
    public Long join(Member member) {

        validateDuplicateMember(member); //중복 회원 검증
        memberRepository.save(member);
        return member.getId();
    }

}
```
트랜잭션은 @Transactional 애노테이션이 적용된 메서드가 실행되는 동안 활성화된다. 
메서드가 완료되고 예외가 발생하지 않으면 트랜잭션은 커밋된다.  

위 내용을 예제에 적용해보면 트랜잭션과 관련이 있는 부분은 join 메서드이다.  
이 메서드 내에서 memberRepository.save(member)가 호출되고 있다.  
해당 메서드는 @Transactional 어노테이션이 적용되어 있으므로, 메서드가 실행되는 동안 트랜잭션이 활성화 되고, 메서드 실행이 완료될 때 트랜잭션이 커밋된다.
즉 join 메서드가 끝날 때 트랜잭션이 커밋되고, 이 때 데이터베이스에 실제로 Member가 저장되는 것이다.  

##### 정리하자면, memberService.join(member) 종료가 바로 트랜잭션이 커밋되는 시점이다.

그런데 여기서 주의할점은 트랜잭션이 커밋됬다고 해서 DB와의 커넥션이 바로 끊기는지 여부이다.  

이와 관련된 설정으로 `spring.jpa.open-in-view`가 있다. 이 설정은 영속성 컨텍스트와 데이터베이스 커넥션의 수명을 제어하며, 기본값은 `true`로 설정되어 있다.  

보통 진행중인 프로젝트를 보면 이설정을 명시적으로 false로 해두었을 것이다.  

`open-in-view`가 `true`일 때는 API 응답 시점까지 커넥션과 영속성 컨텍스트가 유지된다.  
이로 인해, 비즈니스 로직에서 데이터를 처리하는 동안 커넥션을 유지하게되는데 이로인해 **성능 문제**와 **커넥션 풀 고갈** 문제를 유발할 수 있다.  
반면에 `false`로 설정하면 트랜잭션 종료 시점에 커넥션과 영속성 컨텍스트가 반환된다.  
때문에 [토비의 본 TV - JPA의 신 김영한 님](https://www.youtube.com/watch?v=00qwDr_3MC4) 영상을 보면 영한님께서는 트래픽이 많이들어오는 서버에서는 해당옵션을 끄고, 
어드민같이 커넥션을 오래 유지(커넥션을 오래물고있어도되는)서버에서는 켜도 무방하다고 셨다.


### 마무리
트랜잭션이 커밋되는 시점에 데이터베이스에 변경사항이 반영된다!
