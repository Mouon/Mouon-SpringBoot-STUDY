# Querydsl 로 페이징 처리기능 타입 세이프하게 구현하기

프로잭트를 진행하던중 깃허브 이슈리스트 조회 기능을 구현하게되었다.  
그런데 이슈 리스트라는 기능 특성상 무한 스크롤로 구현해야하는 소요가 생겼고, 그에따라 서버사이드에서 페이징 처리를 해줘야했다.  
  
일반적으로 페이징처리라 하면 `OffSet`방식을 많이 떠올릴 것이다.  
하지만 이 방식은 이전의 데이터를 모두 조회하고 그 ResultSet에서 오프셋을 조건으로 잘라내는 방식이라 
데이터가 터지면 성능저하가 발생하고, 실시간 처리에서 데이터 누락 등의 문제를 발생시킨다.  
때문에 나는 `Cursor` 방식의 페이징을 구현하기로 했다.  

`OffSet` 방식을 구현하는 것은 Page 인터페이스 등을 사용하면 되기에 간단하지만
`Cursor` 방식의 페이징은 특정 조건 이후에 있는 데이터들만 선별해서 가져오는 방식이기떄문에
직접 구현하기로 하였다.  
  
여기서 이번 기록의 주인공이 등장한다!  
바로 Querydsl이다. Querydsl은 복잡한 조건으로 쿼리를 다룰때 타입세이프한 환경에서 개발이 가능하게 해준다.  
따라서 코드가 직관적이고, `JPQL`과 다르게 컴파일시에 오류들을 잡아주기에 `Cursor` 방식의 페이징과 같은 복잡한 조건의 쿼리를 작성할때
아주 간단하게 작성이 가능하게 해준다.  

우선 아래는 작성한 페이징 처리 코드이다.  
```java
@Repository
@RequiredArgsConstructor
public class GithubIssueDSL implements GithubIssueRepositoryCustom{
    private final JPAQueryFactory queryFactory;
    QGithubIssue githubIssue = QGithubIssue.githubIssue;
    @Override
    public List<GithubIssueListResponse.GithubIssues> getIssueListByStudyroom(Long studyroomId, BaseStatus status, Long lastgithubIssueId, int limit) {
        BooleanBuilder builder = new BooleanBuilder();
        builder.and(githubIssue.studyroom.studyroomId.eq(studyroomId))
                .and(githubIssue.status.eq(status));

        if(lastgithubIssueId!=null){
            builder.and(githubIssue.githubIssueId.lt(lastgithubIssueId));
        }

        return queryFactory.select(Projections.fields(GithubIssueListResponse.GithubIssues.class,
                        githubIssue.githubIssueId,
                        githubIssue.title,
                        githubIssue.body,
                        githubIssue.url,
                        githubIssue.state))
                .from(githubIssue)
                .where(builder)
                .orderBy(githubIssue.githubIssueId.desc())
                .limit(limit)
                .fetch();
    }
}
```

이번 구현에서 `Querydsl`의 장점이 꽤(?)드러나는 것 같다.  
우선 코드를 간단히 설명하겠다.
#### `Querydsl`은 역시 빌더패턴과 함께 사용된다.  
```java
BooleanBuilder builder = new BooleanBuilder();
builder.and(githubIssue.studyroom.studyroomId.eq(studyroomId))
.and(githubIssue.status.eq(status));
```
위 코드를 통해 where절에 들어갈 조건들을 추가한다.  
여기서 첫번째 `Querydsl`의 장점이 드러난다.  
이렇게 빌더패턴으로 where절이 작성되기때문에 조건의 추가 혹은 삭제가 
동적으로 작성될 수 있으며, 동적이 아니더라고 추가 또는 수정이 간단하고 안정적으로 이루어질 수 있다.
  

#### 다음은 커서페이징의 핵심인 조건문이다.
```java
        if(lastgithubIssueId!=null){
            builder.and(githubIssue.githubIssueId.lt(lastgithubIssueId));
        }
```
이 조건문은 파라미터로 이전 페이지의 마지막 요소의 아이디를 받아 그 이후의 값만 조회하도록하는 역할 을 한다.  
이 역시 앞서 언급한 `Querydsl`의 장점이 담겨있다.   
첫번째 페이지는 lastgithubIssueId가 null이기 때문에 이때는 그 이후의 값만 조회하도록하는 조건을 추가할 수 없는데  
`Querydsl`은 기본적으로 빌더패턴을 통해 where절 내부의 조건을 관리하므로 조건에때라 위에처럼 
조건을 동적으로 추가할 수 있다.  

이 부분에서는 동적으로 where적을 관리하는 장점외에도 다른 장점이 드러나는데 
그것은 바로 `githubIssue.githubIssueId.lt(lastgithubIssueId)` 이 부분과 같이
쿼리문을 직관적으로 코드 수준에서 작성할 수 있다는 점이다.  
JPQL이라면 쿼리가 String으로 작성되기때문에 
개발 과정에서 이름불일치 혹은 공백 불일치 등의
오류를 잡기 어렵다는 문제가 있었는데,
`Querydsl`은 쿼리를 엔티티 객체를 다루는 방식으로 작성하고 특수조건을 제공하는 메서드가 있기때문에  
안정적인 개발이 가능하게 해준다.  


#### 다음 부분은 쿼리를 조립하는 부분이다.
```java
        return queryFactory.select(Projections.fields(GithubIssueListResponse.GithubIssues.class,
                        githubIssue.githubIssueId,
                        githubIssue.title,
                        githubIssue.body,
                        githubIssue.url,
                        githubIssue.state))
                .from(githubIssue)
                .where(builder)
                .orderBy(githubIssue.githubIssueId.desc())
                .limit(limit)
                .fetch();
```
`Querydsl`은 최종적으로 `queryFactory`를 통해 쿼리를 생성한다.  
위 코드에서 마지막 `Querydsl`의 장점이 보이는데!!
그것은 바로 다시한번 안정적인 쿼리의 작성이다.  
`JPQL`의 단점을 모두 커버하고있다.  
`where`,`from`,`orderBy`등을 모두 메서드로 제공하고 있기때문에 쿼리를 작성할때 너무나도 편안하다.  
개발 과정에서 오류를 잡을 수 있다는 아주 강력한 장점이 드러나는 부분이다.  

그 다음은 `DTO`와의 매핑이다.  
엔티티가 아닌 DTO를 직접조회하여 조회 성능을 개선할 수 있는데, 이때 `JPQL`보다
`Querydsl`이 좀더 유리하다. 물론 `JPQL`도 클래스기반 DTO매핑 등을 너무나도 잘 지원하고 있지만  
그 역시 String으로 작성해야한다는 부담이 있다.  
하지만 `Querydsl`은 `Projections.fields`라는 것을 통해 안정적으로 `DTO`와의 매핑을 구현할 수 있게 해준다.  


## 마무리  

이외에도 `Querydsl`은 많겠지만 이번 글에서는 내가 느낀 `Querydsl`의 장점에 대해 정리해보았다.  
어떤 기술이던 장단점은 존재한다.  
이번 글에서 `Querydsl`을 너무 찬양하는 듯 했지만 `Querydsl`도 분명 단점이 존재한다.  
대표적인 단점으로는 글을 보면서 느꼈을 수도 있지만 기본적으로 빌더패턴을 사용하기 때문에 코드가 길어진다는 단점이 존재한다.  
따라서 기본적인 `CRUD`와 같은 간단한 기능은 `DataJPA`가 유리하다.  
또한 복잡하지않은 기능도 `DataJPA`를 통해 작성하는 것이 더 적절하다고 볼 수 있다.  

언제나 그렇듯 여러 기술들을 적재 적소에 도입하는 것이 현명한 개발이라고 생각한다!



