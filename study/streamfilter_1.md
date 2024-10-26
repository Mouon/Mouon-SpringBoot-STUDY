# JPA Fetch Join과 Java Stream의 조합

스프링과 JPA를 사용하다 보면 연관 관계가 있는 엔티티들을 조회할 때 fetch join을 자주 사용하게 된다. 
이때 경우에 따라서 fetch join으로 가져온 데이터 중 특정 조건에 맞는 데이터만 필터링해야 하는 경우가 있다.(추가 필터링 쿼리 없이)  
이런 상황에서 Java의 Stream API가 얼마나 깔끔한 해결책이 될 수 있는지 실제 프로젝트를 진행하면서 겪은 사례를 통해 알아보겠다.

## 🧐 문제 상황

스터디룸 상세 정보를 조회하는 API를 구현하던 중 있던일이다.
대략적으로 설명하자면, 스터디룸에는 여러 멤버가 있고, 각 멤버는 ACTIVE 또는 DELETE 상태를 가질 수 있다.

```java
// 스터디룸 상세 조회 시 모든 멤버 정보를 한 번에 가져오기
MemberStudyroom memberStudyroom = memberstudyroomRepository
    .getStudyroomDetail(studyroomId, memberId, BaseStatus.ACTIVE)
    .orElseThrow(() -> new MemberStudyroomException(NOT_FOUND_MEMBER_STUDYROOM));
```

처음에는 이렇게 조회한 나의 특정 스터디룸 정보를 바탕으로 현재 함께 속한 스터디원들의 정보를 조회할때 아래와 같이 구현했다.  
방식은 특정유저의 `memberStudyroom` 정보를 바탕으로 연관관계를 타고 타고 들어가 `studyroom`을 거쳐 다시 그 스터디룸의 `memberStudyroomList`의 정보를 조회한 다음  
다시 맴버정보를 조회해서 DTO에 매핑하는 그런 방식이었다. 물론 이 정보를 한번의 쿼리로 가져오기위해 fetch join으로 연관관계를 모드 로드했다.:

```java
/**
 * 🤯 문제 코드 
  */
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

위 코드에서는 중요한 문제가 있었다. 
fetch join으로 가져온 MemberStudyroomList에는 **DELETE 상태인 멤버도 포함**되어 있다는 것이다.  
즉, 이미 탈퇴한 멤버도 응답에 포함되는 문제가 발생했다.  
이유는 조회를 요청한 유저의 `memberStudyroom`은 `getStudyroomDetail(studyroomId, memberId, BaseStatus.ACTIVE)`을 통해 `BaseStatus.ACTIVE`만 조회한 반면에  
해당 `memberStudyroom`의 연관 관계를 타고들어가서 `getMemberStudyroomList()`을 통해 다시 조회한 해당 스터디룸의 `memberStudyroom`들은 
`BaseStatus.ACTIVE`라는 조건에 해당하지 않아도 되기 때문이다.


## Stream API를 활용한 깔끔한(센스있는) 해결책

Java의 Stream API는 이런 상황에서 매우 깔끔한 해결책을 제공한다. 
`filter` 메소드를 사용하면 원하는 조건의 데이터만 쉽게 필터링할 수 있다:

```java
/**
 * 🥳 개선된 코드
 */

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

## 왜 이 방식이 유용하지?

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

## 🤔 주의사항

물론 이 방식이 항상 최선은 아니다.  
데이터 건수가 매우 많은 경우와 같은 상황에서는 DB 단에서 필터링하는 것이 더 좋을 수 있다.  
하지만 일반적인 조회 API에서 멤버 수가 100명 이하라면, 현재 방식이 더 실용적이고 효율적인 선택이 될 수 있다고한다.

## 마무리

JPA의 fetch join과 Java Stream API의 조합은 개인적으로 잘 어울리는 조합이라고 생각한다.
특히 적절한 규모의 데이터를 다룰 때는, 추가 쿼리 없이 메모리에서 처리하는 것이 더 효율적인데
이때 직관적이며 깔끔하고 유지보수가 용이한 코드를 작하게 해준다는 점을 기억해두면 좋을 것 같다.

참고 : [[우아콘2020] 수십억건에서 QUERYDSL 사용하기](https://www.youtube.com/watch?v=zMAX7g6rO_Y)