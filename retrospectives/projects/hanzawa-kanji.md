# 일본 상용 한자 학습 앱

## 프로젝트 요약

### 일본 상용 한자 학습 게임 앱

- `기간` `2025-03-07 ~ 2025-05-14`
- `목표` UX 중심의 반복 학습 기능 제공
- `개발 범위`
  - 한자 목록 스크롤 모드 구현
  - 문제 생성 및 보기 랜덤 출력 로직
  - React 프론트 + Kotlin 백엔드 API 정합
- `기술 스택`: React, CSS-module, Kotlin
  <br><br>

# 🖥️ 프론트엔드 (React)

### 1. 무한스크롤 & 성능 개선

#### 문제 상황

학습 모드를 구현할 때 `KanjiCard`마다 UI 항목이 바뀌다보니,<br>
**렌더링이 과도하게 발생하여 성능 저하가 심각했음.**<br>
`react-window`를 도입해 `lazy-rendering`의 힘을 빌리려했으나<br>
복잡한 스타일과 효과로 인해 오히려 **버벅임이 느껴졌음.**

#### 개선 전략

기존 라이브러리의 사용은 유지하되,<br>
React 내부에서 **렌더링 최소화 조치를 적극적으로 적용**함으로써 성능을 끌어올렸다.

| 항목                        | 설명                                             |
| --------------------------- | ------------------------------------------------ |
| ✅ `React.memo()`           | `KanjiCard`, `ReadingRow` 컴포넌트 렌더링 최소화 |
| ✅ `areEqual` 커스텀 비교   | props가 변하지 않으면 리렌더 차단                |
| ✅ `will-change: transform` | CSS GPU 가속 힌트로 렌더링 부하 완화             |
| ✅ `throttle()`             | resize 시 과도한 렌더링 방지                     |
| ✅ `key` 개선               | `index` 대신 `kanji.id` 사용                     |
| ✅ `box-shadow` 최소화      | 렌더링 비용이 큰 CSS 효과 제거                   |

#### 결과

- `react-window`는 그대로 유지하면서도,  
  컴포넌트 단위에서 렌더링을 세밀하게 조절한 덕분에  
  **전체적으로 훨씬 부드러운 UX를 구현할 수 있었음.**

### 📌 배운 점

> _성능 개선은 도구를 바꾸는 것보다, **도구를 어떻게 쓰느냐**에 달려 있다._

### 2. 문제 보기(Choice) 설계

#### 문제 상황

문제 개수를 선택하면 `limit={choice}`만큼 문제를 가져오고,<br>
그 안에서 보기(choice)를 랜덤하게 뽑았더니
**보기들이 자주 겹치면서 난이도가 급격하게 낮아졌음.**

#### 시도한 솔루션

1. **문제마다 UUID 생성 후 해당 UUID 기반으로 보기 랜덤 출력**
   <br>⚠️ 렌더링이 과도하게 발생
2. **choicePool 상태값 생성 후, 50개 보기를 미리 셔플해 저장**
   <br>✅ 중복 보기를 기대치만큼 줄였으며 성능 타협점 역시 확보
3. 전체 상용한자 2300개를 choicePool에 넣고 그 중 랜덤 출력
   <br>💡 가장 단순한 구조였지만 성능이 가장 뛰어났음
   <br>💡 데이터 규모가 작고 변동이 크지 않다면 단순함이 최선일 수 있다는 걸 느낌

#### 📌 고민 지점

- 이렇게 단순한 구조를 선택하면 유지보수성에 좋지 않을 것 같아 두번째 방식을 선택했지만 <br>변동 가능성이 낮은 데이터 셋에서도 더 좋은 성능을 포기하는 것이 과연 `개발자로서 맞는 선택`일까?
- **비즈니스 로직의 유연성과 현재 성능 최적화의 균형점**에 대해 생각하게 됨
  <br><br>

## 🧘 배우고 느낀 것

- 성능 최적화는 라이브러리보다 **렌더링 원칙**과 **CSS 최적화**에서 시작된다.
- 보기 로직 설계에서 **UX 난이도**와 **성능** 사이 밸런스가 중요하다는 걸 체감했다.
- 단순한 로직이 성능 면에서 유리한 경우가 있을 때 그 선택을 정당화하는 **상황 판단 능력** 역시 개발자의 몫이라는 것을 느꼈다.
  <br><br>

## 개선 아이디어

- 한자 즐겨찾기 기능
- 사용자 맞춤 문제 제공
- 로딩 성능 개선을 위한 `SSR/CSR` 교차 사용
- 백엔드에서 `choicePool`도 캐싱할 수 있는 구조로 개선

---

# 🔧 백엔드 API 설계 (Kotlin)

### 구조

이 프로젝트에서 처음으로 **백엔드 구조를 레이어드 아키텍처로 분리**해서 설계해봤다. <br><br>
Controller, Service, Repository, DB 계층을 구분하고,  
단순한 데이터 소스 기반이지만 구조적으로 확장 가능한 형태를 고민했다.

- `Controller` 요청 수신
- `Service` 퀴즈 모드에 따라 데이터 처리
- `Persistence` 한자 리스트 및 문제 조회
- `DB` 데이터 저장소 및 도메인 모델

### 요약

| 항목        | 설명                                                                  |
| --------- | ------------------------------------------------------------------- |
| 레이어드 아키텍처 | Controller → Service → Repository → DataSource 구조                   |
| 비즈니스 분리   | - `controller` 받고 넘기기<br>- `service` 로직 판단<br>- `repository` 데이터 조회 |
| CORS 설정   | React 애플리케이션과 서로 다른 `origin`에 위치하더라도 API 수신이 가능                     |

### API 예시

```http
GET /api/v1/hanzawa-kanji?quizId={id}&mode=RANDOM
```

> Restfull한 API를 설계하기 위해서는 GET 메소드와 <br>Kanji 또는 hanza 등의 명시적인 리소스를 사용해야했지만 <br>애플리케이션의 이름에 맞추어 리소스는 hanzawa-kanji로 명명했다🔥🔥

#### 쿼리 파라미터 설계
- `quizId` 현재 퀴즈 ID
- `mode` Random / Infinite 모드 분기
- `cursor` cursor 기반 페이지네이션을 위한 인자

### 📁 디렉토리 구조

```bash
kotlin/com/mindaaaa/hanzawakanji/
├── db
│   ├── model
│   │   └── Kanji.kt              # 도메인 모델 정의
│   └── KanjiDataSource.kt        # data.json 불러온 후 Kanji 리스트 변환

├── persistence
│   ├── model
│   │   └── Mode.kt
│   ├── KanjiRepository.kt        # Repository 인터페이스 (데이터 조회)
│   └── KanjiRepositoryImpl.kt    # Repository 인터페이스 구현체

├── presentation
│   ├── KanjiController.kt        # HTTP 요청 수신
│   └── WebConfiguration.kt       # CORS 설정

├── service
│   ├── 📁 dto
│   │   ├── ListRequestDto.kt     # 요청 DTO
│   │   └── ListResponseDto.kt    # 응답 DTO
│   └── KanjiService.kt           # 애플리케이션 로직 처리 → repository로부터 조회 후 결과 가공
├── Application.kt
└── Configuration.kt              # 계층별 의존성 주입
```

### 아키텍처 설계 및 계층 구조

이번 프로젝트는 처음으로 백엔드 구조를 직접 설계해보는 경험이었다.
<br>특히 레이어드 아키텍처 구조를 이해하고 적용하는 데 중점을 뒀다.

#### 계층별 역할

| 레이어          | 폴더명             | 역할            |
| ------------ | --------------- | ------------- |
| Presentation | `presentation/` | API 진입점       |
| Service      | `service/`      | 비즈니스 로직 중심 계층 |
| Persistence  | `persistence/`  | 데이터 접근 계층     |
| Data         | `db/`           | 한자 데이터 계층     |


### API 흐름

```
[프론트 요청]
   ↓
KanjiController (REST API 진입점)
   ↓
KanjiService (비즈니스 로직)
   ↓
KanjiRepositoryImpl (데이터 정제 / 페이징 / 모드 분기)
   ↓
KanjiDataSource (JSON 데이터 로딩)
   ↑
[응답: List<Kanji> → DTO 변환 → JSON]
```

1. `KanjiCotroller`가 요청을 수신함
2. `KanjiService`가 **DTO**로 받은 요청을 **Repository에 위임**
3. `KanjiRepositoryImpl`이 DataSource로부터 데이터를 불러와 가공함
4. `KanjiDataSource`는 `data.json`을 읽어 **List**로 반환함
5. 최종 응답을 `ListResponseDto`로 변환해 클라이언트에 반환

> Controller → Service → Repository → DataSource 구조로 분리

### 🌐 CORS 설정

프론트엔드(React)와 백엔드를 각각 다른 `origin`에서 동작하게 되면 브라우저의 **CORS** 정책에 따라 API 요청이 차단되는 경우가 발생할 수 있다.

이를 위해 Spring의 `WebMvcConfigurer`를 활용해 **CORS 설정**을 추가했다.

```kotlin
registry.addMapping("/**")
    .allowedOrigins("*")
    .allowedMethods("*")
    .allowedHeaders("*")
```

#### 📌 고민 지점

`addMapping`에 모든 `origin`을 허용하도록 했는데 실제 배포시에도 이런 설정이 적절할까?

### Service 계층의 이해

핵심은 `로직의 흐름을 조합하는 허브`라는 점이다.

```kotlin
fun list(dto: ListRequestDto): ListResponseDto {
    val kanjiList = repository.list(...)
    return ListResponseDto(...) // if 분기 처리 생략
}
```

1. `Service`는 요청 파라미터를 받아 도메인 로직을 조합한다.
2. `Repository`에 필요한 조건을 전달한다.
3. 그 결과를 **응답용 DTO**로 가공해 `Controller`에 반환한다.

> 실제 로직은 Repository에 위임하면서도, <br>전체 흐름을 관리하고 Controller와 비즈니스 계층 사이의 중개 역할을 수행함

### 🔧 레이어 간 책임과 추상화에 대한 고찰

#### 추상화

`Repository`와 구현체를 나눈 이유는 **테스트나 확장을 고려한 구조적 유연성**을 위해서 인터페이스를 도입한 것이다.<br>
Service 계층이 구현체와 결합하지 않도록 설계함으로써 `테스트 작성 / DI 적용 / 향후 교체`에 유리하도록 구성했다.

#### 책임 분리

`KanjiDataSource`는 JSON 파일을 불러와 `Kanji` 도메인 모델로 매핑하는 역할을 담당한다.

이 클래스는 비즈니스 레이어에 **직접** 관여하지 않으며,<br>
`KanjiRepository`를 통해서만 접근`필터링/페이징`하게 하여 계층 간의 책임을 분리할 수 있도록 설계했다.

`Kanji` 클래스는 한자 정보를 표현하는 **도메인 모델**을 정의하기 위해 사용되었다.

### 🔧 DTO를 통한 계층 간의 통신

#### DTO 도입

요청 / 응답에 필요한 정보를 명확히 전달하기 위해
Controller ↔ Service 간에는 DTO를 사용해 의도를 분리했다.

이 결과,<br>
사용자의 요청과 서비스 인터페이스를 **독립적으로 유지**할 수 있었고<br>
도메인 구조가 변경되더라도 **안정적으로 API 스펙이 유지**될 수 있는 효과를 기대할 수 있다.

> `Kanji` → 개별 데이터 <br> `RequestDto`→ 프론트에서 보낸 쇼핑 리스트<br> `ResponseDto`→ 포장해서 출고되는 택배 상자

> 설계 의도를 분명히 표현했을 뿐만 아니라
> API와 내부 도메인 로직을 안전하게 다룰 수 있게 됐다.

### 🔧 의존성 주입과 DIP 적용

Spring의 DI(의존성 주입)을 활용해 `KanjiRepository` 인터페이스의 구현체로 `KanjiRepositoryImpl`을 주입받는 구조로 설정했다.

```kotlin
@Configuration
class Configuration(
    private val dataSource: KanjiDataSource,
) {
    @Bean
    fun kanjiRepository(): KanjiRepository {
        return KanjiRepositoryImpl(dataSource)
    }
}
```

이를 통해 **구현체 교체, 테스트 주입, 유지 보수**에 유리한 구조를 만들 수 있었다.


## 🧘 배우고 느낀 것

- 기능을 `작동하게 만드는 것`이 아니라 **어떻게 설계하고 분리할 것인가**를 진지하게 고민하게 된 프로젝트였다.
- `Kotlin/Spring` 환경에서 레이어드 구조 설계 감각을 익혔다.
- `DTO`, `Repository`, `CORS` 같은 개념이 어떻게 구현되는지 감을 잡을 수 있었다.
  <br><br>

## 총평

기능 구현보다도 **어떻게 만들지**를 끊임없이 고민했던 시간이었다.

> 단순히 *작동하는 것*이 아니라 **좋게 작동하는 방법**을 고민하면서,  
>  개발자로서의 기준과 태도가 한 단계 성장한 경험이었다👊🏻

## 인사이트

- 성능 개선은 새로운 기술보다 **기본기 튜닝**이 먼저다.
- 비즈니스 로직 설계에서 **단순함**이 때로는 최고의 선택이 될 수 있다.
- 앞으로 어떤 구조를 채택할지 **개발자로서의 판단 기준**을 고민하는 계기를 얻었다🔥🔥
