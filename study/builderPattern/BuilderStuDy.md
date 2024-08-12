# 빌더 패턴 도입기

스프링을 고민하던 중 어떻게 하면 더 가독성 좋고 깔끔한 코드를 짤 수 있을까?' 라는 고민을 하게되었다.  
그러다가 그 질문을 해결하기위해! 나는 다른 개발자들의 깃허브를 뒤지기 시작했다. (실력자들의 깃허브를 보며 배우는 점이 많은 것 같다, 고민할 지점도 많아지고)


그러다 발견한 것이 바로 @Builder 애노테이션이었다. 
사실 이 애노테이션을 몰랐던 것은 아니었다.  
처음엔 그저 '아, 이런 게 있구나'하고 넘겼다. 
솔직히 말하면, 생성자로 값을 주입하는 게 익숙했고 
생소한 걸 도입하기엔 기능을 구현 하는 것만으로도 벅차다고 생각했다..  

하지만 이제는 builder를 사용해서 코드의 가독성과 개발 안정성을 높이는 도전을 해도 되겠다는 생각이 들었다.  

처음에는 조금 어색했다. 기존에 쓰던 코드를 이렇게 바꾸는 게 맞나 싶기도 했고.   
하지만 조금씩 적용해 나가면서 장점이 많은 친구라는 것을 깨달았다.  

특히 DTO를 만들 때 빛을 발했다. 예전에는 필드가 추가될 때마다 생성자도 여러 개 만들어야 했는데,   
빌더를 쓰니 그럴 필요가 없어졌다. 
원하는 필드만 쏙쏙 골라서 객체를 만들 수 있으니 말이다!!  

먼저 기존에 사용하던 코드를 살펴보겠다. 
먼저 DTO 이다.  
```java
@Getter
@AllArgsConstructor
@NoArgsConstructor
public class DetailStudyroomResponse {

    private Long studyroomId;
    private MemberRole role;
    private List<Member> members;

    @Getter
    @AllArgsConstructor
    public static class Member{
        Long memberId;
        String nickname;
        Long avatarId;
    }

}
```

아래 코드는 스터디룸의 정보와 그 스터디룸에 속한 맴버들의 정보를 가져와 DTO 객체에 저장하는 메서드이다.  
```java

public DetailStudyroomResponse getStudyroomDetail(long studyroomId, long memberId) {
    List<Object[]> results = memberstudyroomRepository.getStudyroomDetail(studyroomId, memberId, BaseStatus.ACTIVE);

    if (results.isEmpty()) {
        throw new MemberStudyroomException(NOT_FOUND_MEMBER_STUDYROOM);
    }
    Object[] firstRow = results.get(0);
    MemberRole role = (MemberRole) firstRow[0];

    List<DetailStudyroomResponse.Member> members = results.stream()
            .map(row -> new DetailStudyroomResponse.Member(
                    (Long) row[1],   // memberId
                    (String) row[2], // nickname
                    (Long) row[3]    // avatarId
            ))
            .collect(Collectors.toList());

    return new DetailStudyroomResponse(studyroomId, role, members);
}

```
이 코드에서는 생성자를 사용해 객체를 생성하고 있다. 그리 복잡해 보이지 않을 수 있지만,   
필드가 많아질수록 생성자도 커지고, 어떤 값이 어떤 필드에 들어가는지 한눈에 파악하기 어려워진다.  
(작성자인 나도 각 값이 어떤값인지 명시하기위해 주석을 달아놓았다..)  
이렇게되면 타입세이프한 개발이 어려워진다. (실제로 개발하다 파라미터를 반대로 넣어 발생하는 오류는 빈번하다...)   


이제 빌더 패턴을 적용해 보자.  
적용하기 위해선 DTO 클래스의 코드먼저 수정이 필요하다.  
```java
@Getter
@NoArgsConstructor
public class DetailStudyroomResponse {

    private Long studyroomId;
    private MemberRole role;
    private List<Member> members;

    @Builder
    public DetailStudyroomResponse(Long studyroomId, MemberRole role, List<Member> members){
        this.studyroomId = studyroomId;
        this.role = role;
        this.members = members;
    }

    @Getter
    public static class Member{
        Long memberId;
        String nickname;
        Long avatarId;
        Long colorId;

        @Builder
        public Member(Long memberId, String nickname, Long avatarId, Long colorId) {
            this.memberId = memberId;
            this.nickname = nickname;
            this.avatarId = avatarId;
            this.colorId = colorId;
        }
    }
}
```
생성자에 `@Builder`애노테이션을 붙여주게되면, 스프링에서는 자동으로 빌더패턴을 생성해준다.  
이제 서비스 계층에 이를 사용해보겠다.  

```java
public DetailStudyroomResponse getStudyroomDetail(long studyroomId, long memberId) {
    List<Object[]> results = memberstudyroomRepository.getStudyroomDetail(studyroomId, memberId, BaseStatus.ACTIVE);

    if (results.isEmpty()) {
        throw new MemberStudyroomException(NOT_FOUND_MEMBER_STUDYROOM);
    }
    Object[] firstRow = results.get(0);
    MemberRole role = (MemberRole) firstRow[0];

    List<DetailStudyroomResponse.Member> members = results.stream()
            .map(row -> DetailStudyroomResponse.Member.builder()
                    .memberId((Long) row[1])
                    .nickname((String) row[2])
                    .avatarId((Long) row[3])
                    .colorId((Long) row[4])
                    .build())
            .collect(Collectors.toList());

    return DetailStudyroomResponse.builder()
            .role(role)
            .studyroomId(studyroomId)
            .members(members)
            .build();
}
```
확실히 @Builder를 사용한 코드가 훨씬 더 읽기 쉽고, 어떤 값을 설정하는지 명확하게 보이는 것을 확인할 수 있다.  


## 생성자 vs 빌더: 어떤 차이가 있을까?
위에서 보았 듯이 처음에 사용하던 방식은 다음과 같았다.  

```java
public DetailStudyroomResponse(Long studyroomId, MemberRole role, List<Member> members) {
    this.studyroomId = studyroomId;
    this.role = role;
    this.members = members;
}
```  
필요에 따라 여러 개의 생성자를 만들고, 값을 주입했다.  
이렇게 코드를 작성하다 보니, 시간이 지날수록 생성자 메소드가 길어지고,   
어떤 값이 필요한지 헷갈리는 일이 생겼다.  

하지만 아래처럼 @Builder를 사용하면 확실히 코드가 더 깔끔하고 어떤 값을 주입하는지 명시적으로 알 수 있다.  
```java
return DetailStudyroomResponse.builder()
    .role(role)
    .studyroomId(studyroomId)
    .members(members)
    .build();
```
게다가 코드가 길어지더라도 필요한 부분만 설정할 수 있어서 불필요한 생성자 오버로드도 피할 수 있었다.  


## 빌더 패턴의 장점
내가 느끼는 빌더 패턴의 가장 큰 장점은,
객체를 생성할 때 어떤 값을 설정하고 있는지가 명확하다는 점이다.  

또 다른 장점은 새로운 기능을 추가할 때도, 여러 생성자를 만들 필요 없이 
@Builder를 사용하면 되니 코드가 훨씬 간결해진다는 점이다.
그리고 무언가를 수정하거나 확장할 때도, 코드의 수정이 용이하다는 장점이 있다.  

## 정리

@Builder 애노테이션을 사용하는 빌더패턴은 유지보수,가독성 측면에서 많은 장점을 보여주고 있다.  
DTO뿐만 아니라 도메인 클래스에도 사용한다면 플젝트의 안정적인 개발을 도울 수 있을 것이라고 생각된다!  











