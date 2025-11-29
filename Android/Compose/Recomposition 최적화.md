# Recomposition 최적화

> Compose에서 불필요한 Recomposition을 방지하여 성능을 개선하는 방법

## 핵심 개념

Recomposition은 State가 변경될 때 해당 State를 읽는 Composable 함수가 다시 실행되는 것이다. 불필요한 Recomposition은 UI 성능 저하의 주요 원인이 된다.

### Recomposition이 발생하는 조건
1. State 값이 변경됨
2. 해당 State를 읽는 Composable이 호출됨
3. 새로운 값과 이전 값이 `equals()`로 비교했을 때 다름

## 최적화 기법

### 1. State 읽기 지연 (Defer State Read)

#### 위반 사례

```kotlin
@Composable
fun BadScrollList(scrollState: ScrollState) {
    val offset = scrollState.value // 여기서 State 읽음
    Column {
        Header(offset = offset)
        Content()
    }
}
```

**문제점:**
- 매 스크롤마다 전체 Column이 Recomposition된다
- Content()도 불필요하게 다시 실행된다

#### 개선안

```kotlin
@Composable
fun GoodScrollList(scrollState: ScrollState) {
    Column {
        Header(scrollStateProvider = { scrollState.value })
        Content() // Recomposition 안 됨
    }
}

@Composable
fun Header(scrollStateProvider: () -> Int) {
    val offset = scrollStateProvider() // 여기서만 Recomposition
    // ...
}
```

### 2. derivedStateOf 사용

#### 위반 사례

```kotlin
@Composable
fun BadList(items: List<Item>) {
    val showButton = items.size > 10 // items 변경마다 Recomposition
    // ...
}
```

#### 개선안

```kotlin
@Composable
fun GoodList(items: List<Item>) {
    val showButton by remember {
        derivedStateOf { items.size > 10 }
    }
    // showButton 값이 실제로 바뀔 때만 Recomposition
}
```

### 3. Stable/Immutable 어노테이션

```kotlin
// Compose 컴파일러가 안정성을 추론할 수 없는 경우
@Stable
class UserState(
    val name: String,
    val onNameChange: (String) -> Unit
)

@Immutable
data class UiModel(
    val id: Long,
    val title: String,
    val items: List<String> // List는 기본적으로 unstable
)
```

### 4. key() 사용

#### 위반 사례

```kotlin
items.forEach { item ->
    ItemRow(item)
}
```

#### 개선안

```kotlin
items.forEach { item ->
    key(item.id) {
        ItemRow(item)
    }
}
```

변경된 항목만 Recomposition된다.

### 5. remember로 람다 안정화

#### 위반 사례

```kotlin
@Composable
fun Parent(viewModel: ViewModel) {
    Child(onClick = { viewModel.onClick() }) // 매번 새 인스턴스
}
```

#### 개선안

```kotlin
@Composable
fun Parent(viewModel: ViewModel) {
    val onClick = remember { { viewModel.onClick() } }
    Child(onClick = onClick)
}
```

## Recomposition 확인 방법

```kotlin
// Layout Inspector 사용
// Android Studio > Layout Inspector > Show Recomposition Counts

// 코드로 확인
@Composable
fun DebugRecomposition(tag: String) {
    val recomposeCount = remember { mutableIntStateOf(0) }
    SideEffect { recomposeCount.intValue++ }
    Log.d("Recomposition", "$tag: ${recomposeCount.intValue}")
}
```

## 면접 포인트

- **Q: Compose에서 Recomposition이 언제 발생하나요?**
- A: State가 변경되고, 그 State를 읽는 Composable이 있을 때 발생한다. Compose 컴파일러는 어떤 Composable이 어떤 State를 읽는지 추적하여 필요한 부분만 다시 실행한다.

- **Q: 불필요한 Recomposition을 어떻게 방지하나요?**
- A: State 읽기 지연(람다로 전달), derivedStateOf로 파생 상태 최적화, @Stable/@Immutable 어노테이션으로 안정성 명시, key()로 리스트 아이템 식별이 있다.

- **Q: derivedStateOf는 언제 사용하나요?**
- A: State에서 계산된 값이 원본 State보다 덜 자주 변경될 때 사용한다. 예: 리스트 크기가 10 이상인지 체크할 때, 리스트가 변경되어도 결과(true/false)가 같으면 Recomposition을 건너뛴다.

## 참고

- [Jetpack Compose Performance](https://developer.android.com/jetpack/compose/performance)
- [Compose Compiler Metrics](https://github.com/androidx/androidx/blob/androidx-main/compose/compiler/design/compiler-metrics.md)
