---
title: "Navigation3 가이드"
date: 2026-03-31 00:00:00 +0900
categories: [Android, Navigation 3]
tags: [Android, Jetpack Compose, Navigation 3]
description: "Navigation3을 활용해 Compose 환경에서 백스택을 완전히 제어하고 직관적인 상태 기반 라우팅을 설계하는 모범 패턴을 다룹니다."
---

Navigation 3는 Compose 환경에서 화면 이동 패러다임을 완전히 뒤바꾼 라이브러리입니다. 기존의 블랙박스 형태였던 라우팅 로직을 버리고, 개발자가 직접 **화면 상태(State)**를 소유하고 관리하는 매끄럽고 선언적인 방식으로 진화했습니다.

단순한 버전업을 넘어 아키텍처의 전환이 이루어진 Navigation 3의 핵심 철학과, 처음부터 끝까지 실전 프로젝트에 적용할 수 있는 단계별 모범 패턴을 예제와 함께 정리했습니다.

## 왜 Navigation 3여야 하는가?

Navigation 3는 Compose 네이티브한 아키텍처를 지향하며 다음과 같은 강점을 제공합니다:

1. **백스택의 소유권 이동**: 내부 프레임워크가 숨기고 있던 백스택이 코드 위로 올라왔습니다. `SnapshotStateList` 구조에 데이터를 추가하거나 제거하면 화면이 알아서 렌더링됩니다.
2. **강력한 스코프 격리**: 화면 항목 단위로 생명주기가 철저하게 관리되므로, 특정 화면이 스택에서 빠지는 순간 연관된 ViewModel 역시 깔끔하게 소멸됩니다.
3. **타입 안정성과 확장성 보장**: 별도의 런타임 검사나 직렬화 플러그인에 맞출 필요 없이, 순수 코틀린 `@Serializable` 데이터 구조로 목적지를 명확하게 정의합니다.
4. **반응형 다중 레이아웃 기능**: 큰 화면에서 여러 라우트를 분할 화면으로 띄울 수 있는 `Scenes API`를 기본 제공합니다.

## 1. 개발 환경 설정 가이드

기존 프로젝트에 Navigation 3 라이브러리를 추가하려면 `libs.versions.toml`에 다음 내용을 추가합니다.

```toml
[versions]
nav3Core = "1.0.1"
lifecycleViewmodelNav3 = "2.11.0-alpha03"
kotlinSerialization = "2.2.21"
kotlinxSerializationCore = "1.9.0"
material3AdaptiveNav3 = "1.3.0-alpha09"
compileSdk = "36"

[libraries]
androidx-navigation3-runtime = { module = "androidx.navigation3:navigation3-runtime", version.ref = "nav3Core" }
androidx-navigation3-ui = { module = "androidx.navigation3:navigation3-ui", version.ref = "nav3Core" }
androidx-lifecycle-viewmodel-navigation3 = { module = "androidx.lifecycle:lifecycle-viewmodel-navigation3", version.ref = "lifecycleViewmodelNav3" }
kotlinx-serialization-core = { module = "org.jetbrains.kotlinx:kotlinx-serialization-core", version.ref = "kotlinxSerializationCore" }
androidx-material3-adaptive-navigation3 = { group = "androidx.compose.material3.adaptive", name = "adaptive-navigation3", version.ref = "material3AdaptiveNav3" }

[plugins]
jetbrains-kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlinSerialization"}
```

```kotlin
plugins {
    alias(libs.plugins.jetbrains.kotlin.serialization)
}

dependencies {
    implementation(libs.androidx.navigation3.ui)
    implementation(libs.androidx.navigation3.runtime)
    implementation(libs.androidx.lifecycle.viewmodel.navigation3)
    implementation(libs.androidx.material3.adaptive.navigation3)
    implementation(libs.kotlinx.serialization.core)
}
```

## 2. 화면 목적지 키(Key) 설계

대규모 프로젝트에서는 화면 구현체와 라우팅 식별자를 물리적으로 분리하는 방식이 권장됩니다.

컴포넌트를 호출할 때 타입 세이프(Type-Safe)를 다루기 위해 모든 목적지를 `NavKey` 인터페이스를 구현하는 타입으로 규정합니다.

```kotlin
@Serializable
data object ExpenseListKey : NavKey

@Serializable
data class ExpenseDetailKey(val expenseId: String) : NavKey

@Serializable
data class ExpenseSettingsKey(val darkTheme: Boolean = false) : NavKey
```

> **설계 팁**: 데이터를 전달할 때는 기본 타입이나 ID 값 같은 고유 식별자를 넘기는 것이 유리합니다. 거대한 도메인 객체를 담게 되면 직렬화에 부담이 생길 수 있습니다.

## 3. 백스택 제어와 AppState 래퍼

라우팅을 상태 리스트 조작의 관점으로 다룸으로써, 여러 곳에 퍼진 로직을 모아 줄 수 있습니다. 모범적인 아키텍처 패턴(Now In Android 참고)에서는 백스택을 직접 조작하는 `NavigationState` 객체와 앱 전체의 상태를 아우르는 State Holder(`TrackerAppState`)를 깔끔하게 분리하여 관리합니다.

먼저 순수하게 네비게이션 백스택에 대한 액션(`navigate`, `goBack` 등)만 캡슐화하는 별도의 `NavigationState`를 정의합니다.

```kotlin
@Stable
class NavigationState(
    val currentBackStack: MutableList<NavKey>
) {
    fun navigate(key: NavKey) {
        currentBackStack.add(key)
    }

    fun goBack() {
        if (currentBackStack.size > 1) {
            currentBackStack.removeLastOrNull()
        }
    }
}

@Composable
fun rememberNavigationState(
    currentBackStack: MutableList<NavKey> = rememberNavBackStack(ExpenseListKey)
): NavigationState = remember(currentBackStack) {
    NavigationState(currentBackStack)
}
```

이후 화면 크기 변화(`windowAdaptiveInfo`)나 코루틴 등 앱의 다른 전역 환경과 함께 `NavigationState`를 하나로 묶어주는 래퍼(`TrackerAppState`)를 구축합니다.

```kotlin
@Stable
class TrackerAppState(
    val navigationState: NavigationState,
    val coroutineScope: CoroutineScope,
    val windowAdaptiveInfo: WindowAdaptiveInfo
) {
    fun navigateToTopLevel(targetKey: NavKey) {
        if (navigationState.currentBackStack.lastOrNull() != targetKey) {
            navigationState.currentBackStack.clear()
            navigationState.navigate(targetKey)
        }
    }
}

@Composable
fun rememberTrackerAppState(
    navigationState: NavigationState = rememberNavigationState(),
    coroutineScope: CoroutineScope = rememberCoroutineScope(),
    windowAdaptiveInfo: WindowAdaptiveInfo = currentWindowAdaptiveInfo()
): TrackerAppState = remember(navigationState, coroutineScope, windowAdaptiveInfo) {
    TrackerAppState(navigationState, coroutineScope, windowAdaptiveInfo)
}
```

## 4. UI 렌더링: NavDisplay 패턴 연결

준비된 상태(`App State` 안의 라우트 목록)를 시각적인 화면과 동기화해주는 엔진이 `NavDisplay`입니다. 백스택의 최상단 요소를 인식해 그에 맞는 컴포저블 화면을 새로 그립니다.

프로덕션 수준에서는 화면별 스코프 관리를 위해 `entryDecorators` 속성에 **두 가지 핵심 데코레이터**를 제공하는 것이 좋습니다.

```kotlin
@Composable
fun TrackerAppEntry() {
    val appState = rememberTrackerAppState()

    NavDisplay(
        backStack = appState.navigationState.currentBackStack,
        onBack = { appState.navigationState.goBack() },
        entryDecorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator(),
            rememberViewModelStoreNavEntryDecorator()
        ),
        entryProvider = entryProvider {
            entry<ExpenseListKey> {
                ExpenseListScreen(
                    onItemClick = { id -> 
                        appState.navigationState.navigate(ExpenseDetailKey(id)) 
                    },
                    onSettingsClick = { 
                        appState.navigationState.navigate(ExpenseSettingsKey()) 
                    }
                )
            }

            entry<ExpenseDetailKey> { key ->
                val detailViewModel: ExpenseDetailViewModel = hiltViewModel()
                
                ExpenseDetailScreen(
                    expenseId = key.expenseId,
                    viewModel = detailViewModel,
                    onBackClick = { appState.navigationState.goBack() }
                )
            }
            
            entry<ExpenseSettingsKey>(
                metadata = NavDisplay.transitionSpec {
                    slideInVertically { it } togetherWith slideOutVertically { -it }
                }
            ) { key ->
                ExpenseSettingsScreen(
                    isDarkTheme = key.darkTheme,
                    onNavigateBack = { appState.navigationState.goBack() }
                )
            }
        }
    )
}
```

## 5. 고급 전략

단순 라우팅 외에도 앱 구조를 확장하기 좋은 응용 전략들을 소개합니다.

#### A. DeepLink (딥링크) 대응
순수 코틀린 `@Serializable` 규칙을 사용하므로 외부 URL의 파라미터가 쉽게 매핑됩니다. 긴 파싱 로직을 추가로 둘 필요 없이 바로 시스템 내부에서 타입이 확정된 객체로 통신하게 됩니다.

#### B. 파싱이 필요 없는 직관적인 파라미터 전달
이전 세대에서는 ViewModel 안에서 전달된 파라미터를 꺼내기 위해 `SavedStateHandle`을 조회해야 했지만, Navigation 3에서는 그 방식이 눈에 띄게 간소화되었습니다. `entryProvider`가 뷰를 그리는 시점에 이미 타입이 완벽히 확정된 `NavKey` 인스턴스 자체를 람다의 인자로 넘겨주므로, 구시대적인 런타임 캐스팅이나 복잡한 파싱 없이 팩토리나 `Hilt` 주입 단계에 곧바로 안전한 파라미터를 꽂아 넣을 수 있습니다.

#### C. 멀티 Pane 대응 (Scenes API) 활용
폴더블 기기나 태블릿 화면 구축 시 Navigation 3의 뛰어난 생태계 기능을 살릴 수 있습니다.
화면을 쪼개 백스택의 목적지를 동시 출력하는 `ListDetailSceneStrategy`나 다이얼로그 계층을 지원하는 `DialogSceneStrategy`가 네이티브 차원으로 탑재되어 있어 반응형 레이아웃 설계에 유리합니다.

## 마무리

Navigation 3는 완전히 Compose 친화적인 프레임워크로 구성됐습니다. 개발자가 닫혀 있는 컨테이너 객체에서 벗어나 모든 백스택 구성을 직접 소유하고 제어하게 됨으로써 선언형 UI가 갖는 본연의 이점을 온전하게 누릴 수 있습니다.

AppState 관리 패턴 및 공간 변환을 직관적으로 다루는 네비게이션 전략을 통해, 프로젝트에 한층 더 유연하고 유지보수가 쉬운 환경을 도입해 보시길 바랍니다.

## Learn More

본문에서 다룬 내용은 Navigation 3의 핵심 구조와 패턴입니다. 여기서 한 발 더 나아가, 실전 프로젝트에서 마주하게 될 심화 주제들을 소개합니다.

1. `hiltViewModel`의 `creationCallback`을 활용하면 `NavKey`의 파라미터를 ViewModel 생성 시점에 직접 주입할 수 있습니다. 본문의 `entry<ExpenseDetailKey>` 블록에서 `key.expenseId`를 ViewModel에 어떻게 전달할 수 있을지 찾아서 적용해 보세요.

2. Navigation 3는 `NavDisplay`의 `transitionSpec`, `popTransitionSpec`, `predictivePopTransitionSpec` 파라미터를 통해 글로벌 수준의 화면 전환 애니메이션과 예측형 뒤로가기 제스처를 지원합니다. 특정 화면에만 적용하는 `metadata` 방식과 글로벌 방식을 비교해 보세요.

3. `rememberListDetailSceneStrategy`와 `DialogSceneStrategy`를 사용하면 태블릿이나 폴더블 기기에서 여러 목적지를 동시에 렌더링할 수 있습니다. Scenes API를 활용한 적응형 레이아웃 구현을 직접 시도해 보세요.

4. 멀티 모듈 아키텍처에서는 `NavKey`를 `:feature:api` 모듈에 단독 배치하고, UI와 ViewModel은 `:feature:impl` 모듈에 격리하여 의존성을 단방향으로 제한하는 것이 권장됩니다. 이유를 찾아보세요.

5. 프로덕션 환경에서 자주 발생하는 실수들을 미리 인지해 두세요.
    - `rememberSaveableStateHolderNavEntryDecorator()`와 `rememberViewModelStoreNavEntryDecorator()` 데코레이터를 누락하면 화면 상태와 ViewModel 스코프가 정상적으로 관리되지 않습니다.
    - 코루틴의 비동기 컨텍스트에서 백스택을 직접 조작하면 예기치 않은 오류가 발생할 수 있으므로, 반드시 `Dispatchers.Main`에서 수행해야 합니다.
    - `backStack.removeLastOrNull()` 호출 전에 항상 `backStack.size > 1` 여부를 확인하여, 마지막 항목까지 제거되어 빈 화면이 되는 상황을 방지해야 합니다.
