title: 使用 Android 性能分析工具优化性能
date: 2016-03-13 16:11:16
updated: 2017-02-07 21:47:58
categories: [Android Development]
tags: [android]
---

之前公司基于 android 6.0 的短信开发了一个短信 app，但是 android 原生的短信滑动很卡，刚开始怀疑是头像处理图片的问题，但是把头像去掉后，滑动依旧卡。感觉很奇怪，于是就尝试使用 DDMS 自带的几个工具分析了一下，发现竟然是由于一个 String 使用的问题（当然原生的短信头像处理那里也是有问题的，不优化就算解决了这里的 String 问题照样卡，但是不在这里讨论了），这里稍微记录下这些工具如何使用。由于之前没有当场记录下来，现在想还远现场分析，发现可能是由于后面的修改竟然没办法简单的重新了，于是就自己仿照写了个类似的例子来说明了 ╮("╯₃╰)╭

## 例子

例子很简单就是一个 ListView，每个 item 是一个 TextView，然后固定显示一个字符串，但是在 getView 的时候会进行一个操作，去获取一个很长的字符串，在原生短信中是在 RecyclerView 中的 onBindViewHolder 中获取这个字符串（一个很长的 SQL 语句）然后去查询数据库。下面把关键代码贴一下：

```java

    private void initData() {
        mIsSlow = false;
        mString = getResources().getString(R.string.app_name);
    }

private class MyAdapter extends BaseAdapter {

        private LayoutInflater mLayoutInflater;

        private MyAdapter() {
            super();
            mLayoutInflater = (LayoutInflater) MainActivity.this
                .getSystemService(MainActivity.this.LAYOUT_INFLATER_SERVICE);
        }

        // 2000 个 item 方便滑动测试
        @Override
        public int getCount() {
            return 2000;
        }

        @Override
        public long getItemId(int position) {
            return position;
        }

        // 随便测试的，返回 null 没事
        @Override
        public Object getItem(int position) {
            return null;
        }

        @Override
        public View getView(int position, View convertView, ViewGroup parent) {
            TextView textView = null;
            if (null == convertView) {
                convertView = mLayoutInflater.inflate(R.layout.list_item, null);
            }
            if (null == convertView) {
                return convertView;
            }
            // 每个 item 就只有一个 TextView，这里很简单就没用 ViewHolder 了，影响不大的
            // 然后 mString 这个变量一开始就获取一个固定的字符串
            textView = (TextView) convertView.findViewById(R.id.text_view);
            textView.setText(mString);
            // 比较关键的是下面这个函数的调用
            if (mIsSlow) {
                StringDef.getNotificationQuerySql();
            } else {
                StringDef.getNotificationQuerySql2();
            }
            return convertView;
        }
    }
```

然后贴下获取字符串的函数：

```java
public class StringDef {

    public static final int BUGLE_STATUS_OUTGOING_DRAFT                   = 3;
    public static final int BUGLE_STATUS_INCOMING_COMPLETE                = 100;
    public static final int BUGLE_STATUS_INCOMING_YET_TO_MANUAL_DOWNLOAD  = 101;

    private static final Character QUOTE_CHAR = '\'';
    private static final char DIVIDER = '|';

    private static final String CONVERSATION_MESSAGE_VIEW_PARTS_COUNT =
        "count(" + DatabaseHelper.PARTS_TABLE + '.' + PartColumns._ID + ")";

    private static final String CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL =
        DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns._ID
            + " as " + ConversationMessageViewColumns._ID + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.CONVERSATION_ID
            + " as " + ConversationMessageViewColumns.CONVERSATION_ID + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.SENDER_PARTICIPANT_ID
            + " as " + ConversationMessageViewColumns.PARTICIPANT_ID + ", "

            + makeCaseWhenString(PartColumns._ID, false,
            ConversationMessageViewColumns.PARTS_IDS) + ", "
            + makeCaseWhenString(PartColumns.CONTENT_TYPE, true,
            ConversationMessageViewColumns.PARTS_CONTENT_TYPES) + ", "
            + makeCaseWhenString(PartColumns.CONTENT_URI, true,
            ConversationMessageViewColumns.PARTS_CONTENT_URIS) + ", "
            + makeCaseWhenString(PartColumns.WIDTH, false,
            ConversationMessageViewColumns.PARTS_WIDTHS) + ", "
            + makeCaseWhenString(PartColumns.HEIGHT, false,
            ConversationMessageViewColumns.PARTS_HEIGHTS) + ", "
            + makeCaseWhenString(PartColumns.TEXT, true,
            ConversationMessageViewColumns.PARTS_TEXTS) + ", "

            + CONVERSATION_MESSAGE_VIEW_PARTS_COUNT
            + " as " + ConversationMessageViewColumns.PARTS_COUNT + ", "

            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.SENT_TIMESTAMP
            + " as " + ConversationMessageViewColumns.SENT_TIMESTAMP + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.RECEIVED_TIMESTAMP
            + " as " + ConversationMessageViewColumns.RECEIVED_TIMESTAMP + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.SEEN
            + " as " + ConversationMessageViewColumns.SEEN + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.READ
            + " as " + ConversationMessageViewColumns.READ + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.PROTOCOL
            + " as " + ConversationMessageViewColumns.PROTOCOL + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.STATUS
            + " as " + ConversationMessageViewColumns.STATUS + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.SMS_MESSAGE_URI
            + " as " + ConversationMessageViewColumns.SMS_MESSAGE_URI + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.SMS_PRIORITY
            + " as " + ConversationMessageViewColumns.SMS_PRIORITY + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.SMS_MESSAGE_SIZE
            + " as " + ConversationMessageViewColumns.SMS_MESSAGE_SIZE + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.MMS_SUBJECT
            + " as " + ConversationMessageViewColumns.MMS_SUBJECT + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.MMS_EXPIRY
            + " as " + ConversationMessageViewColumns.MMS_EXPIRY + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.RAW_TELEPHONY_STATUS
            + " as " + ConversationMessageViewColumns.RAW_TELEPHONY_STATUS + ", "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.SELF_PARTICIPANT_ID
            + " as " + ConversationMessageViewColumns.SELF_PARTICIPANT_ID + ", "
            + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns.FULL_NAME
            + " as " + ConversationMessageViewColumns.SENDER_FULL_NAME + ", "
            + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns.FIRST_NAME
            + " as " + ConversationMessageViewColumns.SENDER_FIRST_NAME + ", "
            + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns.DISPLAY_DESTINATION
            + " as " + ConversationMessageViewColumns.SENDER_DISPLAY_DESTINATION + ", "
            + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns.NORMALIZED_DESTINATION
            + " as " + ConversationMessageViewColumns.SENDER_NORMALIZED_DESTINATION + ", "
            + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns.PROFILE_PHOTO_URI
            + " as " + ConversationMessageViewColumns.SENDER_PROFILE_PHOTO_URI + ", "
            + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns.CONTACT_ID
            + " as " + ConversationMessageViewColumns.SENDER_CONTACT_ID + ", "
            + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns.LOOKUP_KEY
            + " as " + ConversationMessageViewColumns.SENDER_CONTACT_LOOKUP_KEY + " ";


    private static final String CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL =
        " FROM " + DatabaseHelper.MESSAGES_TABLE
            + " LEFT JOIN " + DatabaseHelper.PARTS_TABLE
            + " ON (" + DatabaseHelper.MESSAGES_TABLE + "." + MessageColumns._ID
            + "=" + DatabaseHelper.PARTS_TABLE + "." + PartColumns.MESSAGE_ID + ") "
            + " LEFT JOIN " + DatabaseHelper.PARTICIPANTS_TABLE
            + " ON (" + DatabaseHelper.MESSAGES_TABLE + '.' +  MessageColumns.SENDER_PARTICIPANT_ID
            + '=' + DatabaseHelper.PARTICIPANTS_TABLE + '.' + ParticipantColumns._ID + ")"
            // Exclude draft messages from main view
            + " WHERE (" + DatabaseHelper.MESSAGES_TABLE + "." + MessageColumns.STATUS
            + " <> " + BUGLE_STATUS_OUTGOING_DRAFT;

    private static final String CONVERSATION_MESSAGES_QUERY_SQL = "SELECT "
        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        // 这里为了把问题放大，就稍微夸张了点，原来是只有一次的
        // 我多加了9次，因为原来的场景每次 bind view 的时候还有很多别的操作的
        // 所以就算没这里这么夸张，但是只要每次 bind view 慢了 2、3 ms 就能体现出卡了
        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL

        + CONVERSATION_MESSAGES_QUERY_PROJECTION_SQL
        + CONVERSATION_MESSAGES_QUERY_FROM_WHERE_SQL
        ;

    private static final String NOTIFICATION_QUERY_SQL_GROUP_BY =
        " GROUP BY " + DatabaseHelper.PARTS_TABLE + '.' + PartColumns.MESSAGE_ID
            + " ORDER BY "
            + DatabaseHelper.MESSAGES_TABLE + '.' + MessageColumns.RECEIVED_TIMESTAMP + " DESC";

    private final static String NOTIFICATION_QUERY_SQL =
        CONVERSATION_MESSAGES_QUERY_SQL
            + " AND "
            + "(" + DatabaseHelper.MessageColumns.STATUS + " in ("
            + BUGLE_STATUS_INCOMING_COMPLETE + ", "
            + BUGLE_STATUS_INCOMING_YET_TO_MANUAL_DOWNLOAD + ")"
            + " AND "
            + DatabaseHelper.MessageColumns.SEEN + " = 0)"
            + ")"
            + NOTIFICATION_QUERY_SQL_GROUP_BY;

    private static String makeCaseWhenString(final String column,
        final boolean quote,
        final String asColumn) {
        final String fullColumn = makeIfNullString(makePartsTableColumnString(column));
        final String groupConcatTerm = quote
            ? makeGroupConcatString(quote(fullColumn))
            : makeGroupConcatString(fullColumn);
        return "CASE WHEN (" + CONVERSATION_MESSAGE_VIEW_PARTS_COUNT + ">1) THEN " + groupConcatTerm
            + " ELSE " + makePartsTableColumnString(column) + " END AS " + asColumn;
    }

    private static String makeIfNullString(final String column) {
        return "ifnull(" + column + "," + "''" + ")";
    }

    private static String makePartsTableColumnString(final String column) {
        return DatabaseHelper.PARTS_TABLE + '.' + column;
    }

    private static String makeGroupConcatString(final String column) {
        return "group_concat(" + column + ", '" + DIVIDER + "')";
    }

    private static String quote(final String columnName) {
        return "quote(" + columnName + ")";
    }

    // 对，你没看错，原来的这个 SQL 语句就是很复杂的，很长，是通过很多次拼接完成的
    public static final String getNotificationQuerySql() {
        return CONVERSATION_MESSAGES_QUERY_SQL
            + " AND "
            + "(" + DatabaseHelper.MessageColumns.STATUS + " in ("
            + BUGLE_STATUS_INCOMING_COMPLETE + ", "
            + BUGLE_STATUS_INCOMING_YET_TO_MANUAL_DOWNLOAD + ")"
            + " AND "
            + DatabaseHelper.MessageColumns.SEEN + " = 0)"
            + ")"
            + NOTIFICATION_QUERY_SQL_GROUP_BY;
        //return NOTIFICATION_QUERY_SQL;
    }

    // 下面这个是我优化后的开法
    public static final String getNotificationQuerySql2() {
        return NOTIFICATION_QUERY_SQL;
    }

}
```

我界面上弄了个按钮，点击来回切换优化前和优化后的情况，便于观察：

```java
    @Override
    public void onClick(View view) {
        if (view.equals(mBtnSwitch)) {
            mIsSlow = !mIsSlow;
            System.gc();
            mAdapter.notifyDataSetChanged();
        }
    }
```

原生短信每次 bind view 就要通过 getNotificationQuerySql 这个方法去获取 SQL 语句后去查询数据库，我这里只是获取了 SQL 语句，然后什么都没做。结果是我点击按钮回来切换，发现没优化前滑动明显拖慢，并且滑过一定的 item 后就开始卡顿一下。下面通过 android 自带的性能分析工具来分析一下。

## 分析

### Memory

首先在滑动的时候观察 Memory 的变动情况（idea 和 android studio 有），发现没优化前，内存变动很频繁，gc 很多，虽然内存涨幅不大，但是变化很快，稍微涨上去点就 gc 掉下来，很频繁（因为是动态图，我就不截图了）。

### TraceView

然后使用 TraceView 进行分析。idea 中好像没有直接调用的按钮，可以点击工具栏上的 Android Device Monitor 调出单独的 DDMS 界面，左边选中要调试的进程，然后简单工具栏上面的 Start Method Profiling 按钮，会弹出一个框让你选择采样的方式，我是选择第一种默认的每隔 1000ms 采样一次（我还没试过第二种方式，以后再说咯）。点击确定后，就开始采样了，然后去滑动一下，感觉差不多了，就选择 Stop Method Profiling，会在 /tmp/ 下生成一个 ddmsxxxxx.trace 的文件（linux 版下的目录），并且会自动帮你打开（你可以把这个文件保存到别的地方，然后以后也可用 DDMS open file 打开来查看的）。打开后就是下面这个样子的：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/profiling-tools/1.png)

其实结果是分上下2部分的，上面的是时间轴，我不太会看上面的时间轴，目前只是看下面的方法分析的。下面会根据耗费 cpu 的时间（单位 ms）来列出排在前面的方法。我们稍微找一下（这里主要是找自己写的方法，一般排在前面几个的都是 android framework 的一些方法），发现排在 20 位的是我们自己写的 Adapter 的 getView 方法。这里稍微介绍一下后面几个指标的含义（我也是从网上看来的，官方文档竟然没有这些指标的介绍）：

* **1. Incl Cpu Time:**
Incl Cpu Time表示方法top执行的总时间，假如说方法top的执行时间为10ms，方法a执行了1ms，方法b执行了2ms，方法c执行了3ms，方法d执行了4ms（这里是为了举个列子，实际情况中方法a、b、c、d的执行总时间肯定比方法top的执行总时间要小一点）。而且调用方法top的方法的执行时间是100ms，那么：

```java
public void top() {
    a();
    b();
    c();
    d();
}
```

<pre>
Incl Cpu Time
top	10%
a	10%
b	20%
c	30%
d	40%
</pre>

top 方法总共占用了 10% 的 cpu 时间，其中里面的 a、b、c、d 方法分别占用了 top 方法的 10%、20%、30%、40% 。这个指标表示：这个方法以及这个方法的子方法（比如top方法中的a、b、c、d方法）一共执行的时间。例如说，我们看到我们的 getView 方法一共占用了 26.4% 的 cpu 时间（1461ms），其中里面的方法 getNotificationQuerySql 和 setText 分别占用了 88.8%(1298ms) 和 11.2%(164ms)（简单的 View findViewById 这种采样直接就被忽略了）。

* **2. Excl Cpu Time:**
理解了Incl Cpu Time以后就可以很好理解Excl Cpu Time了，还是上面top方法的例子：方法 top 的 Incl Cpu Time 减去 方法 a、b、c、d 的Incl Cpu Time 的时间就是方法 top 的Excl Cpu Time 了。

* **3. Incl Real Time:**
这个感觉和Incl Cpu Time 差不多，第7条会讲到。

* **4. Excl Real Time:**
同上

* **5. Calls + Recur Calls / Total:**
这个指标非常重要！它表示这个方法执行的次数，这个指标中有两个值，一个 Call 表示这个方法调用的次数，Recur Call 表示递归调用次数，继续来看我们的 getView： 可以看到 Calls + Recur Calls 值是 183/0，表示这个方法调用了 183 次，但是没有递归调用。看一下 Children（就是这个方法调用的子方法）： getNotificationQuerySql 是 173/173 表示被调用了 173 次，递归调用了 173 次（好像是由于 String 的拼接，重复调用 StringBuilder 的 append 造成的）。

* **6. Cpu Time / Call:**
重点来了！这个指标应该说是最重要的，我们继续看我们的 getView： 被调用次数为 183 次，而它的 Incl Cpu Time 为 1461ms，那么 getView 每一次执行的时间是 7.988ms。将近 8ms，这个是什么概念，app 要想流畅运行 60fps 的话，那么留给每一帧的时间大概为 16.6 ms。一个 getView 方法就耗了将近一半的时间，其它还有渲染，其它的系统调用，所以卡顿就很正常了。

* **7. Real Time / Call:**
Real Time 和 Cpu Time 我现在还不太明白它们的区别，我的理解应该是: Cpu Time 应该是某个方法占用CPU的时间，Real Time 应该是这个方法的实际运行时间。为什么它们会有区别呢？可能是因为CPU的上下文切换、阻塞、GC等原因方法的实际执行时间要比Cpu Time 要稍微长一点。

### Allocation Tracking

我们通过 TraceView 大概知道 getView 中的 getNotificationQuerySql 有问题，然后根据之前的 Memroy 工具发现 gc 很频繁。所以猜想是不是有什么地方在频繁的 new 对象。所以应该是 getNotificationQuerySql 函数里面有什么地方在不停的 new 对象出来导致卡顿。至于具体原因，我们就再通过另外一个工具来分析一下，这个工具就是 Allocation Tracking。这个工具能够分析出一段时间内，应用申请了哪些对象，还能精确到函数，这样就能知道你写的函数是不是在频繁的申请对象，导致频繁的 gc。

点击 idea 中的 Memory 监视工具旁边有一个 Start Allocation Tracking 的按钮（就在 dump heap 按钮的下面），就开始监视了，然后滑动一段时间后，再点击 Stop Allocation Tracking（同一个按钮），就会在你的工程目录下创建一个 captures 的文件夹，然后创建一个 Allocations_xxxx.xx.xx_xx.xx.xx.alloc（以时间命名） 的文件，并且会帮你自动打开，打开后是这个样子的：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/profiling-tools/2.png)

它能按照2种方式显示，一种是按照申请的对象类型：例如说是 java.lang.String、java.lang.Integer 等这些。我们看到按照类型来看，AbstractStringBuilder 这个类占了将近 71% 的内存，这很夸张啊，字符串占了这么多的内存。接下来我们换第二种显示，按照函数显示：

![](http://7u2hy4.com1.z0.glb.clouddn.com/android/profiling-tools/3.png)

发现在 getNotificationQuerySql 这个方法中申请的对象将近占了 86%，而下面的 StringBuilder 中的 append 方法占了将近 61%，toString 方法占了将近 24%。回头去看下 getNotificationQuerySql 这个方法中，每次返回的 SQL 语句都是通过字符串拼接出来的（使用 String 重载的 + 号），并且这个语句很复杂，拼接次数很多。其实每一次的 String 拼接都会创建新的 String 对象返回。所以这个做法特别在 getView 中造成大量的对象创建，所以滑动的时候伴随着大量 String 对象的创建，会频繁的引发 gc（本身大量的字符串拼接也比较耗时），所以消耗了大量 cpu 时间，造成 UI 卡顿。解决方法很简单，不要每次都拼接就行了，我的改进方法是，使用一个 static 变量保存拼接好的对象，每次返回这个 static 对象就能避免频繁的字符串拼接操作。

## 小结

其实可以看到有些时候优化其实代码只需要改动一点，同时也说明，平常要是不注意的话，一、二句代码就能引发很严重的问题（上次说的内存分析中就是一个系统回调没有注销就能引发内存泄漏）。当然这次例子其实不是很好，被我把问题放大了，其实如果不放大的话，在简单的 ListView 中不会很卡的 ╮("╯₃╰)╭ 。但是这个问题确实是我工作中遇到过的，只不过当时没记录下来而已，这里就当抛砖引玉，记录下如何使用这些工具排查 app 性能问题了。



