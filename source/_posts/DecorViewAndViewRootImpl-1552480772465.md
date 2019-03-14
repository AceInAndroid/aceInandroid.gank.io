---
title: '#系统源码解读:深入理解DecorView与ViewRootImpl'
tags: [Android源码解析,DecorView,ViewRootImpl]
categories:
  - Android
date: 2019-03-13 20:39:32
---


# 系统源码解读:深入理解DecorView与ViewRootImpl

## 前言
对于Android开发者来说，View无疑是开发中经常接触的，**包括它的事件分发机制、测量、布局、绘制流程等**，如果要自定义一个View，那么应该对以上流程有所了解、研究。本系列文章将会为大家带来View的工作流程详细解析。在深入接触View的测量、布局、绘制这三个流程之前，我们从Activity入手，看看从Activity创建后到View的正式工作之前，所要经历的步骤。以下源码均取自Android API 21。
<!--more-->
## setContentView
从`setContentView`说起
一般地，我们在`Activity`中，会在`onCreate()`方法中写下这样一句：

```JAVA
setContentView(R.layout.main);
```

显然，这是为`activity`设置一个我们定义好的main.xml布局，我们跟踪一下源码，看看这个方法是怎样做的，`Activity`的`setContentView()`:

```JAVA
public void setContentView(@LayoutRes int layoutResID) {
     getWindow().setContentView(layoutResID);  //调用getWindow方法，返回mWindow
     initWindowDecorActionBar();
}
...
public Window getWindow() {   
     return mWindow;
}
```

从上面看出，里面调用了`mWindow`的`setContentView`方法，那么这个**“mWindow”**是何方神圣呢？尝试追踪一下源码，发现`mWindow`是`Window`类型的，但是它是一个抽象类，`setContentView`也是抽象方法，所以我们要找到Window类的实现类才行。我们在Activity中查找一下mWindow在哪里被赋值了，可以发现它在Activity#attach方法中有如下实现：

```JAVA
 final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window) {
    ...
        mWindow = new PhoneWindow(this, window);//创建一个Window对象
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);//设置回调，向Activity分发点击或状态改变等事件
        mWindow.setOnWindowDismissedCallback(this);
     ...
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);//给Window设置WindowManager对象
 ...
    }
```
我们只看关键部分，这里实例化了`PhoneWindow`类，由此得知，`PhoneWindow`是`Window`的实现类，那么我们在`PhoneWindow`类里面找到它的`setContentView`方法，看看它又实现了什么，`PhoneWindow`的`setContentView`:

```JAVA
@Override
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) { // 1
        installDecor();//mContentParent为空，创建一个DecroView
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();//mContentParent不为空，删除其中的View
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent); //为mContentParent添加子View,即Activity中设置的布局文件
    }
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();//回调通知，内容改变
    }
}
```

首先判断了`mContentParent`是否为`null`，如果为空则执行`installDecor()`方法，那么这个`mContentParent`又是什么呢？我们看一下它的注释：

```
// This is the view in which the window contents are placed. It is either
// mDecor itself, or a child of mDecor where the contents go.
private ViewGroup mContentParent;
```

它是一个`ViewGroup`类型，结合2的代码处得知,这个`mContentParent`是我们设置的布局(即main.xml)的父布局。注释还提到了，这个`mContentParent`是`mDecor`本身或者是`mDecor`的一个子元素，这句话什么意思呢？这里先留一个疑问，下面会解释。

这里先梳理一下以上的内容：**通过上面的流程我们大致可以了解先在`PhoneWindow`中创建了一个`DecroView`，其中创建的过程中可能根据`Theme`不同，加载不同的布局格式，例如有没有Title，或有没有`ActionBar`等，然后再向`mContentParent`中加入子View,即Activity中设置的布局。到此位置，视图一层层嵌套添加上了。**

## 创建DecorView
接着上面提到的`installDecor()`方法，我们看看它的源码，**PhoneWindow#installDecor:**

```JAVA
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor(); // 1 生成DecorView
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor); // 2 为DecorView设置布局格式，并返回mContentParent
        ...
        } 
    }
}
```

首先，会执行1的代码，调用`PhoneWindow#generateDecor`方法：

```JAVA
protected DecorView generateDecor() {
    return new DecorView(getContext(), -1);
}
```

可以看出，这里实例化了`DecorView`，而`DecorView`则是`PhoneWindow`类的一个内部类，继承于`FrameLayout`，由此可知它也是一个`ViewGroup`。 
那么，DecroView到底充当了什么样的角色呢？ 
其实，`DecorView`是整个`ViewTree`的最顶层`View`，它是一个`FrameLayout`布局，代表了整个应用的界面。在该布局下面，**有标题view和内容view这两个子元素**，而内容view则是上面提到的mContentParent。

我们接着看2处的代码，**PhoneWindow#generateLayout方法**:


```JAVA
protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.
        // 从主题文件中获取样式信息
        TypedArray a = getWindowStyle();

        ...

        if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestFeature(FEATURE_ACTION_BAR);
        }

        if(...){
            ...
        }

        // Inflate the window decor.
        // 根据主题样式，加载窗口布局
        int layoutResource;
        int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
        } else if(...){
            ...
        }

        View in = mLayoutInflater.inflate(layoutResource, null);    //加载layoutResource
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT)); //往DecorView中添加子View，即mContentParent
        mContentRoot = (ViewGroup) in;

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT); // 这里获取的就是mContentParent  @android:id/content
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        if ((features & (1 << FEATURE_INDETERMINATE_PROGRESS)) != 0) {
            ProgressBar progress = getCircularProgressBar(false);
            if (progress != null) {
                progress.setIndeterminate(true);
            }
        }

        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            registerSwipeCallbacks();
        }

        // Remaining setup -- of background and title -- that only applies
        // to top-level windows.
        ...

        return contentParent;
    }
```

由以上代码可以看出，该方法还是做了相当多的工作的，首先根据设置的主题样式来设置`DecorView`的风格，比如说有没有`titlebar`之类的，接着为`DecorView`添加子`View`，而这里的子`View`则是上面提到的`mContentParent`，如果上面设置了`FEATURE_NO_ACTIONBAR`，那么`DecorView`就只有`mContentParent`一个子`View`，这也解释了上面的疑问：`mContentParent`是`DecorView`本身或者是`DecorView`的一个子元素。 
用一幅图来表示DecorView的结构如下：
![](https://ws1.sinaimg.cn/large/007lnl1egy1g11ffv2vb6j30er0gjgn9.jpg)

**小结：** `DecorView`是顶级`View`，内部有`titlebar`和`contentParent`两个子元素，c`ontentParent`的`id`是`content`，而我们设置的`main.xml`布局则是`contentParent`里面的一个子元素。

在`DecorView`创建完毕后，让我们回到P`honeWindow#setContentView`方法，直接看2处代码： `mLayoutInflater.inflate(layoutResID, mContentParent)`;这里加载了我们设置的`main.xml`布局文件，并且设置`mContentParent`为main.xml的父布局，至于它怎么加载的，这里就不展开来说了。

到目前为止，通过`setContentView`方法，创建了`DecorView`和加载了我们提供的布局，但是这时，**我们的View还是不可见的**，因为我们仅仅是加载了布局，并没有对View进行任何的**测量、布局、绘制**工作。在View进行测量流程之前，还要进行一个步骤，那就是把`DecorView`添加至`window`中，然后经过一系列过程触发`ViewRootImpl#performTraversals`方法，在该方法内部会正式开始测量、布局、绘制这三大流程。至于该一系列过程是怎样的，因为涉及到了很多机制，这里简单说明一下：

## 将DecorView添加至Window

每一个`Activity`组件都有一个关联的`Window`对象，用来描述一个应用程序窗口。每一个应用程序窗口内部又包含有一个`View`对象，用来描述应用程序窗口的视图。上文分析了创建`DecorView`的过程，现在则要把`DecorVie`w添加到`Window`对象中。而要了解这个过程，我们首先要简单先了解一下`Activity`的创建过程： 
首先，在`ActivityThread`#`handleLaunchActivity`中启动`Activity`，在这里面会调用到`Activity`#`onCreate`方法，从而完成上面所述的`DecorView`创建动作，当o`nCreate()`方法执行完毕，在`handleLaunchActivity`方法会继续调用到
`ActivityThread#handleResumeActivity`方法，我们看看这个方法的源码：

```JAVA
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward) { 
    //...
    ActivityClientRecord r = performResumeActivity(token, clearHide); // 这里会调用到onResume()方法

    if (r != null) {
        final Activity a = r.activity;

        //...
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow(); // 获得window对象
            View decor = r.window.getDecorView(); // 获得DecorView对象
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager(); // 获得windowManager对象
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                wm.addView(decor, l); // 调用addView方法
            }
            //...
        }
    }
}
```
在该方法内部，获取该`activity`所关联的`window`对象，`DecorView`对象，以及`WindowManager`对象，而`WindowManager`是抽象类，它的实现类是`WindowManagerImpl`，所以后面调用的是
`WindowManagerImpl#addView方法`，我们看看源码：

```JAVA
public final class WindowManagerImpl implements WindowManager {    
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    ...
    @Override
    public void addView(View view, ViewGroup.LayoutParams params) {
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }
}
```

接着调用了`mGlobal`的成员函数，而`mGlobal`则是`WindowManagerGlobal`的一个实例，那么我们接着看
`WindowManagerGlobal#addView`方法：


```JAVA
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ...

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ...

            root = new ViewRootImpl(view.getContext(), display); // 1

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView); // 2
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }
```

先看1号代码处，实例化了`ViewRootImpl`类，接着，在2号代码处，调用`ViewRootImpl#setView`方法，并把`DecorView`作为参数传递进去，在这个方法内部，会通过跨进程的方式向`WMS（WindowManagerService）`发起一个调用，从而将`DecorView`最终添加到`Window`上，在这个过程中，ViewRootImpl、DecorView和WMS会彼此关联，至于详细过程这里不展开来说了。 
最后通过`WMS`调用`ViewRootImpl#performTraverals`方法,然后依照下图流程层层调用，完成绘制，最终界面才显示出来,下偏文章讲View的绘制.

![](https://ws1.sinaimg.cn/large/007lnl1egy1g11g5op2u0j30iv0c0juh.jpg)



