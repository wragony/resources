# Android多任务模糊化处理方案

## 多任务介绍

安卓任务列表是系统对App当前页面进行截图然后再显示的；主流的银行APP有些手机上也并没有实现我们所说的高斯模糊的效果，包括在华为的手机、三星的手机上面；


## 实现方案

### 方案一

通过系统自带的安全属性设置

- 方案实现: 

在 Activity onCreate() 的回调中设置如下代码：

```

getWindow().setFlags(WindowManager.LayoutParams.FLAG_SECURE, WindowManager.LayoutParams.FLAG_SECURE)

```

- 优点：

1. 无适配问题

- 缺点：

1. 任务列表时页面会显示白屏，但是体验较差；
2. 此属性连带了一个防截屏的功能，设置后App不能截屏了；


### 方案二

1. 捕获多任务按键的回调行为；
2. 然后截取当前App的屏幕截图，压缩并进行高斯模糊处理；
3. 然后动态设置到当前界面的最上层；
4. 当App切换到前台时主动移除模糊图层；

- 优点：

1. 在支持的手机上能够实现和设计一致的效果，体验佳；

- 缺点：

1. 捕获多任务按键的回调行为需要作一定的适配；
2. 截图高斯模糊处理存在一定的耗时；
3. 目前测试的情况来看不能做到全部机型适配；


- 方案实现：

捕获多任务按键的回调

```
/**
 * 系统按键监控
 * @author wragony
 * @property context
 * @property mDeviceKeyListener
 */
@Suppress("DEPRECATION")
class DeviceKeyMonitor(
    private val context: Context,
    val mDeviceKeyListener: OnDeviceKeyListener?,
) {

    companion object {

        private const val SYSTEM_REASON = "reason"
        private const val SYSTEM_HOME_RECENT_APPS = "recentapps"
        private const val SYSTEM_HOME_RECENT_APPS_XIAOMI = "fs_gesture"
        private const val SYSTEM_HOME = "homekey"
        private const val SYSTEM_LOCK = "lock"

    }

    private var mDeviceKeyReceiver: BroadcastReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            val action = intent?.action
            if (action.isNullOrEmpty()) {
                return
            }
            if (action == Intent.ACTION_CLOSE_SYSTEM_DIALOGS) {
                val reason = intent.getStringExtra(SYSTEM_REASON)
                if (reason.isNullOrEmpty()) {
                    return
                }
                when (reason) {
                    SYSTEM_HOME_RECENT_APPS,
                    SYSTEM_HOME_RECENT_APPS_XIAOMI -> {
                        mDeviceKeyListener?.onRecentAppClick()
                    }
                    SYSTEM_HOME -> {
                        mDeviceKeyListener?.onHomeClick()
                    }
                    SYSTEM_LOCK -> {
                        mDeviceKeyListener?.onLockClick()
                    }
                }
            }
        }
    }

    fun register() {
        context.registerReceiver(mDeviceKeyReceiver, IntentFilter(Intent.ACTION_CLOSE_SYSTEM_DIALOGS))
    }

    fun unregister() {
        context.unregisterReceiver(mDeviceKeyReceiver)
    }

}

```

多任务模式下对App高斯模糊处理

```
/**
 * 多任务模式下对App截图高斯模糊处理
 * @author wragony
 * @param currentActivity
 */
class RecentViewBlur(currentActivity: Activity) {

    private var mKeyReceiver: DeviceKeyMonitor

    private var contentParent: ViewGroup? = null
    private var imageView: ImageView? = null

    @Volatile
    private var isRecentApps: Boolean = false

    private var params: FrameLayout.LayoutParams? = null

    private val overlayColor = "#728E9190"


    init {
        contentParent = currentActivity.window.decorView as ViewGroup
        imageView = ImageView(currentActivity)
        params = FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT)
        imageView?.layoutParams = params
        imageView?.scaleType = ImageView.ScaleType.FIT_XY

        mKeyReceiver = DeviceKeyMonitor(currentActivity, object : OnDeviceKeyListener {

            override fun onRecentAppClick() {
                LogUtil.d(">>>>>>按了多任务键--->isRecentApps:$isRecentApps")
                if (isRecentApps) {
                    return
                }
                isRecentApps = true
                handleBlur()
            }

        })
    }

    private fun handleBlur() {
        LogUtil.d(">>>>>>开始创建高斯模糊图层")
        ThreadUtils.executeBySingle(object : ThreadUtils.SimpleTask<Bitmap?>() {
            override fun doInBackground(): Bitmap? {
                return try {
                    // 截屏并高斯模糊处理
                    if (contentParent?.contains(imageView!!) == true) {
                        contentParent?.removeView(imageView)
                    }
                    var bitmap = ImageUtils.view2Bitmap(contentParent)
                    bitmap = ImageUtils.compressBySampleSize(bitmap, 16)
                    ImageUtils.renderScriptBlur(bitmap, 8.0F)
                } catch (e: Exception) {
                    null
                }
            }

            override fun onSuccess(bitmap: Bitmap?) {
                addBlurView(bitmap)
            }
        })
    }

    private fun addBlurView(bitmap: Bitmap?) {
        removeBlurView()
        LogUtil.d(">>>>>>开始添加高斯模糊图层")
        imageView?.setImageBitmap(bitmap)
        imageView?.colorFilter = PorterDuffColorFilter(Color.parseColor(overlayColor), PorterDuff.Mode.SRC_OVER)
        contentParent?.addView(imageView, params)
    }

    private fun removeBlurView() {
        LogUtil.d(">>>>>>开始移除高斯模糊图层")
        contentParent?.removeView(imageView)
    }

    /**
     * 注册广播接收器
     */
    fun register() {
        mKeyReceiver.register()
    }

    /**
     * 解除广播接收器
     */
    fun unregister() {
        mKeyReceiver.unregister()
    }

    /**
     * 全面屏手势可能会导致onResume不回调，
     * 通过onWindowFocusChanged回调来处理模糊图层的移除
     *
     * @param hasFocus
     */
    fun onWindowFocusChanged(hasFocus: Boolean) {
        LogUtil.d(">>>>>>onWindowFocusChanged,hasFocus:$hasFocus")
        if (hasFocus) {
            removeBlurView()
            isRecentApps = false
        }
    }

}

```

方案二的测试如下：

各手机厂商rom的机制不一样，在全面屏手势导航和虚拟按键导航操作后台挂起的响应时间不一样，从测试来看，部分手机按下了虚拟按键（全面屏手势）后App立马挂起，此时无法在对App进行任何UI更新操作；

**表格中“没效果”的解释如下：**

没效果：在收到多任务广播回调后进行高斯模糊的时机已经在系统的截图操作之后了，所以没效果

| 机型  | 虚拟按键导航  | 全面屏导航  | 备注  |
|:----------|:----------|:----------|:----------|
| google | yes    | yes    |    |
| samsung   | yes | yes   |  | 
| huawei   | no（没效果） | yes   |   | 
| vivo   | yes | no（没效果）   |  | 
| oppo   | yes | yes   | 虚拟按键导航支持（大部分情况支持，偶现来不及处理没效果的情况） | 
| xiaomi   | yes | yes   | 全面屏需要单独适配多任务监听，系统级别也提供了模糊预览图选项（设置-》桌面-》模糊预览图-》选择对应的应用开启开关） | 



