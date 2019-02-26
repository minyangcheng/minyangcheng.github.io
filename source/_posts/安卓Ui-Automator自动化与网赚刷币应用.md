---
title: 安卓Ui-Automator自动化与网赚刷币应用
date: 2018-10-26 16:07:00
tags: android
---

前端时间接触了几款看新闻即可赚钱的app(趣看点、东方头条等)，运用Ui Automator做了一个可以自动化刷币App，你只需要安装该App，点击App上的启动脚本按钮，即可开启自动化刷币操作。这篇文章主要是介绍刷币应用涉及所需要的一些知识和实现思路。

> 该刷币应用需要运行在已经Root过的手机上，如果你有闲置的手机，装上应用即可每天自动进行自动化刷币，如果想停止手动杀掉进程即可。

<!-- more -->

### 自动化测试

Android有很多自动化测试工具Monkey、MonkeyRunner、Robotium、Ronaorex、Appium、UI Automator、TestBird，这里我选用的是Ui Automator2.0版本，这款工具是谷歌发布的，提供了一组API来构建UI测试，用于在用户应用和系统应用中执行交互，适合编写黑盒自动化测试，其测试代码不依赖于目标应用的内部实现详情，要求Android 4.3（API >= 18）

> 官方文档链接 https://developer.android.google.cn/reference/android/app/UiAutomation

#### 基本使用方法

* 利用uiautomatorviewer工具（位于%android-sdk%/tools/bin目录中），查看和分析Android设备上当前显示的UI组件和布局层次结构，查找自己需要操作的控件信息

![](/images/20190226_11.webp)


* 新建Android项目，添加依赖androidTestCompile 'com.android.support.test.uiautomator:uiautomator-v18:2.1.2'，在androidTest模块下书写自动化代码

```
@RunWith(AndroidJUnit4.class)
public class TestCase {

    private static final String PACKAGE_NAME = "package_name";
    private static final int LAUNCH_TIMEOUT = 5000;
    private UiDevice mDevice;

    @Before
    public void before() throws Exception {
        //获取设备实例
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        //点击Home键
        mDevice.pressHome();
        //启动App
        Context context = InstrumentationRegistry.getContext();
        Intent intent = context.getPackageManager().getLaunchIntentForPackage(PACKAGE_NAME);
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TASK);
        context.startActivity(intent);
        //确保应用已启动
        mDevice.wait(Until.hasObject(By.pkg(PACKAGE_NAME).depth(0)), LAUNCH_TIMEOUT);
    }

    @Test
    public void auto() throws InterruptedException {
        //通过resource-id查找到控件，并点击
        mDevice.findObject(By.res("package_name:id/xx")).click();
    }

    @After
    public void after() {am instrument -w -r   -e debug false -e class com.min.money.test.AutoScript com.min.money.test/android.support.test.runner.AndroidJUnitRunner
    }

}
```

* 运行测试用例

> 通过android studio运行测试用例时，生成的脚本命令可以看出来，Ui Automator自动化运行的过程其实就是运行一个测试Apk的过程。我们可以在手机上安装Android studio生成的apk文件app-debug-androidTest.apk，然后执行adb命令行adb shell am instrument -w -r   -e debug false -e class com.min.money.test.AutoScript com.min.money.test/android.support.test.runner.AndroidJUnitRunner，即可运行自动化测试脚本。

![](/images/20190226_12.webp)

### 刷币应用

这里主要介绍下实现此刷币应用的一些思路和核心实现代码，如需源码可以联系我。

#### 实现思路

1. 书写自动化刷币脚本，脚本执行要像真人操作一样，控件点击、页面停留时间、页面跳转都需要做随机操作，还需要兼容运行过程中各种异常情况如忽然有一个推送弹出出现，脚本的运行时间也需要做限制（07：00-23：00），如果你想多个手机一起刷还需考虑ip代理，这些操作都是为了防止被对方后台反欺诈系统发现
2. 在android-studio gradle窗口点击运行assembleAndroidTesttask或直接执行gradle assembleAndroidTest，生成自动化脚本apk文件，将生成的测试Apk文件app-debug-androidTest.apk，放入Appassets目录中，命名为auto.apk
3. App通过Runtime.getRuntime().exec执行adb shell am instrument -w -r   -e debug false -e class com.min.money.test.AutoScript com.min.money.test/android.support.test.runner.AndroidJUnitRunner即可

#### 主要代码介绍

##### 自动化脚本

脚本入口

```
@RunWith(AndroidJUnit4.class)
public class AutoScript {

    private UiDevice mDevice;

    @Before
    public void before() throws Exception {
        String startTimeStr = Utils.nowTimeStr();
        L.i("start time = %s", startTimeStr);
        Context context = InstrumentationRegistry.getTargetContext();
        PreferencesUtils.putString(context, "start_time", startTimeStr);
        PreferencesUtils.putString(context, "end_time", "");

        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
    }

    @Test
    public void startAutoScript() {
        while (true) {
            try {
                once();
                Utils.sleepRandomSecond(10, 20);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public void once() {
        try {
            BaseAuto auto = null;
            check();
            auto = new Qtt();
            auto.run();
            mDevice.pressHome();
            Utils.sleepRandomSecond(3, 6);
            ......
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void check() throws Exception {
        while (!Utils.checkRightTime()) {
            Utils.sleepRandomSecond(60);
        }
    }

    @After
    public void after() {
        String endTimeStr = Utils.nowTimeStr();
        L.i("end time = %s", endTimeStr);
        Context context = InstrumentationRegistry.getTargetContext();
        PreferencesUtils.putString(context, "end_time", endTimeStr);
    }

}
```

基类

```
public abstract class BaseAuto {

    protected int mMinCycleValue = 20;
    protected int mMaxCycleValue = 40;

    protected String mPackageName;
    protected UiDevice mDevice;

    public BaseAuto(String packageName) {
        this.mPackageName = packageName;
    }

    public void startApp() throws Exception {
        mDevice = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation());
        Utils.wakeUpDevice(mDevice);
        mDevice.pressHome();
        Utils.sleepRandomSecond(2);
        Utils.openAppSafe(mDevice, mPackageName);
    }

    /**
     * 启动操作
     */
    public void run() {
        try {
            L.i("start packageName=%s , time = %s", mPackageName, Utils.nowTimeStr());
            startApp();
            operate();
            L.i("end packageName=%s , time = %s", mPackageName, Utils.nowTimeStr());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 实现操作
     */
    public abstract void operate() throws Exception;

}
```

具体实现

```
public class Qtt extends BaseAuto {

    public Qtt() {
        super("package_name");
    }

    @Override
    public void operate() throws Exception {
        for (int i = 0; i < 2; i++) {
            //随机休眠2s-4s
            Utils.sleepRandomSecond(2, 4);
            newsPage();
            Utils.click(mDevice, By.text("我的"));
            Utils.sleepRandomSecond(2, 4);
            minePage();
            Utils.click(mDevice, By.text("任务"));
            Utils.sleepRandomSecond(2, 4);
            taskPage();
            Utils.click(mDevice, By.text("视频"));
            Utils.sleepRandomSecond(2, 4);
            videoPage();

            Utils.click(mDevice, By.text("头条"));
        }
    }

    public void newsPage() throws Exception {
        int step = Utils.getRandom(mMinCycleValue, mMaxCycleValue);
        for (int i = 0; i < step; i++) {
            //处理异常弹窗
            Utils.click(mDevice, By.res("package_name:id/p4"));
            Utils.click(mDevice, By.res("package_name:id/pl"));
            Utils.click(mDevice, By.res("package_name:id/og"));

            List<UiObject2> uiObjList = mDevice.findObjects(By.res("package_name:id/aag"));
            for (UiObject2 uiObj : uiObjList) {
                try {
                    //随机点击概率为85%
                    boolean flag = Utils.clickRandom(uiObj, 0.85f);
                    if (flag) {
                        Utils.sleepRandomSecond(4, 6);
                        newsDetailPage();
                        mDevice.pressBack();
                    }
                } catch (Exception e) {
                }
            }
            Utils.clickRandom(mDevice, By.res("package_name:id/sk").text("领取"));
            Utils.slideVerticalUp(mDevice);
            Utils.sleepRandomSecond(1);
        }
    }

    public void minePage() throws Exception {
        Utils.click(mDevice, By.res("package_name:id/pl"));
        Utils.click(mDevice, By.res("package_name:id/nu"));
        //随机打开界面，并在界面里随机滑动停留
        String[] textArr = {"我的金币", "提现兑换", "提现记录", "历史记录"};
        Utils.openPageRandom(mDevice, textArr);
    }

    public void newsDetailPage() throws Exception {
        Utils.click(mDevice, By.res("package_name:id/pl"));

        Utils.readPage(mDevice);
    }

    public void taskPage() throws Exception {
        Utils.click(mDevice, By.res("package_name:id/pl"));

        Utils.slideVertical(mDevice, 0.4f, 0.2f);
        Utils.sleepRandomSecond(2, 4);
        Utils.slideVertical(mDevice, 0.4f, 0.2f);
    }

    public void videoPage() throws Exception {
        int step = Utils.getRandom(mMinCycleValue, mMaxCycleValue);
        for (int i = 0; i < step; i++) {
            Utils.click(mDevice, By.res("package_name:id/pl"));

            List<UiObject2> uiObjList = mDevice.findObjects(By.res("package_name:id/a3b"));
            for (UiObject2 uiObj : uiObjList) {
                try {
                    if (Utils.clickRandom(uiObj, 0.85f)) {
                        Utils.sleepRandomSecond(20, 30);
                    }
                } catch (Exception e) {
                }
            }
            Utils.slideVerticalUp(mDevice);
            Utils.sleepRandomSecond(1);
        }
    }

}
```

##### App应用

```
public class MainActivity extends AppCompatActivity {

    private static final String TAG = MainActivity.class.getSimpleName();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
    }

    //引导用户安装看新闻赚钱App
    @OnClick(R.id.btn_install_Qtt)
    void installQtt() {
        Utils.toast("准备安装中....");
        File destFile = new File(this.getExternalCacheDir(), "Qtt.apk");
        FileUtil.copyAssetsFileToStorage(this, "Qtt.apk", destFile.getAbsolutePath());
        Logger.d(TAG, "copy Qtt apk success = %s", destFile.getAbsolutePath());
        Utils.installApk(this, destFile.getAbsolutePath());
    }

    //引导用户安装测试apk
    @OnClick(R.id.btn_install_auto)
    void installAuto() {
        Utils.toast("准备安装中....");
        File destFile = new File(this.getExternalCacheDir(), "auto.apk");
        FileUtil.copyAssetsFileToStorage(this, "auto.apk", destFile.getAbsolutePath());
        Logger.d(TAG, "copy auto apk success = %s", destFile.getAbsolutePath());
        Utils.installApk(this, destFile.getAbsolutePath());
    }

    @OnClick(R.id.btn_start)
    void startAuto() {
        if (RootCmd.haveRoot()) {
            RootCmd.execRootCmdSilent("am instrument -w -r   -e debug false -e class com.min.money.test.AutoScript com.min.money.test/android.support.test.runner.AndroidJUnitRunner");
        } else {
            Utils.toast("由于启动自动化脚本，需要root权限，请下载king root应用软件进行root手机操作");
        }
    }

    @OnClick(R.id.btn_query_run_time)
    void queryRunTime() {
        String startTime = PreferencesUtils.getString(this, "start_time");
        String endTime = PreferencesUtils.getString(this, "end_time");
        if (!TextUtils.isEmpty(startTime)) {
            String s = "启动时间: " + startTime + "\n\n";
            s += "结束时间: " + endTime + "\n\n";
            Utils.showAlertDialog(this, "查看运行情况", s);
        } else {
            Utils.toast("暂无数据....");
        }
    }

    @OnClick(R.id.btn_help)
    void help() {
        Utils.showAlertDialog(this, "帮助", this.getString(R.string.more_help));
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        ButterKnife.unbind(this);
    }

}
```


