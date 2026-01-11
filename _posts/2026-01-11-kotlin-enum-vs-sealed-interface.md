---
title: "Kotlin: Enum의 한계와 Sealed Type을 통한 상태 모델링"
date: 2026-01-11 00:00:00 +0900
categories: [Kotlin]
tags: [Kotlin]
description: "Enum은 단순한 상태 식별자로는 훌륭하지만, 상태별로 서로 다른 데이터를 함께 표현하기에는 한계가 있습니다. Sealed Type을 사용하여 State와 Data를 안전하게 묶는 방법을 소개합니다."
---

## The "Fat Enum" Anti-Pattern

흔히 볼 수 있는, 속성이 비대해진 Enum 패턴입니다.

상태마다 필요한 속성이 다르지만, 이를 하나의 클래스에 무리하게 통합하려다 보니 모든 필드가 `Nullable`하고 `Mutable`해지면서, 타입 안정성이 크게 떨어집니다.

```kotlin
enum class MembershipStatus {
    GUEST,
    PREMIUM,
    BANNED;

    var expirationDate: Long? = null
    var banReason: String? = null
}

fun checkAccess(status: MembershipStatus) {
    if (status == MembershipStatus.PREMIUM) {
        val date = status.expirationDate ?: throw IllegalStateException("Date missing!")
    }
}
```

위 코드는 치명적인 문제가 있습니다.

1. `GUEST` 상태인데 `banReason`에 접근하거나, `PREMIUM`인데 `expirationDate`가 `null`인 논리적 오류를 허용합니다.
2. Enum은 싱글턴입니다. 만약 누군가 `PREMIUM.expirationDate`를 변경하면, 앱 내에서 `PREMIUM` 상태를 사용하는 모든 곳의 값이 바뀝니다. 이는 추적하기 힘든 버그를 만듭니다.

## The Idiomatic Way: Sealed Hierarchies

Kotlin의 `sealed interface` (또는 `sealed class`)를 사용하면, **State**와 **Data**를 하나의 타입으로 묶어 불가능한 상태를 컴파일 시점에 차단할 수 있습니다.

```kotlin
sealed interface MembershipState {

    data object Guest : MembershipState

    data class Premium(
        val id: String,
        val expirationDate: Long
    ) : MembershipState

    data class Banned(
        val reason: String
    ) : MembershipState
}
```

코드가 훨씬 명확해졌습니다. `expirationDate`는 오직 `Premium` 상태일 때만 존재하며, `reason`은 `Banned` 상태일 때만 존재합니다.

## Sealed Type이 제공하는 이점

이 패턴의 진가는 데이터를 소비하는 곳에서 드러납니다. 스마트 캐스팅 덕분에 불필요한 null 체크가 사라지고 코드가 직관적으로 변합니다.

```kotlin
fun getMessage(state: MembershipState): String {
    return when (state) {
        is MembershipState.Guest -> "Sign up now!"

        is MembershipState.Premium -> "Expires on ${state.expirationDate}"

        is MembershipState.Banned -> "You are banned because: ${state.reason}"
    }
}
```

## 결론
Enum은 요일이나 메뉴 옵션처럼 단순한 상수의 집합을 정의할 때 훌륭합니다. 하지만 Enum의 타입들이 서로 다른 속성을 가져야 한다면, 주저 없이 **Sealed Type**을 선택하세요.

**"유효하지 않은 상태는 아예 코드 레벨에서 표현할 수 없게 만들자."**<br>
이것은 방어적인 시스템을 설계할 때 기억하면 좋은 원칙입니다.

## Learn More

더 나은 개발 여정을 원한다면 다음 주제들을 스스로 탐구해보시길 권합니다.

1. **Sealed Class vs Sealed Interface:**

    두 방식은 비슷해 보이지만 설계 의도가 다릅니다. <br>
    언제 무엇을 선택해야 할지 자기만의 기준을 세워보세요. 
2. **상태 관리와 동시성:**

    어떤 상태나 속성이 `Immutable`이라고 해서 동시성 문제에서 완전히 자유로운 것은 아닙니다. <br>
    여러 스레드가 동시에 접근할 때 어떤 문제가 발생하는지, 그리고 이를 `Atomic`이나 `Mutex` 등으로 어떻게 방어할 수 있는지 찾아보세요.
3. **Android에서의 활용:**

    Sealed Type을 안드로이드 개발에서 어떻게 적용할지 생각해보세요. <br>
    ex) LCE Pattern , Result 등