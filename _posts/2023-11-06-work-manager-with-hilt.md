---
layout: post
title: '[Android, Kotlin] WorkManager 에 Hilt로 Dependency 주입하기'
categories: Android
tags: [Android, Kotlin, WorkManager]
comments: true
banner:
    image: "/assets/images/banners/workmanager_with_galaxy.png"
#     height: "100vh"
#     min_height: "38vh"
---

안드로이드 버전 12 부터 특정 상황을 제외하고는 Background 에서 [Foreground Service][android12] 사용이 제한되었고, 
안드로이드 버전 14 부터 [Foreground Service][android14]를 사용하기 위해선 service type 을 필수로 지정하고 
해당 service type을 사용하기 위해서는 권한 설정을 해주어야 작동하도록 변경되었습니다.

이런 제약과 함께 구글에서는 사용자와의 상호작용이 없는 `Foreground Service` 는 `WorkManager` 를 사용하도록 권고하고 있으며,
`WorkManager` 에서 즉시 실행 Worker 가 업데이트 되면서 대부분의 Backrground 작업은 [WorkManager][backgroundRecommend]를 사용하는 것이 추천되고 있습니다.


그런 `WorkManager`를 사용할 때 기존 프로젝트에서 Hilt 라이브러리로 dependency를 주입하고 있는 경우
`Worker` 클래스 에도 Hilt 를 사용할 수 있게 하려면 아래의 여러가지 설정이 필요합니다.

## 세팅

### Build.gradle

```gradle
dependencies {
    implementation("androidx.hilt:hilt-work:1.0.0")
    kapt("androidx.hilt:hilt-compiler:1.0.0")
}
```

### Worker class
```kotlin
@HiltWorker
class PaymentWorker @AssistedInject constructor(
    @Assisted private val context: Context,
    @Assisted workerParams: WorkerParameters,
    private val dataManager: DataManager, // 사용할 Dependency 삽입 ↓
    private val promoRepo: PromoRepository
) : CoroutineWorker(context, workerParams) {
...
}
```

기존 Worker 클래스 `@HiltWorker` annotation 을 추가하고 생성자에 `@AssistedInject` annotation 을, context 와 workerParams 는 `@Assisted` annotation 을 달아줍니다.

### Application Class
```kotlin
@HiltAndroidApp
class GlobalApplication : Application(), Configuration.Provider {

	@Inject
    lateinit var workerFactory: HiltWorkerFactory

    override fun getWorkManagerConfiguration(): Configuration {
        return Configuration.Builder().setWorkerFactory(workerFactory).build()
    }

}
```

Application Class 에 (없으면 새로 생성) `Configuration.Provider` interface 를 상속 시켜주고 `getWorkManagerConfiguration` 을 구현해 줍니다. 

#### work 2.9.0 버전

2.9.0 버전에서는 `Configuration.Provider` 에 있던 abstract `getWorkManagerConfiguration` 함수가 `workManagerConfiguration` 상수를 지니는 interface 형태로 바뀌었습니다.

기존 application class 안에 구현했던 getWorkManagerConfiguration 함수를 아래처럼 바꿔줍니다.
```kotlin
@HiltAndroidApp
class GlobalApplication : Application(), Configuration.Provider {

	@Inject
    lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder().setWorkerFactory(workerFactory).build()

}
```

### AndroidManifest.xml

위의 설정은 `WorkManager` 를 `HiltWorkerFactory` 로 자체 구성 하기 때문에 기존의 `start-up` 기본 초기화를 삭제 해야 합니다.

- androidx `start-up` 라이브러리 자체를 끄는법

```xml
  <provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="remove">
  </provider>
```

만약 기존에 `androidx start-up` 라이브러리로 다른 라이브러리를 초기화 시켜야 한다면 아래처럼 `WorkManagerinitalizer` 만 비활성화 할 수 있습니다.

- `WorkManagerinitalizer` 비활성화

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    
    <meta-data
        android:name="androidx.work.WorkManagerInitializer"
        android:value="androidx.startup"
        tools:node="remove" />

		...
</provider>
```

## 주의

`SingletonComponent` 범위로 지정된 dependency 들 만 inject 할 수 있습니다.

### KSP annotation processor를 사용 하는 경우

단순히 kapt 를 ksp 로 변경하면 `Worker class` 를 초기화 할 수 없다는 오류가 발생합니다.

```kotlin
Could not instantiate .worker.PaymentWorker
java.lang.NoSuchMethodException: .worker.PaymentWorker.<init> [class android.content.Context, class androidx.work.WorkerParameters]
	at java.lang.Class.getConstructor0(Class.java:3325)
	at java.lang.Class.getDeclaredConstructor(Class.java:3063)
	at androidx.work.WorkerFactory.createWorkerWithDefaultFallback(WorkerFactory.java:95)
	at androidx.work.impl.WorkerWrapper.runWorker(WorkerWrapper.java:243)
	at androidx.work.impl.WorkerWrapper.run(WorkerWrapper.java:145)
	at androidx.work.impl.utils.SerialExecutorImpl$Task.run(SerialExecutorImpl.java:96)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:644)
	at java.lang.Thread.run(Thread.java:1012)

Could not create Worker .worker.PaymentWorker
```

2023-10-30 기준 `hilt-compiler` dependency 가 아직 ksp 를 지원하는 정식버전이 release 되지 않아서 발생하는 오류입니다.

현재 RC 버전에서 지원 중이라 버전을  `1.1.0 rc` 버전으로 변경해주면 정상으로 작동합니다.

```gradle
dependencies {
    implementation("androidx.hilt:hilt-work:1.0.0")
    ksp("androidx.hilt:hilt-compiler:1.1.0-rc01")
}
```

+ 추가 2023-11-06 기준 `1.1.0` 정식 버전이 출시되었습니다. 해당 버전을 사용하면 정상적으로 작동합니다.

```gradle
dependencies {
    implementation("androidx.hilt:hilt-work:1.1.0")
    ksp("androidx.hilt:hilt-compiler:1.1.0")
}
```

### 참조
[Android Developers](https://developer.android.com/training/dependency-injection/hilt-jetpack?hl=ko#workmanager)

[android12]: https://developer.android.com/about/versions/12/foreground-services?hl=ko
[android14]: https://developer.android.com/about/versions/14/changes/fgs-types-required?hl=ko
[backgroundRecommend]: https://developer.android.com/guide/background?hl=ko