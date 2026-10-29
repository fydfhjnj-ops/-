高德地图（魔改横屏版）· 车机稳定加固版
=================================================

产物
----
v4: 高德_车机稳定加固版_v4.apk    203.51 MB   <-- 装这个
    SHA256: 392A3D97FC3F7664E8EB2C67E4D6770D53F88B1CA59310A12B241AD8C240011B

v3: 已废弃(v1 日志层的加剧版, 别装)

v2: 高德_车机稳定加固版_v2.apk    203.51 MB
    SHA256: BC839F26B5A5047445146B685072E74D29F0A517DC17DFD9BDA87E825CC229A2

v1: 高德_车机稳定加固版_v1.apk    203.51 MB   (保留备查)
    SHA256: E0284E3BD658FD57139083461633F1418363EBE183542A43C305E47569AB6A04

v2 相比 v1 只多了一处改动: 首页搜索框下移到状态栏安全区
------------------------------------------------------
原因: 魔改层 UcarHomeInline 用 UcarHomeCard.anchor() 拿到的高德内容视图
      矩形来摆放自己的搜索框，而这个矩形的 top 是窗口顶(0)，正好落在
      系统状态栏底下，导致搜索框被状态栏盖住、点不到。
修法: 在 UcarHomeInline 新增 safeTop()，取系统 status_bar_height
      (取不到时回落 40dp) 再加 6dp 间距，然后对 anchor 执行
      Rect.offset(0, safeTop)，把整块内容整体下移到状态栏安全区以下。
      用 offset 而不是只加 top，是为了保持 anchor 的高度不变，
      否则搜索框会被压扁。

目标环境: 吉利星瑞 2025 / FlymeAuto 2.3 / Android 9 (API 28) / arm64


安装（重要）
------------
1. 必须先卸载车机上原来那个魔改高德。
   原因: 本包重新签名了(原包是 MINIMAP_ 签名，本包是通用 debug 签名)，
   同包名不同签名无法覆盖安装，会报"应用未安装/签名冲突"。
2. 卸载会清掉高德的应用数据(登录状态、离线地图)，需要重新登录。
3. 用 U 盘把本 APK 拷到车机安装。
4. 首次启动会明显变慢(正常现象): 它要在 /data 里重建一个 185MB 的
   "原始APK"用于绕过签名校验。这期间不要杀进程、不要断电。
   建议第一次启动前把车打着火或保持通电，给它 1~2 分钟。


本次改了什么
------------
只替换了 2 个 dex 文件(classes9.dex / classes10.dex)，其余 5292 个
条目逐字节保持不变 —— 这是刻意的，因为该包的签名伪装机制依赖这些
条目字节不变。已用包内自带的原始APK重建算法验证:
重建结果 SHA256 = 03cc8fc8b6966de1bbe3028c8c066885404b81d1abbe605cabb44c082b712984
与包内登记值完全一致，说明签名伪装链未被破坏。

1) 崩溃自诊断(新增 com.marcus.amapfix.Fix)
   安装全局 UncaughtExceptionHandler，崩溃时把完整 Java 堆栈写到:
     /sdcard/amap-ucar-crash.log
     /sdcard/Android/data/com.autonavi.minimap/files/ucar-crash.log
   同时打到 logcat，tag = AmapUcarFix。
   启动成功时会先写一行 "==== AmapUcarFix(o) installed ..."，
   看到这行就说明补丁生效了。

2) 签名伪装链改成"失败不死"
   bin.mt.signature.KillerApplication 的静态初始化里，
   killPM(签名伪装) 与 killOpen(重建原始APK + 加载 hook so)
   各自包了 try/catch。原版这里是 throw 直接抛，任一步失败就会
   ExceptionInInitializerError 秒退；现在只记录日志继续启动。

3) 渲染/启动层 19 个入口 fail-safe
   com.marcus.amapphone 下这些方法统一包了 try/catch，异常只落日志:
     UcarStartup : prepare / splash / mapActivity / destroyed
     MapReveal   : beforeShow / continueReveal / afterShow / hide /
                   moved / beforeAdd / destroyed / owns
     UcarDensity : wrap / configured / syncDisplay / nativeDensity
     FloatPip    : changed / enter / showing
   其中返回值非空的方法给了安全兜底(不返回 null):
     UcarDensity.wrap       -> 返回传入的 Context
     UcarDensity.configured -> 返回传入的 Configuration
     UcarDensity.nativeDensity -> 返回传入的 density
     MapReveal.owns / FloatPip.enter / FloatPip.showing -> false
   另外在 UcarStartup.prepare 里补装了一次崩溃处理器，
   因为高德自己会在 Application 初始化时覆盖系统默认处理器。

车道级导航: 完全没有改动，功能保留。


没有做什么(必须说清楚)
----------------------
- 没有做真正的"渲染性能调优"。没有设备、没有 GPU/帧率数据、没有堆栈，
  任何所谓调参都是瞎猜。这次做的是"别崩 + 崩了能查"，不是提速。
- 如果崩溃是 native 层(渲染引擎 libamapr.so、GL 驱动)的段错误，
  或者内存爆掉被系统杀，上面这套 Java 层加固救不了，只能靠日志先定性。


装完请按这个顺序测
------------------
1. 打开一次，等它建完原始APK，确认能进地图。
2. 进 mod 自己的「车机设置」，把这两项各试一遍(无需重装):
     · 地图场景 -> 选「UCAR原场景」或「导航地图（UCAR组件保留）」
     · 3D建筑   -> 关闭
   车道级导航在这几种场景下都保留。如果某一种不闪退，就是那条渲染
   路径的锅，后续我按那个方向精简。
3. 复现车道级导航，看是否还卡住闪退。


如果还闪退，请给我这个文件
--------------------------
  /sdcard/amap-ucar-crash.log
车机文件管理器直接能打开 /sdcard 根目录。里面有 FATAL 段就是崩溃堆栈，
"---- caught in mod layer" 段是魔改层被拦下来的异常(说明拦住了但功能失效)。
把整个文件发我即可。

v4 修的是什么（重要）
------------------------------------------------------
起因: v1 在车机上出现过"整台车机卡死、点什么都没反应"，视频确认是
      画面还在正常渲染但触摸全无响应 —— 主线程 ANR，不是闪退。

责任在我: v1 给魔改层加的 try/catch 一旦命中高频回调(比如 MapReveal
的 moved/beforeAdd/afterShow, 每次布局/每帧都可能调)，每次都会走
note(): 现场拼完整堆栈 + 同步写 3 个文件 + Log，全在主线程。
车机 eMMC 上单次就可能几十到几百毫秒，连续触发直接堵死主线程。

v4 把日志层改造成"不可能阻塞主线程":
  · install() 加 ready 标志，全进程只真正执行一次(原来 UcarStartup.prepare
    每次调用都会 banner+写3个文件)
  · note() 全进程最多写 12 次，且相邻两次至少间隔 1500ms
  · 堆栈裁剪到 3000 字符(原来可能写进几十 KB)
  · proc() 结果缓存，不再每次读 /proc/self/cmdline
  · Toast 上限 4 次

业务改动一概未动: 19 个渲染/启动入口的 try/catch 仍然保留(吞异常策略不变)，
v2 的搜索框状态栏安全区修复继续有效。

如果 v4 仍出现卡死(判断方法: 卡死前有没有弹 "UcarFix: xxx" Toast)
  · 弹了 -> 是我的吞异常让某个流程卡在半状态，我按那条类名精准处理
  · 没弹 -> 基本可排除日志层，嫌疑转向魔改包自身的浮窗/抓帧
            (FloatBridge/WindowAttachment/MapBufferProbe) 在 FlymeAuto 上
            把主线程锁死；那种情况我出"安全模式"版本: 直接禁用这些
            抓帧与浮窗 hook(不吞、不记录、干脆不执行)，代价是少一部分
            魔改特效，换稳定。

从卡死里恢复车机
------------------------------------------------------
长按车机电源键/音量旋钮约 10 秒强制重启；或等系统 ANR 超时自己退出。
测试时建议先停车通电、不要在行驶中试新包。
