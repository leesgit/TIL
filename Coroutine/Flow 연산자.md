# Flow Operators

> Flow에서 **데이터를 변환·결합·필터링·제어**할 때 사용하는 중간 연산자들이다.

## 핵심 개념

**Flow 연산자(Flow Operators)** 는 스트림 중간에서 데이터를 가공하는 **중간 연산자**다.
각 연산자는 **항상 새로운 Flow를 반환**하며, `collect`가 호출되기 전까지는 **실제로 실행되지 않는다** (지연(Lazy) 수행).

생각할 때는 이렇게 정리하면 편하다:

* Flow는 “데이터가 시간 순서대로 흘러가는 파이프라인”
* 연산자는 “그 사이에 끼워 넣는 필터/변환기”
* 실제로 동작하는 시점은 `collect { ... }` 를 호출했을 때

---

## 주요 연산자 분류

**변환(Transformation)**

* `map`, `mapLatest`, `flatMapLatest`, `flatMapMerge`, `flatMapConcat` …

**결합(Combination)**

* `combine`, `zip`, `merge` …

**필터링(Filtering)**

* `filter`, `filterNotNull`, `distinctUntilChanged` …

**제한 / 타이밍(Control)**

* `take`, `takeWhile`, `debounce`, `sample`, `throttleFirst` …

아래부터는 실제로 안드로이드에서 자주 쓰이는 것들 위주로 정리해본다.

---

## map

가장 기본적인 **1:1 변환 연산자**다.
들어온 값 하나를 다른 값 하나로 바꿔서 다시 Flow로 흘려보낸다.

```kotlin
val userIds = flowOf(1, 2, 3)

userIds
    .map { id -> repository.getUser(id) } // Int → User 변환
    .collect { user -> println(user.name) }
```

* `map` 내부는 **순수 함수**로 유지하는 게 좋다. (가능하면 side-effect 줄이기)
* 네트워크/DB 호출처럼 무거운 작업은 `map` 안에서 하기보다는
  `flowOn(Dispatchers.IO)` 등으로 컨텍스트를 분리해주면 더 안전하다.

---

## flatMapLatest

**새 값이 들어오면 이전 내부 Flow를 취소하고, 최신 값에 대한 Flow만 유지하는 연산자.**
검색 자동완성, 텍스트 입력 같은 **“이전 요청은 버리고 최신 것만 처리”**해야 하는 케이스에 딱 맞다.

### 안 좋은 예 (flatMapMerge 사용)

```kotlin
// flatMapMerge 사용 시 이전 검색 결과가 뒤늦게 도착할 수 있음
searchQuery
    .flatMapMerge { query ->
        repository.search(query) // Flow<List<Result>>
    }
    .collect { results ->
        // "abc" 검색 결과가 "ab" 검색 결과보다 먼저 올 수 있음
    }
```

* `flatMapMerge` 는 내부 Flow들을 **동시에 병렬로 다 돌려버리기 때문에**,
  오래 걸리는 이전 검색 결과가 나중에 도착해서 UI를 덮어버릴 수 있다.
* 검색창에서 이런 일이 발생하면, 사용자는 “분명 `abc`를 쳤는데 `ab` 결과가 보이는” 이상한 경험을 하게 된다.

### 개선안 – flatMapLatest + debounce

```kotlin
searchQuery
    .debounce(300) // 300ms 동안 입력이 없을 때만 다음 단계로 흘려보냄
    .flatMapLatest { query ->
        if (query.isEmpty()) {
            flowOf(emptyList())
        } else {
            repository.search(query) // 항상 최신 검색어만 처리
        }
    }
    .collect { results ->
        // 항상 "가장 마지막에 입력된 검색어" 결과만 수신
        updateUI(results)
    }
```

* `debounce(300)` : 사용자가 계속 타이핑하는 동안은 잠깐 기다렸다가,
  **입력이 멈춘 뒤 300ms 지나면** 그때 실제 검색 요청을 보낸다.
* `flatMapLatest` : 입력이 바뀌면 이전 검색 Flow는 바로 `cancel` 되고,
  **최신 query에 대한 Flow만 살아남는다.**

---

## flatMapMerge / flatMapConcat 간단 비교

실무에서 헷갈리기 쉬우니까 한 번에 정리:

* `flatMapLatest`

    * 최신 값만 유지, 이전 것은 **취소**
    * 검색, 실시간 입력 등 → **UI에 최신 값만 중요할 때**

* `flatMapMerge`

    * 내부 Flow들을 **동시에 병렬로 실행**
    * 여러 파일 동시에 업로드, 여러 API 병렬 호출 등 → **결과 순서 중요하지 않을 때**

* `flatMapConcat`

    * 내부 Flow를 **순차적으로** 실행 (하나 끝나야 다음 시작)
    * 순서가 중요한 작업, 백프레셔에 민감한 경우에 사용

---

## filter

값들을 **조건에 맞는 것만 통과**시키는 연산자.

```kotlin
val numbers = (1..10).asFlow()

numbers
    .filter { it % 2 == 0 } // 짝수만 통과
    .collect { println(it) } // 2, 4, 6, 8, 10
```

* UI에서 “체크박스 ON일 때만 이벤트 처리” 같은 경우에 자주 쓴다.
* `filterNotNull()` 을 같이 써서 **null 제거 → non-null Flow** 로 만드는 패턴도 많이 사용.

---

## distinctUntilChanged

**연속된 같은 값을 무시**하는 연산자.
“값이 실제로 바뀌었을 때만 처리해라”라는 의미.

```kotlin
val searchQuery = MutableStateFlow("")

searchQuery
    .distinctUntilChanged()
    .flatMapLatest { query -> repository.search(query) }
    .collect { results -> updateUI(results) }

searchQuery.value = "kotlin"
searchQuery.value = "kotlin" // 무시됨 (같은 값)
searchQuery.value = "android" // 새 값, 실행됨
```

* TextField의 입력을 Flow로 감싼 뒤 `distinctUntilChanged()` 걸어두면,
  똑같은 값이 반복해서 흘러 들어와도 **불필요한 API 호출/렌더링을 막을 수 있다.**

---

## take

Flow에서 **앞에서부터 N개만 받고 끝내는** 연산자.

```kotlin
(1..100).asFlow()
    .take(3)
    .collect { println(it) } // 1, 2, 3만 출력되고 Flow 종료
```

* “프리뷰로 앞에 몇 개만 보고 싶다”
* “특정 개수까지만 처리하고 스트림 끊고 싶다” 같은 경우에 사용.

---

## debounce

**일정 시간 동안 새 값이 안 들어올 때만 값 방출**하는 연산자.

```kotlin
searchQuery
    .debounce(300)
    .collect { query ->
        // 300ms 동안 입력이 멈췄을 때만 여기로 내려옴
        viewModel.search(query)
    }
```

* 검색, 자동완성, 실시간 검증 로직에서 필수.
* `distinctUntilChanged()` 와 같이 쓰면:

    * **값이 바뀐 경우 + 입력이 잠시 멈춘 경우**에만 처리되는 깔끔한 패턴.

---

## combine

여러 Flow의 **“최신 값”을 묶어서 하나의 값으로 만드는 연산자.**
어느 한쪽 Flow라도 새 값을 내보내면,
각 Flow의 **가장 최근 값들**을 모아서 다시 방출한다.

```kotlin
val name = MutableStateFlow("홍길동")
val age = MutableStateFlow(30)

combine(name, age) { n, a ->
    "이름: $n, 나이: $a"
}.collect { println(it) }

// 출력: 이름: 홍길동, 나이: 30

name.value = "김철수"
// 출력: 이름: 김철수, 나이: 30

age.value = 25
// 출력: 이름: 김철수, 나이: 25
```

* UI 관점에서는 **여러 상태를 하나의 `UiState` 로 합치는 데** 거의 기본으로 쓰는 연산자.

### 실무 활용 – UI 상태 결합

```kotlin
class UserProfileViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val isLoadingFlow = MutableStateFlow(false)

    private val userFlow = repository.observeUser()
    private val postsFlow = repository.observePosts()

    val uiState = combine(
        userFlow,
        postsFlow,
        isLoadingFlow
    ) { user, posts, isLoading ->
        UserProfileState(
            user = user,
            posts = posts,
            isLoading = isLoading
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = UserProfileState()
    )
}
```

* `combine` 으로 여러 데이터 소스를 `UserProfileState` 하나로 묶고,
* `stateIn` 으로 **StateFlow로 승격**해서 UI에서 `collectAsState()` 로 바로 사용.

---

## zip

두 Flow의 값을 **순서대로 1:1 매칭**하는 연산자.
둘 다에서 값이 하나씩 나와야 매칭해서 방출한다.

```kotlin
val names = flowOf("A", "B", "C")
val numbers = flowOf(1, 2, 3, 4, 5)

names.zip(numbers) { name, number ->
    "$name$number"
}.collect { println(it) }

// 출력: A1, B2, C3 (4, 5는 무시됨)
```

* 한쪽이 먼저 많이 나와도 **반대쪽에서 값이 올 때까지 기다렸다가** 같이 방출한다.
* 한쪽 Flow가 먼저 끝나면, zip 전체도 같이 종료된다.

---

## combine vs zip 비교

| 구분    | `combine`                                  | `zip`                 |
| ----- | ------------------------------------------ | --------------------- |
| 동작    | 각 Flow의 **최신 값**들을 결합                      | **순서대로 1:1 매칭**       |
| 방출 시점 | 어느 Flow든 새 값이 나오면 (각 Flow가 최소 1번은 emit한 뒤) | 양쪽 Flow 모두 값이 있을 때    |
| 종료 조건 | **모든 Flow가 종료**되면 종료                       | **하나라도 종료**되면 종료      |
| 주 사용처 | UI 상태 결합, 여러 상태를 하나의 State로                | 요청-응답, 1:1 매칭이 중요한 경우 |

실무에서 보통:

* **UI 상태** 만들 때 → `combine`
* **요청 리스트 + 응답 리스트** 순서대로 매칭해야 할 때 → `zip`

---

## stateIn + SharingStarted.WhileSubscribed(5000)

위에서 `uiState` 를 만들 때 사용한 부분을 한 번 더 짚고 가면:

```kotlin
val uiState = someFlow
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5_000),
        initialValue = UserProfileState()
    )
```

* `stateIn`

    * 일반 Flow를 **StateFlow로 승격**시키는 연산자
    * 항상 “마지막 값”을 캐싱하고, 구독 즉시 최신 값을 받을 수 있다.
* `SharingStarted.WhileSubscribed(5_000)`

    * **구독자가 하나도 없으면** upstream Flow를 잠시 멈춘다.
    * 마지막 구독자가 사라진 시점부터 **최대 5초까지는 유지**했다가,
      진짜로 아무도 안 들고 있으면 그때 cancel.
    * 안드로이드에서 화면 회전 등으로 **잠깐 구독이 끊겼다가 다시 생길 때**,
      매번 네트워크를 다시 타는 걸 방지할 수 있다.

---

## 정리 – 자주 나오는 면접 포인트

### Q. `flatMapLatest` 와 `flatMapMerge` 차이는?

* `flatMapLatest`

    * 새 값이 들어오면 이전 내부 Flow를 **cancel** 하고
      **“가장 최신 값에 대한 Flow”만 유지**
    * 검색 자동완성, 텍스트 입력처럼 **최신 값만 의미 있는 경우**에 사용

* `flatMapMerge`

    * 내부 Flow들을 **동시에 병렬로 실행**
    * 오래 걸리는 작업 여러 개를 동시에 돌리고 싶을 때 사용
      (여러 파일 업로드, 여러 API 병렬 호출 등)
    * 대신, **결과가 뒤섞여서 도착해도 괜찮은 상황**이어야 한다.

---

### Q. `combine` 과 `zip` 차이는?

* `combine`

    * 각 Flow의 **가장 최신 값들을 합쳐서** 방출
    * 어느 한 Flow에서 새 값이 나와도 (모든 Flow가 최소 1번 emit 된 이후) 바로 다시 계산
    * UI에서 **여러 상태를 하나의 UiState로 묶는 용도**에 최적

* `zip`

    * 각 Flow에서 나온 값들을 **순서대로 1:1 짝 맞추기**
    * 양쪽 Flow 모두에서 값이 있어야 한 번 방출
    * 한쪽이 먼저 끝나면 전체 종료
    * 요청 리스트와 응답 리스트를 **순서대로 매칭**해야 할 때 유용

---

### Q. `stateIn(..., SharingStarted.WhileSubscribed(5000), ...)` 의미는?

* Flow를 **StateFlow로 바꿔서** UI에서 바로 쓸 수 있게 만든다.
* `WhileSubscribed(5000)`:

    * 마지막 구독자가 사라진 뒤, **5초 동안은 upstream을 유지**한다.
    * 이 시간 안에 구독자가 다시 붙으면 **다시 처음부터 만들지 않고 이어서 사용**.
    * 안드로이드에서 화면 회전 / 일시적인 재구독 때문에
      **API 재호출이나 불필요한 초기화가 반복되는 걸 막는 패턴**이다.

---

## 참고 링크

* Kotlin 공식 문서 – Intermediate operators
  [https://kotlinlang.org/docs/flow.html#intermediate-flow-operators](https://kotlinlang.org/docs/flow.html#intermediate-flow-operators)

* Android Developers – Flows in practice
  [https://developer.android.com/kotlin/flow](https://developer.android.com/kotlin/flow)
    