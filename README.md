# AndroidProcess


此项目 Fork 自 [AndroidProcess by wenmingvs](https://github.com/wenmingvs/AndroidProcess/blob/master/README.md),
我为其添加了“第六种方法”，其他并无区别。
以下为原介绍：

-----
提供一个判断App是否处于前台的工具类,拥有多达5种判断方法,最后一种方法堪称Android黑科技（并非我原创）,既可以突破Android5.0以上的权限封锁,获取任意前台App的包名,又不需要权限

AndroidProcess App, require Android 4.0+, GPL v3 License  
[Download Link ](https://github.com/wenmingvs/AndroidProcess/raw/master/demo.apk)  

五种判断方法展示
-----
![enter image description here](http://ww3.sinaimg.cn/large/691cc151gw1f09mz3iz2cg20bc0h0b2d.gif)

用法
-----
传入Context参数与想要判断是否位于前台的App的包名,会返回ture或者false表示App是否位于前台

``` java

//五种方法任选其一

//使用方法一
Boolean isForeground = BackgroundUtil.getRunningTask(context, packageName);
//使用方法二
Boolean isForeground = BackgroundUtil.getRunningAppProcesses(context, packageName);
//使用方法三
Boolean isForeground = BackgroundUtil.getApplicationValue(context);
//使用方法四
Boolean isForeground = BackgroundUtil.queryUsageStats(context, packageName);
//使用方法五
Boolean isForeground = BackgroundUtil.getLinuxCoreInfo(context, packageName);
```

五种方法的区别
-----
|方法|判断原理|是否需要权限读取|是否可以判断其他应用位于前台|特点
| ------ | ------ | ------ | ------ | ------ |
|方法一|RunningTask|否|Android4.0系列可以,5.0以上机器不行|5.0此方法被废弃
|方法二|RunningProcess|否|当App存在后台常驻的Service时失效|无
|方法三|ActivityLifecycleCallbacks|否|否|简单有效,代码最少
|方法四|UsageStatsManager|是|是|最符合Google规范的判断方法
|方法五|读取/proc目录下的信息|否|是|当proc目录下文件夹过多时,此方法是耗时操作


方法一：通过RunningTask
-----

![enter image description here](http://ww2.sinaimg.cn/large/691cc151gw1f09z4gz35mg20bc0h01kx.gif)

**原理**  
当一个App处于前台的时候，会处于RunningTask的这个栈的栈顶，所以我们可以取出RunningTask的栈顶的任务进程，看他与我们的想要判断的App的包名是否相同，来达到效果

**缺点**    
getRunningTask方法在Android5.0以上已经被废弃，只会返回自己和系统的一些不敏感的task，不再返回其他应用的task，用此方法来判断自身App是否处于后台，仍然是有效的，但是无法判断其他应用是否位于前台，因为不再能获取信息

**验证**  
下面是在小米的机子上打印的结果，Android的版本是Android4.4.4的，我们是可以拿到全部的正在运行的应用的信息的。其中包名为com.whee.wheetalklollipop的就是我们需要判断是否处于前台的App，如今他位于第一个，说明是处于前台的

![enter image description here](https://raw.githubusercontent.com/wenmingvs/AndroidProcess/master/sample/1.PNG)

下面是Android5.0上打印的结果，我们虽然打开了很多诸如新浪微博，网易新闻，QQ等App，可是打印出来后却完全看不到了，只有自身的App的信息和一些系统进程的信息，这说明Android5.0的确是做了这么一重限制了，只返回一小部分调用者本身的task和其他一些不太敏感的task。

![enter image description here](https://raw.githubusercontent.com/wenmingvs/AndroidProcess/master/sample/2.PNG)


方法二：通过RunningProcess
-----

![enter image description here](http://ww1.sinaimg.cn/mw690/691cc151gw1f09z4vmmcgg20bc0h07wh.gif)

**原理**  
通过runningProcess获取到一个当前正在运行的进程的List，我们遍历这个List中的每一个进程，判断这个进程的一个importance 属性是否是前台进程，并且包名是否与我们判断的APP的包名一样，如果这两个条件都符合，那么这个App就处于前台

**缺点**：  
在聊天类型的App中，常常需要常驻后台来不间断的获取服务器的消息，这就需要我们把Service设置成START_STICKY，kill 后会被重启（等待5秒左右）来保证Service常驻后台。如果Service设置了这个属性，这个App的进程就会被判断是前台，代码上的表现就是appProcess.importance的值永远是 ActivityManager.RunningAppProcessInfo.IMPORTANCE_FOREGROUND，这样就永远无法判断出到底哪个是前台了。

方法三：通过ActivityLifecycleCallbacks
------

![enter image description here](http://ww2.sinaimg.cn/mw690/691cc151gw1f09z59b1pzg20bc0h04qp.gif)

**原理**  
AndroidSDK14在Application类里增加了ActivityLifecycleCallbacks，我们可以通过这个Callback拿到App所有Activity的生命周期回调。
``` java
 public interface ActivityLifecycleCallbacks {
        void onActivityCreated(Activity activity, Bundle savedInstanceState);
        void onActivityStarted(Activity activity);
        void onActivityResumed(Activity activity);
        void onActivityPaused(Activity activity);
        void onActivityStopped(Activity activity);
        void onActivitySaveInstanceState(Activity activity, Bundle outState);
        void onActivityDestroyed(Activity activity);
    }
```
知道这些信息，我们就可以用更官方的办法来解决问题，当然还是利用方案二里的Activity生命周期的特性，我们只需要在Application的onCreat（）里去注册上述接口，然后由Activity回调回来运行状态即可。

可能还有人在纠结，我用back键切到后台和用Home键切到后台，一样吗？以上方法适用吗？在Android应用开发中一般认为back键是可以捕获的，而Home键是不能捕获的（除非修改framework）,但是上述方法从Activity生命周期着手解决问题，虽然这两种方式的Activity生命周期并不相同，但是二者都会执行onStop（）；所以并不关心到底是触发了哪个键切入后台的。另外,Application是否被销毁,都不会影响判断的正确性

方法四:通过使用UsageStatsManager获取
-----
![enter image description here](http://ww1.sinaimg.cn/mw690/691cc151gw1f09z5v4g7mg20bc0h0npd.gif)

**原理**  
通过使用UsageStatsManager获取，此方法是Android5.0之后提供的新API，可以获取一个时间段内的应用统计信息，但是必须满足一下要求

**使用前提**  
  1. 此方法只在android5.0以上有效 
  2. AndroidManifest中加入此权限 
```  java
<uses-permission xmlns:tools="http://schemas.android.com/tools" android:name="android.permission.PACKAGE_USAGE_STATS" tools:ignore="ProtectedPermissions" />
```
  3. 打开手机设置，点击安全-高级，在有权查看使用情况的应用中，为这个App打上勾

![enter image description here](https://raw.githubusercontent.com/wenmingvs/AndroidProcess/master/sample/3.PNG)


方法五：读取Linux系统内核保存在/proc目录下的process进程信息
----
![enter image description here](http://ww3.sinaimg.cn/mw690/691cc151gw1f09z6bjz9rg20bc0h0b29.gif)



**原理**  
无意中看到乌云上有人提的一个漏洞，Linux系统内核会把process进程信息保存在/proc目录下，Shell命令去获取的他，再根据进程的属性判断是否为前台。国外已经有大神把这个漏洞做成对应的功能，具体的项目在这里，https://github.com/jaredrummler/AndroidProcesses ，最后一种方法引用的是他所提供的代码

**优点**  
1. 不需要任何权限  
2. 可以判断任意一个应用是否在前台，而不局限在自身应用

**缺点**  
1. 当/proc下文件夹过多时,此方法是耗时操作

**用法**  
获取一系列正在运行的App的进程
``` java
List<AndroidAppProcess> processes = ProcessManager.getRunningAppProcesses();
```

获取任一正在运行的App进程的详细信息
``` java
AndroidAppProcess process = processes.get(location);
String processName = process.name;

Stat stat = process.stat();
int pid = stat.getPid();
int parentProcessId = stat.ppid();
long startTime = stat.stime();
int policy = stat.policy();
char state = stat.state();

Statm statm = process.statm();
long totalSizeOfProcess = statm.getSize();
long residentSetSize = statm.getResidentSetSize();

PackageInfo packageInfo = process.getPackageInfo(context, 0);
String appName = packageInfo.applicationInfo.loadLabel(pm).toString();
```

判断是否在前台
``` java
if (ProcessManager.isMyProcessInTheForeground()) {
  // do stuff
}
```

获取一系列正在运行的App进程的详细信息
``` java
List<ActivityManager.RunningAppProcessInfo> processes = ProcessManager.getRunningAppProcessInfo(ctx);
```

方法六：通过 Android 辅助功能(Accessibility Service) 检测任意前台界面
详情参阅：

Gradle 构建
------
- 版本
	- 最新 Android SDK
	- Gradle
- 环境变量
	- ANDROID_HOME
	- GRADLE_HOME，同时把bin放入path变量
	- Android SDK 安装，都更新到最新
	- Android SDK Build-tools更新到最新
	- Google Repository更新到最新
	- Android Support Repository更新到最新
	- Android Support Library更新到最新


相信未来
-----
当蜘蛛网无情地查封了我的炉台   
当灰烬的余烟叹息着贫困的悲哀   
我依然固执地铺平失望的灰烬   
用美丽的雪花写下：相信未来   

当我的紫葡萄化为深秋的露水   
当我的鲜花依偎在别人的情怀   
我依然固执地用凝霜的枯藤   
在凄凉的大地上写下：相信未来   

我要用手指那涌向天边的排浪  
我要用手掌那托住太阳的大海  
摇曳着曙光那枝温暖漂亮的笔杆   
用孩子的笔体写下：相信未来   

我之所以坚定地相信未来  
是我相信未来人们的眼睛  
她有拨开历史风尘的睫毛  
她有看透岁月篇章的瞳孔  

不管人们对于我们腐烂的皮肉  
那些迷途的惆怅、失败的苦痛  
是寄予感动的热泪、深切的同情   
还是给以轻蔑的微笑、辛辣的嘲讽   

我坚信人们对于我们的脊骨  
那无数次的探索、迷途、失败和成功   
一定会给予热情、客观、公正的评定   
是的，我焦急地等待着他们的评定  

朋友，坚定地相信未来吧  
相信不屈不挠的努力  
相信战胜死亡的年轻  
相信未来、热爱生命  
