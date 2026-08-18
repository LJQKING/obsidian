
## 一、库架构概览

### 核心组件关系

```
RxBleClient (入口点)
    ├── RxBleScanner (扫描管理)
    ├── RxBleDevice (设备管理)
    │   └── RxBleConnection (连接和操作)
    │       ├── RxBleConnectionState (状态观测)
    │       ├── RxBleGattIO (GATT操作)
    │       └── RxBleNotification (通知订阅)
    └── RxBleAdapterStateObservable (蓝牙适配器状态)
```

---

## 二、交互方法详解

### 1. 初始化与客户端获取

```java
// RxBleClient 是单例
public static RxBleClient create(Context context) {
    // 内部使用 RxBleClientImpl 实现
    // 初始化时做的事：
    // 1. 创建 BluetoothAdapter 引用
    // 2. 初始化 RxBleAdapterStateObservable（监听BT状态变化）
    // 3. 创建 ScanSetup 和 DeviceProvider
    // 4. 设置内部的 Scheduler 用于线程管理
}

RxBleClient rxBleClient = RxBleClient.create(context);
```

**关键点**：

- `RxBleClient` 应该是单例模式，因为重复创建会造成资源浪费
- 内部维护一个 `BluetoothManager` 引用，所有的蓝牙操作都通过它

### 2. 扫描流程（Cold Observable）

```java
// Observable 链路
rxBleClient.scanBleDevices(scanSettings, filters)
    .subscribe(scanResult -> {...}, error -> {...})

// 内部实现原理
```

#### 扫描的 Observable 分发流程

```
scanBleDevices()
    ├─ 检查权限和蓝牙状态（observeStateChanges()）
    ├─ 若状态不可用 → 发送 onError
    └─ 若状态可用 → 创建 BluetoothLeScanner
        └─ 调用 BluetoothLeScanner.startScan()
            └─ 通过 ScanCallback 接收结果
                └─ 转化为 RxBleScanResult Observable
                    ├─ device: RxBleDevice
                    ├─ rssi: int (信号强度)
                    ├─ scanRecord: ScanRecord
                    └─ timestampNanos: long
```

#### 核心源码（伪代码）

```java
public Observable<RxBleScanResult> scanBleDevices(
    ScanSettings scanSettings, 
    ScanFilter... scanFilters) {
    
    return new Observable<RxBleScanResult>() {
        @Override
        protected void subscribeActual(Observer<? super RxBleScanResult> observer) {
            // 1. 检查状态
            observeStateChanges()
                .switchMap(state -> {
                    if (state == BleAdapterState.READY) {
                        // 2. 创建扫描 Observable
                        return scanSetup.scanBleDevices(scanSettings, scanFilters);
                    }
                    return Observable.error(/*异常*/);
                })
                .subscribe(observer);
        }
    };
}

// 若使用 API < 21，会使用 BluetoothAdapter.startLeScan() 的模拟实现
```

**扫描特性**：

- **Cold Observable**: 每次 subscribe 都会启动新的扫描
- **自动清理**: dispose 时自动调用 `stopScan()`
- **状态感知**: 通过 `switchMap` 监听 BT 状态，状态变化时自动结束当前扫描

### 3. 连接流程（单值 Observable）

```java
RxBleDevice device = rxBleClient.getBleDevice("AA:BB:CC:DD:EE:FF");

device.establishConnection(autoConnect: boolean)
    .subscribe(
        rxBleConnection -> {/* 拥有连接对象 */},
        error -> {/* 连接失败 */}
    );
```

#### 连接状态转移图

```
Initial State
    ↓
[检查蓝牙状态] → BLUETOOTH_NOT_ENABLED/NOT_AVAILABLE → onError
    ↓ (状态OK)
[调用 BluetoothDevice.connectGatt()]
    ↓
Connecting (GATT Callback: onConnectionStateChange)
    ├─ Status 0 → Connected
    │    ├─ 发起 Service Discovery
    │    └─ Discovering Services...
    │        ↓
    │    Services Discovered (onServicesDiscovered)
    │        ↓
    │    [发送 RxBleConnection] → onNext(connection)
    │        ↓
    │    [保持连接] (Observable 不完成)
    │        ↓
    │    Device Disconnect or Error
    │        └─ onError(异常)
    │
    └─ Status != 0 → onError(连接失败异常)
```

#### 核心实现细节

```java
// RxBleConnection 的创建是关键
class GattCallback extends BluetoothGattCallback {
    private Subject<RxBleConnection> connectionRelay;
    
    @Override
    public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
        if (status == BluetoothGatt.GATT_SUCCESS) {
            if (newState == BluetoothProfile.STATE_CONNECTED) {
                // 开始 Service Discovery
                gatt.discoverServices();
            } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
                connectionRelay.onError(new DisconnectedException());
            }
        } else {
            connectionRelay.onError(new BleGattException(status));
        }
    }
    
    @Override
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
        if (status == BluetoothGatt.GATT_SUCCESS) {
            RxBleConnection connection = new RxBleConnectionImpl(gatt);
            // 发送单一值，但不完成 Observable
            connectionRelay.onNext(connection);
        } else {
            connectionRelay.onError(new BleGattException(status));
        }
    }
}
```

**关键特性**：

- **`autoConnect=false`**: 直接连接，超时时间较短（~30s），设备必须正在广告
- **`autoConnect=true`**: 等待设备出现，更耗电但更灵活，会获取 wake lock
- **连接不会自动重连**: 与原生 Android API 不同，即使设置 autoConnect=true

### 4. GATT 操作（读写通知）

#### 4.1 读操作

```java
rxBleConnection.readCharacteristic(UUID characteristicUUID)
    // 返回 Single<byte[]>
    .subscribe(
        bytes -> {/* 读取成功 */},
        error -> {/* 读取失败 */}
    );
```

#### 读操作的队列化机制

```
readCharacteristic(UUID)
    ↓
[加入 GattOperationQueue]
    ├─ 若队列为空，立即执行
    │  └─ BluetoothGatt.readCharacteristic()
    └─ 若队列不空，等待前序操作完成
       └─ onCharacteristicRead() 回调后执行
            ↓
       [从队列移除，执行下一个操作]
```

**实现原理** (伪代码)：

```java
public Single<byte[]> readCharacteristic(UUID uuid) {
    return Single.create(emitter -> {
        // 1. 通过 UUID 查找 Characteristic
        BluetoothGattCharacteristic characteristic = 
            findCharacteristicByUUID(uuid);
        
        if (characteristic == null) {
            emitter.onError(new CharacteristicNotFound(uuid));
            return;
        }
        
        // 2. 加入操作队列
        gattOperationQueue.queueOperation(() -> {
            // 3. 发起读操作
            boolean readInitiated = bluetoothGatt.readCharacteristic(characteristic);
            
            if (!readInitiated) {
                emitter.onError(new BleGattOperationException("Read failed"));
            }
            // 4. 等待回调 onCharacteristicRead()
        });
        
        // 5. GATT 回调
        @Override
        void onCharacteristicRead(BluetoothGatt gatt, 
                                 BluetoothGattCharacteristic characteristic, 
                                 int status) {
            if (status == GATT_SUCCESS) {
                byte[] value = characteristic.getValue();
                emitter.onSuccess(value);
            } else {
                emitter.onError(new BleGattException(status));
            }
            gattOperationQueue.dequeue(); // 触发下一个操作
        }
    })
    .observeOn(Schedulers.io()); // 确保回调在 IO 线程
}
```

**为什么需要队列化？**

- Android GATT 操作是异步的，但不能并发
- 同时发起多个读/写会导致操作失败或被忽略
- RxAndroidBle 内部维护一个串行队列确保操作顺序

#### 4.2 写操作

```java
rxBleConnection.writeCharacteristic(UUID uuid, byte[] bytes)
    // 返回 Single<byte[]> （回显写入的数据）
    .subscribe(
        writtenBytes -> {/* 写入成功 */},
        error -> {/* 写入失败 */}
    );
```

**长写操作** (用于超过 MTU 大小的数据)：

```java
rxBleConnection.createNewLongWriteBuilder()
    .setCharacteristicUuid(uuid)
    .setBytes(largeByteArray)
    .setMaxBatchSize(20) // 每批 20 字节
    .setWriteOperationRetryStrategy(retryStrategy) // 失败重试
    .build()
    .subscribe(
        allBytes -> {/* 全部写入完成 */},
        error -> {/* 某个批次失败 */}
    );
```

**实现**：

```
LongWriteOperation
    ├─ 将数据分割为 batch（每个 batch <= MTU）
    ├─ 顺序执行每个 batch 的 writeCharacteristic()
    ├─ 若某个 batch 失败，执行 retryStrategy
    │  ├─ IMMEDIATE_RETRY: 立即重试
    │  ├─ DELAYED_RETRY: 延迟后重试
    │  └─ SKIP: 跳过此 batch
    └─ 全部完成或重试次数超限 → onSuccess/onError
```

#### 4.3 通知/指示（Notifications & Indications）

```java
rxBleConnection.setupNotification(UUID characteristicUUID)
    // 返回 Observable<Observable<byte[]>>
    .doOnNext(notificationObservable -> {
        // 通知已设置，此时已经发送 CCCD 写入命令
    })
    .flatMap(notificationObservable -> notificationObservable)
    // 现在 flatMap 后拿到的是实际的通知流
    .subscribe(
        bytes -> {/* 收到通知 */},
        error -> {/* 通知错误 */}
    );
```

**双层 Observable 的设计意图**：

```
setupNotification(UUID)
    ↓
[第一层 Observable]
    ├─ 作用: 设置通知
    └─ onNext(第二层 Observable)
        ↓
       [第二层 Observable]
           ├─ 作用: 持续接收通知
           └─ 每收到一个通知值，发送一次 onNext
```

**完整流程**：

```
setupNotification(UUID)
    ├─ 写入 CCCD (Client Characteristic Configuration Descriptor)
    │  └─ 值: 0x0001 (启用通知)
    ├─ 写入成功 → onNext(innerObservable)
    └─ 等待设备端发送通知
        ↓
    @Override
    onCharacteristicChanged(BluetoothGatt gatt, 
                           BluetoothGattCharacteristic characteristic) {
        byte[] value = characteristic.getValue();
        innerObservable.onNext(value);
        // innerObservable 继续持有
    }
    
    // dispose 时
    ├─ 写入 CCCD 为 0x0000（禁用通知）
    └─ innerObservable.onComplete()
```

**Indication vs Notification**：

```
Notification (setupNotification)
    ├─ 单向: 设备 → 手机
    ├─ 可靠性: 不保证传递（可能丢失）
    ├─ 延迟: 低
    └─ 功耗: 低

Indication (setupIndication)
    ├─ 双向: 设备 → 手机 → 设备（手机需要确认）
    ├─ 可靠性: 保证传递
    ├─ 延迟: 高
    └─ 功耗: 高
```

---

## 三、错误处理机制

### 1. 错误类型体系

```java
// 顶层异常
BleException (所有 BLE 错误的基类)
    ├── BleDisconnectedException
    │   ├── 原因: 连接在操作中被中断
    │   └─ GATT status != SUCCESS
    │
    ├── BleGattException (GATT 状态错误)
    │   ├── status: int (GATT 状态码)
    │   └─ 可能的值:
    │       ├─ 0x0000 (GATT_SUCCESS)
    │       ├─ 0x0002 (GATT_READ_NOT_PERMITTED)
    │       ├─ 0x0003 (GATT_WRITE_NOT_PERMITTED)
    │       ├─ 0x0008 (GATT_REQUEST_NOT_SUPPORTED)
    │       ├─ 0x0E (GATT_INSUFFICIENT_AUTHENTICATION)
    │       └─ ... 更多状态码
    │
    ├── BluetoothDisabledException
    │   └─ 原因: 蓝牙被关闭
    │
    ├── LocationPermissionNotGranted
    │   └─ 原因: 缺少定位权限 (API >= 23 扫描需要)
    │
    ├── CharacteristicNotFound
    │   └─ 原因: 指定的 Characteristic UUID 不存在
    │
    └── OnSubscriptionNotSupported
        └─ 原因: 通知不可用 (设备不支持)
```

### 2. 错误处理的最佳实践

#### 案例 1: 区分错误类型

```java
device.establishConnection(false)
    .flatMap(rxBleConnection -> 
        rxBleConnection.readCharacteristic(characteristicUUID)
    )
    .subscribe(
        bytes -> {
            // 成功处理
        },
        error -> {
            if (error instanceof BleDisconnectedException) {
                // 处理断开连接
                Log.e(TAG, "Device disconnected during read");
            } else if (error instanceof BleGattException) {
                BleGattException gattError = (BleGattException) error;
                int status = gattError.getStatus();
                
                switch (status) {
                    case 0x03: // GATT_WRITE_NOT_PERMITTED
                        Log.e(TAG, "Read not permitted by device");
                        break;
                    case 0x0E: // GATT_INSUFFICIENT_AUTHENTICATION
                        Log.e(TAG, "Need to bond/pair first");
                        // 可能需要配对
                        break;
                    default:
                        Log.e(TAG, "GATT error: " + status);
                }
            } else {
                Log.e(TAG, "Unknown error", error);
            }
        }
    );
```

#### 案例 2: 重试机制

```java
device.establishConnection(false)
    .flatMapSingle(rxBleConnection -> 
        rxBleConnection.readCharacteristic(characteristicUUID)
            // 重试最多 3 次，每次间隔 500ms
            .retry((retryCount, exception) -> {
                if (retryCount < 3) {
                    if (exception instanceof BleGattException) {
                        BleGattException gattError = (BleGattException) exception;
                        // 某些错误可以重试
                        int status = gattError.getStatus();
                        if (status == 0x0E) { // INSUFFICIENT_AUTHENTICATION
                            return true; // 重试
                        }
                    }
                }
                return false; // 不重试，传播错误
            })
    )
    .retryWhen(throwableObservable ->
        throwableObservable.zipWith(
            Observable.range(1, 3),
            (throwable, retryCount) -> retryCount
        )
        .flatMap(retryCount -> 
            Observable.timer(500 * retryCount, TimeUnit.MILLISECONDS)
        )
    )
    .subscribe(...);
```

#### 案例 3: 超时控制

```java
device.establishConnection(false)
    .timeout(30, TimeUnit.SECONDS) // 30 秒超时
    .flatMapSingle(rxBleConnection -> 
        rxBleConnection.readCharacteristic(characteristicUUID)
            .timeout(5, TimeUnit.SECONDS) // 单个读操作 5 秒
    )
    .subscribe(
        bytes -> { /* ... */ },
        error -> {
            if (error instanceof TimeoutException) {
                Log.e(TAG, "Operation timeout");
            }
        }
    );
```

### 3. 常见错误场景

#### 错误 1: Status 133 (GATT_ERROR - 0x85)

```
原因: 连接失败，通常是硬件或驱动问题
解决方案:
  1. 重新扫描设备
  2. 清除配对信息后重新配对
  3. 重启手机蓝牙
  4. 尝试不同的 autoConnect 值
```

```java
device.establishConnection(false)
    .retryWhen(errors -> 
        errors.zipWith(
            Observable.range(1, 3),
            (throwable, retryCount) -> {
                if (throwable instanceof BleGattException) {
                    BleGattException e = (BleGattException) throwable;
                    if (e.getStatus() == 0x85) { // Status 133
                        return retryCount;
                    }
                }
                throw throwable;
            }
        )
        .flatMap(retryCount -> 
            Observable.timer(1000 * retryCount, TimeUnit.MILLISECONDS)
        )
    )
    .subscribe(...);
```

#### 错误 2: UndeliverableException

```
原因: Observable 发送 onError 时，没有订阅者或订阅者已 dispose
症状: App Crash，日志中看到 "UndeliverableException"

解决方案:
  1. 及时 dispose Subscription
  2. 使用 CompositeDisposable 管理多个 Subscription
  3. 在 Activity/Fragment 的 onDestroy 中 dispose
```

```java
private CompositeDisposable disposables = new CompositeDisposable();

public void startScanning() {
    disposables.add(
        rxBleClient.scanBleDevices()
            .subscribe(scanResult -> { /* ... */ })
    );
}

@Override
public void onDestroy() {
    disposables.clear(); // 清理所有订阅
    super.onDestroy();
}
```

#### 错误 3: 连接断开导致的级联错误

```
场景: 
  1. 正在读取数据
  2. 设备突然断开连接
  3. 读操作返回 onError

处理方式:
```

```java
device.observeConnectionStateChanges()
    .subscribe(connectionState -> {
        Log.d(TAG, "Connection state: " + connectionState);
        // CONNECTED, CONNECTING, DISCONNECTED, DISCONNECTING
    });

device.establishConnection(false)
    .flatMapSingle(connection -> 
        readDataContinuously(connection)
    )
    .doOnError(error -> {
        if (error instanceof BleDisconnectedException) {
            // 可以在此重新发起连接
            attemptReconnect();
        }
    })
    .subscribe(...);
```

---

## 四、原理深度解析

### 1. 线程模型

```
RxAndroidBle 的线程策略:

┌─────────────────────────────────────────────────┐
│          User's Thread (Main/Any)               │
│         (subscribe() 调用者)                     │
└────────────────┬────────────────────────────────┘
                 │ Observer
                 ↓
┌─────────────────────────────────────────────────┐
│     RxJava Default Scheduler                    │
│   (通常是 Schedulers.computation())              │
└────────────────┬────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────┐
│    BluetoothGattCallback 线程 (Binder Pool)      │
│  (系统内部线程，不是 Main Thread)                 │
│                                                 │
│  onConnectionStateChange()                      │
│  onCharacteristicRead()                         │
│  onCharacteristicWrite()                        │
│  onCharacteristicChanged()                      │
└────────────────┬────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────┐
│  Schedulers.io() (或其他指定的 Scheduler)       │
│  (RxAndroidBle 内部线程管理)                     │
└────────────────┬────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────┐
│          Observer's Thread                      │
│     (如指定了 observeOn 则在该线程)              │
└─────────────────────────────────────────────────┘
```

**具体实现**：

```java
// RxAndroidBle 内部的 Scheduler 管理
public Single<byte[]> readCharacteristic(UUID uuid) {
    return Single.create(emitter -> {
        // BluetoothGattCallback 线程
        gattOperationQueue.queueOperation(() -> {
            bluetoothGatt.readCharacteristic(characteristic);
        });
    })
    .subscribeOn(Schedulers.io()) // 订阅在 IO 线程
    .observeOn(Schedulers.io()); // 观测也在 IO 线程
}

// 用户代码
readCharacteristic(uuid)
    .observeOn(AndroidSchedulers.mainThread()) // 转到主线程
    .subscribe(bytes -> {
        // 现在在主线程中，可以安全更新 UI
        updateUI(bytes);
    });
```

### 2. GATT 操作队列机制

**为什么需要队列？**

Android GATT 实现的限制：

- 同时只能有一个待处理的 GATT 操作
- 发起操作前必须等待前一个操作的回调
- 违反此规则会导致操作被静默忽略或返回 false

**实现原理**：

```java
class GattOperationQueue {
    private Queue<Runnable> operationQueue = new LinkedList<>();
    private AtomicBoolean isOperationInProgress = new AtomicBoolean(false);
    
    public void queueOperation(Runnable operation) {
        synchronized (operationQueue) {
            operationQueue.add(operation);
            if (!isOperationInProgress.get()) {
                executeNext();
            }
        }
    }
    
    private void executeNext() {
        Runnable operation = operationQueue.poll();
        if (operation != null) {
            isOperationInProgress.set(true);
            try {
                operation.run(); // 执行 GATT 操作
            } catch (Exception e) {
                Log.e(TAG, "Operation failed", e);
                // 错误后继续执行下一个操作
                handleNext();
            }
        }
    }
    
    public void onOperationComplete() {
        // 由 GATT 回调触发
        isOperationInProgress.set(false);
        synchronized (operationQueue) {
            executeNext();
        }
    }
    
    // GATT Callback
    @Override
    public void onCharacteristicRead(BluetoothGatt gatt, 
                                    BluetoothGattCharacteristic characteristic,
                                    int status) {
        // 处理结果...
        
        gattOperationQueue.onOperationComplete(); // 触发下一个操作
    }
}
```

### 3. 连接生命周期管理

```
RxBleConnectionImpl
    ├─ 维持 BluetoothGatt 对象引用
    ├─ 提供 GATT 操作方法（read, write, etc.）
    └─ 管理连接资源清理
    
当 Observable 被 dispose 时:
    ├─ 调用 BluetoothGatt.close()
    ├─ 清理所有待处理的 GATT 操作
    ├─ 禁用所有通知/指示
    └─ 释放 wake lock (如果有)
```

**资源泄漏风险**：

```java
// ❌ 错误: 不 dispose 会导致连接泄漏
Disposable d = device.establishConnection(false)
    .subscribe(...);
// ... 没有调用 d.dispose()，连接永远打开

// ✅ 正确: 及时 dispose
CompositeDisposable disposables = new CompositeDisposable();
disposables.add(
    device.establishConnection(false)
        .subscribe(...)
);

@Override
public void onDestroy() {
    disposables.clear();
}
```

### 4. 状态观测原理

```java
// observeConnectionStateChanges() 是 Hot Observable
device.observeConnectionStateChanges()
    .subscribe(state -> {
        // 立即收到当前状态
        // 后续每当状态变化时收到新状态
    });
```

**实现方式**：

```java
public Observable<RxBleConnectionState> observeConnectionStateChanges() {
    // 返回 ReplaySubject (Hot Observable)
    // - ReplaySubject 的作用: 保存最后一个值，新订阅者立即收到
    // - 这样后订阅的观察者也能知道当前状态
    
    return connectionStateRelay
        .asObservable()
        .distinctUntilChanged(); // 只在状态变化时发送
}

// GATT 回调更新状态
@Override
public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
    RxBleConnectionState state = convertGattStateToRxBleState(newState);
    connectionStateRelay.accept(state); // 发送新状态给所有订阅者
}
```

### 5. 扫描的 Cold Observable 特性

```
Cold Observable vs Hot Observable

Cold Observable (扫描)
    ├─ 每个订阅都会产生新的扫描操作
    ├─ 多个订阅 = 多次扫描
    └─ 场景: 需要独立的扫描进程
    
例如:
    Observable scanObservable = rxBleClient.scanBleDevices();
    
    // 订阅 1: 开始扫描 A
    scanObservable.subscribe(subscriber1);
    
    // 订阅 2: 开始另一个扫描 B（与 A 并行）
    scanObservable.subscribe(subscriber2);
    
    // 现在有两个并行扫描进程

Hot Observable (连接状态变化)
    ├─ 所有订阅都共享同一个源
    ├─ 已发出的值不会重新发送（除非使用 ReplaySubject）
    └─ 场景: 共享蓝牙适配器状态
```

---

## 五、性能优化建议

### 1. 避免重复连接

```java
// ❌ 每次读写都建立新连接
device.establishConnection(false)
    .flatMapSingle(connection -> connection.readCharacteristic(uuid1))
    .subscribe(...);

device.establishConnection(false)
    .flatMapSingle(connection -> connection.readCharacteristic(uuid2))
    .subscribe(...);

// ✅ 复用连接
device.establishConnection(false)
    .flatMap(connection ->
        connection.readCharacteristic(uuid1)
            .flatMapSingle(data1 -> 
                connection.readCharacteristic(uuid2)
                    .map(data2 -> Pair.of(data1, data2))
            )
    )
    .subscribe(...);
```

### 2. 缓存 GATT 结果

```java
// ✅ 缓存 Service 列表避免重复 discovery
public Single<List<BluetoothGattService>> getServices(RxBleConnection connection) {
    return Single.fromCallable(() -> connection.getBluetoothGatt().getServices())
        .cache(); // 缓存结果
}
```

### 3. 智能重试策略

```java
// ✅ 只重试可恢复的错误
readCharacteristic(uuid)
    .retryWhen(errors ->
        errors.zipWith(
            Observable.range(1, 3),
            (error, retryCount) -> {
                // 只重试 GATT 错误，不重试权限错误
                if (error instanceof BleGattException && retryCount < 3) {
                    return retryCount;
                }
                throw error;
            }
        )
        .flatMap(retryCount ->
            Observable.timer(100L * retryCount, TimeUnit.MILLISECONDS)
        )
    )
    .subscribe(...);
```

---

## 六、面试重点总结

### 核心概念

1. **Cold vs Hot Observable**: 扫描是 cold（每次订阅新扫描），状态是 hot（共享状态）
2. **GATT 操作队列**: 必须串行执行，library 内部维护队列
3. **线程管理**: 回调在 Binder 线程，转换到 IO 线程再给用户
4. **连接生命周期**: dispose 时自动清理资源，需要及时 dispose 避免泄漏
5. **错误处理**: 区分 GATT 错误、断开错误、权限错误等

### 常见问题

- Q: 为什么 setupNotification 返回二层 Observable？ A: 第一层控制通知的启用，第二层推送实际数据
    
- Q: 如何避免 Status 133？ A: 使用 autoConnect=true，清除配对，重启蓝牙
    
- Q: 为什么需要 Characteristic UUID？ A: GATT 是树形结构，需要 UUID 定位具体属性