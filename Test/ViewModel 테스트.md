# ViewModel 테스트 전략

> ViewModel의 State 변화와 SideEffect를 검증하는 테스트 작성법

## 핵심 개념

ViewModel 테스트의 목적은 다음과 같다:
1. Intent 입력에 따른 State 변화 검증
2. SideEffect가 올바르게 발생하는지 검증
3. UseCase 호출이 정확히 이루어지는지 검증

## 테스트 환경 설정

### 의존성

```kotlin
// build.gradle.kts
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
testImplementation("io.mockk:mockk:1.13.8")
testImplementation("app.cash.turbine:turbine:1.0.0")
testImplementation("org.junit.jupiter:junit-jupiter:5.10.0")
```

### TestDispatcher 설정

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class ViewModelTest {

    private val testDispatcher = StandardTestDispatcher()

    @BeforeEach
    fun setup() {
        Dispatchers.setMain(testDispatcher)
    }

    @AfterEach
    fun tearDown() {
        Dispatchers.resetMain()
    }
}
```

## State 테스트

### Turbine으로 Flow 테스트

```kotlin
@Test
fun `영화 로드 성공 시 State에 영화 목록이 설정된다`() = runTest {
    // Given
    val movies = listOf(
        Movie(1, "Movie 1"),
        Movie(2, "Movie 2")
    )
    coEvery { getMoviesUseCase() } returns Result.success(movies)

    val viewModel = HomeViewModel(getMoviesUseCase)

    // When & Then
    viewModel.state.test {
        // 초기 상태
        awaitItem() shouldBe HomeState()

        // 로드 실행
        viewModel.processIntent(HomeIntent.LoadMovies)

        // 로딩 상태
        awaitItem() shouldBe HomeState(isLoading = true)

        // 성공 상태
        awaitItem() shouldBe HomeState(
            isLoading = false,
            movies = movies
        )
    }
}
```

### 에러 케이스 테스트

```kotlin
@Test
fun `영화 로드 실패 시 에러 메시지가 설정된다`() = runTest {
    // Given
    coEvery { getMoviesUseCase() } returns Result.failure(
        Exception("Network error")
    )

    val viewModel = HomeViewModel(getMoviesUseCase)

    // When & Then
    viewModel.state.test {
        awaitItem() // 초기 상태

        viewModel.processIntent(HomeIntent.LoadMovies)

        awaitItem() shouldBe HomeState(isLoading = true)
        awaitItem() shouldBe HomeState(
            isLoading = false,
            error = "Network error"
        )
    }
}
```

## SideEffect 테스트

```kotlin
@Test
fun `영화 클릭 시 상세 화면 네비게이션 이벤트가 발생한다`() = runTest {
    // Given
    val viewModel = HomeViewModel(getMoviesUseCase)
    val movieId = 123L

    // When & Then
    viewModel.sideEffect.test {
        viewModel.processIntent(HomeIntent.MovieClicked(movieId))

        awaitItem() shouldBe HomeSideEffect.NavigateToDetail(movieId)
    }
}
```

## Mock vs Fake

### Mock (MockK)

```kotlin
// Mock: 행위 검증에 초점
@Test
fun `새로고침 시 UseCase가 forceRefresh=true로 호출된다`() = runTest {
    // Given
    val getMoviesUseCase = mockk<GetMoviesUseCase>()
    coEvery { getMoviesUseCase(any()) } returns Result.success(emptyList())

    val viewModel = HomeViewModel(getMoviesUseCase)

    // When
    viewModel.processIntent(HomeIntent.Refresh)
    advanceUntilIdle()

    // Then
    coVerify { getMoviesUseCase(forceRefresh = true) }
}
```

### Fake (선호)

```kotlin
// Fake: 실제와 유사한 동작, 상태 검증에 초점
class FakeMovieRepository : MovieRepository {
    private val movies = mutableListOf<Movie>()
    var fetchCount = 0
        private set

    fun setMovies(list: List<Movie>) {
        movies.clear()
        movies.addAll(list)
    }

    override suspend fun getMovies(): List<Movie> {
        fetchCount++
        return movies.toList()
    }
}

@Test
fun `새로고침 시 레포지토리에서 다시 조회한다`() = runTest {
    // Given
    val fakeRepository = FakeMovieRepository()
    fakeRepository.setMovies(listOf(Movie(1, "Test")))

    val viewModel = HomeViewModel(GetMoviesUseCase(fakeRepository))

    // When
    viewModel.processIntent(HomeIntent.LoadMovies)
    advanceUntilIdle()
    viewModel.processIntent(HomeIntent.Refresh)
    advanceUntilIdle()

    // Then
    fakeRepository.fetchCount shouldBe 2
}
```

## 테스트 패턴

### Given-When-Then 구조

```kotlin
@Test
fun `필터 선택 시 해당 카테고리 영화만 표시된다`() = runTest {
    // Given: 테스트 조건 설정
    val allMovies = listOf(
        Movie(1, "Action Movie", category = Category.ACTION),
        Movie(2, "Drama Movie", category = Category.DRAMA)
    )
    coEvery { getMoviesUseCase() } returns Result.success(allMovies)

    val viewModel = HomeViewModel(getMoviesUseCase)
    viewModel.processIntent(HomeIntent.LoadMovies)
    advanceUntilIdle()

    // When: 액션 실행
    viewModel.processIntent(HomeIntent.SelectFilter(Category.ACTION))
    advanceUntilIdle()

    // Then: 결과 검증
    viewModel.state.value.filteredMovies shouldHaveSize 1
    viewModel.state.value.filteredMovies[0].category shouldBe Category.ACTION
}
```

## Mock vs Fake 선택 기준

| 상황 | 선택 |
|------|------|
| 메서드 호출 여부/횟수 검증 | Mock |
| 실제 로직 동작 검증 | Fake |
| 외부 의존성 격리 | Mock |
| 복잡한 상태 시뮬레이션 | Fake |

## 면접 포인트

- **Q: ViewModel 테스트에서 Dispatcher를 왜 교체하나요?**
- A: viewModelScope는 기본적으로 Main Dispatcher를 사용하는데, 테스트 환경에는 Main Dispatcher가 없다. TestDispatcher로 교체하면 코루틴 실행을 제어할 수 있고 테스트가 결정적(deterministic)으로 동작한다.

- **Q: Turbine은 무엇이고 왜 사용하나요?**
- A: Flow를 테스트하기 위한 라이브러리다. `test {}` 블록 안에서 `awaitItem()`으로 방출된 값을 순서대로 검증할 수 있다. Flow의 비동기 특성을 동기적으로 테스트할 수 있게 해준다.

- **Q: Mock과 Fake의 차이는?**
- A: Mock은 메서드 호출과 인자를 검증하는 데 초점을 둔다. Fake는 실제 구현과 유사하게 동작하는 테스트용 구현체다. Fake는 상태 기반 테스트에, Mock은 행위 기반 테스트에 적합하다.

## 참고

- [Turbine](https://github.com/cashapp/turbine)
- [MockK](https://mockk.io/)
- [Testing Coroutines](https://developer.android.com/kotlin/coroutines/test)
