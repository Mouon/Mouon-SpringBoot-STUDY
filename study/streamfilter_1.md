# JPA Fetch Join과 Java Stream의 만남: 실전 필터링 케이스 스터디

스프링과 JPA를 사용하다 보면 연관 관계가 있는 엔티티들을 조회할 때 fetch join을 자주 사용하게 됩니다. 하지만 때로는 fetch join으로 가져온 데이터 중 특정 조건에 맞는 데이터만 필터링해야 하는 경우가 있습니다. 이런 상황에서 Java의 Stream API가 얼마나 우아한 해결책이 될 수 있는지 실제 사례를 통해 알아보겠습니다.

## 🔍 문제 상황

스터디룸 상세 정보를 조회하는 API를 구현하던 중 발생한 재미있는 케이스를 공유합니다. 스터디룸에는 여러 멤버가 있고, 각 멤버는 ACTIVE 또는 DELETE 상태를 가질 수 있습니다.

```java
// 스터디룸 상세 조회 시 모든 멤버 정보를 한 번에 가져오기
MemberStudyroom memberStudyroom = memberstudyroomRepository
    .getStudyroomDetail(studyroomId, memberId, BaseStatus.ACTIVE)
    .orElseThrow(() -> new MemberStudyroomException(NOT_FOUND_MEMBER_STUDYROOM));
```

처음에는 이렇게 조회한 멤버 목록을 변환할 때 다음과 같이 구현했습니다:

```java
// ❌ 문제가 있는 코드
List<DetailStudyroomResponse.Member> members = memberStudyroom
    .getStudyroom()
    .getMemberStudyroomList()
    .stream()
    .map(member -> DetailStudyroomResponse.Member.builder()
        .memberId(member.getMember().getMemberId())
        .nickname(member.getMember().getNickname())
        .avatarId(member.getMember().getAvatar().getAvatarId())
        .colorId(member.getMember().getColor().getColorId())
        .build())
    .collect(Collectors.toList());
```

## 🚨 발견된 문제

위 코드에서는 중요한 문제가 있었습니다. fetch join으로 가져온 MemberStudyroomList에는 **DELETE 상태인 멤버도 포함**되어 있었죠. 즉, 이미 탈퇴한 멤버도 응답에 포함되는 문제가 발생했습니다.

## ✨ Stream API를 활용한 우아한 해결책

Java의 Stream API는 이런 상황에서 매우 우아한 해결책을 제공합니다. `filter` 메소드를 사용하면 원하는 조건의 데이터만 쉽게 필터링할 수 있습니다:

```java
// ✅ 개선된 코드
List<DetailStudyroomResponse.Member> members = memberStudyroom
    .getStudyroom()
    .getMemberStudyroomList()
    .stream()
    .filter(ms -> ms.getStatus().equals(BaseStatus.ACTIVE))  // 👈 활성 상태인 멤버만 필터링
    .map(member -> DetailStudyroomResponse.Member.builder()
        .memberId(member.getMember().getMemberId())
        .nickname(member.getMember().getNickname())
        .avatarId(member.getMember().getAvatar().getAvatarId())
        .colorId(member.getMember().getColor().getColorId())
        .build())
    .collect(Collectors.toList());
```

## 💡 왜 이 방식이 좋은가?

1. **DB 조회 최소화**
    - 스터디룸 정보를 가져올 때 이미 fetch join으로 모든 데이터를 한 번에 조회
    - 추가 쿼리 없이 메모리에서 필터링 수행
    - 데이터가 100건 이하인 경우 DB 커넥션 비용 > 메모리 처리 비용

2. **코드의 가독성**
    - Stream API의 선언적 프로그래밍 방식으로 의도가 명확
    - filter와 map을 통한 데이터 처리 흐름이 직관적

3. **유지보수성**
    - 필터링 조건 변경이 필요할 때 filter 부분만 수정하면 됨
    - 체이닝된 메소드로 처리 단계가 명확히 구분됨

## 🤔 고려사항

물론 이 방식이 항상 최선은 아닙니다. 다음과 같은 상황에서는 DB 단에서 필터링하는 것이 더 좋을 수 있습니다:

- 데이터 건수가 매우 많은 경우
- 메모리 사용량이 중요한 경우
- 네트워크 대역폭이 제한적인 경우

하지만 일반적인 스터디룸 서비스에서 멤버 수가 100명 이하라면, 현재 방식이 더 실용적이고 효율적인 선택이 될 수 있습니다.

## 📝 결론

JPA의 fetch join과 Java Stream API의 조합은 실무에서 자주 마주치는 데이터 필터링 요구사항을 우아하게 해결할 수 있는 강력한 도구입니다. 특히 적절한 규모의 데이터를 다룰 때는, 추가 쿼리 없이 메모리에서 처리하는 것이 더 효율적일 수 있다는 점을 기억해두면 좋겠습니다.