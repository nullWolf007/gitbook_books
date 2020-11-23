[TOC]

## View的事件分发机制

#### 参考链接

* [**Android开发艺术探索**](https://book.douban.com/subject/26599538/)
* [Android事件分发机制 详解攻略，您值得拥有](https://blog.csdn.net/carson_ho/article/details/54136311)

#### 说明

* 基于Android8.0

### 一、基础知识

#### 1.1 事件分发中事件指的是？

* **点击事件(Touch事件)**
  * 当用户触摸屏幕时（`View` 或 `ViewGroup`派生的控件），将产生点击事件（`Touch`事件）

#### 1.2 同一事件序列

* 同一个事件序列指的是从手指接触到屏幕起，到手指离开屏幕结束，这个过程中所产生的一系列事件。从`down`开始，多个`move`，`up`结束。

#### 1.3 事件分发的本质

* **将点击事件（MotionEvent）传递到某个具体的`View` & 处理的整个过程**

#### 1.4 事件在哪些对象之间进行传递？

* **Activity、ViewGroup、View**
* `Android`的`UI`界面由`Activity`、`ViewGroup`、`View` 及其派生类组成

![](..\..\images\自定义UI\自定义View\Activity,View,ViewGroup.png)

#### 1.5 事件分发核心方法

**public boolean dispatchTouchEvent(MotionEvent ev)**

* 用来进行事件的分发。如果事件能够传递给当前View，那么此方法就一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法影响，表示是否消耗当前事件

**public boolean onInterceptTouchEvent(MotionEvent event)**

* 在dispatchTouchEvent方法内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列中，此方法不会被再次调用，返回结果表示是否拦截当前事件
* 只存在ViewGroup中，普通的View无该方法

**public boolean onTouchEvent(MotionEvent event)**

* 在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件

### 二、源码解析

* `Android`事件分发流程 = **Activity -> ViewGroup -> View**
* 要想充分理解Android分发机制，本质上是要理解：
  * `Activity`对事件的分发机制
  * `ViewGroup`对事件的分发机制
  * `View`对事件的分发机制

#### 2.1 Activity的事件分发机制

##### 2.1.1 dispatchTouchEvent

* Activity.java#dispatchTouchEvent

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
	if (ev.getAction() == MotionEvent.ACTION_DOWN) {
    	onUserInteraction();//1
	}
    if (getWindow().superDispatchTouchEvent(ev)) {//2
        //如果Activity将触摸事件分发给它所包含的视图的时候，如果有视图拦截或消费了该事件，就不会轮到Activity来处理该事件了；即，不会执行Activity的onTouchEvent()了。
    	return true;
	}
    return onTouchEvent(ev);//3
}
```

* 在上面代码中，分别调用了三个方法，下面将一一讲解

##### 2.1.2 onUserInteraction

* Activity.java#onUserInteraction

```java
public void onUserInteraction() {
}
```

* 该方法为一个空方法，当Activity在栈顶的时候，触屏点击，按Home，back，menu都会触发此方法
* 可以重写这个方法，实现用户操作时的交互效果

##### 2.1.3 getWindow().superDispatchTouchEvent

* getWindow获取的是Window对象，然而Window类是抽象类，所以要找其实现类，而PhoneWindow是其唯一的实现类，所以调用的就是PhoneWindow的superDispatchTouchEvent
* PhoneWindow.java#superDispatchTouchEvent

```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
	return mDecor.superDispatchTouchEvent(event);
}
```

* mDecor是DecorView(顶层View)对象，调用了其superDispatchTouchEvent方法

##### 2.1.4 superDispatchTouchEvent

* DecorView.java#superDispatchTouchEvent

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
	return super.dispatchTouchEvent(event);
}
```

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
```

* DecorView继承自FrameLayout，是对FrameLayout的扩展，是所有界面的根View即顶级View

![界面关系](..\..\images\自定义UI\自定义View\界面关系.png)

* 调用了super的dispatchTouchEvent，那么在这里指的就是FrameLayout，FrameLayout又继承自ViewGroup，又没有重写此方法，故而值得是ViewGroup的dispatchTouchEvent

```java
public class FrameLayout extends ViewGroup {
```

##### 2.1.5 dispatchTouchEvent

* ViewGroup.java#ViewGroup

* 这部分在接下来的ViewGroup对事件分发机制会详细讲解

##### 2.1.6 onTouchEvent

* Activity.java#onTouchEvent

```java
public boolean onTouchEvent(MotionEvent event) {
	if (mWindow.shouldCloseOnTouch(this, event)) {
    	finish();
        return true;
	}

    return false;
}
```

* 同理可得，调用的是PhoneWindow的shouldCloseOnTouch方法，然而shouldCloseOnTouch不是抽象方法，有没有重写，所以指的实际上是Window的shouldCloseOnTouch

##### 2.1.7 shouldCloseOnTouch

```java
public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
	if (mCloseOnTouchOutside && event.getAction() == MotionEvent.ACTION_DOWN
    		&& isOutOfBounds(context, event) && peekDecorView() != null) {
		return true;
	}
    return false;
}
```

* android:windowCloseOnTouchOutside属性为true，则mCloseOnTouchOutside为true
*  isOutOfBounds(context, event)是判断该event的坐标是否在context(对于本文来说就是当前的Activity)之外。是的话，返回true
* peekDecorView()则是返回PhoneWindow的mDecor。
* 也就是说，如果设置了android:windowCloseOnTouchOutside属性为true，并且当前事件是ACTION_DOWN，而且点击发生在Activity之外，同时Activity还包含视图的话，则返回true；则Activity的onTouchEvent方法会调用finish方法，表示该点击事件会导致Activity的结束。
* 默认返回false

##### 2.1.8 总结

![Activity事件分发机制流程图](..\..\images\自定义UI\自定义View\Activity事件分发机制流程图.png)



#### 2.2 ViewGroup事件分发机制

##### 2.2.1 dispatchTouchEvent

* ViewGroup.java#dispatchTouchEvent
* **mFirstTouchTarget**：此对象是一个单链表结构，存储这一系列的事件（ACTION_DOWN、ACTION_MOVE...、ACTION_UP）发生时所涉及到的子View，因此触摸事件 ACTION_DOWN 发生后如果这个对象还是为null，那么就表示 ViewGroup 没有将事件传递到子View。
* **mGroupFlags**：mGroupFlags 可以理解为很多个标志位的组合。mGroupFlags & FLAG_DISALLOW_INTERCEPT != 0 表示这个标志位组合内有「不允许拦截事件」这个标志位

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	if (mInputEventConsistencyVerifier != null) {
    	mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
	}

    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
    	ev.setTargetAccessibilityFocus(false);
	}

    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
    	final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        //当ACTION_DOWN事件来临的时候
        if (actionMasked == MotionEvent.ACTION_DOWN) {
        	//表示这是一个新的事件序列 需要重置手势、触摸状态等 开始一个新的事件序列
            cancelAndClearTouchTargets(ev);
            resetTouchState();
		}

		//此标志位代表自己是否要拦截这个事件
		final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
        		|| mFirstTouchTarget != null) {
            //判断是否允许进行拦截
			final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                //调用onInterceptTouchEvent方法
                //onInterceptTouchEvent基本返回false 代表不拦截
                //当然自定义的ViewGroup可以设置拦截 返回true
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

		// Check for cancelation.
		final boolean canceled = resetCancelNextUpFlag(this)
				|| actionMasked == MotionEvent.ACTION_CANCEL;

		// Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        if (!canceled && !intercepted) {

			// If the event is targeting accessiiblity focus we give it to the
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
                if (newTouchTarget == null && childrenCount != 0) {
                	final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                    	&& isChildrenDrawingOrderEnabled();
                    //遍历ViewGroup的所有子View
					final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                    	final int childIndex = getAndVerifyPreorderedIndex(
                        		childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                        		preorderedList, children, childIndex);

                        // If there is a view that has accessibility focus we want it
                        // to get the event first and if not handled we will perform a
                        // normal dispatch. We may do a double iteration but this is
                        // safer given the timeframe.
                        if (childWithAccessibilityFocus != null) {
                        	if (childWithAccessibilityFocus != child) {
                            	continue;
							}
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
						}
						
                        //通过canViewReceivePointerEvents和isTransformedTouchPointInView
                        //来判断点击时坐标落在哪一个子View上
                        if (!canViewReceivePointerEvents(child)
                        		|| !isTransformedTouchPointInView(x, y, child, null)) {
							ev.setTargetAccessibilityFocus(false);
                            continue;
						}

                        //
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                        	// Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
						}

                        resetCancelNextUpFlag(child);
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                        	//记录找到的子View，以便之后的事件序列可以直接使用目标View
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
                            break;
						}

                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
					}
                    if (preorderedList != null) preorderedList.clear();
				}

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                	// Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                    	newTouchTarget = newTouchTarget.next;
					}
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
				}
			}
		}

        //触摸事件ACTION_DOWN发生后如果这个对象还是为null
        //那么就表示 ViewGroup 没有将事件传递到子View
        //所以要接着执行其他动作(滑动等)的处理
        if (mFirstTouchTarget == null) {
        	// No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
            		TouchTarget.ALL_POINTER_IDS);
		} else {
        	// Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
            	final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                	handled = true;
				} else {
                	final boolean cancelChild = resetCancelNextUpFlag(target.child)
                    		|| intercepted;
					if (dispatchTransformedTouchEvent(ev, cancelChild,
                    		target.child, target.pointerIdBits)) {
						handled = true;
					}
                    if (cancelChild) {
                    	if (predecessor == null) {
                        	mFirstTouchTarget = next;
						} else {
                        	predecessor.next = next;
						}
                        target.recycle();
                        target = next;
                        continue;
					}
				}
                predecessor = target;
                target = next;
			}
		}

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
        		|| actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
			resetTouchState();
		} else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
        	final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
		}
	}

    if (!handled && mInputEventConsistencyVerifier != null) {
    	mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
	}
    return handled;
}
```

* 整体总结：首先判断拦截与否，不拦截的话遍历ViewGroup的所有子View，找到点击事件落在的子View，调用子View的dispatchTransformedTouchEvent方法,，进而调用View的dispatchTouchEvent

##### 2.2.2 dispatchTransformedTouchEvent

* ViewGroup.java#dispatchTransformedTouchEvent

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
		View child, int desiredPointerIdBits) {
	final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
    	event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
        	handled = super.dispatchTouchEvent(event);
		} else {
        	handled = child.dispatchTouchEvent(event);
		}
        event.setAction(oldAction);
        return handled;
	}

    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
    	return false;
	}

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
    	if (child == null || child.hasIdentityMatrix()) {
        	if (child == null) {
                //View来处理事件
            	handled = super.dispatchTouchEvent(event);
			} else {
                //将触摸点的坐标转换成子视图的坐标
            	final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
               
                event.offsetLocation(offsetX, offsetY);
				//子View处理事件
                handled = child.dispatchTouchEvent(event);
				//又将触摸坐标还原，之前转换的坐标只适合那个子视图
                event.offsetLocation(-offsetX, -offsetY);
			}
            return handled;
		}
        transformedEvent = MotionEvent.obtain(event);
	} else {
    	transformedEvent = event.split(newPointerIdBits);
	}

    // Perform any necessary transformations and dispatch.
    if (child == null) {
    	handled = super.dispatchTouchEvent(transformedEvent);
	} else {
    	final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
        	transformedEvent.transform(child.getInverseMatrix());
		}

        handled = child.dispatchTouchEvent(transformedEvent);
	}

    // Done.
    transformedEvent.recycle();
    return handled;
}
```

* 会调用View的dispatchTouchEvent或者子View的dispatchTouchEvent

##### 2.2.3 dispatchTouchEvent

* View.java#dispatchTouchEvent 或者 子View的dispatchTouchEvent 

![ViewGroup事件分发机制流程图](..\..\images\自定义UI\自定义View\ViewGroup事件分发机制流程图.png)

#### 2.3 View事件分发机制

##### 2.3.1 dispatchTouchEvent 

* View.java#dispatchTouchEvent 

```java
public boolean dispatchTouchEvent(MotionEvent event) {
	// If the event should be handled by accessibility focus first.
    if (event.isTargetAccessibilityFocus()) {
    	// We don't have focus or no virtual descendant has it, do not handle the event.
        if (!isAccessibilityFocusedViewOrHost()) {
        	return false;
		}
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
	}

    boolean result = false;

    if (mInputEventConsistencyVerifier != null) {
    	mInputEventConsistencyVerifier.onTouchEvent(event, 0);
	}

    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
    	// Defensive cleanup for new gesture
        stopNestedScroll();
	}

    if (onFilterTouchEventForSecurity(event)) {
    	if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
        	result = true;
		}
        //首先判断 OnTouchListener 是否为空
        //再判断这个 View 是否可以用（即setEnable属性，默认都是true）
        //然后调用 OnTouchListener.onTouch 方法执行我们自定义的触摸操作
        //如果此方法返回 true, 则代表事件被消费，接下来不需要执行 onTouchEvent
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
        		&& (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
			result = true;
		}

        //我们使其返回 false, 那么可以继续传递给 onTouchEvent 去消费
        if (!result && onTouchEvent(event)) {
        	result = true;
		}
	}

    if (!result && mInputEventConsistencyVerifier != null) {
    	mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
	}

    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
    		actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
		stopNestedScroll();
	}

    return result;
}
```

* 关键部分已经做了注释解释，如果事件不被消费，就会调用onTouchEvent

##### 2.3.2 onTouchEvent

* View.java#onTouchEvent

```java
public boolean onTouchEvent(MotionEvent event) {
	final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

	final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
    		|| (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

	if ((viewFlags & ENABLED_MASK) == DISABLED) {
    	if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        	setPressed(false);
		}
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return clickable;
	}
    if (mTouchDelegate != null) {
    	if (mTouchDelegate.onTouchEvent(event)) {
        	return true;
		}
	}

    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
    	switch (action) {
        	case MotionEvent.ACTION_UP:
            	mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                	handleTooltipUp();
				}
                if (!clickable) {
                	removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
				}
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                	// take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                    	focusTaken = requestFocus();
					}

                    if (prepressed) {
                    	// The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true, x, y);
					}

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                    	// This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                        	// Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                            	mPerformClick = new PerformClick();
							}
                            if (!post(mPerformClick)) {
                            	performClick();
							}
						}
					}

                	if (mUnsetPressedState == null) {
                    	mUnsetPressedState = new UnsetPressedState();
					}

                    if (prepressed) {
                    	postDelayed(mUnsetPressedState,
                        		ViewConfiguration.getPressedStateDuration());
					} else if (!post(mUnsetPressedState)) {
                    	// If the post failed, unpress right now
                        mUnsetPressedState.run();
					}

                    removeTapCallback();
				}
                mIgnoreNextUpEvent = false;
                break;
                .....
		}
		return true;
	}
    return false;
}
```

* 其中最关键的部分就是我们最常用的 click 事件，一些长按等事件的逻辑这里就不再分析。通过属性判断 View 是否可点击，并且在手指抬起时即 ACTION_UP 执行 performClick 方法，其内部就是判断用户是否设置了 OnClickListener 监听器，如果有则调用 onClick 方法。

##### 2.3.3 performClick 

* PerformClick.java#performClick 

```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    return result;
}
```

* 有就会执行 OnClickListener.onClick 方法，这个就是常见的new OnClickListener中自己代码逻辑的onClick 方法

##### 2.3.4 总结

* 如果用户设置了 OnTouchListener, 那么就会调用 onTouch 方法，并且如果 onTouch 方法返回true, 那么就不会执行 onTouchEvent 了，也就不会执行 onClick 了；
* 如果来到 onTouchEvent 方法，那么就会执行performClick，进而有机会去执行 OnClickListener.onClick 方法，除非你执行了长按之类的操作；
* 最后回调到 onClick；

![View事件分发机制流程图](..\..\images\自定义UI\自定义View\View事件分发机制流程图.png)

### 三、总结

#### 3.1  伪代码解析

```java
/**
  * 点击事件产生后
*/ 
// 步骤1：调用dispatchTouchEvent（）
public boolean dispatchTouchEvent(MotionEvent ev) {

	boolean consume = false; //代表 是否会消费事件
    
    // 步骤2：判断是否拦截事件
    if (onInterceptTouchEvent(ev)) {
      	// a. 若拦截，则将该事件交给当前View进行处理
      	// 即调用onTouchEvent (）方法去处理点击事件
		consume = onTouchEvent (ev) ;
    } else {
      // b. 若不拦截，则将该事件传递到下层
      // 即 下层元素的dispatchTouchEvent（）就会被调用，重复上述过程
      // 直到点击事件被最终处理为止
      consume = child.dispatchTouchEvent (ev) ;
    }
    // 步骤3：最终返回通知 该事件是否被消费（接收 & 处理）
    return consume;
}
```

* 对于根`ViewGroup`而言，点击事件产生之后，它的`dispatchTouchEvent`就会被调用。此时判断根`ViewGroup`的`onInterceptTouchEvent`方法的返回值，如果返回值为`true`，拦截当前事件，这个事件就会交给根`ViewGroup`，即它的`onTouchEvent`方法就会被调用；如果返回值为`false`，则当前时间传递给它的子元素，此时子元素的`dispatchTouchEvent`方法会被调用，反复下去直到被处理。

* 当一个点击事件产生后，它的传递过程遵循如下顺序：`Activity----Window----View`。如果该View下的所有元素都不处理这个事件（`onTouchEvent为false`），此时会传递给上层的`View`，如果一直都不处理这个事件，最后会交给`Activity`，此时`Activity`的`onTouchEvent`方法会被调用。

#### 3.2 结论：

* 除去特殊手段处理，一个事件序列只能被一个`View`拦截并消耗。

* 某个`View`一旦决定拦截，那么这一个事件序列都只能由它处理，并且它的`onInterceptTouchEvent`不会再被调(因为再被调用也失去了意义)。

* 某个`View`一旦开始处理事件，如果他不消耗`ACTION_DOWN`事件（`onTouchEvent`返回了`false`），那么同一事件序列都不会给他处理。并且事件会重新交给它的父元素处理，即父元素的onTouchEvent会被调用。

* 如果`View`不消耗除`ACTION_DOWN`以外的其他事件，那么这个点击事件会消失，此时父元素的`onTouchEvent`并不会被调用，并且当前`View`可以持续收到后续的事件，最终这些消失的点击事件会传递给`Activity`处理。

* `ViewGroup`默认不拦截任何事件。

* `View`没有`onInterceptTouchEvent`方法，一旦有点击事件传递给他，那么它的`onTouchEvent`方法就会被调用。

* `View`的`onTouchEvent`默认都会消耗事件（返回`true`），除非它是不可点击的(`clickable`和`longClickable`同时为`false`)。`View`的`longClickable`属性默认为`false`，`clickable`属性要分情况，`Button`为`true`，`TextView`为`false`。

* `View`的`enable`属性不影响`onTouchEvent`的默认返回值。

* `onClick`会发生的前提是当前`View`是可点击的，并且它收到了`down`和`up`的事件。

* 事件传递过程是由外向内的，即事件总是传递给父元素，然后再有父元素分发给子`View`，通过`requestDisallowInterceptTouchEven`t方法可以在子元素中干预父元素的事件分发过程，除了`ACTION_DOWN`事件。


### 四、View的滑动冲突

#### 4.1 处理规则

* 根据情况，让特定的`View`拦截点击事件

#### 4.2 滑动冲突的解决方式

##### 4.2.1 外部拦截法---推荐

```java
//父类进行点击事件的拦截---重写父容器的onInterceptTouchEvent方法  
//通过x和y的滑动距离来判断是上下滑动还是左右滑动
//如果父容器需要当前点击事件，返回true，就不会往下传递
//DOWN和UP都返回false 
@Override	
public boolean onInterceptTouchEvent(MotionEvent ev) {
	boolean intercepted = false;
    int x = (int) ev.getX();
    int y = (int) ev.getY();
    switch (ev.getAction()) {
    	case MotionEvent.ACTION_DOWN:
        	intercepted = false;
            break;
		case MotionEvent.ACTION_MOVE:
        	if (父容器需要当前点击事件) {
            	intercepted = true;
			} else {
            	intercepted = false;
			}
            break;
		case MotionEvent.ACTION_UP:
        	intercepted = false;
        	break;
		default:
        	break;
	}
    return intercepted;
}
```

##### 4.2.2 内部拦截法

```java
//子元素进行点击事件的处理，不消耗就交给父容器进行处理----重写子元素的dispatchTouchEvent方法
//父容器不拦截ACTION_DOWN事件，子元素通过requestDisallowInterceptTouchEvent(true)方法让父元素不再拦截其他的点击事件
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
	int x = (int) ev.getX();
    int y = (int) ev.getY();

    switch (ev.getAction()) {
    	case MotionEvent.ACTION_DOWN:
        	//一旦设置  父容器无法拦截点击事件（除了ACTION_DOWN）
            getParent().requestDisallowInterceptTouchEvent(true);
            break;
		case MotionEvent.ACTION_MOVE:
        	if (父容器需要当前点击事件) {
            	getParent().requestDisallowInterceptTouchEvent(false);
			}
            break;
		case MotionEvent.ACTION_UP:
        	break;
		default:
        	break;
	}
    return super.dispatchTouchEvent(ev);
}
```

```java
    //父容器不拦截ACTION_DOWN事件，重写父容器的onInterceptTouchEvent
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        int action = ev.getAction();
        if (action == MotionEvent.ACTION_DOWN) {
            return false;
        } else {
            return true;
        }
    }
```