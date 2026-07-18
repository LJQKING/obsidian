这份JD对应的是标准的"车企/Tier1智能座舱Android工程师"技术栈，整体链路可以先建立一个框架，后面每一节基本都是这条链路上的某一层：

```
App层（Kotlin/Java：IVI、仪表、HUD应用）
    ↓ Binder / AIDL
Framework适配层（CarService、AdaptApi）
    ↓ HIDL/AIDL
VHAL（Vehicle HAL）
    ↓ 私有寄存器/协议
BSP驱动 → MCU → CAN/LIN总线
```

按JD原文四大块逐条拆解。

## 一、车载安卓应用开发

### 1. IVI应用（导航、音乐、车辆设置、空调控制）

**原理**：和普通App最大的区别不在UI，而在数据源——车速、挡位、空调温度这些不是本地传感器读出来的，是跨进程调用拿到的。标准AAOS的链路是 `CarPropertyManager`（App进程）→ Binder → `CarPropertyService`（系统进程）→ `VehicleHal` → `HalClient` → `IVehicle`（HAL实现，最终对接CAN）。调用会先经过Binder跨进程转发给车辆属性服务，由服务端做权限校验和回调注册，最终由VehicleHal层去和HAL实现交互完成实际的读写。这层暴露给App的AIDL接口很薄，只有注册/反注册监听、拿属性列表、读写单个属性以及查询读写权限这几个方法，属性用整型常量区分（定义在`VehiclePropertyIds`），区域用AreaId区分。

```kotlin
val car = Car.createCar(context)
val propertyManager = car.getCarManager(Car.PROPERTY_SERVICE) as CarPropertyManager

// 设置主驾空调目标温度
propertyManager.setProperty(
    Float::class.java,
    VehiclePropertyIds.HVAC_TEMPERATURE_SET,
    VehicleAreaSeat.ROW_1_LEFT,
    24.0f
)

// 订阅车速变化（导航、仪表都靠这个）
propertyManager.registerCallback(object : CarPropertyManager.CarPropertyEventCallback {
    override fun onChangeEvent(value: CarPropertyValue<*>) {
        val speed = value.value as Float
    }
    override fun onErrorEvent(propId: Int, zone: Int) {}
}, VehiclePropertyIds.PERF_VEHICLE_SPEED, CarPropertyManager.SENSOR_RATE_NORMAL)
```

对应源码：`packages/services/Car/car-lib/src/android/car/hardware/property/CarPropertyManager.java`、`packages/services/Car/car-lib/src/android/car/VehiclePropertyIds.java`、`packages/services/Car/service/src/com/android/car/hal/VehicleHal.java`。

**导航**：本质是GPS + 总线信号辅助修正——隧道/立交这类弱GPS场景，用车速信号配合方向盘转角做航位推算，弥补定位漂移，通常由地图SDK自己实现，应用层只需要把车速信号喂给它。

**音乐**：走标准`MediaSession`框架，不是车载专属机制。方控"上一曲/下一曲"本质是`KeyEvent.KEYCODE_MEDIA_NEXT`按键事件，经`MediaSessionManager`分发给当前"激活"的Session，播放器/语音助手/方控三方谁在控制播放，靠的是Session焦点机制。

### 2. 多屏互动（中控屏、仪表屏、副驾屏、HUD）

**原理**：Framework层由`DisplayManagerService`统一管理多屏，每块物理屏对应一个`displayId`。跨屏拉起Activity靠`ActivityOptions`带上目标屏幕ID：

```kotlin
val options = ActivityOptions.makeBasic()
options.launchDisplayId = instrumentDisplayId
startActivity(intent, options.toBundle())
```

同进程往副屏渲染用`Presentation`：

```kotlin
class InstrumentPresentation(context: Context, display: Display) : Presentation(context, display) {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.instrument_cluster)
    }
}
```

**HUD比较特殊**：物理上要把画面投影到挡风玻璃，光路存在透视畸变，所以HUD一般不是独立Activity直出物理屏，而是仪表/导航App先用`VirtualDisplay`+`Surface`渲染出内容子集（车速、转向箭头、ADAS预警），再由HUD Service做透视变形后交给HUD控制器输出。

对应源码：`frameworks/base/services/core/java/com/android/server/display/DisplayManagerService.java`、`frameworks/base/core/java/android/app/ActivityOptions.java`（`setLaunchDisplayId`）、`frameworks/base/core/java/android/app/Presentation.java`。

### 3. 语音交互（对接车载语音助手）

**原理**：车载语音引擎基本不走AOSP标准的`VoiceInteractionService`（那是给系统级语音助手用的），而是各家私有SDK，常见两种落地方式：

**结构化广播**——语音引擎做完ASR+NLU后，把"领域-意图-槽位"通过广播下发：

```kotlin
class VoiceCommandReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.getStringExtra("domain") == "climate" &&
            intent.getStringExtra("intent") == "set_temperature") {
            val temp = intent.getIntExtra("value", 24).toFloat()
            carPropertyManager.setProperty(Float::class.java,
                VehiclePropertyIds.HVAC_TEMPERATURE_SET, VehicleAreaSeat.ROW_1_LEFT, temp)
        }
    }
}
```

**Deep Link**——语音引擎解析出意图后直接发带自定义scheme的Intent，App只需在manifest声明intent-filter接住：

```xml
<activity android:name=".NavigationActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <data android:scheme="myapp" android:host="navigate"/>
    </intent-filter>
</activity>
```

两种方式共同点：App不做语音识别，只暴露"能力"给语音中台，接收结构化指令后执行。

### 4. 车联网（T-Box/V2X）

**原理**：T-Box是独立于主控之外的通信模块（自带基带，负责4G/5G连TSP云平台），与Android主控通常走UART或CAN，协议是各家私有帧格式，没有公开标准。典型分层：

```
T-Box硬件 ←UART/CAN→ /dev/ttyHSx（内核串口驱动）
    → native daemon（解析私有帧协议）
    → HIDL/AIDL（暴露为HAL服务，或写入VHAL vendor property）
    → Java系统服务（TboxManagerService）
    → App（Manager类，Binder Proxy）
```

V2X原理类似，只是接入的是C-V2X芯片，通过以太网/Socket把路侧单元广播的预警事件转成上层事件，喂给ADAS或仪表展示。这两块因为协议私有，"对应源码"更多指驱动/daemon这层的架构范式，不是某个公开类。

### 5. 车机互联（HiCar、Carlink、FlymeLink）

这三个名字虽然都叫"车机互联"，实际是三种不同性质的方案：

**HiCar（华为）**：本质是投屏系统，连接方式是USB直连，或者蓝牙配对后转WiFi，画面以H.264视频流的形式传给车机解码显示。部分手机在无线场景下的发现机制是：车机一边向手机发送USB控制指令，一边启动mDNS服务广播自身IP和监听端口供手机发现，USB断开后该服务关闭；主机厂接入SDK时还要向其开放定位、摄像头、麦克风等敏感权限。驾驶安全上有联动限制：行驶状态下，图标右下角标了"P"的应用会被置灰、禁止点击，只有车辆静止时才能打开——这和后面第三部分`CarUxRestrictionsManager`是同一层设计思路的两种实现：一个是投屏SDK自己按车速限制，一个是系统级强制。

**Carlink（ICCOA Carlink）**：和HiCar/FlymeLink不同，这不是单一厂商方案，而是智慧车联开放联盟（ICCOA）牵头制定的标准化互联协议，由OPPO、vivo、小米几家头部国产手机厂商共同发起。价值在于车机侧只需要接入一套SDK，就能同时兼容OPPO、vivo、小米等多个安卓品牌，不用每换一个手机品牌就重做一遍适配，连接方式上有线、无线都支持，功能覆盖画面投屏和桌面融合两种交互形态（桌面融合类似把手机应用以原生控件形式渲染进车机桌面，而不是单纯镜像）。车机产品要上量产车需要通过联盟认证。

**FlymeLink（魅族）**：定位比单纯投屏更进一步，官方强调这不是像CarPlay那样的简单画面映射，而是把算力、数据、硬件和生态应用这几项能力打通共享。最典型的体验是状态无感流转：上车后手机正在播的音乐、已设好的导航目的地会自动接续到车机继续，下车时又无缝交还回手机，相当于打通了一条常驻的高速数据通道，而不只是一次性画面推送。

三者协议各自私有闭源，但落地范式一致：**传输层**（USB类AOA协议 或 WiFi P2P/蓝牙+WiFi）→ **视频层**（`MediaCodec`硬解H.264/H.265）→ **交互层**（触控坐标换算后反向注入手机）→ **安全层**（与驾驶状态联动，行驶中屏蔽非白名单应用）。车机侧接收端骨架：

```kotlin
val decoder = MediaCodec.createDecoderByType("video/avc")
decoder.configure(format, projectionSurfaceView.holder.surface, null, 0)
decoder.start()
// 从USB/WiFi通道收到编码帧后送入 decoder.dequeueInputBuffer/queueInputBuffer

projectionSurfaceView.setOnTouchListener { v, event ->
    val scaledX = (event.x * phoneScreenWidth / v.width).toInt()
    val scaledY = (event.y * phoneScreenHeight / v.height).toInt()
    sendTouchToPhone(scaledX, scaledY, event.action)  // 走各自私有的反向控制通道
    true
}
```

### 6. 平台适配层交互（CarService、AdaptApi）

**原理**：同一份座舱App要跑在高通、瑞萨、恩智浦三种完全不同的SoC上，每家VHAL实现、属性编码甚至CAN信号定义都不一样，直接调某平台VHAL接口，换平台就得重写。

标准AAOS给出的答案是`CarService`——独立系统进程，通过`car-lib`把`VehicleHal`包装成统一API，App不关心底层是哪家实现。但很多国内车企/Tier1并没有完整采用AAOS，而是在标准Android上自建一个功能类似的服务，并在其上再叠一层`AdaptApi`：纯Adapter/Facade模式，把"读车速""开空调"这种业务语义和"具体平台上是哪个属性ID/寄存器"隔离开。

```kotlin
interface IVehicleAdapter {
    fun getVehicleSpeed(): Float
    fun setHvacTemperature(zone: Int, temp: Float)
}

class QualcommVehicleAdapter : IVehicleAdapter { /* 走高通平台CarService属性ID */ }
class RenesasVehicleAdapter : IVehicleAdapter { /* 走瑞萨平台信号映射 */ }

val adapter: IVehicleAdapter = when (SystemProperties.get("ro.vendor.platform")) {
    "qcom" -> QualcommVehicleAdapter()
    "renesas" -> RenesasVehicleAdapter()
    else -> NxpVehicleAdapter()
}
```

这也是JD里"CarService、AdaptApi"两个词分开写的原因——前者对应车辆信号服务本身，后者对应跨芯片平台的适配抽象，是两个层级。

## 二、系统适配与优化

### 1. Profiler工具优化（启动速度、内存占用、多任务调度）

**启动速度**：`adb shell am start -W -n com.xxx.ivi/.MainActivity`，输出的`ThisTime/TotalTime/WaitTime`来自`ActivityTaskManager`真实记录，`TotalTime`包含Zygote fork到首帧绘制全过程；更细粒度用Perfetto抓类加载、`onCreate/onResume`到`Choreographer`首帧的时间线。

**内存占用**：`adb shell dumpsys meminfo <pkg>`，PSS按Dalvik Heap/Native Heap/Graphics/Code分类，车机场景重点看Graphics（多屏叠加渲染）和Native Heap（如果有Qt/C++混合栈）。

**多任务调度**：核心是OOM_ADJ机制——`OomAdjuster`给每个进程计算`adj`值，数值越小代表越重要、越不容易被杀，前台进程的adj被固定在最低档，缓存/空进程处在adj较高、随时可能被系统回收的区间，这个值会同步给用户态`lmkd`，内存紧张时按adj从大到小杀进程。车机场景的坑在于：中控音乐/导航App切后台后adj会往上涨，不做前台Service保活的话，切一圈设置页面回来可能已经被杀重启——所以定制系统里常见做法是给白名单App做adj保护或者锁前台Service。

对应源码：`frameworks/base/services/core/java/com/android/server/am/ProcessList.java`（adj常量）、`frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java`（计算逻辑）。

### 2. Linux指令调查问题

|命令|原理|
|---|---|
|`logcat`|读`logd`用户态日志服务维护的环形缓冲区（按main/system/crash/radio/events分buffer）|
|`dumpsys <service>`|对某系统服务的Binder对象调用其`dump(FileDescriptor, PrintWriter, args)`方法|
|`top`/`procrank`|直接读`/proc/[pid]/stat`、`/proc/[pid]/smaps`等procfs节点|
|`dmesg`|读内核环形缓冲区，排查显示/CAN驱动层（BSP层）问题比logcat更有用|
|`getprop`|读属性服务，常用来判断当前跑在哪个芯片平台|

### 3. 硬件平台适配（高通、瑞萨、恩智浦）

每家BSP的显示参数、编解码能力、核数都不一样，需要适配的点：屏幕分辨率/DPI不能写死尺寸；部分平台某些编码格式缺硬解，要判断`MediaCodecList`能力做软解降级；线程池大小用`Runtime.getRuntime().availableProcessors()`动态取，不写死；属性映射差异靠前面的`AdaptApi`屏蔽。运行时平台探测：

```kotlin
val adapter: IVehicleAdapter = when {
    SystemProperties.get("ro.vendor.platform").contains("qcom") -> QualcommVehicleAdapter()
    SystemProperties.get("ro.vendor.platform").contains("renesas") -> RenesasVehicleAdapter()
    else -> NxpVehicleAdapter()
}
```

## 三、安全与合规开发

### 1. 功能安全与网络安全标准

**功能安全（对应ISO 26262）**：座舱Android应用绝大部分是QM等级，真正ASIL功能（转向、制动）不放在Android系统里，但应用层要对上游信号做防御性处理：超时未收到车速/挡位更新就回落安全默认值而不是显示旧值；CAN信号解析要做校验避免总线干扰导致误显示；关键服务要有心跳看门狗。

**网络安全（对应ISO/SAE 21434）**：SELinux domain隔离让每个进程限制在安全域内，即使App被攻破也无法直接访问车控设备节点；应用签名校验+Keystore硬件密钥管理；与TSP云端通信走HTTPS+证书锁定。SELinux策略示意：

```
type car_service, domain;
allow car_service vehicle_device:chr_file rw_file_perms;
neverallow { domain -car_service } vehicle_device:chr_file *;
```

### 2. 驾驶模式防分心设计

这是JD里能给出最完整、可验证AOSP源码链路的一点。AAOS把"车速→驾驶状态→UI限制"标准化成`CarDrivingStateManager`+`CarUxRestrictionsManager`：

第一步，系统把挂P挡定义为Parked、不挂P挡但车速为零定义为Idling、车辆在动定义为Moving，一共三种驾驶状态。

第二步，驾驶状态映射成一组UX限制位，规则来自可配置XML（不同车厂/市场标准不同）。App侧的正确做法是订阅抽象出来的限制位，而不是自己监听车速写if-else——这样App不用因为不同地区安全法规差异各自写一套判断逻辑，只需要对系统告知的限制状态做反应：

```kotlin
val car = Car.createCar(context)
val uxrManager = car.getCarManager(Car.CAR_UX_RESTRICTION_SERVICE) as CarUxRestrictionsManager

uxrManager.registerListener { restrictions ->
    if (restrictions.isRequiresDistractionOptimization) {
        val active = restrictions.activeRestrictions
        if (active and CarUxRestrictions.UX_RESTRICTIONS_NO_VIDEO != 0) player.pause()
        if (active and CarUxRestrictions.UX_RESTRICTIONS_NO_KEYBOARD != 0) keyboardInput.visibility = View.GONE
    } else {
        keyboardInput.visibility = View.VISIBLE
    }
}
```

一旦这个值变为true，前台Activity如果没有被标注为"可分心优化"，系统会直接把它停掉——这就是"限制复杂交互、语音优先"的系统级实现，比HiCar自己按"P角标"做的应用内限制更底层、更强制。

对应源码：`packages/services/Car/car-lib/src/android/car/drivingstate/CarUxRestrictionsManager.java`、`packages/services/Car/service/src/com/android/car/CarUxRestrictionsManagerService.java`。

## 四、跨团队协作

### 1. 与UI/UX协作，HMI体验

这块"源码"主要落在自定义View：仪表指针、能量流动画这类座舱特有控件标准View库覆盖不了，一般重写`onDraw()`用`Canvas`+`Path`手工画，指针角度用`ValueAnimator`驱动：

```kotlin
class SpeedGaugeView(context: Context) : View(context) {
    private var needleAngle = 0f
    fun setSpeed(speed: Float) {
        ValueAnimator.ofFloat(needleAngle, speedToAngle(speed)).apply {
            duration = 300
            addUpdateListener { needleAngle = it.animatedValue as Float; invalidate() }
        }.start()
    }
    override fun onDraw(canvas: Canvas) {
        canvas.save()
        canvas.rotate(needleAngle, width / 2f, height / 2f)
        canvas.drawPath(needlePath, needlePaint)
        canvas.restore()
    }
}
```

### 2. 与底层系统团队对接（FrameWork、BSP、MCU信号层）

本质是把前面"平台适配层"那条链路再往下打通一层——MCU负责车身低速I/O（门窗、灯光），信号定义在DBC文件里（CAN Database，定义每个CAN ID里哪几个bit对应哪个物理量）：

```
CAN Frame (ID=0x120, 8 bytes)
  --DBC解码--> 车速原始信号 = raw × 0.01 km/h
  --VHAL--> VehiclePropValue{ prop: PERF_VEHICLE_SPEED, value: 65.5 }
  --Binder--> CarPropertyManager.onChangeEvent(...)
  --> App更新UI
```

Android侧"信号集成"的具体工作是跟BSP团队对齐DBC到VHAL property的映射表，跟FrameWork团队确认新增property的权限声明——DBC变更、VHAL适配、应用消费是三方交接的三个环节，任何一环字段定义对不齐，读出来的车速/挡位就是错的。

### 3. 支持HIL、实车路测

**HIL**：用dSPACE、Vector CANoe这类台架模拟CAN信号（不需要真车），重点验证边界场景——信号丢失时的默认值、数值突变时UI有没有异常、总线负载拉满时Binder调用是否超时。

**实车路测支持**：主要靠现场可回溯性：

```kotlin
Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
    crashLogger.log(thread, throwable, System.currentTimeMillis())  // 带时间戳，方便和CAN抓包对齐
    defaultHandler?.uncaughtException(thread, throwable)
}
```

Native崩溃对应`/data/tombstones/`，ANR对应`/data/anr/traces.txt`，路测阶段这两类文件配合车辆总线时间戳做关联分析，是定位偶现问题的主要手段。