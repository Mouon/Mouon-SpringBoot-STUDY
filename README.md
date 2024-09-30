# 🍃 스프링 부트 공부기록 🍃


## 글 목록
-----------------------------
### 🍃스프링 공부 - 원리에 가까운

- [🚨JPA SELECT 개선해보기🚨](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/QueryImprove/QueryImprove.md)
- [⭐️지연로딩과 프록시 그리고 리플렉션⭐️](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/LazyloadingNProXy.md)
- [Querydsl 맛보기](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/queryDSL/qdslBasic.md)
- [영속성 컨텍스트에 대한](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/PersistentContext.md)
- [⭐️@Async 기본⭐️](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/AsyncBacic.md)
- [⭐️@Async로 Non/Blocking, Synchronous/Asynchronous 짧게 알아보기⭐️](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/AsyncBasic2.md)
- [흔한 어노테이션-1탄](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/Anotation_1.md)
- [DTO사용하는 이유](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/whyDto.md)
- [세터사용 주의](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/WhyNoSetter.md)
- [⚡️예외로 스프링 작동 흐름 살펴보기](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/ewptionStudy/exceptionStudy.md)
- [🚀@Builder로 안정적인 개발하기🚀](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/builderPattern/BuilderStuDy.md)
- [정적팩토리메소드와 빌드패턴](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/builderPattern/StaticFactory.md)
- [💆🏻‍♂️💆🏻‍♂️더티체킹에대한 고민](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/DirtyChecking.md)  
- [🔥🔥Querydsl로 페이징 처리🔥🔥<Querydsl의 강력함>](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/queryDSL/DslPractice.md)
- [WebClient 맛보기](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/webClient.md)  
- [😲 Querydsl 오픈소스 컨트리뷰션이 되다! 😲](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/queryDSL/firstContridutor.md)
- [🚀 자료실 조회 성능에 인덱싱 ](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/index/Index_Query_1.md)
  
---------------------------
### 🍃스프링 공부 - 논리에 가까운

- [🗂️파일검증기 만들기 논리](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/fileValidaterStudy/MakeValidater.md)
- [파일검증기 예외처리 개선](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/fileValidaterStudy/ImproveFilevalidater.md)  
- [API 설계에 대한 고민 1탄](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/WhatRestfulAPI.md)

---------------------------
![image](https://github.com/user-attachments/assets/e53610ea-83b6-4e11-99a0-0843fcaf5a57)

# JPA 위주의 공부하다가 서블릿까지..


스프링부트를 이용해서 여러 프로젝트를 진행하던 중 문득 스프링을 처음배울때 크게 중요하게 공부하지않아 잊고있던 

DispatcherServlet의 작동 메커니즘이 궁금해졌다.

 

정확하게 말하자면 스프링부트가 어떻게 HTTP요청을 주고받는지 궁금해졌다.

 

스프링부트는 기본적으로 DispatcherServlet을 통해 HTTP 요청을 받아들이고 처리 과정을 조정한다. 

DispatcherServlet은 웹 애플리케이션의 중앙 진입점으로, 모든 요청을 받아 핸들러로 전달하고, 결과를 다시 클라이언트에게 응답한다.


대략적인 흐름이 이러하다.


 

- HTTP 요청 수신: 클라이언트가 HTTP 요청을 보내면 이 요청은 서블릿 컨테이너(예: Tomcat)로 들어온다. 서블릿 컨테이너는 이 요청을 DispatcherServlet으로 전달한다.
- 핸들러 매핑: DispatcherServlet은 요청을 처리할 핸들러(컨트롤러 메서드)를 찾기 위해 HandlerMapping에게 요청을 전달한다. HandlerMapping은 요청 URL, HTTP 메서드, 기타 매핑 정보를 기반으로 적절한 핸들러를 찾는다.
- 핸들러 어댑터: DispatcherServlet은 찾은 핸들러를 실행하기 위해 적절한 HandlerAdapter를 사용한다. HandlerAdapter는 실제로 핸들러를 호출하고, 결과를 DispatcherServlet에 반환한다.
- 핸들러 실행: HandlerAdapter는 컨트롤러의 메서드를 호출하여 요청을 처리하고, 처리 결과를 반환한다.
 
## 여기서 말하는 DispatcherServlet은 무엇일까?
 

 

### DispatcherServlet은 Spring MVC에서 프론트 컨트롤러로 작동하는 서블릿이다.
![image](https://github.com/user-attachments/assets/fe81e2da-0bfd-4f66-9a17-e2d209fdddbd)

여기서 서블릿이란 자바 기반의 웹 요청 및 응답을 처리하는 서버 측 프로그램이다. 즉 서블릿(Servlet)은 자바 기반의 웹 애플리케이션에서 클라이언트의 요청을 처리하고, 서버 측에서 동적 콘텐츠를 생성하여 응답을 반환하는 자바 프로그램이다. 

![image](https://github.com/user-attachments/assets/2546f50a-68e6-4fc6-a0e6-97022c8dad43)

 

다시 DispatcherServlet로 넘어와보면 DispatcherServlet은 Spring MVC에서 프론트 컨트롤러로 작동하는 서블릿으로  
모든 HTTP 요청을 받아서 처리 과정을 조정하는 역할을 한다. 이때 DispatcherServlet을 도와주는 다양한 구성요소가 있는데  



그것들이 바로 HandlerMapping, HandlerAdapte 등이다.


 

스프링부트기반의 서버 프로그램에 클라이언트가 HTTP 요청을 보내면 DispatcherServlet은 클라이언트의 모든 HTTP 요청을 수신한다. 그후 그 요청을 처리할 핸들러 즉 컨트롤러 메서드를 찾는데, 이때 이용하는 것이 바로 HandlerMapping이다.  
그 후 HandlerAdapter를 사용하여 적절한 핸들러를 호출하고 요청을 처리한다. 그 후 이제 클라이언트에게 요청을 반환해야하는데 이때 HttpMessageConverter 인터페이스의 구현체를 사용하여 객체를 JSON으로 변환한다. 기본적으로 Jackson 라이브러리를 이용한다.


 

 

 

 

### DispatcherServlet 구성요소 각각을 좀 더 깊게 알아보자 !

 
 
 
## HandlerMapping
먼저 HandlerMapping을 알아보자.

HandlerMapping은 요청 URL과 관련된 핸들러(컨트롤러 메서드)를 매핑하는 역할을 하는 인터페이스이다.

즉 요청이 들어오면 URL, HTTP 메서드, 파라미터 등을 기반으로 적절한 핸들러를 찾아 반환하는 역할을 한다고 볼 수 있다. 

 

우리가 스프링프로젝트를 진행할때 아래같이 @RequestMapping 애노테이션을통해 기본 URL주소를 구현했는데 이때 애노테이션 내부의 주소를 이용하는것이 HandlerMapping인 것이다. 

 

Spring MVC에서 @RequestMapping 애노테이션 등을 사용하여 매핑 정보를 설정하는데, 그 정보를 기반으로 @RequestMapping 애노테이션이 붙은 클래스에 속하는 메서드 중 HandlerMapping이 적절한 핸들러를 반환한다.

 

여기서 @RestController 애노테이션은 클래스의 메서드를 HTTP 요청을 처리할 핸들러로 인식하게 해준다.

```java

@RestController
@RequestMapping("/user")
public class MemberController {


}
 
```
정확하게 말하면 HandlerMapping는  기본 인터페이스이고 RequestMappingHandlerMapping라는 구현체가 @RequestMapping 애노테이션을 처리하는 기본 구현체이다.

 

 

 

- HandlerMapping 인터페이스: 요청 URL과 관련된 핸들러(컨트롤러 메서드)를 찾는 역할
- RequestMappingHandlerMapping 클래스: HandlerMapping 인터페이스를 구현한 클래스로, @RequestMapping, @GetMapping, @PostMapping 등의 애노테이션을 처리하여 적절한 핸들러를 찾는 역할.
 

 
 
 

 

## HandlerAdapter
그 다음은 HandlerAdapter이다.

HandlerAdapter는 DispatcherServlet이 요청을 실제 핸들러(컨트롤러 메서드)로 위임하여 처리할 수 있도록 도와주는 어댑터이다. HandlerMapping이 핸들러를 찾아 줬다면 HandlerAdapter는 요청을 찾은 핸들러에게 위임하는 역할을 한다. 그리고 핸들러의 결과를 DispatcherServlet에 반환하는 역할도 한다.

 

구체적으로, HandlerAdapter는 handle 메서드를 통해 핸들러를 호출하고 결과를 반환한다.

Spring MVC에서 자주 사용되는 구현체는 RequestMappingHandlerAdapter로,

이 어댑터는 @RequestMapping을 처리하는 핸들러 메서드를 호출한다.

 

 

이제 각 요소들의 실제 스프링에서 작동을 보자 

아래는 단순히 맴버가 속한 스터디룸의 리스트를 반환하는 메서드이다.  
```java
@RestController
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/studyroom")
public class StudyroomController {

    StudyroomService studyroomService;
    MemberStudyroomService memberStudyroomService;
    JwtProvider jwtProvider;

    @GetMapping("")
    public BaseResponse<MemberStudyroomListResponse> getStudyroomList(@RequestHeader("Authorization") String authorization){
        log.info("[StudyroomController.getStudyroomList]");
        Long memberId = jwtProvider.extractIdFromHeader(authorization);
        return new BaseResponse<>(memberStudyroomService.getStudyroomList(memberId));
    }

}
 
```
- 클라이언트가 /studyroom 경로로 GET 요청을 보냄.
- DispatcherServlet이 요청을 받아 HandlerMapping을 통해 getStudyroomList 메서드를 찾음.
- HandlerAdapter가 메서드를 실행하고, BaseResponse 객체를 반환.
- 반환된 객체는 Jackson 라이브러리에 의해 JSON으로 변환되어 HTTP 응답으로 클라이언트에게 전송.
 

### Jackson 라이브러리?
![image](https://github.com/user-attachments/assets/f0c8576d-69e5-430f-9641-62e318bde90c)

 

아! 여기서 또 생소한 개념이 등장했다. 바로 Jackson 라이브러리이다. 

Jackson은 Java 객체를 JSON으로 직렬화하거나 JSON을 Java 객체로 역직렬화하는 데 사용되는 라이브러리이다.

Spring MVC에서 주로 @RestController나 @ResponseBody 애노테이션과 함께 사용된다.
여기서 직렬화, 역직렬화의 개념을 한번 짚고 넘어가 보자!

- 직렬화: Java 객체를 JSON 문자열로 변환
- 역직렬화: JSON 문자열을 Java 객체로 변환
 

 

 

정리
좀 정신없는 글 같아서 마지막으로 정리하자면 ..

 

- 클라이언트의 HTTP 요청이 DispatcherServlet에 전달되기 전, 서블릿 컨테이너(Tomcat 등)에서의 처리 과정을 거침.

- 서블릿 컨테이너는 요청을 필터와 리스너를 통해 처리한 후 DispatcherServlet으로 전달

- DispatcherServlet은 웹 애플리케이션의 중앙 진입점

- HandlerMapping에게 요청에게 요청을전달

- HandlerMapping은 적절한 핸들러를 찾아 DispatcherServlet에게반환

- HandlerAdapter는 HandlerMapping찾은 핸들러메서드에게 요청에대한 처리를 위임

- 핸들러 메서드가 요청을처리하고 생성한 응답을 HandlerAdapter가 DispatcherServlet에게반환

- 생성한 응답은 주로 자바 객체일텐데 이를 HttpMessageConverter 인터페이스의 구현체가 Jackson 라이브러리를이용하여 JSON으로 변환되어 HTTP 응답으로 클라이언트에게 전송.

이렇게 이해하면 될 것 같다!!

2탄 -> [⚡️예외로 스프링 작동 흐름 살펴보기](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/ewptionStudy/exceptionStudy.md)
                      
--------------------


