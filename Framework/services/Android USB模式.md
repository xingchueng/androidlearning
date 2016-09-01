## 问题引出

最近做一个需求，要求修改默认的USB开机模式，但是不能重启adb服务。所以有必要弄清楚Android在USB模式的初始化工作是如何进行的，模式是如何切换的。

## USB常用模式

一般厂商会定制一些模式的，常用的模式就是mtp以及adb了。具体有哪些，可以在代码里找`init.usb.rc`以及`ini.[customname].usb.rc`。

一般情况就是`mtp,adb`模式了。
有时候需要定制一些模式，就需要在`init.[].usb.rc`里面添加。具体格式可以参考相应的文件。

## USB模式初始化

在手机运行的时候，可以通过`adb shell getprop |grep usb`命令查看手机的USB模式，一般可以查看以下两个属性值对
    
    sys.usb.config
    persis.sys.usb.config
    
而通常`persis.sys.usb.config`是初始的属性，如下所示

    # Used to set USB configuration at boot and to switch the configuration
    # when changing the default configuration
    on property:persist.sys.usb.config=*
    setprop sys.usb.config ${persist.sys.usb.config}
    
而 `persis.sys.usb.config`这个属性，一般是解析根分区的`default.prop`文件而来。以下是该文件的部分内容：

    ro.debuggable=1
    persist.sys.usb.config=mtp,adb
    ro.zygote=zygote64_32

在`system/core/init/property_service.cpp`中有以下代码：

    #define PROP_PATH_RAMDISK_DEFAULT  "/default.prop"    //_system_properties.h
    load_properties_from_file(PROP_PATH_RAMDISK_DEFAULT, NULL);
    
就是将相应的属性值对“key-value”解析为系统属性。

## Frameworks层对USB模式的处理

在完成了系统属性的解析之后，上层也会对USB模式做进一步的流程管理。主要是完成状态机，以及功能或者模式的切换规则与方法。

大致流程是：在`SystemServer`里面会启动`UsbService`的服务，这个服务主要是向上层提供接口，同时封装具体的系统服务`UsbDeviceManager`，在系统服务`UsbDeviceManager`里面，会读取默认的属性作为初始化模式，然后判断OEM厂商会不会做修改，如果修改就切换相应的模式，并且把最终的模式报告给系统。

另一方面，也做了模式切换规则，使用监听模式来监听相应的文件，初始化状态机来根据具体模式切换不同状态等。

下面跟着代码走走初始化流程。
在` SystemServe `里启动了USB服务，

    private static final String USB_SERVICE_CLASS =
            "com.android.server.usb.UsbService$Lifecycle";
    mSystemServiceManager.startService(USB_SERVICE_CLASS);
找到`UsbService`这个类，可以知道`Lifecycle`是其一个内部类，

    public static class Lifecycle extends SystemService {
        private UsbService mUsbService;
        
        public Lifecycle(Context context) {
            super(context);
        }
        
        @Override
        public void onStart() {
            mUsbService = new UsbService(getContext());
            publishBinderService(Context.USB_SERVICE, mUsbService);
        }
        
        @Override
        public void onBootPhase(int phase) {
            if (phase == SystemService.PHASE_ACTIVITY_MANAGER_READY) {
                mUsbService.systemReady();
            } else if (phase == SystemService.PHASE_BOOT_COMPLETED) {
                mUsbService.bootCompleted();
            }
        }
    }
在调用`startService()`这个函数的时候，其参数是一个`SystemService`的子类，会触发调用这个子类的`onStart()`函数。

在`SystemServer`里调用`mSystemServiceManager.startBootPhase(                        SystemService.PHASE_ACTIVITY_MANAGER_READY);`时，会触发所有以注册`SysteService`子类的`onBootPhase(int phase)`函数，并将参数传递出来。所以`mUsbService`初始化完成之后，会接着调用自身的`systemReady()`以及`bootCompleted()`函数。接着往下看。

    mDeviceManager = new UsbDeviceManager(context, mAlsaManager);
    public void systemReady() {
        mAlsaManager.systemReady();
        
        if (mDeviceManager != null) {
            mDeviceManager.systemReady();
        }
        if (mHostManager != null) {
            mHostManager.systemReady();
        }
        if (mPortManager != null) {
            mPortManager.systemReady();
        }
    }
    public void bootCompleted() {
        if (mDeviceManager != null) {
            mDeviceManager.bootCompleted();
        }
    }
    
`UsbDeviceManager`的构造函数做的动作有以下几个需要注意：
1. 用sn号初始化adb的地址：`initRndisAddress();`
2. 如果OEM厂商有默认设置，读取并设置(虽然一般没有或者不在这里改)：`readOemUsbOverrideConfig();`
3. 确认是否加密以及数据安全： 

`boolean secureAdbEnabled = SystemProperties.getBoolean("ro.adb.secure", false);`

`boolean dataEncrypted = "1".equals(SystemProperties.get("vold.decrypt"));`
4. 生成异步处理`Handler` ： `mHandler = new UsbHandler(FgThread.get().getLooper());`

第四步比较重要，因为模式的切换等大部分工作都在这里完成。转到`UsbHandler`。一下是它的构造函数：

    private static final String USB_CONFIG_PROPERTY = "sys.usb.config";
    public static final String USB_FUNCTION_NONE = "none";
    private static final String USB_STATE_PROPERTY = "sys.usb.state";
            public UsbHandler(Looper looper) {
            super(looper);
            try {
                // Restore default functions.
                mCurrentFunctions = SystemProperties.get(USB_CONFIG_PROPERTY,
                        UsbManager.USB_FUNCTION_NONE);
                if (UsbManager.USB_FUNCTION_NONE.equals(mCurrentFunctions)) {
                    mCurrentFunctions = UsbManager.USB_FUNCTION_MTP;
                }
                mCurrentFunctionsApplied = mCurrentFunctions.equals(
                        SystemProperties.get(USB_STATE_PROPERTY));
                mAdbEnabled = UsbManager.containsFunction(getDefaultFunctions(),
                        UsbManager.USB_FUNCTION_ADB);
                setEnabledFunctions(null, false);
                String state = FileUtils.readTextFile(new File(STATE_PATH), 0, null).trim();
                updateState(state);
                
                // register observer to listen for settings changes
                mContentResolver.registerContentObserver(
                        Settings.Global.getUriFor(Settings.Global.ADB_ENABLED),
                                false, new AdbSettingsObserver());
                                
                // Watch for USB configuration changes
                mUEventObserver.startObserving(USB_STATE_MATCH);
                mUEventObserver.startObserving(ACCESSORY_START_MATCH);
            } catch (Exception e) {
                Slog.e(TAG, "Error initializing UsbHandler", e);
            }
        }

所做的工作就是，读取默认的模式，如果默认为空，就设置为MTP模式。然后读取默认的状态的属性值。这里要注意一个函数`setEnabledFunctions(null, false);`这个函数就是切换USB功能的函数，这里的功能（跟模式是一个意思）切换意味着状态也是切换的。

正常情况下，这个函数执行的是以下操作，从参数名称可以看出，第二个参数意思是是否强制重启adb server这个服务。

    if (trySetEnabledFunctions(functions, forceRestart)) {
                return;
    }
调用的函数`trySetEnabledFunctions()`，如果调用不成功，会有进一步的处理，这里先忽略。先看看这个函数的函数体：

    private boolean trySetEnabledFunctions(String functions, boolean forceRestart) {
            if (functions == null) {
                functions = getDefaultFunctions();
            }
            functions = applyAdbFunction(functions);
            functions = applyOemOverrideFunction(functions);
            
            if (!mCurrentFunctions.equals(functions) || !mCurrentFunctionsApplied
                    || forceRestart) {
                Slog.i(TAG, "Setting USB config to " + functions);
                mCurrentFunctions = functions;
                mCurrentFunctionsApplied = false;
                
                // Kick the USB stack to close existing connections.
                setUsbConfig(UsbManager.USB_FUNCTION_NONE);
                
                // Set the new USB configuration.
                if (!setUsbConfig(functions)) {
                    Slog.e(TAG, "Failed to switch USB config to " + functions);
                    return false;
                }
                
                mCurrentFunctionsApplied = true;
            }
            return true;
    }
如果`functions`参数为null，则将作为默认功能对待。而如果跟目前的功能是一样的，则直接返回`true`了。以下是获取默认功能的函数体：

    private String getDefaultFunctions() {
            String func = SystemProperties.get(USB_PERSISTENT_CONFIG_PROPERTY,
                    UsbManager.USB_FUNCTION_NONE);
            if (UsbManager.USB_FUNCTION_NONE.equals(func)) {
                func = UsbManager.USB_FUNCTION_MTP;
            }
            return func;
    }