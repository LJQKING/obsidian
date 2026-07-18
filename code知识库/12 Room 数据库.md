# Room 数据库简介

Room 是 Google Jetpack 中提供的 ORM（对象关系映射）数据库框架，本质上是对 SQLite 的封装。

相比直接使用 SQLite：

✅ 编译时 SQL 检查

✅ 自动对象映射

✅ LiveData、Flow 支持

✅ 与 MVVM 完美结合

---

# 一、添加依赖

app/build.gradle

```gradle
dependencies {

    // Room
    implementation "androidx.room:room-runtime:2.6.1"
    annotationProcessor "androidx.room:room-compiler:2.6.1"

    // 可选 RxJava
    implementation "androidx.room:room-rxjava3:2.6.1"

    // 可选 Kotlin 协程
    implementation "androidx.room:room-ktx:2.6.1"
}
```

Java 项目：

```gradle
annotationProcessor "androidx.room:room-compiler:2.6.1"
```

---

# 二、创建实体类(Entity)

例如设备表：

```java
@Entity(tableName = "device")
public class Device {

    @PrimaryKey(autoGenerate = true)
    private int id;

    private String name;

    private String mac;

    private int battery;

    public Device(String name, String mac, int battery) {
        this.name = name;
        this.mac = mac;
        this.battery = battery;
    }

    // Getter Setter
}
```

对应 SQLite：

```sql
CREATE TABLE device(
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    mac TEXT,
    battery INTEGER
)
```

---

# 三、创建 Dao

DAO(Data Access Object)

负责增删改查。

```java
@Dao
public interface DeviceDao {

}
```

---

# 四、插入数据(Insert)

## 单条插入

```java
@Insert
long insert(Device device);
```

使用：

```java
Device device =
        new Device("耳机","AA:BB:CC",90);

long id = dao.insert(device);
```

返回：

```java
id = 1
```

---

## 批量插入

```java
@Insert
List<Long> insertAll(List<Device> devices);
```

```java
dao.insertAll(list);
```

---

# 五、查询数据(Query)

## 查询全部

```java
@Query("SELECT * FROM device")
List<Device> getAll();
```

调用：

```java
List<Device> list = dao.getAll();
```

---

## 根据ID查询

```java
@Query("SELECT * FROM device WHERE id=:id")
Device getDeviceById(int id);
```

```java
Device device = dao.getDeviceById(1);
```

---

## 根据Mac查询

```java
@Query("SELECT * FROM device WHERE mac=:mac")
Device getDeviceByMac(String mac);
```

```java
Device device =
        dao.getDeviceByMac("AA:BB:CC");
```

---

## 模糊查询

```java
@Query("SELECT * FROM device WHERE name LIKE '%' || :name || '%'")
List<Device> search(String name);
```

```java
dao.search("耳机");
```

生成SQL：

```sql
SELECT * FROM device
WHERE name LIKE '%耳机%'
```

---

# 六、更新数据(Update)

```java
@Update
int update(Device device);
```

使用：

```java
device.setBattery(80);

int row = dao.update(device);
```

返回：

```java
1
```

表示影响1行。

---

## SQL更新

```java
@Query("UPDATE device SET battery=:battery WHERE id=:id")
int updateBattery(int id,int battery);
```

```java
dao.updateBattery(1,95);
```

---

# 七、删除数据(Delete)

## 删除单条

```java
@Delete
int delete(Device device);
```

```java
dao.delete(device);
```

---

## 删除全部

```java
@Query("DELETE FROM device")
void deleteAll();
```

---

## 根据ID删除

```java
@Query("DELETE FROM device WHERE id=:id")
int deleteById(int id);
```

---

# 八、创建Database

```java
@Database(
        entities = {Device.class},
        version = 1,
        exportSchema = false
)
public abstract class AppDatabase extends RoomDatabase {

    public abstract DeviceDao deviceDao();
}
```

---

# 九、初始化Room

推荐单例模式

```java
public class DatabaseManager {

    private static volatile AppDatabase database;

    public static AppDatabase getDatabase(Context context){

        if(database == null){

            synchronized (DatabaseManager.class){

                if(database == null){

                    database =
                            Room.databaseBuilder(
                                    context.getApplicationContext(),
                                    AppDatabase.class,
                                    "device_db"
                            )
                            .build();
                }
            }
        }

        return database;
    }
}
```

使用：

```java
AppDatabase db =
        DatabaseManager.getDatabase(this);

DeviceDao dao = db.deviceDao();
```

---

# 十、Room不能在主线程操作

错误：

```java
Cannot access database on the main thread
```

必须子线程：

```java
Executors.newSingleThreadExecutor()
        .execute(() -> {

            Device device =
                    new Device("耳机","AA",90);

            dao.insert(device);

        });
```

---

# 十一、结合LiveData自动刷新

Dao：

```java
@Query("SELECT * FROM device")
LiveData<List<Device>> getAllDevice();
```

Activity：

```java
dao.getAllDevice().observe(this, devices -> {

    adapter.setNewData(devices);

});
```

数据库变化时：

```java
insert()
update()
delete()
```

RecyclerView 自动刷新。

---

# 十二、数据库升级(Migration)

Version 1

```java
@Entity
public class Device {

    @PrimaryKey(autoGenerate = true)
    public int id;

    public String name;
}
```

Version 2 增加字段：

```java
public String mac;
```

数据库版本：

```java
version = 2
```

Migration：

```java
static Migration MIGRATION_1_2 =
        new Migration(1,2) {

            @Override
            public void migrate(
                    @NonNull SupportSQLiteDatabase db) {

                db.execSQL(
                        "ALTER TABLE device " +
                        "ADD COLUMN mac TEXT"
                );
            }
        };
```

注册：

```java
Room.databaseBuilder(
        context,
        AppDatabase.class,
        "device_db"
)
.addMigrations(MIGRATION_1_2)
.build();
```

---

# 十三、实际项目(MVVM)结构

```text
com.xxx.app
│
├── db
│   ├── entity
│   │    └── Device.java
│   │
│   ├── dao
│   │    └── DeviceDao.java
│   │
│   ├── AppDatabase.java
│   │
│   └── DatabaseManager.java
│
├── repository
│   └── DeviceRepository.java
│
├── viewmodel
│   └── DeviceViewModel.java
│
└── ui
    └── DeviceActivity.java
```

---

# Room 完整 CRUD 示例

```java
// 初始化
AppDatabase db =
        DatabaseManager.getDatabase(this);

DeviceDao dao = db.deviceDao();

// 插入
dao.insert(new Device("耳机","AA",90));

// 查询
List<Device> list = dao.getAll();

// 更新
Device device = list.get(0);
device.setBattery(50);
dao.update(device);

// 删除
dao.delete(device);
```

如果你现在的项目是 **蓝牙设备管理（RxAndroidBle + RecyclerView）**，我还可以给你写一个 **Room + MVVM + LiveData + RecyclerView 完整设备缓存案例**（扫描到蓝牙设备自动保存数据库，并在 `EquipmentFragment` 中展示）。