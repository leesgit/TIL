## lateinit vs lazy


### lateinit
lateinit은 늦은 초기화로서 선언과 동시에 초기화를 하지 않아도 되어 Null safety에 있어 자유로우며 var로만 선언 가능하다.

```
lateinit var text: String

fun main() {
    //println(text) 
    text = "hi"
    println(text)
}
```

Null safety로부터 자유롭기 때문에 초기화 전에 사용이 가능하여 실수에 위험이 존재한다.



### lazy
lazy는 람다를 이용하여 lazy의 인스턴스를 반환하는 함수로 처음 호출되었을 때 람다 함수 내에 코드들이 실행되며 값을 정의한다.


```
lateinit var text: String

val textLength: Int by lazy {
    println("$text computed!")
    text.length
}

fun main() {
    text = "hi"
    println(text)
    println(textLength)
    println(textLength)
}
```

출력
```
hi
hi computed!
2
2
```

위와 같이 textLength가 처음 호출 될 때만 hi computed!를 출력한다.
하지만 위와 같이 text의 값의 할당에 따라 textLength의 값이 정해지기 때문에 같은 스레드내에서 작업이 이루어 지는게 안전하다.


참고 
https://kotlinlang.org/docs/delegated-properties.html#lazy-properties