---
layout: default
title: 牛群眼中的下拉刷新
lead:  ""
---

## 本文作者：https://github.com/guqun

---

### 简介

PtrFrameLayout框架是一个用来实现pull to refresh的框架。

PtrFrameLayout可以包含任意两个（或一个）控件。一个header（可选）、一个content。header用于实现pull的下拉特效，content是具体要显示的内容。

PtrIndicator是一个辅助类，主要是用于记录PtrFrameLayout控件的一些位置信息、事件状态等。这些信息为PtrFrameLayout的实现提供了决定性作用。

### 下拉效果原理

重写`public boolean dispatchTouchEvent(MotionEvent e)`，通过move事件来实现下拉效果。具体是根据move移动的具体，实时更新header、content位置来实现。

### 实现回弹效果原理
class ScrollChecker  类用于实现回弹。主要是通过scroller完成回弹效果。scroller的作用主要是模拟一次虚拟的回弹，主要为了获取回弹过程中的中间值。通过这些瞬间位移的值来设置header、content两个布局的位置。

public void tryToScrollTo(int to, int duration)这个函数用来开始回弹（调用了mScroller.startScroll(0, 0, 0, distance, duration);），并post一个runnable。runnable根据scroll滚动过程中计算的值来设置header、content的高度。

### `dispatchTouchEvent`函数分析

1. down事件

    1. 更新PtrIndicator类信息
    2. 取消scroller的滚动操作，即在回弹过程中，如果有down事件，则滚动暂停。
    3. 如果当前位置不是起始位置，则事件不发给child节点。
        否则事件传递给child。主要是为了实现页面发生滚动，则child不能接受点击事件。故而截断down事件。

2. move事件

    1. 更新PtrIndicator信息
    2. 判断是否需要进行水平move事件，如果需要，并且当前是水平滑动，则直接将事件dispatch出去，此次move事件再也无法实现垂直滑动。
    3. 判断是否down方向，并且不确实需要刷新操作，若成立，则将事件dispatch出去，因为此时刷新没有意义。
    3. 判断是否down方向或者是否up方向并且up方向确实有空间，则调用movePos实现滑动效果。

3. up、cancel事件

    1. 更新PtrIndicator信息
    2. 判断是否滑动过，如果没有滑动过，则直接事件dispatch出去。否则，调用onRelease函数，触发刷新或者完成操作
    3. 如果down事件之后又过move操作，则说明发生滑动，则发送一个cancel事件给child，为了取消之前传递给child的down事件。
        否则将事件dispatch出去

### `movePos`, `updatePos`函数分析

1. `movepos`

    1. 已经到达顶部，不移动
    2. 移动后超过顶部位置，将最终位置设置为顶部位置
    3. 更新PtrIndicator信息
    4. 调用updatePos(change)函数，更新位置以及状态信息。

2. `updatePos`

    1. 如果发生了down事件并且已经move，则发送cancel事件给child。主要为了低调之前传递给child的down事件
    2. 如果状态为init并离开初始位置或者刚刚完成刷新并位置大于完成刷新的位置，则将状态修改为prepare状态，并回调onUIRefreshPrepare函数
    3. 如果当前move刚好回到起始位置，则调用tryToNotifyReset进行reset。然后，如果当前有down事件，则生成一个down事件dispatch给child。
    4. 判断是否进入刷新状态，并调用tryToPerformRefresh进行刷新
    5. 更新header、content位置
    6. PtrUIHandlerHolder回调onUIPositionChange函数

### `onRelease`函数分析

1. 调用tryToPerformRefresh进行刷新

2. 当状态为`PTR_STATUS_LOADING时`

    1. 如果header需要刷新时不消失，则判断是否超过刷新所需高度并需要loading，则调用tryToScrollTo回滚到刷新所需高度
    2. 否则，调用`tryScrollBackToTopWhileLoading`回滚到顶部

3. 否则

    1. 如果状态是PTR_STATUS_COMPLETE，则调用notifyUIRefreshComplete通知刷新完成。
    2. 否则，调用tryScrollBackToTopAbortRefresh回滚到顶部并取消刷新

### PtrUIHandler分析

PtrUIHandler主要被headerview继承。当PtrFrameLayout状态发生改变是，通过PtrUIHandler进行回调，保证每个状态headerview都可以进行相应的操作。回调有以下函数：

```
public void onUIReset(PtrFrameLayout frame);
public void onUIRefreshPrepare(PtrFrameLayout frame);
public void onUIRefreshBegin(PtrFrameLayout frame);
public void onUIRefreshComplete(PtrFrameLayout frame);
public void onUIPositionChange(PtrFrameLayout frame, boolean isUnderTouch, byte status, PtrIndicator ptrIndicator);
```

### `PtrUIHandlerHolder`分析

PtrUIHandlerHolder用于管理PtrUIHandler派生类。其自身也是PtrUIHandler的派生类。本质上，如果仅仅需要一个回调给headerview并不需要再封装一个PtrUIHandlerHolder，直接在PtrFrameLayout中回调就可以。与源码作者交流，认为并不只有headerview需要回调，为了实现比较复杂的headerview效果可能需要一些额外的回调。所以仅仅靠一个headerview是不够的。故而封装 了一个PtrUIHandlerHolder。
     PtrUIHandlerHolder用list的方式管理PtrUIHandler派生类。PtrUIHandlerHolder是list中的一个item。回调产生时，对所有PtrUIHandlerHolder对象中的PtrUIHandler派生类进行回调。

### `PtrUIHandlerHook`分析

此类也是一个回调，使用时机比较特别，在notifyUIRefreshComplete的使用时调用。可以分析下notifyUIRefreshComplete函数。发现如果hook存在，则先执行hook的回调。后面hook自身会再次调用一次notifyUIRefreshComplete函数。走回正常流程。
