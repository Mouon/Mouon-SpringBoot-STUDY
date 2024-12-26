# 리팩토링을 시작하며.. 

끝날것 같지않던 졸업프로젝트의 최종 발표가 마무리된지 한달이 지난 시점... 토스 페이먼츠의 서류합격과 과제전형.. 많은 일이있었다...
좋은 기회로 토스 과제 전형을 진행하면서 내게 지금 부족한 게 무엇인지, 집중해야하는 부분이 무엇인지 등 앞으로의 공부방향을 잡을 수 있었고 
때문에 졸업프로젝트의 리팩토링 작업을 진행하게 되었다... 
첫 리팩토링 작업은 아래와 같다.  
[첫 리팩토링 작업 상세 기록](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/Pessimistic_Lock.md)  
<br>  

# 코드에 질문던지기 🤔
[지속 성장 가능한 코드를 만들어가는 방법](https://youtu.be/RVO02Z1dLF8?si=ChmwCOt2DGgqMFuP) 이라는 영상을 보며 기존 코드를 보며 지속적으로 질문을 던지는것이 좋은 자세라는 것을 느끼게 되었다.  


### 코드에 던져보는 간단한 질문

"여러 곳에서 비슷한 기능을 하는 코드가 있네? 이거 뭔가 개선할 수 있지 않을까?"

```java
/** 방장으로 가입 */
@Transactional
public void joinStudyroomAsCaptain(long studyroomId, long memberId){
    JoinStudyroomRequest joinStudyroomRequest = JoinStudyroomRequest.builder()
                    .studyroomId(studyroomId)
                    .memberId(memberId)
                    .memberRole(MemberRole.CAPTAIN)
                    .build();
    joinStudyroom(joinStudyroomRequest);
    log.info("Success Join Studyroom as Captain");
}

/** 초대 코드로 가입 */
@Transactional
public JoinStudyroomByCodeResponse joinStudyroomByCode(JoinStudyroomByCodeRequest request, long memberId){
    log.info("[StudyroomService.joinStudyroomByCode]");
    Long studyroomId = inviteService.findRoomIdByInviteCode(request.getInviteCode())
            .orElseThrow(()->new StudyroomException(INVALID_INVITE_CODE));
    Studyroom studyroom = studyroomRepository
            .findById(studyroomId).orElseThrow(()->new StudyroomException(INVALID_INVITE_CODE));
    if (memberstudyroomRepository.findByMemberIdAndStudyroomIdStatus(memberId,studyroomId,BaseStatus.ACTIVE).isPresent()){
        throw new MemberException(JOINED_STUDYROOM);
    }
    JoinStudyroomRequest joinStudyroomRequest = JoinStudyroomRequest.builder()
            .studyroomId(studyroomId)
            .memberId(memberId)
            .memberRole(MemberRole.CREW)
            .build();

    joinStudyroom(joinStudyroomRequest);
    return JoinStudyroomByCodeResponse.from(studyroom);
}


/** 초대 코드가 필요없는 가입 */
@Transactional
public void joinStudyroom(JoinStudyroomRequest request){
    log.info("[StudyroomService.joinStudyroom]");
    Member member = memberRepository.findById(request.getMemberId())
            .orElseThrow(()->new MemberException(NOT_FOUND_MEMBER));
    synchronized(this) {
        Studyroom studyroom = studyroomRepository.findById(request.getStudyroomId())
                .orElseThrow(() -> new StudyroomException(NOT_FOUND_STUDYROOM));

        long activeCount = studyroom.getMemberStudyroomList().stream()
                .filter(memberStudyroom -> memberStudyroom.getStatus().equals(BaseStatus.ACTIVE))
                .count();

        if (activeCount > 5) {
            throw new StudyroomException(OVER_MEMBER_STUDYROOM);
        }

        MemberStudyroom memberStudyroom = new MemberStudyroom(
                null,
                BaseStatus.ACTIVE,
                request.getMemberRole(),
                member,
                studyroom);
        memberstudyroomRepository.save(memberStudyroom);
        log.info("Success save memberStudyroom");
    }
}
```

코드를 보니 비슷한 패턴이 반복되고 있었다.( joinStudyroom를 만들어서 가입행위를 재사용하게 했지만 비효율은 개선되지 않고있었다.. )  
가입 로직인데, 방장으로 가입하는 경우와 초대 코드로 가입하는 경우... 🤔  
또 가입로직인데 여러 검증을 하나의 메서드에서 진행하고있었다.


### 💡 개선 포인트 찾기

1. **중복되는 로직이 있나?**
   - 멤버 조회
   - 스터디룸 조회
   - 인원 수 체크
   - MemberStudyroom 저장

2. **각 메서드의 고유한 책임은 뭘까?**
   - joinStudyroomAsCaptain: 방장 권한으로 가입
   - joinStudyroomByCode: 초대 코드 확인 후 일반 멤버로 가입

### 🛠 리팩토링 시작!

초음에 가입이라는 기능과 검증이 섞여있어서 공통된 가입관련 기능을 뽑아내기란 어려웠다.
때문에 먼저 가입 메서드의 고유 책임에서 벗어나는 기능을 분리했다!  

```java
    /**
     * 스터디룸 기존 가입여부 검증
     * */
    private void validateAlreadyJoined(long memberId, long studyroomId){
        if (memberstudyroomRepository.findByMemberIdAndStudyroomIdStatus(memberId,studyroomId,BaseStatus.ACTIVE).isPresent()){
            throw new MemberException(JOINED_STUDYROOM);
        }
    }

    /**
     * 스터디룸 인원 파악
     * */
    private void validateHeadCount(Studyroom studyroom){
        long activeCount = studyroom.getMemberStudyroomList().stream()
                .filter(memberStudyroom -> memberStudyroom.getStatus().equals(BaseStatus.ACTIVE))
                .count();

        if (activeCount > 5) {
            throw new StudyroomException(OVER_MEMBER_STUDYROOM);
        }
    }
```

분리하고나니 스터디룸가입정보 저장이라는 공통 작업이 보이기 시작했다.  

```java
    /**가입 공통 로직*/
    public void joinStudyroom(long studyroomId, long memberId, MemberRole role) {
        log.info("[StudyroomService.joinStudyroom]");
        Member member = memberRepository.findById(memberId)
                .orElseThrow(() -> new MemberException(NOT_FOUND_MEMBER));
        synchronized (this) {
            Studyroom studyroom = studyroomRepository.findById(studyroomId)
                    .orElseThrow(() -> new StudyroomException(NOT_FOUND_STUDYROOM));
            validateHeadCount(studyroom);
            MemberStudyroom memberStudyroom = new MemberStudyroom(
                    null,
                    BaseStatus.ACTIVE,
                    role,
                    member,
                    studyroom);
            memberstudyroomRepository.save(memberStudyroom);
        }
    }
```

그리고 각 메서드는 자신만의 책임에 집중하도록하였다.

```java
    /** 방장으로 가입 */
    @Transactional
    public void joinStudyroomAsCaptain(long studyroomId, long memberId){
        joinStudyroom(studyroomId, memberId, MemberRole.CAPTAIN);
        log.info("Success Join Studyroom as Captain");
    }

    /** 초대 코드로 가입 */
    @Transactional
    public JoinStudyroomByCodeResponse joinStudyroomByCode(JoinStudyroomByCodeRequest request, long memberId){
        log.info("[StudyroomService.joinStudyroomByCode]");
        Long studyroomId = inviteService.findRoomIdByInviteCode(request.getInviteCode())
                .orElseThrow(()->new StudyroomException(INVALID_INVITE_CODE));
        Studyroom studyroom = studyroomRepository
                .findById(studyroomId).orElseThrow(()->new StudyroomException(INVALID_INVITE_CODE));

        validateAlreadyJoined(memberId, studyroomId);
        joinStudyroom(studyroomId,memberId,MemberRole.CREW);

        return JoinStudyroomByCodeResponse.from(studyroom);
    }
```


### 🎯 개선된 점

1. **코드 중복 제거**
   - 핵심 가입 로직이 한 곳으로 모음
   - 각 메서드는 자신만의 고유한 책임을 가짐
   - 불필요하게 `JoinStudyroomRequest`만들지 않아도됨

2. **유지보수성 향상**
   - 가입 로직 수정이 필요하면 한 곳만 수정
   - 새로운 가입 유형 추가 용이

3. **안정성 향상**
   - 검증 로직들이 깔끔하게 분리


## 마무리
리팩토링은 단순히 코드를 "깔끔하게 정리"하는 것이 아니라, 더 나은 설계와 유지보수성을 확보하기 위한 과정임을 다시금 느낄 수 있었다.  
이번 작업을 통해 중복 제거, 책임 분리, 확장 가능성이라는 세 가지 키워드를 기반으로 코드를 개선했다.  

이제 스터디룸 가입 로직은 더 단순하고 명확해졌고 새로운 요구사항이나 기능 추가에도 유연하게 대응할 수 있는 상태가 되었다.  
앞으로도 이런 과정을 반복하며 더 나은 코드를 작성해나가 개발자가 되고자 한다!
