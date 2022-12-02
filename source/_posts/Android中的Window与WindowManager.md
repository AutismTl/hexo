---
title: Android中的Window与WindowManager
date: 2017-05-20
tags: Android
---
---------
## Window
Android中的所有视图，不管是Activity、Dialog还是Toast都是通过Window来呈现的，它们的视图都是附加在Window上面的，Window实际是View的直接管理者。
Window是一个抽象类，它的具体实现是PhoneWindow，WindowManager是外界访问Window的入口，Window的具体实现位于WindowManagerService中，WindowManger与WindowMangerService的交互其实是个IPC过程。
```java
mFloatingButton=new Button(this);
mFloatingButton.setText("button");
mLayoutParams=new WindowManger.LayoutParams(
            LayoutParams.WRAP_CONTENT,LayoutParams.WRAP_CONTENT,0,0,
            PixlFormat.TRANSAPARENT);
mLayoutParams.flags=LayoutParams.FLAG_NOT_TOUCH_MODAL|LayoutParams.FLAG_NOT_FOCUSABLE
                    |LaoutParams.FLAG_SHOW_WHEN_FOCUSABLE;
mLayoutParams.x=100;
mLayoutParams.y=200;
mWindowManger.addView(mFloatingButton,mLayoutParams);
```
上述代码可以将一个Button添加到屏幕坐标为(100,200)的位置上。
<!--more-->

**Flags表示Window的属性，以下为几个常见的属性：**
- FLAG_NOT_TOUCH_MODAL 表示不需要获取焦点，也不需要接收各种输入，最终事件直接传递给下层具有焦点的Window。
- FLAG_NOT_FOCUSABLE：在此Window外的区域单击事件传递到底层Window中。区域内的单击事件则自己处理，这个一般都要设置，否则其他的Window将无法收到单击事件。
- FLAG_SHOW_WHEN_LOCKED：开启可以让Window显示在锁屏界面上。

**Type表示Window的类型，类型分为**：应用Window（Activity）、子Window（不能单独存在，必须依附在父Window中，Dialog等）、系统Window（需声明权限才能创建，Toast等）。

Window中存在分层，层级大会覆盖在层级小的上面，应用Window的层级一般为1-99，子Window的层级一般为1000-1999，系统Window为2000-2999。

## Window内部机制
Window是一个抽象的概念，一个Window对应一个View和一个ViewRootImpl,Window和View通过ViewRootImpl来建立联系，所以Window并不实际存在，它是以View存在的。
Window的操作需要通过WindowManager，WindowManager常用的只有三个方法，即添加/更新/删除View。
WindowManager是一个接口，它的实现类是WindowManagerImpl。WindowManagerImpl也并没有直接实现三大操作，而是委托给WindowManagerGlobal。
```java
//WindowManger继承了这个类
public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}

```

### 添加Window
**addView的实现分为以下几步：**
1.检查参数是否合法。
```java
if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }
 
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent and we're running on L or above (or in the
            // system context), assume we want hardware acceleration.
            final Context context = view.getContext();
            if (context != null
                    && context.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.LOLLIPOP) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
}
```

2.创建ViewRootImpl并将View添加到列表中。
WindowMangerGlobal内部存在如下几个列表：
- ArrayList mViews; 所有Window对应的View
- ArrayList mRoots; 所有Window对应iewRootImpl
- ArrayList mParams; 所有Window对应LayoutParams
- ArraySet mDyingViews; 调用removeView但是删除还未完成的Window对象

```java
root = new ViewRootImpl(view.getContext(), display);
view.setLayoutParams(wparams);
mViews.add(view);
mRoots.add(root);
mParams.add(wparams);
```

3.通过ViewRootImpl来更新界面并完成window的添加过程 。
```java
root.setView(view, wparams, panelParentView);
```
setView内部会通过requestLayout异步刷新，最后通过IPC操作调用WindowManagerService实现Window添加。

### 删除Window
同Window的添加一样，都是最终通过WindowManagerGlobal实现，下面是其中的removeView：
```java
public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
 
        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }
 
            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```
这里首先调用findViewLocked来查找删除view的索引，这个过程就是建立数组遍历。然后再调用removeViewLocked来做进一步的删除。
```java
private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();
 
        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```
真正删除操作是viewRootImpl来完成的。windowManager提供了两种删除接口，removeViewImmediate，removeView。它们分别表示异步删除和同步删除。具体的删除操作由ViewRootImpl的die来完成。
```java
boolean die(boolean immediate) {
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }
 
        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(TAG, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }
```
由上可知如果是removeViewImmediate，立即调用doDie，如果是removeView，用handler发送消息，ViewRootImpl中的Handler会处理消息并调用doDie。

**最终删除主要做四件事：**
1. 垃圾回收相关工作，比如清数据，回调等。
2. 通过Session的remove方法删除Window,最终通过IPC调用WindowManagerService的removeWindow。
3. 调用dispathDetachedFromWindow，在内部会调用onDetachedFromWindow()和onDetachedFromWindowInternal()。当view移除时会调用onDetachedFromWindow，它用于作一些资源回收。
4. 通过doRemoveView刷新数据，删除相关数据，如在mRoot，mDyingViews等中删除对象。

### 更新Window
看下WindowManagerGlobal中的updateViewLayout。
```java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
       if (view == null) {
           throw new IllegalArgumentException("view must not be null");
       }
       if (!(params instanceof WindowManager.LayoutParams)) {
           throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
       }
 
       final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
 
       view.setLayoutParams(wparams);
 
       synchronized (mLock) {
           int index = findViewLocked(view, true);
           ViewRootImpl root = mRoots.get(index);
           mParams.remove(index);
           mParams.add(index, wparams);
           root.setLayoutParams(wparams, false);
       }
   }
```
通过viewRootImpl的setLayoutParams更新ViewRootImpl的LayoutParams,接着scheduleTraversals对View重新布局，包括测量，布局，重绘，此外它还会通过WindowSession来更新Window。这个过程通过IPC由WindowManagerService实现。









