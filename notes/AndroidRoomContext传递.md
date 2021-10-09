
<!-- TOC -->

- [一、背景](#一背景)
- [二、解决思路](#二解决思路)
- [三、方案实现](#三方案实现)
    - [3.1 LayoutInflater.inflate()](#31-layoutinflaterinflate)
- [3.2 LayoutInflater.from(context)](#32-layoutinflaterfromcontext)
- [四、使用方式](#四使用方式)
- [五、注意事项](#五注意事项)

<!-- /TOC -->

## 一、背景

在直播间上下滑需求中，以直播间为单位，提供了直播间级别的资源获取和基础支持的管理类（简称 RoomContext），RoomContext 将可能用于直播间的任何一个业务模块。

在大部分业务需求中，都离不开视图的体现，由于直播间的视图层级较深且视图数量多，面对着未来持续增长的业务需求，如何方便的把 RoomContext 提供给具体的业务模块使用将成为一个问题。

## 二、解决思路

Android 的视图恰好拥有 Context（上下文）的概念和实现，因此可以通过自定义 Context 的思路去解决该问题：让每一个视图的 Context 都是 RoomContext，即可解决大部分场景下的 RoomContext 获取需求。

## 三、方案实现

对于原有的 Context 方法，还是交由传入的 activity 接管实现。

```java
class LiveRoomContext(val fragmentActivity: FragmentActivity) : ContextThemeWrapper(fragmentActivity, fragmentActivity.theme) {
    // 省略相应的资源和基础支持。
}
```

日常编码中，View 的实例化一般通过以下 2 种方式：

第一是使用 new View(Context context) 的方式，直接传入 RoomContext 作为 View 的 Context 即可。

第二是使用 LayoutInflater 解析 xml 的方式，这种方式也是使用较多且涉及到的视图较多的方式，即：

```java
LayoutInflater.from(context).inflate(resource, root, attachToRoot);
```

> 问题就变成通过 LayoutInflater 解析 XML 得到的 View 的 Context 取的值是什么？

### 3.1 LayoutInflater.inflate()

我们先看 LayoutInflater.inflate() 方法，可以发现 View 取的是 LayoutInflater 实例化传入的 Context。

```java
class LayoutInflater {
    protected final Context mContext;

    protected LayoutInflater(Context context) {
        mContext = context;
    }

    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        // inflaterContext 取的是实例化 LayoutInflater 的 Context。
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        View result = root;
        // mContext 是最终通过传入 View 构造方法中的 context，不在深入贴太多代码。
        final View temp = createViewFromTag(root, name, inflaterContext, attrs);

        if (root != null && attachToRoot) {
            root.addView(temp, params);
        }

        if (root == null || !attachToRoot) {
            result = temp;
        }

        return result;
    }
}
```

### 3.2 LayoutInflater.from(context)

LayoutInflater.from(context) 只是调用了context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)，由于 RoomContext 是 activity 的包装类，因此默认交由 activity 处理该服务，最终拿到的是 PhoneLayoutInflater 对象，该对象是用 activity 去实例化得到的。

```java
class LayoutInflater {
    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
}
```

因此需要重写 Context.getSystemService() 方法，传入 RoomContext 去实例化自己的 LayoutInflater 对象。

```java
class LiveRoomContext(val fragmentActivity: FragmentActivity) : ContextThemeWrapper(fragmentActivity, fragmentActivity.theme) {

    override fun getSystemService(name: String): Any {
        if (Context.LAYOUT_INFLATER_SERVICE == name) {
            if (layoutInflater == null) {
                // baseContext 是 Activity，这样能够使用对应 activity 的 inflateFactory， this 是 RoomContext。
                layoutInflater = LayoutInflater.from(baseContext).cloneInContext(this)
            }
            return layoutInflater as LayoutInflater
        }
        return super.getSystemService(name)
    }

}
```

## 四、使用方式

```java
View 使用：
new View(roomContext)；
XML 使用：
LayoutInflater.from(roomContext).inflate(int resource, ViewGroup root, boolean attachToRoot);
```

## 五、注意事项

-  Fragment.getContext() 默认取的是 Activity，且不可修改，因此只能通过重写解决；
-  Activity 重启时，Fragment 会按 Activity 生命周期进行恢复，此时若直播 View 没有在 onCreate 创建，则没有办法去重新赋值，从而导致 LiveRoomContext 为 null，解决办法有 2 种：
   1. 在 Activity.onCreate() 移除会重建的 fragment；
   2. 对 LiveRoomContext 进行判空，去判断是否属于脏 fragment（重启的fragment），从而不去执行初始化的代码。

```java
class FragmentA extends Fragment {

    private LiveRoomContext mContext;

    public static FragmentA newInstance(LiveRoomContext roomContext) {
        Bundle args = new Bundle();

        FragmentA fragment = new FragmentA();
        fragment.setContext(roomContext);
        fragment.setArguments(args);
        return fragment;
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull android.view.LayoutInflater inflater, @Nullable ViewGroup container,
            @Nullable Bundle savedInstanceState) {
        return LayoutInflater.from(getContext()).inflate(int resource, ViewGroup root, boolean attachToRoot);
    }

    public void setContext(LiveRoomContext context) {
        mContext = context;
    }

    @Nullable
    @Override
    public Context getContext() {
        return mContext;
    }

}
```

