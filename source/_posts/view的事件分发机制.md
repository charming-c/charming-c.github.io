---
title: view的事件分发机制
date: 2021-10-16 23:47:42
tags: [view]
categories: Android



---

# 浅谈view、activity、window

## 1、从 setContentView（）出发

对于activity的添加一个布局我们一般都会使用这个方法，我们尝试从源码的角度去解读这样的一方法的工作流程

```java
/**
     * Set the activity content from a layout resource.  The resource will be
     * inflated, adding all top-level views to the activity.
     *
     * @param layoutResID Resource ID to be inflated.
     *
     * @see #setContentView(android.view.View)
     * @see #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)
     */
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

这里就可以看到，我们的布局文件其实并不是加载进activity中的，而是调用了当前 activity 所持有的一个 Window 对象，然后调用该对象 mWindow 的 setContenView() 方法，而这个 mWindow 是在 attach 方法里赋的初值 ` mWindow = new PhoneWindow(this, window, activityConfigCallback); ` 他实际是一个 PhoneWindow 的 对象，对于 PhoneWindow的方法：

<!--more-->

```java
public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
          //移除该mContentParent下的所有View
          //又因为这个的存在  我们可以多次使用setContentView()
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
          //设置动画场景
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

所以最终调用的是上面的一个setContentView方法，解释一下上面的一套流程和参数：

- mContentParent：一个ViewGroup对象，里面存放的就是我们要绘制和显示的view，所以我们在activity中添加显示的view只是默认的ViewGroup的一个子view

- installDector()：这个方法主要就是初始一个DecorView

  ```java
  private void installDecor() {
          mForceDecorInstall = false;
          if (mDecor == null) {
              mDecor = generateDecor(-1);
              mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
              mDecor.setIsRootNamespace(true);
              if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                  mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
              }
          } else {
              mDecor.setWindow(this);
          }
          if (mContentParent == null) {
              mContentParent = generateLayout(mDecor);
  		...
  ```

  检查mDecor 是否为空，是则初始化一个DecorView，否，则将当前的window与初始化的decorView绑定，下面的generateLayout注意到DectorView本身的主题theme，style其实也被加载了，最后返回所有的子view到phoneWindow的mContentParent这样的一个ViewGroup属性里。

第二个方法`initWindowDecorActionBar()`则是将decorView中的actorBar子控件全部初始化

简单梳理一下这样一个方法的调用流程：

![image-20211017015058281](https://tva1.sinaimg.cn/large/008i3skNgy1gvhoo2mz3ej60v70rsjvd02.jpg)

这里就可以讲解一下关于这三样东西的功能：

- Activity是一个工人，它来控制Window；Window是一面显示屏，用来显示信息；View就是要显示在显示屏上的信息，这些View都是层层重叠在一起（通过infalte()和addView()）放到Window显示屏上的。
- Activity并不负责视图控制，它只是控制生命周期和处理事件，真正控制视图的是Window。一个Activity包含了一个Window，Window才是真正代表一个窗口，Window 中持有一个 DecorView，而这个DecorView才是 view 的根布局。
- DecorView可以被认为是Android视图树的根节点视图。DecorView作为顶级View，一般情况下它上面的是标题栏，下面的是内容栏。在Activity中通过setContentView所设置的布局文件其实就是被加到内容栏之中的，而内容栏的id是content，在代码中可以通过ViewGroup content = （ViewGroup)findViewById(R.android.id.content)来得到content对应的layout。

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gvhosj6dpej60eq0eqdgg02.jpg)

# MotionEvent 事件

单点触控事件：

| 事件           | 简介                                 |
| -------------- | ------------------------------------ |
| ACTION_DOWN    | 手指初次触碰屏幕时触发               |
| ACTION_MOVE    | 手指在屏幕上滑动时会触发，可多次触发 |
| ACTION_UP      | 手指离开屏幕时触发                   |
| ACTION_CACEL   | 事件被上层拦截时触发                 |
| ACTION_OUTSIDE | 手指不在控件区域时触发               |

一般来说，从ACTION_DOWN 到ACTION_UP结束为一个事件序列

## 从一个例子出发：

```java
public class MainActivity extends AppCompatActivity {
    Button button;
    int time;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        button = findViewById(R.id.button);

        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.e("clickEvent","time"+time++);
            }
        });


        button.setOnTouchListener(new View.OnTouchListener() {

            @SuppressLint("ClickableViewAccessibility")
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.e("touchEvent","time"+time++);
                return true;
            }
        });
    }
}
```

点击事件的响应结果：

![image-20211017022624260](https://tva1.sinaimg.cn/large/008i3skNgy1gvhpovelsbj60sm07gdi902.jpg)

无论点击按钮多少下，其实最终调用的都是onTouch方法，而没有调用onClick方法而这里的原因就可以从这里的return true 找起。

检查一下onTouch方法的调用情况：

``` java
 if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
```

在view的`public boolean dispatchTouchEvent(MotionEvent event)`方法里onTouch方法被调用，可知，当一个安全的事件发生时，先判断View是否是可用的，并前当前操作是为了操作滚动条的，如果是，就先将返回值确定为true，然后初始化ListenInfo，可以发现，当我们重写了onTouch方法返回true以后，这里的result也被赋值为true，再下面一行的onTouchEvent就不会被调用，而我们去看看这个方法会发现：

![image-20211017025942977](https://tva1.sinaimg.cn/large/008i3skNgy1gvhqnj2mbbj60yw0oiad602.jpg)

![image-20211017030022973](https://tva1.sinaimg.cn/large/008i3skNgy1gvhqo8pf5aj61920fcwgg02.jpg)

![image-20211017030239010](https://tva1.sinaimg.cn/large/008i3skNgy1gvhqqks6ukj60v40beq4002.jpg)

onclick方法其实是在onTouchEvent的某个条件下出发的，这也就是上面为什么会出现不调用onclick方法的原因。

从上面的例子我门引出一个事件是如何在各级的view实现分发的。

首先我们需要了解三个方法：

> - **public boolean dispatchTouchEvent(MotionEvent event)**
>    通过方法名我们不难猜测，它就是事件分发的重要方法。那么很明显，如果一个MotionEvent传递给了View，那么dispatchTouchEvent方法一定会被调用！
>    **返回值：表示是否消费了当前事件。可能是View本身的onTouchEvent方法消费，也可能是子View的dispatchTouchEvent方法中消费。返回true表示事件被消费，本次的事件终止。返回false表示View以及子View均没有消费事件，将调用父View的onTouchEvent方法**

> - **public boolean onInterceptTouchEvent(MotionEvent ev)**
>    事件拦截，当一个ViewGroup在接到MotionEvent事件序列时候，首先会调用此方法判断是否需要拦截。**特别注意，这是ViewGroup特有的方法，View并没有拦截方法**
>    **返回值：是否拦截事件传递，返回true表示拦截了事件，那么事件将不再向下分发而是调用View本身的onTouchEvent方法。返回false表示不做拦截，事件将向下分发到子View的dispatchTouchEvent方法。**

> - **public boolean onTouchEvent(MotionEvent ev)**
>    真正对MotionEvent进行处理或者说消费的方法。在dispatchTouchEvent进行调用。
>    **返回值：返回true表示事件被消费，本次的事件终止。返回false表示事件没有被消费，将调用父View的onTouchEvent方法**



从最开始的内容中我们了解，activity才是处理事件的用来处理事件的，所以我们首先从activity的dispatchTouchEvent看起：

```java
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

通过上面的代码我们可以发现，事件会给Activity附属的Window进行分发。如果返回true，那么事件被消费。如果返回false表示事件发下去却没有View可以进行处理，则最后return Activity自己的onTouchEvent方法。

跟进getWindow().superDispatchTouchEvent(ev)方法发现是Window类当中的一个抽象方法

查看PhoneWindow类，因为这个getWindow返回的其实就是他的一个对象：

```java
@Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```

这里其实又调用了decorView的方法：(吐槽一句，真是套娃)

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

而我们知道decorView其实是继承自frameLayout，而它又继承自ViewGroup，而在viewGroup类中，我们才找到相应的

dispatchTouchEvent方法，接下来简单分析一下这个方法：

```java
if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
              //FLAG_DISALLOW_INTERCEPT是子View通过
            //requestDisallowInterceptTouchEvent方法进行设置的
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
              // 允许拦截  
              if (!disallowIntercept) {
              //调用onInterceptTouchEvent方法判断是否拦截
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

```

对于每个安全的事件，都会判断是否被拦截，如果子view要求viewGroup不拦截，则不会调用拦截的方法，反之，则viewGroup拦截，不再向下分发事件。当ViewGroup决定拦截事件后，后续事件将默认交给它处理并且不会再调用onInterceptTouchEvent方法来判断是否拦截。子View可以通过设置FLAG_DISALLOW_INTERCEPT标志位来不让ViewGroup拦截除ACTION_DOWN以外的事件。

```java
            if (!canceled && !intercepted) {
                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                  // 触摸到的目标不为空，子view数量不为零，开始分发
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x =
                                isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                        final float y =
                                isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                          //判断1，View可见并且没有播放动画。2，点击事件的坐标落在View的范围内
            //如果上述两个条件有一项不满足则continue继续循环下一个View
                            if (!child.canReceivePointerEvents()
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                           //如果有子View处理即newTouchTarget 不为null则跳出循环。
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                           //dispatchTransformedTouchEvent第三个参数child这里不为null
            							//实际调用的是child的dispatchTouchEvent方法
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                //子View处理了事件，然后就跳出了for循环
                                break;
                            }

            }
```

如果没有被取消也没有被拦截的话，则要把这个事件分发给子view去处理了，循环判断是否可以处理这个事件，如果有子view处理，已经处理完毕，跳出大循环

总结一下：

**ViewGroup会遍历所有子View去寻找能够处理点击事件的子View（可见，没有播放动画，点击事件坐标落在子View内部）最终调用子View的dispatchTouchEvent方法处理事件**，**当子View处理了事件则mFirstTouchTarget 被赋值，并终止子View的遍历。**

## view 对事件的处理

从上面的viewGroup的事件分发可以看出来，其实view最终对时间的处理使用的也是dispatchTouchEvent方法

```java
        //如果窗口没有被遮盖
        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            //当前监听事件
            ListenerInfo li = mListenerInfo;
            //需要特别注意这个判断当中的li.mOnTouchListener.onTouch(this, event)条件
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            //result为false调用自己的onTouchEvent方法处理
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
```



这个就是之前看过那部分代码，通过上面代码我们可以看到View会先判断是否设置了OnTouchListener，**如果设置了OnTouchListener并且onTouch方法返回了true，那么onTouchEvent不会被调用。**
 当没有设置OnTouchListener或者设置了OnTouchListener但是onTouch方法返回false则会调用View自己的onTouchEvent方法。而一些时间的消费和对应的处理都封装在onTouchEvent方法中了，这里就不详细阐述了。
