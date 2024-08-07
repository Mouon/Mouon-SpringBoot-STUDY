# 영속성 컨텍스트

영속성 컨텍스트란 JPA가 엔티티를 영구 저장하는 환경을 말한다. 쉽게말해서 JPA가 엔티티를 추적하고 관리하는 영역인셈이다.

이 환경에서 엔티티는 영속 상태로 관리된다.

예를들어 영속성 컨텍스트에서 Member 엔티티를 저장하면, 해당 엔티티는 영속성 컨텍스트 내에 존재하게 된다. 다시말해 JPA가 그 엔티티를 '기억'하고 있게 된다!

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

#### 정리하자면, memberService.join(member) 종료가 바로 트랜잭션이 커밋되는 시점이다.

### 마무리
트랜잭션이 커밋되는 시점에 데이터베이스에 변경사항이 반영된다!
