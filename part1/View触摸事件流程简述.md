
#触摸事件分发流程#
---
首先看一个方法调用的截图

![](https://github.com/getletCodes/StudyNotes/blob/master/part1/picture/touch_event.png)

ViewRootImpl类中中使用内部类ViewPostImeInputStage的processPointerEvent把事件传递给DecorView

```

    private int processPointerEvent(QueuedInputEvent q) {
        final MotionEvent event = (MotionEvent)q.mEvent;

        mAttachInfo.mUnbufferedDispatchRequested = false;
        boolean handled = mView.dispatchPointerEvent(event);//传递事件给DecorView
        //...
        return handled ? FINISH_HANDLED : FORWARD;
    }

```

DecorView的dispatchPointerEvent位于View中

```

    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);//调用DecorView的dispatchTouchEvent方法
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }

```

DecorView$dispatchTouchEvent

```

        @Override
        public boolean dispatchTouchEvent(MotionEvent ev) {
            final Callback cb = getCallback();//Activity中实现了该回调
            return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                    : super.dispatchTouchEvent(ev);
        }

```

Activity$Callback$dispatchTouchEvent

```

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

PhoneWindow$superDispatchTouchEvent

```

    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }



```

DecorView$superDispatchTouchEvent

```

    public boolean superDispatchTouchEvent(MotionEvent event) {
            return super.dispatchTouchEvent(event);//调用ViewGroup的dispatchEvent方法
        }

```

最后ViewGroup的dispatchTouchEvent中首先判断是否需要拦截(onIntercept)，如果不需要拦截则调用dispatchTransformedTouchEvent方法，在该方法中调用child.dispatchTouchEvent(event);向下一级分发事件


接下来就是各个dispatchEvent的调用，拦截处理，如果都不处理，则Activity会处理事件,一个触摸事件就结束了调用过程。

