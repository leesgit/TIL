# MVI에서 SideEffect 처리

> 일회성 이벤트(토스트, 네비게이션, 스낵바)를 MVI 패턴에서 안전하게 처리하는 방법

## 핵심 개념

MVI의 State는 화면의 "상태"를 나타내며 지속적이다. 하지만 토스트, 네비게이션, 스낵바 같은 일회성 이벤트는 State에 담으면 문제가 생긴다.

### State에 이벤트를 담으면 생기는 문제

#### 위반 사례

```kotlin
data class HomeState(
    val navigateToDetail: Long? = null, // 화면 회전하면 다시 네비게이션됨
    val showToast: String? = null       // 화면 회전하면 다시 토스트 뜸
)
```

**문제점:**
- Configuration Change(화면 회전) 시 State가 다시 수집되면서 이벤트가 중복 발생한다
- State를 null로 초기화하는 추가 로직이 필요하다

## SideEffect 패턴

### 개선안: Channel 사용

```kotlin
class HomeViewModel : ViewModel() {

    private val _state = MutableStateFlow(HomeState())
    val state: StateFlow<HomeState> = _state.asStateFlow()

    // SideEffect는 Channel로 분리
    private val _sideEffect = Channel<HomeSideEffect>()
    val sideEffect: Flow<HomeSideEffect> = _sideEffect.receiveAsFlow()

    fun processIntent(intent: HomeIntent) {
        when (intent) {
            is HomeIntent.MovieClicked -> {
                viewModelScope.launch {
                    _sideEffect.send(HomeSideEffect.NavigateToDetail(intent.movieId))
                }
            }
            is HomeIntent.AddToFavorite -> {
                viewModelScope.launch {
                    addToFavorite(intent.movieId)
                    _sideEffect.send(HomeSideEffect.ShowToast("즐겨찾기에 추가됨"))
                }
            }
        }
    }
}

sealed class HomeSideEffect {
    data class NavigateToDetail(val movieId: Long) : HomeSideEffect()
    data class ShowToast(val message: String) : HomeSideEffect()
    object ShowSnackbar : HomeSideEffect()
}
```

### UI에서 수집

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel,
    onNavigateToDetail: (Long) -> Unit
) {
    val context = LocalContext.current

    // SideEffect 수집 (일회성)
    LaunchedEffect(Unit) {
        viewModel.sideEffect.collect { effect ->
            when (effect) {
                is HomeSideEffect.NavigateToDetail -> {
                    onNavigateToDetail(effect.movieId)
                }
                is HomeSideEffect.ShowToast -> {
                    Toast.makeText(context, effect.message, Toast.LENGTH_SHORT).show()
                }
                is HomeSideEffect.ShowSnackbar -> {
                    // snackbarHostState.showSnackbar(...)
                }
            }
        }
    }

    // State 수집 (지속적)
    val state by viewModel.state.collectAsStateWithLifecycle()

    // UI 렌더링
    HomeContent(state = state, onIntent = viewModel::processIntent)
}
```

## Channel vs SharedFlow

| 구분 | Channel | SharedFlow |
|------|---------|------------|
| 구독자 | 단일 | 다중 가능 |
| 버퍼링 | 구독자 없으면 저장 | replay=0이면 유실 |
| 용도 | 일회성 이벤트 | 브로드캐스트 |

### Channel (권장)

```kotlin
private val _sideEffect = Channel<SideEffect>()
val sideEffect = _sideEffect.receiveAsFlow()
```

- 이벤트가 정확히 한 번만 전달됨
- 구독자 없으면 버퍼에 저장 (기본 RENDEZVOUS)

### SharedFlow (대안)

```kotlin
private val _sideEffect = MutableSharedFlow<SideEffect>()
val sideEffect = _sideEffect.asSharedFlow()
```

- 여러 구독자에게 전달 가능
- 구독자 없으면 이벤트 유실 (replay=0일 때)

### 선택 기준

| 상황 | 선택 |
|------|------|
| 단일 구독자, 이벤트 유실 방지 | Channel |
| 다중 구독자, 브로드캐스트 필요 | SharedFlow |
| 최신 이벤트만 필요 | SharedFlow(replay=1) |

## Orbit MVI 라이브러리 활용

```kotlin
// Orbit MVI를 사용하면 더 간결하게 처리 가능
class HomeViewModel : ContainerHost<HomeState, HomeSideEffect>, ViewModel() {

    override val container = container<HomeState, HomeSideEffect>(HomeState())

    fun onMovieClicked(movieId: Long) = intent {
        postSideEffect(HomeSideEffect.NavigateToDetail(movieId))
    }

    fun addToFavorite(movieId: Long) = intent {
        addToFavoriteUseCase(movieId)
        postSideEffect(HomeSideEffect.ShowToast("추가됨"))
    }
}
```

## 면접 포인트

- **Q: MVI에서 일회성 이벤트를 State에 담으면 안 되는 이유는?**
- A: State는 Configuration Change 이후에도 유지되어 이벤트가 중복 발생한다. 화면 회전 후 토스트가 다시 뜨거나 네비게이션이 다시 실행되는 문제가 생긴다.

- **Q: Channel과 SharedFlow 중 SideEffect에 뭘 쓰는 게 좋나요?**
- A: 단일 구독자이고 이벤트 유실을 방지해야 하면 Channel이 적합하다. Channel은 구독자가 없을 때 이벤트를 버퍼에 저장하고, 구독 시 전달한다.

- **Q: SideEffect와 State를 어떻게 구분하나요?**
- A: State는 화면에 표시되는 지속적인 데이터(로딩 상태, 목록), SideEffect는 일회성 이벤트(토스트, 네비게이션, 다이얼로그)다. "화면 회전 후에도 유지되어야 하나?"를 기준으로 판단한다.

## 참고

- [Kotlin Channel](https://kotlinlang.org/docs/channels.html)
- [Orbit MVI](https://orbit-mvi.org/)
