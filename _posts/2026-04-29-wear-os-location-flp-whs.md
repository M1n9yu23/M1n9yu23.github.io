---
title: "Wear OS 위치 정보 최적화: FLP와 WHS의 전략적 활용"
date: 2026-04-29 00:00:00 +0900
categories: [Android, Wear OS]
tags: [Android, Wear OS, Location, Health Services, FLP]
description: "Wear OS에서 전력 효율과 정확도를 모두 잡기 위해 통합 위치 정보 제공자(FLP)와 건강 관리 서비스(WHS)를 어떻게 조합해야 하는지 정리합니다."
---

Wear OS에서 위치 정보를 다루는 일은 스마트폰과 분명한 차이가 있습니다. 손목 위 디바이스의 작은 배터리 용량이 어떤 위치 API를 선택하느냐에 따라 사용자 경험과 기기 유지 시간을 좌우하기 때문입니다.

Wear OS에서 위치를 얻는 경로는 크게 두 가지입니다. **통합 위치 정보 제공자(FLP)**와 **Wear 건강 관리 서비스(WHS)**. 두 API는 경쟁 관계가 아니라 서로 다른 시점을 책임지는 보완 관계에 가깝습니다. 이 글은 두 API의 차이를 정리하고, 운동 추적 앱에서 둘을 어떻게 조합하는지 살펴봅니다.

## 1. FLP와 WHS의 역할 구분

#### 통합 위치 정보 제공자 (FLP)
Google Play Services에서 제공하는 컴포넌트로, GPS와 Wi-Fi, 셀 신호를 조합해 위치를 산출합니다. 캐시된 위치를 빠르게 반환할 수 있어 단발성 좌표 조회나 짧은 추적에 적합합니다.

#### Wear 건강 관리 서비스 (WHS)
Wear OS 디바이스의 저전력 센서 허브를 활용해 위치, 거리, 심박수 등 운동 관련 데이터를 일괄적으로 수신합니다. 운동 세션 안에서 동작하므로 장시간 추적과 센서 간 시점 동기화가 필요한 시나리오에 적합합니다.

| 비교 항목 | FLP | WHS |
| ---- | ---- | ---- |
| 동작 방식 | Play Services 기반 위치 산출 | 운동 세션 + 센서 허브 |
| 장점 | 빠른 초기 응답, 범용성 | 저전력, 다중 센서 동기화 |
| 한계 | 장시간 추적 시 전력 부담 | 첫 위치 수신까지 워밍업 시간 필요 |
| 적합한 시나리오 | 진입 직후 위치 표시, 짧은 추적 | 운동 세션 중 지속 추적 |

## 2. FLP: 빠른 초기 위치 획득

운동 화면 진입 직후의 인상은 첫 좌표가 지도에 찍히는 속도가 결정합니다. WHS 세션이 GPS를 워밍업하는 동안 사용자가 빈 지도를 응시하지 않도록, FLP의 단발성 호출로 좌표를 즉시 확보해 두는 방식이 유리합니다.

```kotlin
val token = CancellationTokenSource()

fusedLocationClient.getCurrentLocation(
    Priority.PRIORITY_HIGH_ACCURACY,
    token.token,
).addOnSuccessListener { location: Location? ->
    location?.let { /* 지도 시작점으로 사용 */ }
}

// 화면 종료 또는 WHS 위치 수신 시
token.cancel()
```

`getCurrentLocation`은 한 번의 콜백으로 종료되지만, 호출 직후 화면이 닫힐 가능성이 있다면 `CancellationTokenSource`로 명시적으로 취소해 GPS 칩이 불필요하게 깨어 있지 않도록 정리해야 합니다.

## 3. WHS: 운동 세션의 지속 추적

장시간 운동 추적은 WHS의 영역입니다. 하드웨어 센서 허브에서 데이터를 일괄 처리해 전달하기 때문에, 동일 시간 동안 FLP를 직접 사용하는 것보다 전력 소모가 크게 줄어듭니다.

#### 매니페스트 설정

운동 추적은 ForegroundService 안에서 수행되며, 위치 데이터를 함께 다룬다면 서비스 타입에 `health`와 `location`을 같이 지정해야 합니다.

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_HEALTH" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />

<service
    android:name=".exercise.ExerciseService"
    android:foregroundServiceType="health|location"
    android:exported="false" />
```

#### 운동 세션 설정과 시작

`prepareExerciseAsync`는 GPS와 심박 센서를 미리 깨워 본 운동이 시작될 때 첫 위치 수신 지연을 줄여 주는 단계입니다.

```kotlin
val warmUpConfig = WarmUpConfig(
    exerciseType = ExerciseType.RUNNING,
    dataTypes = setOf(DataType.LOCATION),
)
exerciseClient.prepareExerciseAsync(warmUpConfig).await()

val config = ExerciseConfig(
    exerciseType = ExerciseType.RUNNING,
    dataTypes = setOf(DataType.LOCATION, DataType.DISTANCE),
    isAutoPauseAndResumeEnabled = true,
    isGpsEnabled = true,
)
exerciseClient.startExerciseAsync(config).await()
```

#### 위치 데이터 수신

운동 데이터 콜백에서는 위치 포인트와 가용성 상태를 함께 다룹니다.

```kotlin
exerciseClient.setUpdateCallback(object : ExerciseUpdateCallback {
    override fun onExerciseUpdateReceived(update: ExerciseUpdate) {
        update.latestMetrics
            .getData(DataType.LOCATION)
            .forEach { dataPoint ->
                val location: LocationData = dataPoint.value
                // location.latitude, location.longitude
            }
    }

    override fun onAvailabilityChanged(
        dataType: DataType<*, *>,
        availability: Availability,
    ) {
        if (availability is LocationAvailability) {
            when (availability) {
                LocationAvailability.ACQUIRING -> { ... }
                LocationAvailability.ACQUIRED_TETHERED,
                LocationAvailability.ACQUIRED_UNTETHERED -> { ... }
                else -> { ... }
            }
        }
    }

    // onLapSummaryReceived, onRegistered, onRegistrationFailed 생략
})
```

`LocationAvailability`의 상태 전이를 UI에 노출하면, 사용자가 첫 좌표 수신 전 대기 시간을 자연스럽게 인지할 수 있어 체감 응답성이 좋아집니다.

## 4. 하이브리드 전략

운동 추적 앱에서는 두 API의 강점을 시점별로 나누어 쓰는 구조가 안정적입니다.

| 시점 | 담당 API | 역할 |
| ---- | ---- | ---- |
| 운동 화면 진입 | FLP | 캐시된 좌표로 지도 즉시 초기화 |
| 운동 준비 | WHS `prepareExerciseAsync` | GPS·심박 센서 워밍업 |
| 운동 진행 | WHS `ExerciseUpdateCallback` | 저전력 위치/거리 수신 |
| 운동 종료 | WHS `endExerciseAsync` | 세션 정리, FLP는 cancel |

FLP에서 얻은 첫 좌표는 WHS의 첫 위치 수신이 도착하기 전까지의 시각적 공백을 메우는 앵커 역할을 합니다. WHS 콜백에 `ACQUIRED_*` 상태가 도착한 이후로는 FLP 좌표를 더 이상 참조할 필요가 없으므로, `CancellationTokenSource.cancel()`로 정리해 GPS가 이중으로 깨어 있지 않도록 합니다.

## 마무리

Wear OS의 위치 추적은 정확도와 응답성, 그리고 배터리라는 세 변수의 균형을 맞추는 작업입니다. FLP가 진입 직후의 응답성을 책임지고, WHS가 운동 세션 내내 효율성을 책임지는 분담 구조는 세 변수를 함께 끌어안을 수 있는 안정적인 패턴입니다. 새로 운동 추적 기능을 설계하고 있다면 이 분담 구조를 출발점으로 삼아 보시길 권합니다.
