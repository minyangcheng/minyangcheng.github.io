---
title: 怎样爬取竞争公司app上的加密数据
date: 2018-08-18 17:07:09
tags: android
---
最近公司产品有个需求，获取竞争公司的app应用上一些数据。当时听了后也有点蒙逼，要知道现在的app都是混淆并且加固过了的，而且数据接口都进行过加密处理的，这样就给爬取数据增加了一些难度。思考了一段时间，想出来本文的解决方案。此方案难点在于需要对app进行脱壳处理，寻找需要hook的方法。

<!-- more -->

### 技术点

1. app加固原理和一些常用的脱壳技术
2. xposed框架使用
3. android中各种网络请求框架的数据返回处理点
4. ui automator自动化技术
5. android应用开发基础
6. node开发基础
7. adb命令用法

### 实现步骤

#### 对加固过app进行脱壳处理

市面上的app大多数app都是用360、腾讯等其他厂商的免费加固，面对这些app的脱壳，有很多方法如ZjDroid、DexHunter、dex2oat法。

我这里采用的是dex2oat法，这种方式对360加固方式脱壳成功率挺高的，其原理：android系统art模式下，apk文件首次打开时后会用dex2oat生成oat时，这个时候内存中的DEX是完整的，所以只需要在这个时候将dex保存下来就ok了。看雪上有个大牛专门采用这种方法做了一个android4.4版本脱壳系统镜像，我们只需要在android虚拟机上安装镜像，修改系统启动模式为ART，重启机器，安装apk，打开后就会自动脱可。

以下是脱壳过程中产生的日志：

```
07-27 02:19:10.532 3484-3484/? I/dex2oat: dex2oat: /data/dalvik-cache/system@app@Exchange2.apk@classes.dex
07-27 02:19:11.652 3484-3484/? I/dex2oat: harvey:dex file name-->/system/app/Exchange2.apk
07-27 02:19:23.712 3484-3484/? W/dex2oat: Verification of void com.google.common.net.TldPatterns.<clinit>() took 1.576255482s
07-27 02:26:48.362 3545-3545/? I/dex2oat: dex2oat: /data/dalvik-cache/system@priv-app@DefaultContainerService.apk@classes.dex
07-27 02:26:48.452 3545-3545/? I/dex2oat: harvey:dex file name-->/system/priv-app/DefaultContainerService.apk
07-27 02:26:54.422 3547-3547/? I/dex2oat: dex2oat: /data/dalvik-cache/data@app@***8@classes.dex
07-27 02:26:54.872 3547-3547/? I/dex2oat: harvey:dex file name-->/data/app/****-1.apk
07-27 02:26:56.592 3565-3565/? I/dex2oat: dex2oat: /data/dalvik-cache/system@app@Gallery2.apk@classes.dex
07-27 02:26:57.232 3565-3565/? I/dex2oat: harvey:dex file name-->/system/app/Gallery2.apk
07-27 02:27:22.152 3592-3592/? I/dex2oat: dex2oat: /data/dalvik-cache/system@app@PicoTts.apk@classes.dex
07-27 02:27:22.282 3592-3592/? I/dex2oat: harvey:dex file name-->/system/app/PicoTts.apk
07-27 02:34:00.692 3677-3677/? I/dex2oat: dex2oat: /data/data/****/code_cache/rocoo-dexes/rocoo.dex
07-27 02:34:00.752 3677-3677/? I/dex2oat: harvey:dex file name-->/data/data/****/files/hotfix/rocoo.dex
07-27 02:34:05.572 3688-3688/? I/dex2oat: dex2oat: /data/data/****/.jiagu/classes.oat
07-27 02:34:16.912 3688-3688/? I/dex2oat: harvey:dex file name-->/data/data/****/.jiagu/classes.dex
07-27 02:34:17.112 3688-3688/? I/dex2oat: harvey:write tartget dex file successfull->>/data/data/****/.jiagu/classes.dex_8238736.dex
07-27 02:34:17.112 3688-3688/? I/dex2oat: harvey:dex file name-->/data/data/****/.jiagu/classes2.dex
07-27 02:34:17.112 3688-3688/? I/dex2oat: harvey:write tartget dex file successfull->>/data/data/****/.jiagu/classes2.dex_138268.dex
07-27 02:34:17.122 3688-3688/? I/dex2oat: harvey:dex file name-->/data/data/****/.jiagu/classes3.dex
07-27 02:34:17.122 3688-3688/? I/dex2oat: harvey:write tartget dex file successfull->>/data/data/****/.jiagu/classes3.dex_214904.dex
```

#### 在代码中寻找请求数据返回的hook点

通过查看apk的mainifest文件，获取包名，用jadx打开脱壳后的classes.dex文件，通过包名寻找到代码，我们这里是为了寻找接口数据的hook点，所以我一般会先去寻找登录界面的代码，然后通过登录界面找出接口调用涉及到的那些类，在通过这些类判断出该app用的是哪个数据请求框架，常见的也就几种：volley、retrofit+okhttp、xutil、asynchttpclient等。找到app用的是哪个请求框架后，我们就可以推断出具体需要hook的位置了。

例如：我发现app用的是retrofit，那就可以推断出它会使用`Converter<ResponseBody, T>`对返回的结果进行json反序列化处理，因此找到了以下类：

```
import com.google.gson.f;
import com.google.gson.s;
import java.io.BufferedReader;
import java.io.Reader;
import java.io.StringReader;
import okhttp3.ae;
import retrofit2.d;

/* compiled from: RsaGsonResponseBodyConverter */
public class i<T> implements d<ae, T> {
    private final f a;
    private final s<T> b;

    public /* synthetic */ Object b(Object obj) {
        return a((ae) obj);
    }

    i(f fVar, s<T> sVar) {
        this.a = fVar;
        this.b = sVar;
    }

    public T a(ae aeVar) {
        String a = a(aeVar.f());
        if (a.endsWith("\r\n")) {
            a = a.replace("\r\n", "");
        }
        try {
            T b = this.b.b(this.a.a(new StringReader(a)));
            return b;
        } finally {
            aeVar.close();
        }
    }

    private String a(Reader reader) {
        BufferedReader bufferedReader = new BufferedReader(reader);
        StringBuilder stringBuilder = new StringBuilder();
        while (true) {
            String readLine = bufferedReader.readLine();
            if (readLine != null) {
                stringBuilder.append(readLine);
                stringBuilder.append("\r\n");
            } else {
                bufferedReader.close();
                return stringBuilder.toString();
            }
        }
    }
}
```

#### 书写xposed模块

该模块主要是为了hook掉指定的方法，进行数据采集，并且采集的数据会在记录在文件里面，以便上传服务器分析。具体xposed的使用请自行查看官方文档。

* 定义一个基类，统一处理mutil-dex hook

```
/**
 * Created by minych on 18-7-24.
 */

public abstract class BaseHook implements IXposedHookLoadPackage {

    protected static final String TAG = "ZEOK";

    public String packageName;
    public String hookClassName;

    public BaseHook(String packageName, String hookClassName) {
        if (TextUtils.isEmpty(packageName) || TextUtils.isEmpty(hookClassName)) {
            throw new IllegalArgumentException();
        }
        this.packageName = packageName;
        this.hookClassName = hookClassName;
    }

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam loadPackageParam) throws Throwable {
        if (loadPackageParam.packageName.equalsIgnoreCase(packageName)) {
            XposedHelpers.findAndHookMethod(Application.class, "attach", Context.class, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    xLog(packageName, "加载成功");
                    ClassLoader classLoader = ((Context) param.args[0]).getClassLoader();
                    Class<?> hookClass = null;
                    try {
                        hookClass = classLoader.loadClass(hookClassName);
                        handleLoadPackage(hookClass);
                    } catch (Exception e) {
                        Logger.e(TAG, e);
                    }
                }
            });
        }
    }

    public void xLog(String tag, Object object) {
        if (tag == null || object == null) {
            return;
        }
        XposedBridge.log(tag + " : " + object.toString());
    }

    public String getCurrentActivityName() {
        try {
            ActivityManager manager = (ActivityManager) AndroidAppHelper.currentApplication().getSystemService(Context.ACTIVITY_SERVICE);
            ActivityManager.RunningTaskInfo info = manager.getRunningTasks(1).get(0);
            String shortClassName = info.topActivity.getClassName();
            return shortClassName;
        } catch (SecurityException e) {
            e.printStackTrace();
        }
        return "";
    }

    public abstract void handleLoadPackage(Class clazz) throws Exception;

}
```
* 定义具体应用的hook

```
/**
 * Created by minych on 18-7-24.
 */

public class ××××CheHook extends BaseHook {

    public MaiHaoCheHook() {
        super("****", "××××.service.b.i");
    }

    @Override
    public void handleLoadPackage(Class clazz) {
        XposedHelpers.findAndHookMethod(clazz, "a", Reader.class, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                Object resultObj = param.getResult();
                if (resultObj != null) {
                    JSONObject jsonObject = new JSONObject();
                    jsonObject.put("packageName", packageName);
                    jsonObject.put("activityName", getCurrentActivityName());
                    jsonObject.put("responseData", resultObj.toString());
                    String content = jsonObject.toString();
                    Logger.d(TAG, content + "\n\n");
                    FileLogger.log(content);
                }
            }
        });
    }

}
```

* 收集数据到本地文件

   ```
   public class FileLogger {
   
       private static final String TAG = "HookLogger";
       private static final String LOG_DIR = "hookLogs";
       private static final long MAX_SIZE = 10 * 1024 * 1024;
   
       public static void log(String content) {
           if (TextUtils.isEmpty(content)) {
               return;
           }
           content = content + "\n";
           try {
               File logFile = getOutputFile();
               FileOutputStream fileOutputStream = new FileOutputStream(logFile, true);
               fileOutputStream.write(content.getBytes());
               fileOutputStream.flush();
               fileOutputStream.close();
           } catch (Exception e) {
               Logger.e(TAG, e);
           }
       }
   
       public static File getOutputFile() throws Exception {
           File hookLogDir = getLogDir();
           SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
           String today = simpleDateFormat.format(Calendar.getInstance().getTime());
           File file = new File(hookLogDir, today + ".log");
           long fileSize = file.length();
           if (file.exists() && fileSize > MAX_SIZE) {
               String indicator = getIndicatorStr();
               File renameFile = new File(hookLogDir, today + "-" + indicator + ".log");
               file.renameTo(renameFile);
           }
           return file;
       }
   
       public static File[] scanCompleteFile() throws Exception {
           File dir = getLogDir();
           SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
           String today = simpleDateFormat.format(new Date());
           File[] fileArr = dir.listFiles(new FilenameFilter() {
               @Override
               public boolean accept(File dir, String name) {
                   return !(today + ".log").equals(name);
               }
           });
           if (fileArr != null && fileArr.length > 0) {
               Arrays.sort(fileArr, new Comparator<File>() {
                   @Override
                   public int compare(File o1, File o2) {
                       return o1.getName().compareTo(o2.getName());
                   }
               });
           }
           return fileArr;
       }
   
       public static File getLogDir() {
           File file = new File(Environment.getExternalStorageDirectory(), LOG_DIR);
           if (!file.exists()) {
               file.mkdirs();
           }
           return file;
       }
   
       public static String getIndicatorStr() {
           return System.currentTimeMillis() + "";
       }
   
   }
   ```

#### 书写定期上传日志文件服务

定义一个前台服务，定期上传xposed模块收集完成的数据

```
/**
 * Created by minych on 18-7-28.
 */

public class UploadHookHttpResponseService extends Service {

    private static final String TAG = "UploadHookHttpResponseService";
    private boolean exitFlag;

    @Override
    public void onCreate() {
        super.onCreate();
        foreground();
        exitFlag = false;
        startThread();
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        return START_STICKY;
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onDestroy() {
        exitFlag = true;
    }

    private void foreground() {
        NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
        Intent intent = new Intent(this, MainActivity.class);
        PendingIntent pIntent = PendingIntent.getActivity(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
        builder.setContentIntent(pIntent).
                setContentTitle("上传爬虫数据服务")
                .setContentText("正在运行中.....")
                .setSmallIcon(R.mipmap.ic_launcher)
                .setAutoCancel(false);
        startForeground(1314, builder.build());
    }

    private void startThread() {
        new Thread() {
            @Override
            public void run() {
                uploadFiles();
            }
        }.start();
    }

    private void uploadFiles() {
        while (true) {
            if (exitFlag) {
                return;
            }
            try {
                try {
                    File[] files = FileLogger.scanCompleteFile();
                    if (files != null && files.length > 0) {
                        File file = null;
                        for (int i = 0; i < files.length; i++) {
                            file = files[i];
                            Logger.d(TAG, "start upload file " + file.getAbsolutePath());
                            RequestBody fileBody = RequestBody.create(MediaType.parse("application/octet-stream"), file);
                            RequestBody requestBody = new MultipartBody.Builder()
                                    .setType(MultipartBody.FORM)
                                    .addFormDataPart("file", file.getName(), fileBody)
                                    .build();

                            Request request = new Request.Builder()
                                    .url(BuildConfig.API_SERVER_URL + "/uploadLog")
                                    .post(requestBody)
                                    .build();
                            Response response = OkHttpUtil.getOkHttpClient().newCall(request).execute();
                            Logger.d(TAG, "end upload file " + file.getAbsolutePath() + " responseCode=" + response.code());
                            if (response.code() == 200) {
                                String responseBody = response.body().string();
                                if ("success".equals(responseBody)) {
                                    saveUploadSuccRecord(file);
                                    file.delete();
                                }
                            }
                        }
                    }
                } catch (Exception e) {
                    Logger.e(TAG, e);
                }
                Thread.sleep(15 * 60 * 1000);
            } catch (Exception e) {
                Logger.e(TAG, e);
            }
        }
    }

    public void saveUploadSuccRecord(File file) {
        String str = PreferencesUtils.getString(this, "succUploadFiles");
        List<String> succUploadList = null;
        if (TextUtils.isEmpty(str)) {
            succUploadList = new ArrayList<>();
        } else {
            succUploadList = GsonUtil.getGson().fromJson(str, new TypeToken<ArrayList<String>>() {
            }.getType());
        }
        succUploadList.add(file.getAbsolutePath());
        PreferencesUtils.putString(this, "succUploadFiles", GsonUtil.toJson(succUploadList));
    }
```

#### 自动化脚本

上面的步骤做完了后，我们就可以就可以进行数据采集啦。但是通过人工手动点击操作app来收集数据的化，肯定是很傻的操作。因此需要通过android自动化框架ui automator来进行自动化操作。


```
@RunWith(AndroidJUnit4.class)
public class NiuNiuQiCheTest {

    private static final String PACKAGE_NAME = "****";

    private UiDevice mDevice;

    @Before
    public void before() throws Exception {
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        Utils.wakeUpDevice(mDevice);

        mDevice.pressHome();
        sleep(3000);

        while (!mDevice.hasObject(By.pkg(PACKAGE_NAME))) {
            Utils.openApp(PACKAGE_NAME);
            sleep(10000);
        }
    }

    @Test
    public void auto() throws Exception {
        startUp();
    }

    public void startUp() throws Exception {
        sleep(4000);
        Utils.click(mDevice, By.text("国产"));
        sleep(2000);
        firstListView();
    }

    public void firstListView() throws Exception {
        int width = mDevice.getDisplayWidth();
        int height = mDevice.getDisplayHeight();
        mDevice.swipe(width / 2, (int) (height * 0.75), width / 2, 0, 100);

        int index = SharedPreferenceUtil.getInt(PACKAGE_NAME);
        for (int i = 0; i < index; i++) {
            if (!Utils.findObject(mDevice, By.res("****:id/****"))
                    .scroll(Direction.DOWN, 0.8f)) {
                sleep(500);
            }
        }

        if (mDevice.hasObject(By.text("中兴"))) {
            SharedPreferenceUtil.putInt(PACKAGE_NAME, 0);
            return;
        }

        List<String> identificationStrList = new ArrayList<>();
        String identificationStr = null;
        List<UiObject2> uiObj2List = null;
        boolean arriveEndFlag = false;

        while (true) {
            sleep(2000);
            UiObject2 itemObj = null;
            uiObj2List = mDevice.findObjects(By.res("****:id/tv_content"));
            for (int i = 0; i < uiObj2List.size(); i++) {
                try {
                    itemObj = uiObj2List.get(i);
                    identificationStr = itemObj.getText();
                    if (!identificationStrList.contains(identificationStr)) {
                        identificationStrList.add(identificationStr);
                        itemObj.click();
                        sleep(2000);
                        secondListView();
                    }
                } catch (Exception e) {
                    L.e(e);
                }
            }

            if (arriveEndFlag) {
                break;
            }

            if (!Utils.findObject(mDevice, By.res("****:id/country_lvcountry"))
                    .scroll(Direction.DOWN, 0.8f)) {
                arriveEndFlag = true;
            }
            SharedPreferenceUtil.putInt(PACKAGE_NAME, ++index);
        }
    }

    public void secondListView() throws Exception {
        if (!mDevice.hasObject(By.textContains("暂无"))) {
            List<String> identificationStrList = new ArrayList<>();
            String identificationStr = null;
            List<UiObject2> uiObj2List = null;
            boolean arriveEndFlag = false;
            while (true) {
                uiObj2List = mDevice.findObjects(By.res("****:id/sbl_title"));
                for (UiObject2 childUiObj2 : uiObj2List) {
                    try {
                        identificationStr = childUiObj2.getText();
                        if (!identificationStrList.contains(identificationStr)) {
                            identificationStrList.add(identificationStr);
                            L.d(identificationStr);
                            childUiObj2.click();
                            sleep(2000);
                            thirdListView();
                        }
                    } catch (Exception e) {
                        L.e(e);
                    }
                }

                if (arriveEndFlag) {
                    break;
                }

                if (!Utils.findObject(mDevice, By.res("****:id/sub_list"))
                        .scroll(Direction.DOWN, 0.8f)) {
                    arriveEndFlag = true;
                }
            }
        }
        mDevice.pressBack();
    }

    public void thirdListView() throws Exception {
        if (!mDevice.hasObject(By.textContains("沒有")) || !mDevice.hasObject(By.textContains("暂无"))) {
            List<String> identificationStrList = new ArrayList<>();
            String identificationStr = null;
            List<UiObject2> uiObj2List = null;
            boolean arriveEndFlag = false;
            while (true) {
                uiObj2List = mDevice.findObjects(By.res("****:id/res_user_name_area"));
                for (UiObject2 childUiObj2 : uiObj2List) {
                    try {
                        sleep(1000 * 60 * 5);

                        identificationStr = childUiObj2.getText();
                        if (!identificationStrList.contains(identificationStr)) {
                            identificationStrList.add(identificationStr);
                            childUiObj2.click();
                            sleep(2000);

                            Utils.click(mDevice, By.text("联系"));
                            sleep(1000);
                            Utils.click(mDevice, By.res("****:id/txt_cancel"));
                            sleep(1000);

                            while (!mDevice.hasObject(By.res("****:id/tab_title"))) {
                                mDevice.pressBack();
                                sleep(1000);
                            }
                            sleep(3000);
                        }
                    } catch (InterruptedException e) {
                        L.e(e);
                    }
                }

                if (arriveEndFlag) {
                    break;
                }

                if (!Utils.findObject(mDevice, By.res("android:id/list"))
                        .scroll(Direction.DOWN, 0.8f)) {
                    arriveEndFlag = true;
                }
            }
        }
        mDevice.pressBack();
    }

    @After
    public void after() {
    }

}
```

#### 自动化脚本定期执行

自动化脚本的运行其实就是执行脚本apk文件的过程，因此只需要将打包好的脚本apk文件，放在服务器定时执行即可。
这里说下我的实现思路：

1. 通过`adb connect`命令让android手机和服务器链接在一起
2. nodejs写一个定时执行任务，将打包好的脚本apk文件定期在android手机运行即可
3. 通过pm2运行node执行文件，当定时任务挂掉后重启定时任务
4. 记录执行过程中导致自动化挂掉的问题日志


我把在android手机执行测试脚本的过程写成一个shell脚本,然后通过shelljs去不断执行脚本

```
adb connect {ip}

adb -s {ip} shell am force-stop ****

sleep 10s

adb -s {ip} push ./shell/app-hook-debug-androidTest.apk /data/local/tmp/com.min.zeok.hook.test

adb -s {ip} shell pm install -t -r "/data/local/tmp/com.min.zeok.hook.test"

adb -s {ip} shell am instrument -w -r   -e debug false -e class com.min.zeok.test.**** com.min.zeok.hook.test/android.support.test.runner.AndroidJUnitRunner

```

```
const shell = require('shelljs');
const fs = require('fs');
const path = require('path');
const moment = require('moment');

let logDir = path.resolve(__dirname, '../logs');
if (!fs.existsSync(logDir)) {
  fs.mkdirSync(logDir);
}

function log(devicesIp, data) {
  data = data + '\n';
  fs.writeFileSync(path.resolve(__dirname, '../logs', devicesIp + '.log'), data, {flag: 'a'});
}

module.exports = function (devicesIp, filePath) {
  const autoStrTemplete = fs.readFileSync(path.resolve(__dirname, filePath)).toString();
  let scriptShell = autoStrTemplete.replace(/{ip}/g, devicesIp);

  while (true) {
    log(devicesIp, '---------------------->' + devicesIp + '启动,' + moment().format('YYYY-MM-DD HH:mm'));
    let result = shell.exec(scriptShell);
    log(devicesIp, result.toString());
    log(devicesIp, '<---------------------' + devicesIp + '退出,code=' + result.code + '   ' + moment().format('YYYY-MM-DD HH:mm'));
  }
}
```

```
const app = require('./app');
const devicesIp = '10.10.12.185:5555';

app(devicesIp, '../shell/auto_shengxinbao.sh');
```

### 闲扯几句

1. 其实通过xposed hook应用可以做很多事情，比如有些人通过它修改一些收费应用商的金币数量等等，让自己可以使用收费版功能。
2. 在通过自动化爬取数据的时候很容易被封号，这个就需要你为你的自动化脚本指定一些执行策略，比如执行多少次后，换一个账户、换imei、换地址位置、换ip地址
3. 通过该方案爬取数据的好处是，你不需要真正的区破解接口的加密逻辑。如果你确实有这方面的需求可以通过xposed上的InspectPackage模块来对应用进行动态分析，在结合源码进行破解，所需要花费的时间也是比较多。
4. 以后还是少熬夜的比较好！





