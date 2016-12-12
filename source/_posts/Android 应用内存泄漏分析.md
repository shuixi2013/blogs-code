title: Android 应用内存泄漏问题分析
date: 2015-11-02 00:35:16
updated: 2016-12-12 23:49:16
categories: [Android Development]
tags: [android]
--- 

## Java 真的有内存泄漏问题么

会 java 的人都知道 java 没有 delete 也没有 free 方法，只有 new，java 的内存是虚拟机来自动管理的，程序员不需要关心内存回收，虚拟机会自动管理。所以大家都认为 java 开发比 C/C++ 方便很多，因为再也不用关心内存释放、泄漏问题了。很多人都是这么认为的，开发过程中也是这门做的。java 真的不存在有内存泄漏问题么，对此我只能话说：呵呵，拿衣服。java 不仅有内存泄漏问题，而且一旦发生了，排查起来比 C/C++ 更加麻烦。你不信，你在写 android 的时候没遇到过 OOM 么，没内存泄漏哪来的 out of memory，所以说拿衣服。

java 虚拟机管理内存的方式是通过引用计数来实现的，就是说虚拟机通过一个计数来识别一个对象是否还在被使用，如果该对象没有人使用（引用计数为0），那么虚拟机会在合适的时机回收这个对象占用的内存（俗称 GC）。换句话说，只要有一个对象的引用计数不为 0，那么它就无法被回收掉，如果这种对象越来越多，那可用内存就越来越少，最后就会 OOM。也就说如果你的 app 里面存在这种对象，那么就算是 java 也会有内存泄漏的。

## 现象

那什么情况下会导致引用计数无法清零呢。其实就我知道的有一种简单的办法，也是比较常见的，就是 java 中的各种容器的使用，例如说：List、Map 之类的。例如说你的 app 中 new 了一个 ArrayList，然后往里面添加了一些对象，然后用完就完了。我们这里先把使用场景说一下，以一个 activity 为单位，activity 退出（onDestroy）就算是用完了。接着上面说的，这种情况下，虽然 activity onDestroy 了，正常来说，就没有人引用这个 activity 了，它里面的各种对象也相应的可以回收，但是注意一点，这里你使用了一个容器：ArrayList， ArrayList 这个容器的对象是没人引用了，但是你往里面加入的对象（add）还是会被这个 ArrayList 引用的，这些 add 的对象的引用计数是无法清零的。需要你调用 ArrayList 的 clear 方法才行，但是 onDestroy 之后，这个 ArrayList 的对象就销毁了，这就有点像 C 里面的野指针了，这些对象就无法回收，就会造成内存泄漏。

还有一种，也和上面的类似，就是一些 static 的对象，如果你的一些对象被一些 static 的对象持有了，也是会造成内存泄漏的，因为 static 的对象生命周期是整个进程范围的，所以只要进程不重新启动（你不会认为按 back 键退出 activity 会退出进程吧，不太清楚的去看下我有关 start activity 的分析），你让这些 static 持有的对象就会一些存在。典型的来说，你把你的 activity 的对象（Context）传给了某个 static 的对象，然后 activity onDestroy 了，你会认为这个 activity 对象应该会被销毁，系统也是这么认为的，所以你下次启动这个 activity 的时候，系统会帮你重新创建一个新的 activity 对象，但是其实老的还在，然后不停的进出几次，内存的就爆满了。

这么说不太形象，来个例子说明一下吧：

```java
public class MainActivity extends Activity implements View.OnClickListener,
    ClipboardManager.OnPrimaryClipChangedListener {

	// 测试用的图片
    private final static int[] DR_LIST = {
        R.drawable.poster_1,
        R.drawable.poster_2,
        R.drawable.poster_3,
        R.drawable.poster_4,
        R.drawable.poster_5
    };

    private ArrayList<Drawable> mDrList;
    private ClipboardManager mClipboardManager;

    private Button mBtnLoad;

    private MyAdapter mAdapter;
    private ListView mListView;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        initData();
        initView();
    }

    private void initData() {
        mDrList = new ArrayList<Drawable>();
        mClipboardManager = (ClipboardManager) getSystemService(Context.CLIPBOARD_SERVICE);
        // 假设我们的应用需要监听系统的剪切板，来完成某个功能
        mClipboardManager.addPrimaryClipChangedListener(this);
    }

    private void initView() {
        mBtnLoad = (Button) findViewById(R.id.btn_load);

        mListView = (ListView) findViewById(R.id.lv_image);
        mAdapter = new MyAdapter();
        mListView.setAdapter(mAdapter);

        mBtnLoad.setOnClickListener(this);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        mDrList.clear();
        mAdapter.notifyDataSetChanged();
    }

    @Override
    public void onClick(View view) {
        if (view.equals(mBtnLoad)) {
            onBtnLoadClick();
        }
    }

    @Override
    public void onPrimaryClipChanged() {
        // 系统剪切版的内容变了，我们来完成我们自己的业务逻辑
    }

    private void onBtnLoadClick() {
        Resources res = getResources();
        for (int id : DR_LIST) {
            mDrList.add(res.getDrawable(id));
        }
        mAdapter.notifyDataSetChanged();
    }

    private class MyAdapter extends BaseAdapter {

        private LayoutInflater mLayoutInflater;

        private MyAdapter() {
            super();
            mLayoutInflater = (LayoutInflater) MainActivity.this
                .getSystemService(MainActivity.this.LAYOUT_INFLATER_SERVICE);
        }

        @Override
        public int getCount() {
            return mDrList.size();
        }

        @Override
        public long getItemId(int position) {
            return position;
        }

        @Override
        public Object getItem(int position) {
            return mDrList.get(position);
        }

        @Override
        public View getView(int position, View convertView, ViewGroup parent) {
            ImageView iv = (ImageView) convertView;
            if (null == convertView) {
                iv = (ImageView) mLayoutInflater.inflate(R.layout.list_item, null);
            }
            iv.setImageDrawable(mDrList.get(position));
            return iv;
        }
    }
}
```

上面这个例子算得上去我工作中遇到的一个活生生例子的缩写。上面代码很简单，就是按一个 button 给 ListView 加载一些图片，然后就是给系统的剪切版设置了一个监听器，因为需要监控系统的剪切版的变化。大家是不是发现没什么问题？对，这个跑起来确实没什么问题（抛去我只是写例子，少了一些 null、越界的判断）。但是就是这个一个简单的 activity 你每次进去点击加载图片的 button，然后多进出几次就会发现 OOM 了。这是怎么回事咧。下面我就这个例子来分析 andorid 应用中的内存泄漏问题。

## 分析

### 分析内存趋势

android 内存分析有很多工具，但是我使用的是官方推荐的那几个。第一个就是 ideal（android studio） 带的 **Memory Monitor**。先运行上面的程序，然后点开 Memory Monitor，最左边是你设备上能够调试（debugable）的程序。要想调试内存问题，需要能够调试的程序，一般你用 ideal 运行 debug 版的都是可以调试的（活活，我的 nexus 是 user-debug 版的，所有程序都是 debugable 的）。选中我们的例子（com.gmail.killer.mingming.oomtest），然后最右边那个图就会动态的显示你选中的程序的内存情况，现在看到一开始什么都没干的情况下，在我的 nexus 上是 7M 左右：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mm_1.png)

然后点击下 load 按钮，发现会涨到了 24M 左右：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mm_2.png)

然后按 back 键退出 activity，这个时候点击 Memory Monitor 那个窗口左上角那个小货车的图标，这个是强制 GC 的功能，注意这个和你在 app 中调用 System.GC 是不一样的，按这个按钮是一定会执行 GC 操作，当然一次可能不会把所有能释放掉的内存回收掉，就可以多点击几次，这个时候发现内存降到 17M 了：


![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mm_3.png)

内存不是降下来了么，别高兴得太早，你应该注意到现在 activity 是按 back 键退出的，就是执行了 onDestroy 方法的，系统会认为这个 activity 已经销毁了，其实正常也应该是这样，正常来说点击 GC 数次之后，内存应该回到最开始 7M 左右的水平。那这是怎么回事咧。

这个时候我们要配合使用 dumpsys 这个 android 的系统命令行工具，输入： 

<pre>
adb shell dumpsys meminfo com.gmail.killer.mingming.oomtest（后面这个是你应用的包名）
</pre>

然后会看到下面的内容：

```bash
Applications Memory Usage (kB):
Uptime: 168253826 Realtime: 2415297707

** MEMINFO in pid 29036 [com.gmail.killer.mingming.oomtest] **
                   Pss  Private  Private  Swapped     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap     5889     5804        0        0     8056     8056    15495
  Dalvik Heap    11445    11240        0        0    23805    17945     5860
 Dalvik Other      332      332        0        0                           
        Stack      108      108        0        0                           
    Other dev     9924     9920        4        0                           
     .so mmap      871      300        4        0                           
    .apk mmap      112        0        8        0                           
    .ttf mmap       50        0       44        0                           
    .dex mmap       24        0       20        0                           
    code mmap     1017        0      232        0                           
   image mmap      818      508        8        0                           
   Other mmap        4        4        0        0                           
     Graphics     7744     7744        0        0                           
           GL    14188    14188        0        0                           
      Unknown      132      132        0        0                           
        TOTAL    52658    50280      320        0    31861    26001    21355
 
 Objects
               Views:       19         ViewRootImpl:        0
         AppContexts:        3           Activities:        1
              Assets:        2        AssetManagers:        2
       Local Binders:        9        Proxy Binders:       16
    Death Recipients:        0
     OpenSSL Sockets:        0
 
 SQL
         MEMORY_USED:        0
  PAGECACHE_OVERFLOW:        0          MALLOC_SIZE:        0

```

注意看 Objects 的内容，这里会发现 Views 和 Activities 的个数分别是 19 和 1。这 1 个 activity 就是刚刚那个应该别销毁的 activity，然后由于这个 activity 残留了下来，导致他使用的 view 也残留了下来，然后因为上面我们给 ImageView 设置了不少 Drawaable（我例子中都是比较大的图片），所以你会发现内存比最开始占用了不少。如果你多进出几次，然后重复上面的操作，你发现内存涨得非常快。然后敲一下上面的 dumpsys 看一下会发现，你进出了几次 activity 就会留下几个 activity 的对象：

```bash
Applications Memory Usage (kB):
Uptime: 168592867 Realtime: 2415636748

** MEMINFO in pid 29036 [com.gmail.killer.mingming.oomtest] **
                   Pss  Private  Private  Swapped     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap     7011     6928        0        0     8605     8605    16994
  Dalvik Heap    18869    18664        0        0    33502    25342     8160
 Dalvik Other      336      336        0        0                           
        Stack      112      112        0        0                           
    Other dev     7552     4788        4        0                           
     .so mmap      879      304        4        0                           
    .apk mmap      112        0        8        0                           
    .ttf mmap       50        0       44        0                           
    .dex mmap       24        0       20        0                           
    code mmap     1088        0      280        0                           
   image mmap     2116      516      360        0                           
   Other mmap        4        4        0        0                           
     Graphics     7744     7744        0        0                           
           GL    14636    14636        0        0                           
      Unknown      132      132        0        0                           
        TOTAL    60665    54164      720        0    42107    33947    25154
 
 Objects
               Views:       76         ViewRootImpl:        0
         AppContexts:        6           Activities:        4
              Assets:        2        AssetManagers:        2
       Local Binders:       12        Proxy Binders:       19
    Death Recipients:        0
     OpenSSL Sockets:        0
 
 SQL
         MEMORY_USED:        0
  PAGECACHE_OVERFLOW:        0          MALLOC_SIZE:        0

```

这为什么会有 activity 残留咧，下面我们使用另一个工具来排查问题：

### 分析对象持有

这个工具就是 [Memory Analyzer](http://www.eclipse.org/mat/ "Memory Analyzer") ，使用它之前，你需要导出应用当前的内存。导出使用 ideal（android studio）自带的 DDMS 就行，上面的图最左边选进程那里，的左侧工具栏，有一个绿色下载箭头的图标（Dump Java Heap），点击一下，会让你选一个路径，然后保存的是一个 hprof 文件。打开 Memory Analyzer（mat），打开刚刚导出的 hprof 文件，会看到一个 Overview 的界面：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mat_1.png)


上面大致显示了应用当前内存的概要，我们点击下面的 Top Consumers，会出现最占内存的对象和类：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mat_2.png)

会发现是 Drawalbe 对象，这里如果没有任何头绪的，可能会去找为什么会有 Drawable，谁持有它，这会很麻烦。不过前面我们使用 dumpsys 工具，已经知道了是因为有 activity 残留。这里可以看到从 mat 看最占内存的对象，不一定就能找到问题的本质。查找内存问题，其实你有了一定的经验就知道套路了。既然我们知道是因为 activity 有残留，那就要找到导致 activity 残留的原因。mat 有一个很强大的功能，叫 OQL（Object Query Language），和 SQL 类似，能查询内存中的对象。通过上面的工具栏打开 OQL，然后会出现一个类似 SQL 中的输入栏，在里面输入：

<pre>
select * from instanceof android.app.Activity
</pre>

这个很像 SQL 吧，这句话的意思是查找所有继承自 Activity 的对象，并显示所有的属性（关于 OQL 的语法和功能可以看 mat 自带的帮助文档）。然后按上面的 ！ 号执行查询语句（快捷键 F5），就会出现查询结果：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mat_3.png)

会发现有4个我们的 activity 的对象，和 dumpsys 看到的 activity 残留个数是一样的。然后 mat 有一个功能可以查看谁持有了这个对象的引用，右点击一个 activity 的对象，选择：

<pre>
Merge Shortes Paths to GC Roots 
-->
exclude weak/soft references
</pre>

这个命令的意思字面意思是（exclude weak/soft references 是排除弱应用和软应用，只有强引用[strong references]才会造成内存泄漏）：从 GC 根上合并显示最近的一条道路，至于要怎么理解这个意思可以去看 mat 自带的帮助文档中有关 java GC 回收的说明。其实就是话说 GC 回收有一定顺序的，举个例子： 有3个对象 A、B、C，C 被 B 引用，B 被 A 引用，如果要回收 C，那么首先 B 要释放对 C 的引用。那么我们假设 B 要在自己被释放的时候才会不引用 C，那么 C 要回收就首先要 B 被回收。同理我们再假设 A 对 B 的引用也是要到 A 被释放的时候才会释放。那么 C 被回收的路径就是 A->B->C ，上面的命令的意思大概就是这个意思（其实 java 的引用有很复杂的情况的，具体的看 mat 的文档说明吧）。反正这个命令能帮助找到上面例子中最近的那个路径，其实也就是谁持有了这个对象。如果这个命令显示的数据不太对，你也可以使用 Paths to GC Roots，这个就会显示所有的路径，对分析会造成一些干扰。下面我们来看结果：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mat_4.png)

竟然是系统的 ClipboardManager 持有我们的 activity 对象！！这个是怎么回事咧，仔细看下下面这段代码：

```java
        // 假设我们的应用需要监听系统的剪切板，来完成某个功能
        mClipboardManager.addPrimaryClipChangedListener(this);
```

我们把 activity 对象传递给了系统的 ClipboardManager。查阅一下 API 文档会发现 ClipboardManager 还有一个 removePrimaryClipChangedListener 的接口。很多 android 新手几乎不会调用系统提供的一些 remove 接口，就只会 add。因为他们认为 java 会自动帮他们管理内存，java 不存在内存泄漏，但是你想想看，为什么 android 还会提供一个 remove 的接口？为什么 ClipboardManager 的监听接口叫 add，而 View 的 OnClickListener 叫 set。很简单啊，有 add 就有 remove 么，就是要让你成对调用（还记得 C++ 要成对调用 new 和 delete 么），但是 set 就不需要。所以这种系统服务的 add 接口很猫腻的。

我就来稍微解释下，为什么不调用 remove 方法会造成 activity 残留（进而导致内存泄漏），因为 ClipboardManager 里面那个了一个 list 来保存要监听剪切版变化的接口，因为系统服务只有一个，要让大家都能监听，所以只好拿一个 list 来保存咯，然后系统服务是一直存在的（你可以理解为是 static 的），所以你只 add，不 remove 的话，你传递过去的 activity 对象会被这个系统服务持有，但是你的 activity onDestroy 了，系统认为它应该被销毁掉了，所以下次会重新创建一个 activity 对象，这样来来回回，activity 对象就会越来越多，activity 持有的对象也会越来越来，典型的就是 activity 中的 view ，然后那些 view 的背景是图片的之类的话，就会很恐怖了。

要改这个问题很简单，在 onDestroy 加上 ClipboardManager 的 removePrimaryClipChangedListener 方法调用就行了：

```java
    @Override
    protected void onDestroy() {
        super.onDestroy();

        mDrList.clear();
        mAdapter.notifyDataSetChanged();

        mClipboardManager.removePrimaryClipChangedListener(this);
    }
```

然后重复上面的例子，会发现多次 GC 后，内存又回到了最开始的样子：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mm_4.png)

然后用 dumpsys 来查看，activity 个数是0：

```bash
Applications Memory Usage (kB):
Uptime: 172539344 Realtime: 2419583224

** MEMINFO in pid 3092 [com.gmail.killer.mingming.oomtest] **
                   Pss  Private  Private  Swapped     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap     5882     5812        0        0     8455     8455    15096
  Dalvik Heap      775      584        0        0     9369     7059     2310
 Dalvik Other      316      316        0        0                           
        Stack      112      112        0        0                           
    Other dev     6264     6260        4        0                           
     .so mmap      923      308        8        0                           
    .apk mmap      112        0        8        0                           
    .ttf mmap       50        0       44        0                           
    .dex mmap       24        0       20        0                           
    code mmap      890        0      140        0                           
   image mmap     2104      516      360        0                           
   Other mmap        4        4        0        0                           
     Graphics     7744     7744        0        0                           
           GL    10588    10588        0        0                           
      Unknown      132      132        0        0                           
        TOTAL    35920    32376      584        0    17824    15514    17406
 
 Objects
               Views:        0         ViewRootImpl:        0
         AppContexts:        2           Activities:        0
              Assets:        2        AssetManagers:        2
       Local Binders:        8        Proxy Binders:       16
    Death Recipients:        0
     OpenSSL Sockets:        0
 
 SQL
         MEMORY_USED:        0
  PAGECACHE_OVERFLOW:        0          MALLOC_SIZE:        0

```

### 导入泄漏的 Bitmap

使用 mat 一般发现最大的泄漏都是 Bitmap，例如：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mat_5.png)

除了上面的说的分析 object 引用链，一般我们都想把 Bitmap 导出来看一下到底是哪些图片被持有，无法释放，这样就很直观了。在 mat 中是可以倒出来的。找到泄漏的 Bitmap 对象，其中有一个成员变量是 **mBuffer**（pixel datas），右键然后 --> Copy --> Save Value To File，选择保存路径，保存的文件后缀名为 .data ，同时注意记住 Bitmap 的 mWidth 和 mHeight ，导出需要用到的： 

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mat_6.png)

在 linux 上可以使用 **GIMP** 打开刚刚导出的 .data 文件（window，mac 我就不知道用什么软件可以打开了，应该支持 raw 格式的都可以吧），然后填写正确的参数： Width，Height，Image Type 。 Width，Height 刚刚在 mat 里可以看得到。至于 Image Type，一般 Bitmap 是 png 的就选 RGB Alpha，是 jpeg 的就选 RGB （或者 RGB565）。你要说你怎么知道是什么格式的，每一个都试一下，能正确显示图片就算对了。参数设置正确后，就能看到导出的图片了：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/oom-note/mat_dump_image.png)


## 总结

总结一下，android 应用中内存泄漏其实就是对象生命周期的问题，要想避免内存泄漏问题，首先写代码的人从意识上就要保留有**资源什么时候加载，什么时候释放**的意识。保持有这个意识才会去注意到会有内存泄漏的问题，这里给出几个写 android 代码的建议：

* 凡是 List、Map 容器类，add 了对象后，不用了，一定要 remove、clean

* 凡是系统有 add 的接口，注意找 API 文档有没有 remove 的方法，有的话，不用了，一定要调用 remove 方法

* 尽量少用 static 对象，因为 static 对象的生命周期是最长的

* 传递 Context 对象的时候，能传递 Application 的，就不要传递 Activity 的，把 Activity 被持有的概率降到最低


然后所下我遇到过的坑。在一些三星的手机上会有一些奇怪的问题，一些系统服务器总会持有最近一个 activity 的对象，不知道一些三星系统开了什么东西，所以排查内存问题只好使用 nexus 系列，或者说以 nexus 系统作为标准。但是如果实现要兼顾一些其它的手机，就算 activity 会残留一些，可以手动把一些 Drawable 释放掉，例如说一些 view 的背景啊，ImageView 设置 drawable 为 null 啊（其实最占内存的就是 Bitmap）。但是注意不要再多做一些别的多余事情了，例如说在 onDestroy 的时候手动把 activity 的 contentview 删掉之类的，这些多余的操作反而会造成在正常的 nexus 系列上产生问题。

然后可以编写一些 monkey runner 的脚本，自己的进出应用的各种 activity ，进行操作，然后让这个测试脚本循环跑个十几次，之后再使用上面的工具查看内存和 activity 的残留情况。如果没有残留的话，基本上你的应用就没什么内存泄漏问题了。 
 

