---
title: "Kotlin: Value Class로 Type Safety와 성능 동시에 잡기"
date: 2026-01-23 00:00:00 +0900
categories: [Kotlin]
tags: [Kotlin]
description: "기본 타입을 그대로 사용하면 성능은 좋지만 타입 안정성이 떨어지고, 객체로 감싸면 힙 메모리 할당 비용이 발생합니다. Value Class를 통해 해결하는 방법을 소개합니다."
---

## Primitive Obsession

개발을 하다 보면 도메인의 특정 값을 표현하기 위해 `Int`, `Double`, `String`같은 기본 타입을 습관적으로 사용하곤 합니다.

예를 들어, 환자의 활력징후를 기록하는 함수를 살펴봅시다. 심박수와 산소 포화도는 둘 다 정수 타입의 데이터를 가집니다.

```kotlin
fun recordVitalSign(heartRateBpm: Int, spO2Percent: Int) {
    ...
}

fun main() {
    val currentHeartRate = 95
    val currentSpO2 = 98

    recordVitalSign(currentSpO2, currentHeartRate)
}
```

위 코드의 문제점은 명확합니다. `recordVitalSign`함수를 호출할 때 인자의 순서를 실수로 바꿔서 넣었습니다. 심박수 자리에 산소 포화도를, 산소 포화도 자리에 심박수를 넣은 것입니다.

하지만 컴파일러는 이를 막아주지 못하죠. 둘 다 `Int`타입이기 때문입니다. 이런 실수는 런타임에 심각한 상황을 초래할 수 있습니다.

### 지명 인자로 해결할 수 있지 않나요?

코틀린의 지명 인자를 사용하면 해결된다고 생각할 수 있습니다.

```kotlin
recordVitalSign(
    heartRateBpm = currentSpO2, 
    spO2Percent = currentHeartRate
)
```

지명 인자는 **가독성**을 높여줄 뿐 **타입 안정성**을 보장하지 않습니다. 위 코드처럼 실수로 잘못된 변수를 던져줘도, 매개변수의 타입과 일치하기 때문에 컴파일러는 알맞은 로직인 줄 압니다. 지금 필요한 것은 "심박수 자리에 산소 포화도를 넣으면 컴파일 에러가 나는 구조"입니다.

## The Cost of Wrapper Classes

타입 안정성을 확보하는 정석적인 방법은 각각을 클래스로 감싸는 것입니다.

```kotlin
data class HeartRate(val value: Int)
data class SpO2(val value: Int)
```

이렇게 하면 타입은 안전해지지만, 성능 비용이 발생합니다. `Int`는 스택 영역에서 효율적으로 처리되는 기본 타입인 반면, 일반 클래스는 힙 영역에서 객체를 할당하고 GC의 대상이 됩니다. 성능에 민감한 루프 안이나 대용량 데이터 처리 시에는 이러한 오버헤드가 부담이 될 수 있습니다.

## 해결: Inline Value Classes

`value class`는 객체의 타입 안정성과 기본 타입의 성능을 모두 제공합니다.<br>
`JvmInline`애노테이션과 `value`키워드를 사용하여 정의합니다.

```kotlin
@JvmInline
value class HeartRate(val bpm: Int) {
    init {
        require(bpm > 0) { "심박수는 0보다 커야 합니다." }
    }

    val isTachycardia: Boolean
        get() = bpm > 100
}

@JvmInline
value class SpO2(val percent: Int) {
    init {
        require(percent in 0..100) { "산소 포화도는 0과 100 사이여야 합니다." }
    }
}

fun recordVitalSign(heartRate: HeartRate, spO2: SpO2) {
    ...
}
```

이제 `recordVitalSign`을 호출할 때 잘못된 데이터(heartRate 자리에 SpO2 data)를 넘기면 컴파일 에러가 발생하여 실수를 방지할 수 있습니다.

### 특징 및 제약 사항

1. **단일 속성**: 주 생성자에는 반드시 `val`로 선언된 하나의 불변 속성만 가져야 합니다.
2. **백킹 필드 없음**: 클래스 내부의 속성은 계산된 속성(Computed property)만 가능하며, 상태를 저장하는 별도의 필드를 가질 수 없습니다. 
3. **상속 불가**: 다른 클래스를 상속받을 수 없으며, `final`클래스로 취급됩니다. (단, 인터페이스 구현은 가능.)
4. **식별자 없음**: 식별자가 없으며 값만 저장할 수 있습니다.

## 내부 동작 검증

`value class`가 정말로 객체를 생성하지 않는지 궁금하여 직접 바이트코드를 확인해 보았습니다. 코틀린 컴파일러는 가능한 경우 `value class`를 내부의 기본 타입으로 치환합니다.

즉, 위의 `HeartRate`클래스는 런타임에 객체가 생성되지 않고, 단순한 `int`(Java)로 취급됩니다.

```java
public static final void recordVitalSign(int heartRate, int spO2) {
    ...
}
```

코드상으로는 `HeartRate`라는 타입을 사용했지만, 실제 런타임에는 기본 타입 `int`로 동작합니다. 이것이 바로 Zero-Cost Abstraction 입니다. `inline`이라는 이름처럼 호출 지점에 값이 직접 인라인 됩니다.

### Mangling

여기서 흥미로운 문제가 발생합니다. 만약 다음과 같이 오버로딩된 함수들이 있다면 어떨까요?

```kotlin
fun analyze(data: HeartRate) {
    ...
}

fun analyze(data: SpO2) { 
    ...  
}
```

런타임에 두 클래스 모두 `int`로 변환된다면, JVM 시그니처 상으로는 `analyze(int)`라는 똑같은 함수가 두 개 생기는 충돌이 발생합니다.

이를 방지하기 위해 코틀린 컴파일러는 맹글링 기법을 사용합니다. 함수 이름 뒤에 해시코드를 붙여 고유한 식별자를 만듭니다. 

```java
public static final void analyze-YJ1PKq0(int data)
public static final void analyze-LlWpVQZ(int data)
```

이 때문에 자바 코드에서 코틀린의 `value class`를 사용하는 함수를 호출하려면 `@JvmName`을 통해 명시적으로 이름을 지정해 주어야 합니다.

## Boxing vs Unboxing

항상 기본 타입으로 최적화되는 것은 아닙니다. 상황에 따라 `Integer`처럼 래퍼 클래스로 감싸지는 **박싱**이 발생합니다.

다음과 같은 경우 박싱이 일어납니다.
1. **Null 가능한 타입으로 사용될 때**: `HeartRate?`
2. **제네릭 타입으로 사용될 때**: 제네릭 타입에 전달될 때
3. **인터페이스로 다뤄질 때**: 인터페이스를 구현하고 그 타입으로 넘길 때

```kotlin
interface Vitals

@JvmInline
value class Temperature(val celsius: Double) : Vitals

fun <T> processGeneric(item: T) { ... }
fun processInterface(item: Vitals) { ... }
fun processNullable(item: Temperature?) { ... }
fun processInline(item: Temperature) { ... }

fun main() {
    val temp = Temperature(36.5)

    processInline(temp)
    
    processGeneric(temp)
    processInterface(temp)
    processNullable(temp)
}
```

## Value Class vs Type Alias

`typealias`는 기존 타입에 별칭만 붙일 뿐, 새로운 타입을 만드는 것이 아닙니다. 따라서 컴파일러 입장에서 `typealias`는 원본 타입과 완전히 호환됩니다.

```kotlin
typealias Milligrams = Int
typealias Milliliters = Int

fun mixDrug(amount: Milliliters) { ... }

fun main() {
    val powder: Milligrams = 100
    mixDrug(powder)
}
```

반면 `value class`는 완전히 새로운 타입을 정의하여, 섞어 쓰는 것을 컴파일 타임에 차단합니다.

## 결론

ID, Password, 좌표 등 도메인 특화된 값을 다룰 때 `value class` 도입을 적극 고려해보세요. 런타임 성능 저하 없이 강력한 타입 안정성을 확보할 수 있습니다.

## Learn More

더 나은 개발 여정을 원한다면 다음 주제들을 스스로 탐구해보시길 권합니다.

1. **직렬화와 Json:**

    실무에서 아직 많이 쓰이는 Gson은 리플렉션 기반이라 Value Class를 제대로 인지하지 못하고 객체로 감싸거나 맹글링된 필드명을 그대로 내보냅니다.
    Moshi에선 이를 어떻게 처리하는지, 그리고 kotlinx.serialization이 왜 가장 적합한지 비교 분석해보세요.

2. **Room & Parcelable:**

    과거에는 Room에 저장하기 위해 번거로운 TypeConverter가 필수였지만 현재는 KSP에서 value class가 지원됩니다.
    또한, Parcelize를 적용했을 때 IPC 통신 과정에서 객체가 생성되는지, 아니면 기본 타입만 전달되어 오버헤드가 제거되는지 확인해보세요.

3. **Mocking:**

    Value Class는 final이며 컴파일 시점에 정적 메서드로 치환되기 때문에, Mockito 같은 프록시 기반 모킹 라이브러리로는 테스트 대역을 만들기 어렵습니다.

    이러한 제약이 오히려 값 객체는 모킹하는 것이 아니라 실제 값을 사용해야 한다는 테스트 설계 원칙과 어떻게 연결되는지 고민해보세요.

4. **상황에 맞게:**

    Value Class는 만능이 아닙니다. 모든 상황엔 그에 맞는 코드와 문법이 있습니다. 자신만의 명확한 기준을 세우고, 장단점을 이해하여 Value Class /  Data Class / Class 등을 자신의 기준에 알맞게 사용해보세요.

5. **기본 & 참조**

    코틀린에선 모든 타입이 참조 타입인데 왜 기본 타입이라고 제가 표현했을까요? 내부적으로 어떻게 돌아가는지 확인해보세요.