## Android 面试八股文（3 年经验工程师通用版）

------

### 🧠 基础组件与生命周期

#### ✅ Activity 生命周期

- 生命周期方法顺序：`onCreate()` → `onStart()` → `onResume()` → `onPause()` → `onStop()` → `onDestroy()`。
- 特殊场景：横竖屏切换、权限弹窗、跳转其他 Activity 等会触发重新创建或暂停恢复。
- 状态保存：在 `onSaveInstanceState()` 中保存 UI 状态，如输入框内容、滚动位置。
- 注意事项：避免在 `onPause()` 做耗时操作，否则会影响界面流畅性；资源释放应在 `onStop()` 或 `onDestroy()` 中完成。

#### ✅ Fragment 生命周期

- 生命周期流程：`onAttach()` → `onCreate()` → `onCreateView()` → `onViewCreated()` → `onStart()` → `onResume()`，对应销毁阶段：`onPause()` → `onStop()` → `onDestroyView()` → `onDestroy()` → `onDetach()`。
- `onCreateView()` 是创建 UI 结构的关键方法。
- 推荐使用 `viewLifecycleOwner.lifecycleScope` 启动协程，避免 View 销毁后访问空指针。
- 与 ViewPager 联用时，生命周期控制更复杂，需结合 `setUserVisibleHint()` / `FragmentStateAdapter` 优化。

#### ✅ View 的绘制流程

- 三大阶段：`measure()` 计算尺寸 → `layout()` 确定位置 → `draw()` 完成绘制。
- VSYNC 信号驱动绘制：由 `Choreographer` 安排下一帧执行，驱动 ViewRootImpl 发起重绘。
- 自定义 View 时需重写三个方法：`onMeasure()`（设置尺寸）、`onLayout()`（子 View 摆放）、`onDraw()`（具体绘制）。

#### ✅ 事件分发机制

- 事件流转过程：Activity → ViewGroup → 子 View。
- 三个核心方法：
  - `dispatchTouchEvent()`：事件分发，决定是否向子 View 传递。
  - `onInterceptTouchEvent()`：ViewGroup 拦截事件（子 View 收不到）。
  - `onTouchEvent()`：最终处理事件。
- 常见应用：自定义控件滑动冲突处理、点击穿透问题。

------

### 🧵 线程与异步机制

#### ✅ Handler 消息机制

- Handler 的本质是线程间通信工具。
- 每个线程可创建一个 `Looper` + `MessageQueue`。
- 主线程默认初始化了 Looper，其他线程需手动调用 `Looper.prepare()`。
- 通过 `post(Runnable)` 或 `sendMessage()` 向主线程发送任务，实现子线程更新 UI。

#### ✅ Looper 和 MessageQueue

- `Looper.prepare()` 初始化线程的消息循环系统。
- `Looper.loop()` 启动死循环，从 MessageQueue 取出消息并处理。
- 主线程唯一，系统通过它实现 UI 操作同步。
- 常用于：子线程初始化 Handler（如 HandlerThread）、定时任务、延迟执行等。

#### ✅ ANR 原因与优化

- Activity 5 秒无响应输入，Service 20 秒，Broadcast 10 秒未处理完成 → 触发 ANR。
- 常见原因：主线程做耗时操作、死循环、数据库/网络操作阻塞。
- 优化手段：使用协程/线程池，子线程处理耗时逻辑；使用 StrictMode 提前发现慢操作。

------

### 🔄 Kotlin 协程核心

#### ✅ 协程数量与线程关系

- 协程是用户态线程（轻量线程），不是操作系统级线程。
- 一个线程可以同时运行成千上万个协程，因为协程本质上是 **非阻塞的状态机**，仅在需要挂起时交出线程。
- 协程挂起时不会占用线程资源，因此可以大规模并发执行。

#### ❗ 创建过多协程的后果

- 虽然每个协程开销极小（大约几百字节），但过量协程仍可能导致：
  - **内存压力增加**：每个协程都有调用栈、上下文。
  - **调度开销增加**：调度器维护的任务队列变大。
  - **业务逻辑错误**：忘记取消或无限重试可能导致协程泄漏或逻辑重复。
- 建议：使用结构化并发、限制协程数量、配合 `CoroutineDispatcher` 控制调度策略。

```kotlin
// 模拟启动大量协程（注意：不可滥用）
runBlocking {
    repeat(100_000) {
        launch(Dispatchers.Default) {
            delay(1000L)
            println("协程 $it 完成")
        }
    }
}
// 理论上可运行，但根据设备配置可能 OOM 或被 GC 频繁打断
```

#### ✅ 延时任务示例

- `delay(timeMillis)` 是协程挂起函数，不会阻塞线程，而是让协程“暂停”，等待恢复后继续执行。
- `repeat(n)` 是 Kotlin 标准库中的循环函数，会将代码块执行 n 次，常用于轮询、倒计时等。

```kotlin
lifecycleScope.launch {
    Log.d("TAG", "开始倒计时")
    repeat(5) { i ->
        delay(1000) // 挂起当前协程 1 秒，不阻塞线程（本质是挂起函数，由调度器调度恢复）
        Log.d("TAG", "倒计时剩余：${5 - i} 秒")
    }
    Log.d("TAG", "倒计时结束，执行跳转")
    navigateToNextPage()
}
```

#### ✅ 协程调度器详解

- `Dispatchers.Default`：默认调度器，适用于 CPU 密集型任务（如排序、JSON 解析）。背后是一个基于 CPU 核数的共享线程池。
- `Dispatchers.IO`：用于磁盘、数据库、网络等 IO 密集型任务，线程数远大于 Default，内部可调度更多线程处理阻塞任务。
- `Dispatchers.Main`：主线程调度器，适用于更新 UI，仅在 Android 中有效。
- `Dispatchers.Unconfined`：不限制线程，初次运行在调用者线程，挂起后恢复在哪个线程由恢复逻辑决定。适合调试或特定场景，不推荐用于生产。

```kotlin
// 示例：根据任务类型选择调度器
withContext(Dispatchers.IO) {
    val json = URL(url).readText()
    val result = parseJson(json) // IO + CPU 混合任务
}
```

#### ✅ 协程 Job 层级与取消机制

- `Job`：协程的生命周期管理器。每个 `launch`/`async` 默认会绑定一个 Job。
- 子协程依附于父协程，如果父协程取消，所有子协程也将取消（结构化并发核心）。
- 使用 `coroutineContext.job` 可以访问当前 Job，调用 `cancel()` 手动取消。
- 可使用 `CoroutineScope(SupervisorJob())` 实现兄弟协程失败不互相影响。

```kotlin
val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
scope.launch {
    val a = launch { error("A fail") }
    val b = launch { delay(1000); println("B success") }
    // B 仍然会执行，不受 A 的失败影响
}
```

#### ✅ suspend 函数原理简析

- `suspend` 并不会新建线程，而是表示函数可挂起。
- 编译器会将挂起点转换为状态机（StateMachine），生成 `Continuation`。
- 调用挂起函数时协程会“暂停”执行，挂起处会保存上下文，等结果返回后“恢复”继续执行。
- 结合调度器，可挂起执行但继续复用原协程线程，提升性能。

```kotlin
suspend fun fetchUser(): User {
    val json = withContext(Dispatchers.IO) { apiService.getUserJson() }
    return parse(json)
}
```

- `launch{}`：启动协程，不关心返回结果，常用于 UI 操作。
- `async{}`：返回 Deferred，可使用 `await()` 获取结果，适合并发任务结果合并。
- `withContext(Dispatchers.IO)`：切换线程上下文并获取返回值，常用于封装数据库或网络请求操作。
- `runBlocking{}`：阻塞当前线程，常用于单元测试或 main 函数中启动协程。
- 协程优点：轻量级、易组合、线程复用、结构清晰。

**常见实践示例：**

```kotlin
// launch：无需返回结果，适合 UI 操作
lifecycleScope.launch {
    delay(1000)
    Log.d("TAG", "更新 UI")
}

// async + await：适合并发任务并合并结果
lifecycleScope.launch {
    val a = async { loadFromNetworkA() }
    val b = async { loadFromNetworkB() }
    val result = a.await() + b.await()
    updateUI(result)
}

// withContext：切换线程执行任务并返回结果
suspend fun loadData(): List<Item> {
    return withContext(Dispatchers.IO) {
        database.getAllItems()
    }
}

// runBlocking：用于 main 函数或测试中启动协程（注意阻塞）
fun main() = runBlocking {
    val result = withContext(Dispatchers.Default) {
        computeHeavyTask()
    }
    println("结果: $result")
}
```

- `launch{}`：启动协程，不关心返回结果，常用于 UI 操作。
- `async{}`：返回 Deferred，可使用 `await()` 获取结果，适合并发任务结果合并。
- `withContext(Dispatchers.IO)`：切换线程上下文并获取返回值，常用于封装数据库或网络请求操作。
- 协程优点：轻量级、易组合、线程复用、结构清晰。

#### ✅ 协程作用域

- `GlobalScope`：应用全局，不推荐生产使用。
- `CoroutineScope`：可手动控制生命周期，如 `viewModelScope`、`lifecycleScope`。
- 推荐绑定生命周期，防止内存泄漏和无效调用。

#### ✅ 结构化并发

- 父协程负责子协程的生命周期，出现异常可统一取消。
- 子协程失败时，会取消整个作用域内的所有协程，确保状态一致。
- 使用 `supervisorScope` 可实现部分失败不影响其他协程的执行。

#### ✅ 异常处理

- 使用 `try-catch` 包裹挂起函数处理异常。
- `CoroutineExceptionHandler` 可统一处理 `launch` 中未捕获异常。
- `async` 的异常要在调用 `await()` 时才能捕获。

------

### 🧱 架构模式与 Jetpack

#### ✅ MVVM 架构

- Model：数据层，如 Repository、数据源、网络、数据库。
- ViewModel：处理业务逻辑，维护 UI 状态，封装 LiveData/StateFlow。
- View：Activity/Fragment 监听数据变化并刷新 UI。

#### ✅ ViewModel

- 生命周期安全，系统配置变更（如旋转屏幕）不会销毁。
- 持有 LiveData、Flow 等数据源，协程默认运行在 `viewModelScope` 中。
- 避免持有 View 引用，防止内存泄漏。

#### ✅ LiveData vs StateFlow

| 特性         | LiveData                                 | StateFlow                                          |
| ------------ | ---------------------------------------- | -------------------------------------------------- |
| 生命周期感知 | ✅ 生命周期感知强                         | ❌ 需手动取消订阅                                   |
| 背压处理     | ❌ 无内建背压机制，快速更新可能丢失中间值 | ✅ 支持 `buffer()`、`conflate()` 等操作控制数据频率 |
| 多值发送     | 只保留最后值（中间值丢弃）               | 可连续发送多条值，每条都可被处理                   |
| 常用于       | 与 UI 层联动，自动更新                   | 数据流收集、状态管理合并                           |

**术语解释：**

- **背压（Backpressure）**：当数据产生速度远快于消费速度时，如果没有控制手段就可能导致内存爆炸或 UI 卡顿。StateFlow 本身只维护一个值（类似 LiveData），但它作为 Flow 的实现者，可结合 `.buffer()`、`.conflate()` 等操作符实现背压处理策略。而 LiveData 缺乏此类机制，适用于低频、状态类数据绑定。

#### ✅ Room

- SQLite 的抽象封装，提供更安全高效的数据库操作方式。
- 主要组件：`@Entity` 定义数据表、`@Dao` 操作数据库方法、`@Database` 管理数据库实例。
- 支持 `suspend fun`，可直接配合协程使用。
- 支持关系型查询、事务操作、数据监听（Flow）。

#### ✅ Navigation

- 使用 `NavHostFragment` 实现页面导航容器。
- 在 `nav_graph.xml` 中定义页面结构与跳转路径。
- 支持参数传递、动画、返回栈管理、安全导航等功能。



# 共享元素动画（Shared Element Transition）

## 1. 概念
- 共享元素动画是指在两个界面（Activity 或 Fragment）之间切换时，两个界面中相同的视图元素通过动画实现无缝过渡，提升界面切换的流畅度和用户体验。

## 2. 适用场景
- 列表页点击图片跳转详情页，图片放大平滑过渡
- Fragment 或 Activity 之间视觉元素的连续性动画

## 3. 实现要点

### Activity 之间共享元素动画
- 两个 Activity 对应的共享视图设置相同的 `android:transitionName`
- 启动 Activity 时使用 `ActivityOptionsCompat.makeSceneTransitionAnimation()` 指定共享元素
- 只支持 API 21 及以上

### Fragment 之间共享元素动画
- 两个 Fragment 中共享视图的 `transitionName` 一致
- Fragment 切换时使用 FragmentTransaction 的 `setReorderingAllowed(true)`
- 使用 `addSharedElement(view, transitionName)` 添加共享元素
- 兼容性同样是 API 21+

## 4. 示例代码（简要）

```kotlin
// Activity启动示例
val options = ActivityOptionsCompat.makeSceneTransitionAnimation(this, imageView, "shared_image")
startActivity(intent, options.toBundle())

// Fragment切换示例
parentFragmentManager.beginTransaction()
    .setReorderingAllowed(true)
    .addSharedElement(imageView, "shared_image")
    .replace(R.id.container, FragmentB())
    .addToBackStack(null)
    .commit()
```

------

# Android 大型项目常见问题及解决方案

## 1. 模块化与代码组织
### 问题
- 项目体量大，代码耦合严重，维护困难
- 构建时间过长，编译效率低

### 解决方案
- 使用 Gradle 多模块化拆分（app、feature 模块、library 模块）
- 采用清晰的架构模式（如 MVVM、MVI、Clean Architecture）
- 统一代码规范和设计文档
- 使用依赖注入框架（Dagger/Hilt/Koin）降低耦合

---

## 2. 性能优化
### 问题
- 启动时间长，卡顿、ANR
- 内存泄漏，OOM
- UI 绘制卡顿，滚动不流畅

### 解决方案
- 启动优化：减少冷启动任务，延迟加载非关键业务
- 使用内存分析工具（LeakCanary、Android Profiler）定位泄漏
- 避免过度嵌套布局，使用 ConstraintLayout 或自定义绘制
- 合理使用缓存和异步加载，避免主线程阻塞
- 优化 RecyclerView 和列表的性能

---

## 3. 资源管理
### 问题
- 图片资源庞大，导致 APK 体积大
- 多语言、多屏幕适配难度高

### 解决方案
- 图片压缩和格式优化（WebP）
- 使用矢量图（VectorDrawable）减少多分辨率资源
- 采用资源分包或动态交付（Dynamic Delivery）
- 使用多语言资源文件（strings.xml），并严格管理版本更新
- 使用 ConstraintLayout、尺寸限定符，结合适配方案（如 dp/sp）

---

## 4. 网络请求与数据同步
### 问题
- 多网络请求复杂，状态管理困难
- 数据一致性和缓存设计

### 解决方案
- 使用 Retrofit、OkHttp 等稳定网络库
- 利用协程、RxJava 统一异步处理和线程调度
- 设计合理的缓存策略（内存缓存、磁盘缓存、网络缓存）
- 使用响应式编程或状态管理库（LiveData、Flow、MVI）处理数据状态
- 异常统一处理和重试机制设计

---

## 5. 多线程与异步处理

**多个线程修改livedata，尤其历史逻辑，都去修改某个关键livedata，导致状态出错。（梳理逻辑，整理前序逻辑的顺序，livedata和stateflow的选择），数据同步问题，缓存中有很多来源更新数据，在更新的位置增加卡点，根据来源和自带的版本号，在更新时机加一层数据保证。**

### 问题
- 线程安全和资源竞争
- 复杂业务逻辑异步链路管理

### 解决方案
- 使用 Kotlin 协程简化异步代码和异常管理
- 避免共享可变状态，尽量使用不可变对象
- 使用线程安全的数据结构
- 设计清晰的异步流程，做好异常和取消处理

---

## 6. 测试与质量保证
### 问题
- 代码回归风险高，bug 难以复现
- 自动化测试覆盖不足

### 解决方案
- 编写单元测试，覆盖核心业务逻辑
- UI 测试（Espresso、UI Automator）保证界面稳定
- 持续集成（CI）工具，自动化构建和测试
- 使用静态代码分析工具（Lint、SonarQube）提升代码质量

---

## 7. 版本升级与兼容性
### 问题
- Android 系统版本碎片化，API 差异
- 第三方库版本冲突

### 解决方案
- 使用兼容库（AndroidX）
- 按需适配不同 API，合理使用 `@RequiresApi` 和版本判断
- 依赖版本管理，避免冲突，升级前做充分测试
- 采用灰度发布和远程配置降低风险

---

## 8. 用户体验和设计一致性
### 问题
- 多模块、多团队开发导致 UI 风格不统一
- 动画和交互效果不流畅

### 解决方案
- 制定 UI 设计规范和组件库，统一样式和交互
- 使用 Jetpack Compose 或自定义组件实现一致性
- 合理使用动画，避免滥用导致卡顿
- 持续优化响应速度，做到界面流畅

---



### 📦 数据存储与通信

#### ✅ 数据存储方式

- SharedPreferences：传统键值对，适合轻量配置存储。
- DataStore：Jetpack 推荐的新一代存储方案，支持类型安全、协程、Flow 流式读取。
- Room：结构化数据存储，适合列表、复杂表格。
- 文件存储：适用于导出文件、图片缓存、日志记录等。
- MMKV：微信开源的高性能替代品，适合大数据量频繁读写。

#### ✅ 跨组件通信方式

- Intent：最常用方式，支持显式跳转和隐式调用。
- BroadcastReceiver：全局广播通知组件，适合解耦通信。
- AIDL：Android Interface Definition Language，支持跨进程通信（IPC）。
- LiveData/EventBus：用于组件间消息总线，简化通信逻辑。
- SharedViewModel：同一 Navigation Graph 下的 Fragment 共享数据。

------

### 📲 网络与性能优化

#### ✅ Retrofit + 协程封装

- 定义接口：使用 `@GET/@POST` 注解 + `suspend` 函数封装请求。
- 支持自动反序列化 JSON 数据为数据类。
- 可封装网络状态类（sealed class）统一处理 loading/error/success 状态。
- 可结合 OkHttp Interceptor 添加统一 header、日志打印。

#### ✅ 性能优化点

- 冷启动优化：延迟初始化非关键组件、优化 Application onCreate、使用 SplashScreen。
- 内存泄漏防范：弱引用、及时释放 Listener、避免静态持有 Context。
- 卡顿优化：避免 UI 主线程阻塞、布局扁平化、减少嵌套、使用 ConstraintLayout。
- 图片优化：使用 Glide 控制缓存、压缩尺寸、避免内存占用过大。

------

### 🧪 常见面试问题速记

1. **协程和线程的区别？**
   - 协程是用户级轻量线程，由 Kotlin 协程库调度，能复用底层线程池；线程是系统资源，调度由操作系统控制。
2. **View 的绘制流程？**
   - 从 ViewRootImpl 发起 `measure()` → `layout()` → `draw()`，最终通过 GPU 渲染输出图像。
3. **Fragment 为什么容易泄漏？如何避免？**
   - 原因：在 Fragment 中持有 Activity/Context 的静态引用或 View 引用，或注册监听未释放。
   - 避免方法：使用 `viewLifecycleOwner` 管理生命周期、及时解绑资源、避免匿名内部类。
4. **LiveData 的生命周期感知机制？**
   - 基于 `LifecycleOwner` 注册观察者，自动在 `onStart/onStop` 时激活/停止观察，防止内存泄漏。
5. **如何实现应用模块化？**
   - 拆分为多个 Gradle module：如 `data/`, `domain/`, `ui/`
   - 每个 module 职责单一，通过 interface/DI 管理依赖，降低耦合、加快编译。

------

> ✅ 建议打印前将内容粘贴至 Markdown 编辑器如 Typora，或 VSCode + Markdown PDF 插件进行导出。

祝你面试顺利，拿下心仪 offer！