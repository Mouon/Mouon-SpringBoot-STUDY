# 영속성 컨텍스트

영속성 컨텍스트란 JPA가 엔티티를 영구 저장하는 환경을 말합니다. 쉽게말해서 JPA가 엔티티를 추적하고 관리하는 영역입니다.

이 환경에서 엔티티는 영속 상태로 관리됩니다.

예를들어 영속성 컨텍스트에서 Member 엔티티를 저장하면, 해당 엔티티는 영속성 컨텍스트 내에 존재하게 됩니다. 다시말해 JPA가 그 엔티티를 '기억'하고 있게 됩니다.

트랜잭션이 커밋되면 JPA가 영속성 컨텍스트에 있는 엔티티를 데이터베이스에 반영합니다.

따라서 Member 엔티티를 영속성 컨텍스트에 저장하면, 해당 엔티티는 트랜잭션이 커밋되는 시점에 데이터베이스에 삽입됩니다.
```
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
해당 Member 엔티티는 영속성 컨텍스트에 영속 상태로 관리되고, 트랜잭션이 커밋될 때 영속성 컨텍스트에 있는 모든 변경 사항이 데이터베이스에 반영됩니다.


코드상에서 트랜잭션이 커밋되는 경우, 즉 트랜잭션이 끝나는 경우는 아래 코드에서 확인이 가능합니다.

```
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

트랜잭션과 관련이 있는 부분은 join 메서드입니다.

이 메서드 내에서 memberRepository.save(member)가 호출되고 있습니다.

해당 메서드는 @Transactional 어노테이션이 적용되어 있으므로, 메서드 실행이 완료될 때 트랜잭션이 커밋됩니다.

따라서 join 메서드가 끝날 때 트랜잭션이 커밋되고, 이 때 데이터베이스에 실제로 Member가 저장됩니다.

따라서 Member를 저장하고 있는 memberService.join(member) 호출이 바로 트랜잭션이 커밋되는 시점입니다.

이 시점에서 데이터베이스에 변경사항이 반영됩니다.
