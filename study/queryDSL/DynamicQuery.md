# QueryDSL을 활용한 동적 쿼리 적용하기

### 1. 왜 동적 쿼리를 사용하게 되었는가?

이전에 QueryDSL을 사용한 경험이 있었지만 비즈니스 요구 사항으로 인해 적극적으로 동적 쿼리를 활용하지는 못했다.  
하지만 새로운 토이 프로젝트를 진행하면서 채용 공고 검색 기능을 구현하게 되었고 다양한 조건을 조합하여 검색해야 하는 요구 사항이 발생했다.  
이에 따라 QueryDSL을 활용한 동적 쿼리를 적용해 보기로 했다.

---

### 2. 동적 쿼리란?

동적 쿼리(Dynamic Query)는 실행 시점에 조건이 결정되어 쿼리가 동적으로 구성되는 방식이다.  
정적 쿼리는 미리 작성된 SQL을 그대로 실행하지만 동적 쿼리는 사용자의 입력이나 특정 상황에 따라 쿼리의 조건이 달라진다.

동적 쿼리는 다음과 같은 상황에서 유용하다.
- 검색 필터가 다양하고 선택적으로 적용되는 경우
- 특정 조건에 따라 WHERE 절이 추가되거나 제외되어야 하는 경우
- 커서 기반 페이징을 사용할 때 이전 데이터 기준으로 조회해야 하는 경우

기존의 JPQL에서는 `if-else` 문을 활용하여 문자열을 조합하는 방식으로 동적 쿼리를 만들었지만 이는 코드 가독성을 떨어뜨리고 유지보수성을 저하시킨다.  
이를 해결하기 위해 **QueryDSL**을 활용하면 보다 직관적으로 동적 쿼리를 작성할 수 있다.

---

### 3. QueryDSL을 활용한 동적 쿼리 구현

#### 3.1 코드 적용

아래는 QueryDSL을 활용하여 채용 공고 검색을 동적으로 처리하는 코드이다.

#### `JobPostingRepository`
```java
import com.ilil.alba.domain.QJobPosting;
import com.ilil.alba.domain.base.BaseStatus;
import com.ilil.alba.dto.JobPostingSearchRequest;
import com.ilil.alba.dto.JobPostingSearchResponse;
import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.types.Projections;
import com.querydsl.jpa.impl.JPAQueryFactory;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
@RequiredArgsConstructor
public class JobPostingRepository implements JobPostingDslRepository {
    private final JPAQueryFactory queryFactory;
    QJobPosting jobPosting = QJobPosting.jobPosting;

    public List<JobPostingSearchResponse.SearchResults> searchJobPostings(JobPostingSearchRequest request, Long lastId, int limit) {
        BooleanBuilder builder = new BooleanBuilder();

        // 기본적으로 활성화된 공고만 검색
        builder.and(jobPosting.status.eq(BaseStatus.ACTIVE));

        // 커서 기반 페이징을 위한 조건
        if (lastId != null) {
            builder.and(jobPosting.jobPostingId.lt(lastId));
        }

        // 검색 필터 적용
        if (request.getTitle() != null) {
            builder.and(jobPosting.title.containsIgnoreCase(request.getTitle()));
        }

        if (request.getLocation() != null) {
            builder.and(jobPosting.location.containsIgnoreCase(request.getLocation()));
        }

        if (request.getWorkDate() != null) {
            builder.and(jobPosting.workDate.eq(request.getWorkDate()));
        }

        if (request.isOneDayJob()) {
            builder.and(jobPosting.isOneDayJob.isTrue());
        }

        // 프로젝션을 활용하여 필요한 데이터만 조회
        var projections = Projections.fields(JobPostingSearchResponse.SearchResults.class,
                jobPosting.jobPostingId,
                jobPosting.title,
                jobPosting.location,
                jobPosting.workDate,
                jobPosting.dailyWage,
                jobPosting.paymentDate,
                jobPosting.isOneDayJob);

        return queryFactory
                .from(jobPosting)
                .select(projections)
                .where(builder)
                .orderBy(jobPosting.jobPostingId.desc())
                .limit(limit)
                .fetch();
    }
}
```

#### 3.2 QueryDSL을 활용한 동적 쿼리의 장점

1. **BooleanBuilder 활용**: 다양한 조건을 간단하게 추가 가능
   ```java
   BooleanBuilder builder = new BooleanBuilder();
   if (request.getTitle() != null) {
       builder.and(jobPosting.title.containsIgnoreCase(request.getTitle()));
   }
   ```
   위 처럼 빌더패턴을 통해 where절을 동적으로 가독성 좋게 조합할 수 있다.  

2. **빌더 패턴 사용**: 직관적인 코드 작성 및 가독성 향상
   ```java
   return queryFactory.from(jobPosting)
           .where(builder)
           .orderBy(jobPosting.jobPostingId.desc())
           .limit(limit)
           .fetch();
   ```
   
  
4. **QueryDSL의 Projections 기능 활용**: DTO로 직접 매핑 가능
   ```java
   var projections = Projections.fields(JobPostingSearchResponse.SearchResults.class,
           jobPosting.jobPostingId,
           jobPosting.title,
           jobPosting.location,
           jobPosting.workDate,
           jobPosting.dailyWage,
           jobPosting.paymentDate,
           jobPosting.isOneDayJob);
   ```
   DTO를 직접 조회하여 필요한 데이터만 가져옴으로써 성능 최적화 및 코드 유지보수가 용이하다.

5. **JPQL보다 간결하고 안전한 코드 작성 가능**
   - JPQL처럼 문자열 조합이 필요 없고 컴파일 단계에서 오류를 방지할 수 있음. (jpql로 동적쿼리를 하기 어려운 이유중 하나가 문자열이기 떄문인데 이 단점을 극복해준다.)  
   - `BooleanBuilder`를 활용하면 조건을 쉽게 추가 및 제거 가능.

---

### 4. 실제 실행 로그 분석

코드를 실행하면 동적 쿼리가 어떻게 생성되는지 확인할 수 있다.

#### 4.1 필터링 조건 없이 전체 조회

<img width="496" alt="스크린샷 2025-02-26 오후 7 06 06" src="https://github.com/user-attachments/assets/2f40c14e-b790-48bd-8658-b026292cf650" />  
<img width="1012" alt="스크린샷 2025-02-26 오후 7 09 01" src="https://github.com/user-attachments/assets/161a43cd-e71b-466e-99c8-dfcf31c960c4" />  

#### 4.2 특정 필터링 조건 적용 (장소, 제목)
<img width="496" alt="스크린샷 2025-02-26 오후 7 06 42" src="https://github.com/user-attachments/assets/92779aa7-8773-4686-9802-85bec356e0c3" />  
<img width="1012" alt="스크린샷 2025-02-26 오후 7 09 17" src="https://github.com/user-attachments/assets/73444ef6-ccf6-428a-bb16-8d7257b40e63" />  

#### 4.3 커서 기반 페이징 적용
<img width="496" alt="스크린샷 2025-02-26 오후 7 07 13" src="https://github.com/user-attachments/assets/bbad6b41-ab7a-4b02-873d-530ddb9685eb" />
<img width="1012" alt="스크린샷 2025-02-26 오후 7 09 45" src="https://github.com/user-attachments/assets/8c5045d8-f23f-436b-beba-bbd8f16e8e89" />  

---

### 5. 마무리

QueryDSL을 활용하면 동적 쿼리를 보다 쉽게 작성할 수 있고 `BooleanBuilder`와 `Projections` 기능을 이용하면 코드의 가독성과 유지보수성 그리고 효율성을 더욱 높일 수 있다. 
추상적으로 좋아지는게 아니라 빌더패턴 기반이라 정말 편리하다.


앞으로 QueryDSL을 적극 활용하여 복잡한 검색 기능을 효율적으로 구현해 보자!

