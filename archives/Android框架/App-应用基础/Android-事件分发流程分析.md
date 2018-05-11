## 1.1 dispatchTouchEvent onInterceptTouchEvent onTouchEvent 关系
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if(onInterceptTouchEvent(ev)) { // 当前view拦截
        consume = onTouchEvent(ev); 
    } else { // 子view分发
        child.dispatchTouchEvent(ev);
    }
    return consume;
}
```
## 1.2 PhoneWindow分发到DecorView流程

![PhoneWindow2DecorView](http://upload-images.jianshu.io/upload_images/74811-a42771772e2cbbd9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 2.1 ViewGroup::dispatchTouchEvent
```
dispatchTouchEvent(MotionEvent ev) {

    // 当ACTION_DOWN时，将要重置FLAG_DISALLOW_INTERCEPT标记位；
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // 发生ACTION_DOWN事件, 则取消并清除之前所有的触摸targets
        cancelAndClearTouchTargets(ev);
        resetTouchState(); // 重置触摸状态
    }

    ...
    // Check for interception.
    final boolean intercepted;
    if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) { // mFirstTouchTarget 处理事件的子元素
        /*
            mFirstTouchTarget != null , 则由子元素处理，
            否则，需要由 当前group处理，
        */
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        // 滑动冲突 内部拦截法：通过改变 FLAG_DISALLOW_INTERCEPT 设置ViewGroup是否拦截事件；
        if (!disallowIntercept) {
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
    ....

    // 当前viewgrop不拦截，且Action未取消
    if (!canceled && !intercepted) {
        ...
        // 遍历子元素分发
        for (int i = childrenCount - 1; i >= 0; i--) {
            final int childIndex = customOrder
                    ? getChildDrawingOrder(childrenCount, i) : i;
            final View child = (preorderedList == null)
                    ? children[childIndex] : preorderedList.get(childIndex);
    
            // 如果当前视图无法获取用户焦点，则跳过本次循环
            if (childWithAccessibilityFocus != null) {...}
    
            //如果view不可见，或者触摸的坐标点不在view的范围内，则跳过本次循环
            if (!canViewReceivePointerEvents(child)
                    || !isTransformedTouchPointInView(x, y, child, null)) {
                ...
            }
    
            // 已经开始接收触摸事件,并退出整个循环
            newTouchTarget = getTouchTarget(child);
            if (newTouchTarget != null) {
                ...
            }
            //重置取消或抬起标志位
            //如果触摸位置在child的区域内，则把事件分发给子View或ViewGroup
            resetCancelNextUpFlag(child);
			// 递归调用子view的dispatchTouchEvent
            if (dispatchTransformedTouchEvent(ev, false, child/*存在子元素*/, idBitsToAssign)) { //2.2
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
                //如果child处理事件，则添加到mFirstTouchTarget单链表
                newTouchTarget = addTouchTarget(child, idBitsToAssign); //2.3
                alreadyDispatchedToNewTouchTarget = true;
                break;
            }
            ...
        }
        ...
        //
        if (mFirstTouchTarget == null) {
            //子view未消耗事件，则mFirstTouchTarget null, 由当前ViewGroup来处理
            handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS); //2.2
        } else {
            ...
            if (dispatchTransformedTouchEvent(ev, cancelChild, target.child, target.pointerIdBits)) {
                handled = true; // 2.2 target.child处理
            }
            ...
        }
        
    }


    if (mFirstTouchTarget == null) { // 无子元素处理事件时；
        handled = dispatchTransformedTouchEvent(ev, canceled, null/*会调用view.dispatchTouchEvent*/,  TouchTarget.ALL_POINTER_IDS); //2.2
    } else {
      ...//如果View消费ACTION_DOWN事件，那么MOVE,UP等事件相继开始执行
    }
}
```

## 2.2 ViewGroup::dispatchTransformedTouchEvent
```
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
...
    if (child == null) {
        handled = super.dispatchTouchEvent(event); // 返回父组件继续dispatchTouchEvent
        // 最终传递给View处理 参考 View 对事件的处理过程
        // 不存在子视图时，ViewGroup调用View.dispatchTouchEvent分发事件，再调用ViewGroup.onTouchEvent来处理事件
    } else {
        handled = child.dispatchTouchEvent(event); // 子节点 (View //2.4 ViewGroup//2.1) 分发事件 
    }
...
}
```
## 2.3 ViewGroup::addTouchTarget

只有处理ActionDown事件，才会进入addTouchTarget, mFirstTouchTarget被赋值；
```
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target; // mFirstTouchTarget 获取 TouchTarget
    return target;
}
```

## 2.4 View::dispatchTouchEvent
```
public boolean dispatchTouchEvent(MotionEvent event) {
...
    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) { //onTouch 优先处理
            result = true;
        }

        if (!result && onTouchEvent(event)) { //onTouchEvent
            result = true; // onTouchEvent
        }
...
}
```

## 2.5 ViewGroup::dispatchTouchEvent 概览

![ViewGroup_dispatchTouchEvent](http://upload-images.jianshu.io/upload_images/74811-5ade424e765b42d5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.1 Android 滑动冲突
#### 3.1.1 外部拦截法

viewgrop 中重写 onInterceptTouchEvent
```
public boolean onInterceptTouchEvent(MotionEvent ev) {
    boolean result = false;
    detector.onTouchEvent(ev); // 事件传递给 GestureDetector
    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:

            downX = ev.getX();
            downY = ev.getY();
            break;
        case MotionEvent.ACTION_MOVE:
            // 新坐标
            float newdownX = ev.getX();
            float newdownY = ev.getY();

            // 计算距离
            int distanceX = (int) Math.abs(newdownX - downX);
            int distanceY = (int) Math.abs(newdownY - downY);

            // distanceX 防止抖动
            if(distanceX > distanceY && distanceX > 10) // 根据dx dy判断水平 垂直方向滑动
                result =  true;
            else
                scrollToPager(currentIndex);
            break;
        case MotionEvent.ACTION_UP:
            break;

    }
    return result;
}
```

#### 3.1.2 内部拦截法
```
public boolean onTouchEvent(MotionEvent event) {
    getParent().requestDisallowInterceptTouchEvent(true); // 设置父组件(listview)不拦截事件
}
```
