---
title: "Android Testing: Mocks vs Fakes"
date: 2026-02-08 00:00:00 +0900
categories: [Android, Test]
tags: [Android, Test]
description: "테스트가 구현 세부 사항에 결합되면 리팩토링의 걸림돌이 됩니다. 명시적으로 권장되는 'Fakes'를 사용하여, 깨지기 쉬운 테스트를 견고한 'State-based Testing'으로 전환하는 방법을 다룹니다.'"
---

## 1. 행위 검증의 한계.

많은 안드로이드 프로젝트에서 단위 테스트가 유지보수의 대상이 아닌 '걸림돌'로 전락하는 주된 원인은 **테스트가 구현 세부 사항에 강하게 결합되어 있기 때문**입니다.

흔히 사용되는 Mocking Framework를 이용한 테스트 코드를 살펴보겠습니다.

```kotlin
@Test
fun loadData_delegatesToRepository() {
    Mockito.`when`(userRepository.fetchUser(anyString()))
        .thenReturn(flowOf(Result.success(User("Mingyu"))))

    viewModel.loadData()
    
    verify(userRepository, times(1)).fetchUser("Mingyu")
}
```

이러한 테스트 방식은 다음의 문제점을 가집니다:

1. **깨지기 쉬움**: 비즈니스 로직은 정상이어도, 내부 구현이 변경되면 테스트가 실패합니다.
2. **낮은 충실도**: 실제 객체의 로직을 수행하지 않고 흉내 내므로, 실제 환경에서의 동작을 완벽히 보장하지 못합니다.
3. **성능 비용**: Mockito는 런타임에 리플렉션을 사용하므로, 순수 Kotlin 객체보다 실행 속도가 느립니다.

## 2. Fakes over Mocks

Android 공식 문서의 'Test Doubles' 섹션에서는 Fakes를 우선적으로 사용할 것을 명시하고 있습니다.

**Fake**란 프로덕션에는 적합하지 않지만 테스트에는 적합한(In-Memory DB) **Test Double**입니다.

내부 로직은 `List`, `Map`등을 사용하여 단순하고 빠르게 동작합니다.

## 3. 구현 가이드

Fakes를 사용하면 '함수가 호출되었는가'가 아닌 **시스템의 상태가 기대한 대로 변경되었는가**를 검증할 수 있습니다. 

사용자 데이터를 다루는 예시로 살펴보겠습니다.

#### Step 1: 인터페이스 정의
테스트와 프로덕션 코드가 공유할 인터페이스를 정의합니다.

```kotlin
interface UserRepository {
    fun getUser(userId: String): Flow<Result<User>>
    suspend fun saveUser(user: User)
}
```

#### Step 2: Fake 구현 (In-Memory)

`test` 또는 `testFixtures` 소스 셋에 위치하며, 복잡한 DB나 네트워크 의존성 없이 메모리상에서 동작합니다.

```kotlin
class FakeUserRepository : UserRepository {
    private val _users = MutableStateFlow<List<User>>(emptyList())

    override fun getUser(userId: String): Flow<Result<User>> {
        return _users.map { list ->
            list.find { it.id == userId }
                ?.let { Result.success(it) }
                ?: Result.failure(NoSuchElementException("No User"))
        }
    }

    override suspend fun saveUser(user: User) {
        _users.update { currentList ->
            currentList.filterNot { it.id == user.id } + user
        }
    }
    
    suspend fun emitUser(user: User) {
        saveUser(user)
    }
}
```

#### Step 3: 테스트 코드 작성

`when`이나 `verify`없이, 직관적인 로직 흐름과 상태 검증이 가능합니다.

```kotlin
@Test
fun loadUser_whenSuccess_updatesUiState() = runTest {
    val fakeRepo = FakeUserRepository()
    val testUser = User(id = "user_1", name = "Mingyu")
    fakeRepo.emitUser(testUser)
    
    val viewModel = UserViewModel(userRepository = fakeRepo)

    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem()) 
        
        viewModel.loadUser("user_1")
        
        val successState = awaitItem() as UiState.Success
        assertEquals("Mingyu", successState.data.name)
    }
}
```

## 4. 핵심 이점 분석

Fake 사용 시 얻을 수 있는 이점은 다음과 같다고 생각합니다.

1. **실행 속도**: Mocking 프레임워크의 초기화 및 리플렉션 등의 비용이 없으며, I/O 없이 메모리에서 동작하므로 실행 속도가 빠릅니다.
2. **높은 충실도**: 단순히 값을 반환하는 것이 아니라 실제 데이터 조작 로직을 수행하므로, 신뢰성을 확보할 수 있습니다.
3. **리팩토링**: 내부 구현이 변경되어도, 결과 상태가 동일하다면 테스트는 깨지지 않습니다.

## 5. Coroutines Test

비동기 로직을 테스트할 때는 `kotlinx-coroutines-test` 라이브러리의 표준 도구를 사용해야 합니다.

코루틴을 사용하는 단위 테스트 코드는 주의가 필요합니다. 비동기로 실행될 수 있고 여러 스레드에서 발생할 수 있기 때문입니다. 

#### 핵심 도구: `runTest`

`runTest`는 테스트 환경을 위한 코루틴 빌더입니다. 기존의 `runBlocking`과 달리 **가상 시간**을 사용하여 `delay()`와 같은 중단 함수를 실제 시간 대기 없이 즉시 처리합니다.

```kotlin
suspend fun fetchData(): String {
    delay(1000L)
    return "MinGyu!!"
}

@Test
fun fetchData_skipsDelay_inTest() = runTest {
    val data = fetchData() 
    assertEquals("MinGyu!!", data)
}
```

#### TestDispatchers: 실행 제어권 확보

테스트 내에서 새로운 코루틴이 생성될 때, 이들이 언제 실행될지 제어하기 위해 `TestDispatcher`를 사용합니다.

1. **StandardTestDispatcher (기본값):**

    - 새로운 코루틴을 스케줄러의 대기열에 넣습니다. 
    - `advanceUntilIdle()`이나 `runCurrent()`를 호출해야만 실행됩니다. 
    - 장점: 복잡한 비동기 흐름의 순서를 정밀하게 제어하고 검증할 수 있습니다. 

2. **UnconfinedTestDispatcher:**

    - 새로운 코루틴을 즉시 실행합니다.
    - 장점: 간단한 테스트에서 별도의 스케줄링 조작없이 코드를 작성할 수 있습니다.

#### Best Practice 1: Dispatcher 주입

프로덕션 코드에 `Dispatchers.IO`등을 하드코딩하면 테스트에서 제어할 수 없습니다. 

생성자를 통해 Dispatcher를 주입받도록 설계하여, 테스트 시에는 `TestDispatcher`로 교체해야 합니다.

```kotlin
// Bad
class Repository {
    fun load() = CoroutineScope(Dispatchers.IO).launch { ... }
}

// Good
class Repository(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend fun load() = withContext(ioDispatcher) { ... }
}
```

#### Best Practice 2: Main Dispatcher 교체

Local Unit Test는 안드로이드 기기가 아닌 JVM에서 실행되므로, 안드로이드의 `Main`스레드(UI 스레드)가 존재하지 않습니다. 따라서 `ViewModel` 등을 테스트할 때는 `Dispatchers.Main`을 `TestDispatcher`로 교체해야 합니다.

실제 UI 스레드가 제공되는 `Instrumented test`등 에서 Main 디스패처를 교체해서는 안 됩니다.

이를 매번 작성하는 대신, JUnit Rule로 만들어 재사용하는 것이 표준 패턴입니다.

```kotlin
class MainDispatcherRule(
    val testDispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

class HomeViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun init_loadsData_automatically() = runTest {
        val viewModel = HomeViewModel()
        
        // 검증 로직 ...
    }
}
```

## 결론

테스트 코드는 시스템의 무결성을 보장하는 가장 강력한 수단입니다. 가이드라인에 따라 **자신이 소유한 타입**에는 Mock 대신 Fake를 적용하세요. 

테스트는 단순히 버그를 찾는 수단이 아닙니다. 개발자가 해당 소프트웨어의 비즈니스 로직이나 작동 방식을 명확히 정의하는 과정이며, 리팩토링 시 발생할 수 있는 결함을 사전에 차단하는 가장 정교한 **안전장치**입니다.

결국 **테스트 가능한 구조가 곧 코드의 품질이며, 유지보수 가능한 아키텍처의 핵심**임을 잊지 마세요.

## Learn More

테스트는 앱 개발 프로세스에서 빼놓을 수 없습니다. 단위 테스트와 Fake를 통한 상태 검증에 익숙해졌다면, 이제 다음 단계인 UI 테스트(End-to-End)와 통합 테스트, 인스트루먼트 테스트 등으로 확장해보세요.

1. 안드로이드 테스트 피라미드 전략을 찾아보고 단위 테스트, 통합 테스트, UI 테스트의 비율을 확인하세요. <br>
또한, 그 전략을 프로젝트에 어떻게 적용할지 고민해 보세요.

2. 안드로이드의 다양한 테스트 방법을 찾아보고 자신의 프로젝트에 적용해보세요.

3. Espresso를 사용하여 Android UI Test를 작성해보세요. 뷰 기반 UI를 위해 설계되었지만 Compose test의 일부 측면에서는 여전히 유용합니다. 대신 Compose Test 프레임워크를 통해 Compose UI test도 해보세요.

4. 테스트 가능한 아키텍처는 가독성, 유지관리성, 확장성, 재사용성 등을 보장합니다.<br>
앱 아키텍처 가이드를 참고하여 프로젝트에 도입해보세요.

5. **Approaches to decoupling**: 함수, 클래스 또는 모듈의 일부를 나머지 부분에서 추출할 수 있다면 테스트가 더 쉽고 효과적입니다. 이 방법을 디커플링이라고 합니다. 일반적인 분리 기법은 다음과 같습니다.
    - 비즈니스 로직이 포함된 클래스에서 직접적인 프레임워크 종속성을 피해야 합니다.
    - 앱을 레이어로 분할하거나, 기능별로 분할할 수 있습니다.
    - ...

    이렇게 다양한 기법이 있는데 이 외에 어떤 기법이 있는지 스스로 깊이 탐구해 보시길 권합니다.

## 참고

- [LifeLog](https://github.com/M1n9yu23/LifeLog-Compose-Clean-Architecture)

- [android-test-code](https://github.com/M1n9yu23/android-test-code)