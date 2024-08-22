# 정적 팩토리 메서드와 빌더 패턴의 적절한 사용에 대한 고민

개발을 하면서 정적 팩토리 메서드와 빌더 패턴에 대한 고민이 자주 생긴다. 
특히 서비스 레이어에서 이 둘을 어떻게 조합해서 사용하는 것이 더 나은 접근인지에 대해 생각해볼 필요가 있다고 느꼈다. 
최근 깃허브 파도타기를 하던 중 도메인 클래스에 정적 팩토리 메서드를 적용한 사례를 보면서 이 고민이 더욱 깊어졌다.

예를 들어, 도메인을 담당하는 `Member` 클래스에서 아래와 같이 `of`라는 정적 팩토리 메서드가 사용된 코드를 보았다.

```java
public static Member of(String providerId, String name, String email) {
    return Member.builder()
        .providerId(providerId)
        .name(name)
        .email(email)
        .build();
}
```

이 코드에서는 빌더 패턴을 활용하여 객체를 생성하는데, 이를 정적 팩토리 메서드로 감싸서 제공하고 있다. 이렇게 하면 서비스 레이어에서 `of` 메서드를 통해 객체를 쉽게 생성할 수 있다. (new 로 생성자를 만들지 않아도된다.)  

하지만, 파라미터의 순서에 의존하게 된다는 점에서 빌더 패턴을 사용하는 본래의 의도가 다소 희석되는 느낌이 들기도 한다.  
빌더 패턴의 장점 중 하나는 파라미터 순서에 의존하지 않고 객체를 생성할 수 있다는 점인데, 정적 팩토리 메서드를 사용하면 다시 순서에 의존하게 된다.(180 바뀐게아니라 360도 바뀐 느낌이랄까...)  


이와 관련해 [관련 블로그 글](https://jiwondev.tistory.com/193)에서도 비슷한 고민이 언급되어있었다. 
정적 팩토리 메서드를 사용할 때의 장점과 단점, 그리고 그 적절한 사용 시점에 대해 잘 설명되어 있다.   
이 글을 읽으면서 서비스 레이어에서의 사용에 대해 다시 생각해보게 되었다.

나는 정적 팩토리 메서드를 주로 DAO에서 엔티티를 가져와 응답 DTO를 만들 때 사용하는 것이 더 적절하다고 생각해왔다. 
예를 들어, 생성 기능에서 엔티티 객체를 만들고, 그 객체에서 필요한 값만 응답으로 주는 경우에 정적 팩토리 메서드를 사용하는 것이 더 맞지 않나 싶다.  
물론 모든 상황에서 이 방식이 정답은 아니기때문,. 상황에 따라 다른 접근이 필요할 수 있다.

아래는 실제로 정적 팩토리 메소드를 사용한 코드이다.

```java
@Transactional
public UploadDataResponse uploadData(UploadDataRequest request, long memberId) {
    log.info("[DataService.uploadData]");
    MemberStudyroom memberStudyroom = memberStudyroomRepository.findByMemberIdAndStudyroomIdAndStatus(memberId, request.getStudyroomId(), BaseStatus.ACTIVE)
        .orElseThrow(() -> new MemberStudyroomException(NOT_FOUND_MEMBER_STUDYROOM));
    try {
        String[] dataInfo = extractDataNameAndUrl(request);
        String dataName = dataInfo[0];
        DataType dataType = request.getDataType();
        String dataUrl = dataInfo[1];
        Data savedData = saveData(dataName, dataType, dataUrl, memberStudyroom.getMember(), memberStudyroom.getStudyroom());
        return UploadDataResponse.from(savedData);
    } catch (NullPointerException e) {
        throw new DataException(NONE_FILE);
    }
}
```


아래 코드는 데이터 저장 기능을 수행하는 서비스 계층의 코드이다.  
이 경우, 정적 팩토리 메서드보다는 빌더 패턴을 통해 명확하게 객체를 생성하는 것이 더 적절하다고 판단했다.  
그래서 아래와 같이 빌더 패턴을 사용해 엔티티를 생성하고 있다.  
```java
@Transactional
public Data saveData(String dataName, DataType dataType, String dataUrl, Member member, Studyroom studyroom) {
    log.info("[DataService.saveData]");
    OpenGraphData openGraphData = (dataType == DataType.LINK) ? extractOpenGraphData(dataUrl) : new OpenGraphData(null, null, null, null);
    Data data = Data.builder()
        .dataName(dataName)
        .dataType(dataType)
        .status(BaseStatus.ACTIVE)
        .studyroom(studyroom)
        .dataUrl(dataUrl)
        .member(member)
        .ogTitle(openGraphData.getOgTitle())
        .ogDescription(openGraphData.getOgDescription())
        .ogImage(openGraphData.getOgImage())
        .ogType(openGraphData.getOgType())
        .build();
    return dataRepository.save(data);
}
```

이렇게 작성하면 객체 생성 시 파라미터 순서에 의존하지 않으면서도, 필요한 필드만 선택적으로 초기화할 수 있다. 빌더 패턴의 장점을 충분히 활용하는 방식이라고 생각한다.  

결국, 정적 팩토리 메서드와 빌더 패턴의 사용은 상황에 따라 달라질 수 있다.  
두 패턴을 어떻게 조합하여 사용할지에 대한 고민은 계속될 것 같다.  
다만, 각 패턴의 장단점을 잘 이해하고, 그에 따라 유연하게 접근하는 것이 중요하다고 느꼈다.  
이 글을 통해 이러한 학습 과정과 고민을 기록하고, 앞으로 더 나은 방법을 찾아가고자 한다.