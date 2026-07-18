如果你要在 **Android Java 项目中接入商米（Sunmi）人脸识别 SDK**，并实现：

> 点击按钮 → 打开人脸识别页面 → 返回识别结果 → 显示用户信息

建议先参考你提供的仓库：

[SunmiFaceSDK Gitee仓库](https://gitee.com/EpibolyWorkspace/SunmiFaceSDK?utm_source=chatgpt.com)

由于该仓库是商米设备专用 SDK，不同版本接口略有区别，下面按照商米人脸设备常见接入方式给出完整实现思路。

---

# 一、SDK架构

一般流程：

```text
App
 │
 ├── 人脸注册
 │      ↓
 │  SDK提取特征值
 │      ↓
 │  保存到本地数据库
 │
 └── 人脸识别
        ↓
   Camera预览
        ↓
   Face Detect
        ↓
   Face Feature
        ↓
   1:N比对
        ↓
   返回用户
```

---

# 二、导入SDK

## app/build.gradle

例如：

```gradle
dependencies {

    implementation fileTree(
            dir: "libs",
            include: ["*.jar", "*.aar"]
    )

}
```

把：

```text
SunmiFaceSDK.aar
```

放入：

```text
app/libs/
```

---

# 三、申请权限

AndroidManifest.xml

```xml
<uses-permission android:name="android.permission.CAMERA"/>

<uses-permission
    android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>

<uses-permission
    android:name="android.permission.READ_EXTERNAL_STORAGE"/>
```

---

# 四、初始化SDK

Application

```java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        FaceSDKManager.getInstance()
                .init(this);
    }
}
```

---

# 五、打开识别页面

布局：

```xml
<Button
    android:id="@+id/btnFace"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="开始识别"/>
```

Activity

```java
btnFace.setOnClickListener(v -> {

    Intent intent =
            new Intent(MainActivity.this,
                    FaceRecognitionActivity.class);

    startActivityForResult(intent,1001);

});
```

---

# 六、人脸识别Activity

```java
public class FaceRecognitionActivity
        extends AppCompatActivity {

    private FaceCameraView cameraView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_face);

        cameraView =
                findViewById(R.id.faceCamera);

        startFaceRecognition();
    }

    private void startFaceRecognition() {

        FaceSDKManager.getInstance()
                .startRecognition(cameraView,
                        new FaceCallback() {

            @Override
            public void onSuccess(FaceUser user) {

                Intent intent = new Intent();

                intent.putExtra(
                        "userName",
                        user.getName());

                setResult(RESULT_OK,intent);

                finish();
            }

            @Override
            public void onFail(String msg) {

                Toast.makeText(
                        FaceRecognitionActivity.this,
                        msg,
                        Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```

---

# 七、接收识别结果

MainActivity

```java
@Override
protected void onActivityResult(
        int requestCode,
        int resultCode,
        Intent data) {

    super.onActivityResult(
            requestCode,
            resultCode,
            data);

    if(requestCode == 1001
            && resultCode == RESULT_OK){

        String userName =
                data.getStringExtra("userName");

        tvResult.setText(
                "识别成功：" + userName);
    }
}
```

---

# 八、人脸注册

很多商米项目都会先录入人脸。

例如：

```java
Bitmap bitmap =
        BitmapFactory.decodeFile(path);

FaceSDKManager.getInstance()
        .registerFace(
                "10001",
                "张三",
                bitmap,
                new RegisterCallback() {

            @Override
            public void onSuccess() {

                Toast.makeText(
                        MainActivity.this,
                        "注册成功",
                        Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onFail(String msg) {

            }
        });
```

数据库：

```java
id
name
faceFeature
```

存储：

```java
BLOB
```

或：

```java
Base64
```

---

# 九、MVVM封装

## FaceRepository

```java
public class FaceRepository {

    public void startRecognize(
            FaceCameraView cameraView,
            FaceCallback callback){

        FaceSDKManager
                .getInstance()
                .startRecognition(
                        cameraView,
                        callback);
    }
}
```

---

## FaceViewModel

```java
public class FaceViewModel
        extends ViewModel {

    private MutableLiveData<String>
            faceResult =
            new MutableLiveData<>();

    public LiveData<String> getFaceResult() {
        return faceResult;
    }

    public void recognize(
            FaceCameraView cameraView){

        new FaceRepository()
                .startRecognize(
                        cameraView,
                        new FaceCallback() {

            @Override
            public void onSuccess(
                    FaceUser user) {

                faceResult.postValue(
                        user.getName());
            }

            @Override
            public void onFail(
                    String msg) {

                faceResult.postValue(
                        msg);
            }
        });
    }
}
```

---

# 十、工业项目常用封装

实际项目不会直接调用 SDK。

一般会封装：

```java
FaceManager
```

```java
public class FaceManager {

    private static FaceManager instance;

    public static FaceManager get(){

        if(instance == null){
            instance = new FaceManager();
        }

        return instance;
    }

    public void startRecognize(
            FaceCameraView cameraView,
            FaceResultListener listener){

        FaceSDKManager.getInstance()
                .startRecognition(
                        cameraView,
                        listener);
    }
}
```

业务层：

```java
FaceManager.get()
        .startRecognize(...)
```

这样以后更换：

- 商米SDK
    
- Face++
    
- 百度AI
    
- ArcSoft
    
- FaceAISDK
    

业务代码无需修改。([GitHub](https://github.com/FaceAISDK/FaceAISDK_Android?utm_source=chatgpt.com "GitHub - FaceAISDK/FaceAISDK_Android: Android on_device Face Recognition 、 Liveness detection and 1:N & M:N Face Search SDK 离线版设备端人脸识别 活体检测 以及1:N M:N 人脸搜索SDK · GitHub"))

---

# 推荐的项目结构

```text
app
├── ui
│    ├── MainActivity
│    └── FaceRecognitionActivity
│
├── viewmodel
│    └── FaceViewModel
│
├── repository
│    └── FaceRepository
│
├── manager
│    └── FaceManager
│
├── sdk
│    └── SunmiFaceSDK
│
└── room
     ├── FaceEntity
     ├── FaceDao
     └── FaceDatabase
```

如果你能把 **SunmiFaceSDK 仓库中的具体 aar 包、Demo源码或者 API 文档发出来**，我可以直接按照该 SDK 的真实接口，为你写一套 **Android Java + MVVM + Room + RecyclerView + 商米人脸识别完整工程代码**（包括人脸录入、删除、1:1验证、1:N识别）。