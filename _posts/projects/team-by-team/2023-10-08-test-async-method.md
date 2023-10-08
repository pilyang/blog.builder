---
title: JAVA Spring @Async 비동기 테스트하기
date: 2023-10-08 17:05:00 +0900
categories: [Projects, team-by-team]
tags: [JAVA, Spring, Async, test, 자바, 비동기, 테스트, 우테코, 우아한테크코스, 팀바팀, team-by-team]
---

팀바팀 서비스에는 일정 관리 기능이 있고, 우리 서비스를 이용해 일정을 등록한 사용자들이 팀바팀을 통해 등록한 일정을 사용자들의 개인 캘린더에서도 확인할 수 있게 할 필요성을 느끼게 되었다.
그래서 직접 일정전송 표준인 icalendar (.ics)형식으로 일정 파일을 배포하게 되었다.
해당 기능은 우리의 주요 비지니스와 분리되어 실행되어야 하기에 트랜젝션과 스레드를 모두 분리하여 실행하게 되었고,
이를 위해서 spring의 `@Async` 어노테이션을 달아 메서드를 구현하게 되었다.

해당 선택의 이유와 구현 방법들은 추후 팀바팀 icalendar 배포기 포스팅을 통하여 공유하도록 하겠다.

본 글에서는 기능구현을 한 이후 비동기로 (다른 스레드에서) 처리되는 메서드의 테스트 방법에 대해서 공유한다.  
결론부터 말하면 이벤트리스너의 테스트를 위해서는 SycnTaskExcutor를 이용하여 테스트가 동기적으로 처리될 수 있도록 하였고, 인수테스트에서는 AOP를 사용하여 `AsyncAspect`를 만들어 `@Asycn`메서드가 끝나는것을 기다려주는 방식으로 테스트를 진행했다.


## 이벤트 리스너 테스트 : `SycnTaskExecutor` 사용


테스트 코드를 보기 전 icalendar 생성 및 배포를 위해 구현된 EventListner 클래스의 일부는 아래와 같다.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class IcalendarEventListener {

    private final IcalendarPublishService icalendarPublishService;

    ...

    @Async
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void createIcalendar(final CreateIcalendarEvent createIcalendarEvent) {
        final Long teamPlaceId = createIcalendarEvent.teamPlaceId();

        icalendarPublishService.createAndPublishIcalendar(teamPlaceId);
        log.info("ics파일 생성 - teamPlaceId : {}", teamPlaceId);
    }

    @Async
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void updateIcalendar(final ScheduleEvent scheduleEvent) {
        final Long teamPlaceId = scheduleEvent.getTeamPlaceId();

        icalendarPublishService.updateIcalendar(teamPlaceId);
        log.info("ics파일 업데이트 - teamPlaceId : {}", teamPlaceId);
    }
}

```
{: file="IcalendarEventListener.java"}

(자세한 구현에 대한 내용은 앞서 엎급한데로 추후 추가적인 포스팅을 통해서 공우하겠다.)

우선 실패하는 테스트 코드의 일부를 살펴보면 아래와 같다.

```java
class IcalendarEventListenerTest extends ServiceTest {

    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    @MockBean
    private IcalendarPublishService icalendarPublishService;

    ...

    @Test
    @DisplayName("Ical생성에 성공한다.")
    void successCallCreatingIcalendar() {
        // given
        final TeamPlace ENGLISH_TEAM_PLACE = testFixtureBuilder.buildTeamPlace(ENGLISH_TEAM_PLACE());

        final CreateIcalendarEvent createIcalendarEvent = new CreateIcalendarEvent(ENGLISH_TEAM_PLACE.getId());

        // when
        applicationEventPublisher.publishEvent(createIcalendarEvent);

        TestTransaction.flagForCommit();
        TestTransaction.end();

        // then
        verify(icalendarPublishService, times(1)).createAndPublishIcalendar(ENGLISH_TEAM_PLACE.getId());
    }

    ...
}

```
{: file="IcalendarEventListenerTest.java"}


위의 테스트코드를 간단히 설명하면, EventListner의 테스트를 위해 Listhner가 들을 `CreateIcalendarEvent`를 발행해주었다.
이후 해당 이벤트리스너가 `TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`이므로 트렌젝션을 커밋하여줄 수 있도록
```java
TestTransaction.flagForCommit();
TestTransaction.end();
```
{: .nolineno}
위와같이 테스트트렌젝션을 커밋을 해주었다.


만약에 해당 이벤트 발행으로 인한 다른 리스너의 영향이 걱정된다면, `ApplicationEventPublisher`를 통한 이벤트 발행이 아닌, `IcalendarEventListener`를 이용해서 직접 메서드를 실행시켜 줄 수 있다.
이경우에는 위의 `TestTransaction`을 통한 트랜젝션 커밋처리가 없어도 잘 실행되게 된다.

> 사실 위와 똑같은 코드를 작성하여 테스트를 하면 일부 테스트가 그냥 성공하는 경우가 생겼다.
예상하기로는 해당 메서드 내부 코드가 워낙 단순하고 짧아서 검증단계를 거칠때 이미 메서드가 잘 수행이 되었기 때문에 성공하는것으로 생각된다.  
임의로 리스너 메서드에 `Thread.sleep(100);`과 같이 짧은시간의 딜레이만 주어도 테스트가 실패함을 확인할 수 있을 것이다.
{: .prompt-info}


위의 테스트가 실패하는 이유는 이벤트리스너가 `@Asycn`로 인하여 비동기로 실행이 되게 되고,
테스트코드의 검증을 하는 
```java
verify(icalendarPublishService, times(1)).createAndPublishIcalendar(ENGLISH_TEAM_PLACE.getId());
```
{: .nolineno}
시점에 아직 메서드가 수행되지 않아서 테스트가 실패하게 된다.

이를 해결하기 위해서는 테스트코드가 해당 이벤트리스너 메서드가 종료되기까지 기다리게 하거나, 그냥 테스트메서드를 동기로 실행하게 해야한다.
이 IcalendarEventListenerTest는 여러 메서드들의 복합적인 테스트가 아닌 이벤트 리스너 메서드의 단위테스트 이기 때문에, 그냥 해당 메서드를 비동기적으로 실행할 수 있게 만들어주는것이 좋다고 생각하였다.

아래와 같은 `@TestConfiguration`을 추가하면 이제 메서드가 동기적으로 잘 실행되고 모든 테스트가 정장적으로 통과됨을 확인할 수 있다.


### 동기 실행 테스트 코드

```java
class IcalendarEventListenerTest extends ServiceTest {

    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    @MockBean
    private IcalendarPublishService icalendarPublishService;

    @TestConfiguration
    static class TestConfig {
        @Bean
        @Primary
        public Executor executor() {
            return new SyncTaskExecutor();
        }
    }

    ...

    @Test
    @DisplayName("Ical생성에 성공한다.")
    void successCallCreatingIcalendar() {
        // given
        final TeamPlace ENGLISH_TEAM_PLACE = testFixtureBuilder.buildTeamPlace(ENGLISH_TEAM_PLACE());

        final CreateIcalendarEvent createIcalendarEvent = new CreateIcalendarEvent(ENGLISH_TEAM_PLACE.getId());

        // when
        applicationEventPublisher.publishEvent(createIcalendarEvent);

        TestTransaction.flagForCommit();
        TestTransaction.end();

        // then
        verify(icalendarPublishService, times(1)).createAndPublishIcalendar(ENGLISH_TEAM_PLACE.getId());
    }

    ...
}

```
{: file="IcalendarEventListenerTest"}

`SyncTaskExecutor`의 구현을 들어가보면 아래와 같이 문서가 작성되어있음을 확인할 수 있다.
>  implementation that executes each task <i>synchronously</i>
in the calling thread.  
Mainly intended for testing scenarios.  
Execution in the calling thread does have the advantage of participating
in its thread context, for example the thread context class loader or the
thread's current transaction association. That said, in many cases,
asynchronous execution will be preferable: choose an asynchronous
TaskExecutor instead for such scenarios.

해당 설명에도 테스트 시나리오를 위하여 각각의 task들을 스레드에서 동기적으로 실행하게 함을 알 수 있다.

위와같은 방식으로 `@Asycn` 메서드들의 단위테스트를 간단하게 할 수 있게 된다.


## 인수테스트 : `AsyncAspect` 구현

단위테스트는 위와같이 동기적인 상황을 만들어서 쉽게 테스트 할 수 있었다.  
하지만 인수테스트는 가능한 실제 환경과 동일한(비슷한) 상황을 만들어 테스트를 하고 싶었고, `SyncTaskExecutor`를 사용하지 않고 테스트를 할 방법에 대해서 고민을 하였다.  

결론적으로 테스트 중간에 `@Async`어노테이션이 달린 메서드가 실행되고 종료되기를 기다리는 `AsycnAspect`를 만들어서 사용하게 되었다.

### 테스트 시나리오

1. 사용자가 아직 ical이 배포되지 않은 팀에 대하여 ical배포 구독 url을 요청한다.
2. 서버는 아직 ical이 배포가 안되어있음으로 404응답을 하고, 배포 이벤트를 발행한다.
3. 일정 시간이 지난 후 사용자가 ical구독 url을 요청하면 생성된 구독 url을 응답한다.


아래는 해당 시나리오에 해당하는 테스트코드이다.


```java
public class IcalendarAcceptanceTest extends AcceptanceTest {

    @MockBean
    private FileCloudUploader fileCloudUploader;

    @Autowired
    private TestConfig.AsyncAspect asyncAspect;

    @TestConfiguration
    static class TestConfig {

        @Aspect
        @Component
        static class AsyncAspect {

            private CountDownLatch countDownLatch;

            public void init() {
                countDownLatch = new CountDownLatch(1);
            }

            @After("execution(* team.teamby.teambyteam.icalendar.application.IcalendarEventListener.createIcalendar(*))")
            public void afterIcalendarCreation() {
                countDownLatch.countDown();
            }

            public void await() throws InterruptedException {
                countDownLatch.await();
            }
        }
    }

    @Nested
    @DisplayName("icalendar 배포 파일 조회시")
    public class IcalendarUrlGetTest {

        private Member member;
        private TeamPlace teamPlace;
        private String authCode;

        @BeforeEach
        void setup() {
            member = testFixtureBuilder.buildMember(MemberFixtures.PHILIP());
            teamPlace = testFixtureBuilder.buildTeamPlace(TeamPlaceFixtures.FLUID_TEAM_PLACE());
            testFixtureBuilder.buildMemberTeamPlace(member, teamPlace);
            authCode = jwtTokenProvider.generateAccessToken(member.getEmailValue());
        }

        ...

        @Test
        @DisplayName("아직 생성되지 않았으면 404예외가 발생된 후 생성이 된다.")
        void failWhenNotPublished() throws InterruptedException {
            // given
            final String generatedUrl = "https://assets.test.teamby.team/asset/path/icalendar.ics";
            BDDMockito.given(fileCloudUploader.upload(any(byte[].class), any(String.class)))
                    .willAnswer(invocation -> generatedUrl);

            // when
            asyncAspect.init();
            final ExtractableResponse<Response> firstResponse = GET_ICALENDAR_PUBLISHED_URL(authCode, teamPlace.getId());
            asyncAspect.await();

            final ExtractableResponse<Response> secondResponse = GET_ICALENDAR_PUBLISHED_URL(authCode, teamPlace.getId());

            // then
            SoftAssertions.assertSoftly(softly -> {
                softly.assertThat(firstResponse.statusCode()).isEqualTo(HttpStatus.NOT_FOUND.value());
                softly.assertThat(secondResponse.statusCode()).isEqualTo(HttpStatus.OK.value());
                softly.assertThat(secondResponse.jsonPath().getString("url")).isEqualTo(generatedUrl);
            });
        }
    }
}

```
{: file="IcalendarAcceptanceTest.java"}

위에서 `GET_ICALENDAR_PUBLISHED_URL` 는 `RestAssured`를 통해 요청을 보내는 부분을 메서드 분리한 것이다.

해당 테스트에서는 `GET_ICALENDAR_PUBLISHED_URL`최초 요청 후 정상적으로 url이 생성되기까지 테스트코드를 진행하지 않고, 대기하여야 한다.
이를 위해서 아래와 같은 Spring AOP를 구현하였다.

```java
@TestConfiguration
static class TestConfig {

    @Aspect
    @Component
    static class AsyncAspect {

        private CountDownLatch countDownLatch;

        public void init() {
            countDownLatch = new CountDownLatch(1);
        }

        @After("execution(* team.teamby.teambyteam.icalendar.application.IcalendarEventListener.createIcalendar(*))")
        public void afterIcalendarCreation() {
            countDownLatch.countDown();
        }

        public void await() throws InterruptedException {
            countDownLatch.await();
        }
    }
}

```

해당 AOP는 `IcalendarEventListener`의 `createIcalendar`메서드가 실행이 된 이후에 countDownLatch의 숫자를 1회 카운트다운 하는 역할을 한다.
이를 이용하여 해당 `createIcalendar`메서드의 실행을 기다려야 하면, 해당 메서드를 실행하기 전 `init()` 메서드를 실행하여 CountDownLatch를 1로 초기화 한 후,
`await()`메서드를 실행하여 icalendar생성 메서드가 종료되기까지를 기다리게 할 수 있다.

해당 부분의 코드는 아래와 같다.

```java
asyncAspect.init();    // CountDownLatch 초기화
final ExtractableResponse<Response> firstResponse = GET_ICALENDAR_PUBLISHED_URL(authCode, teamPlace.getId());    // ical 생성 전 요청
asyncAspect.await();    // AOP afterIcalendarCreation 종료시까지 기다리기 (countDownLatch 가 0이 될 때까지 기다리기)

final ExtractableResponse<Response> secondResponse = GET_ICALENDAR_PUBLISHED_URL(authCode, teamPlace.getId());    // ical 생성 후 요청
```


## 마무리

이처럼 스프링의 기능을 활용해서 두가지 방법을 이용해 비동기 메서드를 테스트하게 되었다.  
앞으로도 각각의 상황에 맞게 해당 방법들중에 선택을 하여 (혹은 문제가 생긴다면 달느 방법을 생각해보고) 상황에 맞게 비동기 테스트를 처리하게 되면 다양한 상황에서 적절한 테스트가가 가능할 것으로 보인다.

끝!!


---

팀바팀은 우아한테크코스 5기 프로젝트로 진행된 프로젝트로,
한학기에 여러 종류의 팀플을 하는 대학생들이 쉽게 팀플을 할 수 있도록 도움을 주고자 하는 플렛폼 입니다.

[팀바팀 레포 보기](https://github.com/woowacourse-teams/2023-team-by-team)  
[팀바팀 서비스 써보기](https://teamby.team)

