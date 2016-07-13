# SystemServer
---
## 为什么需要分析SystemServer？
---
1. SystemServer是Android系统的中间层核心部分
2. 起到承上启下的作用，向上提供服务，抽象下层的实现；向下提供封装
3. 了解系统的运行机制

## 需要哪些基础?
---
1. Java
2. Binder
3. 设计模式
4. Confidence

## Who is this doc for?
---
1. Those who want to understand Framework
2. Those who are app developers now, want to know more about Android
3. 其实想看都可以看

## How could we be more efficient in reading code about services?
---
1. Abstract principle goes first。不要拘泥于细节
2. Have to know the flow of services。其实流程有时是通过文档了解的，有时还需要通过代码反证文档。
3. 懂得一些设计模式的知识。
4. 尝试实现自己的服务。

## System Services的简单分类.
---
Android系统将服务分为三类: critical, core 以及 other. 我们可以看到代码中的情况` SystemServer.java `:

    // Start services.
        try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "***");
            Slog.e("System", "*** Failure starting system services", ex);
            throw ex;
        }        
这个分类并非严格分类，只是从逻辑结构和相互关系的分类。实际上以上三类服务都是系统所必需的，除非你不想要某些功能，否则以上系统服务都不能少。

### Critial Services
---
在文件`SystemServer.java`的注释中, 这些服务是系统运行起来必备的服务，而且它们之间有着复杂的相互关系。

1. Installer.class
2. ActivityManagerService.Lifecycle.class   //a inner class in ActivityManagerService
3. PowerManagerService.class
4. LightsService.class
5. DisplayManagerService.class
6. UserManagerService
7. startSensorService();

### Core Services
---
注释中是这么解释的："Starts some essential services that are not tangled up in the bootstrap process." 而且从名字来看也知道这部分的服务业非常重要。

1. BatteryService.class
2. UsageStatsService.class
3. UsageStatsManagerInternal.class
4. WebViewUpdateService.class

### Othter Services
---
如果翻译过来，就是其他服务了。其实这些都是为上层提供的服务。以下是代码中的部分服务，因为启动的方式不同，所以可能会有重复的总结。

    AccountManagerService accountManager = null;
    ContentService contentService = null;
    VibratorService vibrator = null;
    IAlarmManager alarm = null;
    IMountService mountService = null;
    NetworkManagementService networkManagement = null;
    NetworkStatsService networkStats = null;
    NetworkPolicyManagerService networkPolicy = null;
    ConnectivityService connectivity = null;
    NetworkScoreService networkScore = null;
    NsdService serviceDiscovery= null;
    WindowManagerService wm = null;
    UsbService usb = null;
    SerialService serial = null;
    NetworkTimeUpdateService networkTimeUpdater = null;
    CommonTimeManagementService commonTimeMgmtService = null;
    InputManagerService inputManager = null;
    TelephonyRegistry telephonyRegistry = null;
    ConsumerIrService consumerIr = null;
    AudioService audioService = null;
    MmsServiceBroker mmsService = null;
    EntropyMixer entropyMixer = null;
    CameraService cameraService = null;

    StatusBarManagerService statusBar = null;
    INotificationManager notification = null;
    InputMethodManagerService imm = null;
    WallpaperManagerService wallpaper = null;
    LocationManagerService location = null;
    CountryDetectorService countryDetector = null;
    TextServicesManagerService tsms = null;
    LockSettingsService lockSettings = null;
    AssetAtlasService atlas = null;
    MediaRouterService mediaRouter = null;

1. SchedulingPolicyService
2. TelecomLoaderService.class
3. CameraService.class
4. Watchdog.getInstance()
5. AriNativeImsService
6. new DropBoxManagerService(context, new File("/data/system/dropbox")
7. GestureLauncherService.class
8. CertBlacklister
9. DreamManagerService.class
10. RestrictionsManagerService.class
11. MediaSessionService.class
12. HdmiControlService.class
13. TvInputManagerService.class
14. TrustManagerService.class
15. FingerprintService.class
16. BackgroundDexOptService
17. LauncherAppsService.class
18. MediaProjectionManagerService.class
19. MmsServiceBroker.class
20. Configuration
21. DisplayMetrics

## Structure of System Service.
---
Framework中采用的是C/S模式的架构，所有服务都作为Server端为其Client提供服务。由于服务数量较多，所以也有必要使用相应的管理类进行生命周期的管理。以下是常见的几个相关文件：

    `services/java/com/android/server/SystemServer.java`
    `services/core/java/com/android/server/SystemServiceManager.java`
    `core/java/com/android/server/LocalServices.java`
    `services/core/java/com/android/server/SystemService.java`


`SystemServer.java` 相当于Server端。

`SystemService.java`系统服务的抽象类，大多数服务都继承至此类，或拥有这个类子类的一个内部类。

`LocalServices.java` and `SystemServiceManager.java` 相当于管理类。

以下代码可以看到一些服务是作为`SystemService.java`的子类的，这样可以便于系统批量管理。

    BluetoothService.java:23:class BluetoothService extends SystemService
    NotificationManagerService.java:143:public class NotificationManagerService extends SystemService
    ActivityManagerService.java:2317:    public static final class Lifecycle extends SystemService 
所以我们可以先看看`SystemService.java` 这个类的具体实现：

    public abstract class SystemService {
        public static final int PHASE_WAIT_FOR_DEFAULT_DISPLAY = 100;
        public static final int PHASE_LOCK_SETTINGS_READY = 480;
        public static final int PHASE_SYSTEM_SERVICES_READY = 500;
        public static final int PHASE_ACTIVITY_MANAGER_READY = 550;
        public static final int PHASE_THIRD_PARTY_APPS_CAN_START = 600;
        public static final int PHASE_BOOT_COMPLETED = 1000;
        private final Context mContext;
        public SystemService(Context context) {
            mContext = context;
        }
        public final Context getContext() {
            return mContext;
        }
        ...
        public abstract void onStart();
        public void onBootPhase(int phase) {}
        public void onStartUser(int userHandle) {}
        public void onStopUser(int userHandle) {}
        public void onCleanupUser(int userHandle) {}
        protected final void publishBinderService(String name, IBinder service) {
            publishBinderService(name, service, false);
        }
        protected final void publishBinderService(String name, IBinder service,
                boolean allowIsolated) {
            ServiceManager.addService(name, service, allowIsolated);
        }
        protected final IBinder getBinderService(String name) {
            return ServiceManager.getService(name);
        }
        protected final <T> void publishLocalService(Class<T> type, T service) {
            LocalServices.addService(type, service);
        }
        protected final <T> T getLocalService(Class<T> type) {
            return LocalServices.getService(type);
        }
        private SystemServiceManager getManager() {
            return LocalServices.getService(SystemServiceManager.class);
        }
    }
可以看出，注册Binder类的服务由ServiceManager管理，而注册本地服务则由LocalServices和SystemServiceManager来管理。它们将不同的服务分别放到自己内部的容器中，在需要的时候取出其实例。而`onBootPhase()`以及`onStart()`等函数则可以用来管理生命周期的启动等行为的批处理。

---
### 服务的启动方式
下面先看SystemServiceManager这个类，他主要负责管理已注册的Service的生命周期。两个主要的方法如下：

	public SystemService startService(String className) 
	public void startBootPhase(final int phase)
	
只要某个Service注册到`SystemServiceManager`中，在`SystemService`中就可以通过`startService`单独启动，或则在某个阶段通过`startBootPhase`批量启动。几个阶段分别在`SystemService`中已`int`值定义的。

	./services/java/com/android/server/SystemServer.java:352:        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
	./services/java/com/android/server/SystemServer.java:1022:        mSystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY);
	./services/java/com/android/server/SystemServer.java:1024:        mSystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY);

以上代码会调用各个注册过的Service的 `onBootPhase(int phase)`函数，如下所示：
	
	for (int i = 0; i < serviceLen; i++) {
            final SystemService service = mServices.get(i);
            try {
                service.onBootPhase(mCurrentPhase);
            } catch (Exception ex) {
                ...
            }
---
### 在需要的时候获取服务
另一种方式是系统将服务加入到`ServiceManager`中，然后在需要的时候可以获取到。

	wm = WindowManagerService.main(context, inputManager,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                    !mFirstBoot, mOnlyCore);
            ServiceManager.addService(Context.WINDOW_SERVICE, wm);
            
上面的代码将`WindowManagerService`加入到 `ServiceManager`中，我们也可以看看哪些地方用到了这个服务：

	WindowManager w = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
	
在`SystemServer.java`中，后面的代码就使用了这个服务。这里将服务与`Context`关联在一起了。也有直接使用`ServiceManager`的例子，如下：

	(IPowerManager)ServiceManager.getService(Context.POWER_SERVICE)
这里获取的是电源管理方面的服务。那么这个电源管理的服务是什么时候注册到`ServiceManager`中的呢？可以看到获取了这个服务之后，将其转换为`IPowerManager`类型了，应该是一个Binder类型，可以搜到以下文件：

	./core/java/android/os/IPowerManager.aidl
所以应该是调用了`SystemService`的`publishBinderService（）`函数，在`PowerManagerService.java`中，有如下代码：

	public void onStart() {
        publishBinderService(Context.POWER_SERVICE, new BinderService());
        ...
    }
而这个`BinderService`就是`IPowerManager`的一个子类，也是`PowerManagerService`类的一个内部类：

	    private final class BinderService extends IPowerManager.Stub {
	    ...
	    ｝


而将系统服务注册到`Context`中调用的是`SystemServiceRegistry.java`中的以下函数：

	 private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }
比如，将`PowerManager`注册到`Context`中，由以下代码执行：

	SystemServiceRegistry.java:366:  registerService(Context.POWER_SERVICE, PowerManager.class,
	....
这样，就可以通过`Context`以及常量Context.POWER_SERVICE来获取这个服务了：
	
	./services/core/java/com/android/server/policy/PhoneWindowManager.java:1380:        mPowerManager = (PowerManager)context.getSystemService(Context.POWER_SERVICE);




## some services could be optmized.
---
1.WallpaperManagerService
R.bool.config_enableWallpaperService

2.GestureLauncherService

3.com.android.server.print.PrintManagerService

## Some points.
---
    // We start this here so that we update our configuration to set watch or television
    // as appropriate.
    mSystemServiceManager.startService(UiModeManagerService.class);
    try {
        mPackageManagerService.performBootDexOpt();
    } catch (Throwable e) {
        reportWtf("performing boot dexopt", e);
    }
    try {
        ActivityManagerNative.getDefault().showBootMessage(
                context.getResources().getText(
                        com.android.internal.R.string.android_upgrading_starting_apps),
                false);
    } catch (RemoteException e) {
    }