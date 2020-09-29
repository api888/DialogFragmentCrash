# 解决showDialogFragment发生的crash

DialogFragment这个控件作为一个Android开发者来说，应该都是再熟悉不过的了。
不过在showDialogFragment发的时候经常会碰到下面这个crash：

```
      java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
        at androidx.fragment.app.FragmentManagerImpl.checkStateLoss(FragmentManager.java:2080)
        at androidx.fragment.app.FragmentManagerImpl.enqueueAction(FragmentManager.java:2106)
        at androidx.fragment.app.BackStackRecord.commitInternal(BackStackRecord.java:683)
        at androidx.fragment.app.BackStackRecord.commit(BackStackRecord.java:637)
        at androidx.fragment.app.DialogFragment.show(DialogFragment.java:144)
        at com.netease.demo.MainActivity$1.run(MainActivity.java:23)
        at android.os.Handler.handleCallback(Handler.java:873)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:193)
        at android.app.ActivityThread.main(ActivityThread.java:6669)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```

对于上面这个问题，首先肯定是得去查看源码为什么会出现这个错误
一般来说使用DialogFragment基本上就是下面这种写法:

```
                DialogFragment dialogFragment = new DialogFragment();
                dialogFragment.show(getSupportFragmentManager(),"");
```

1. 获取宿主即当前Activity所持有的FragmentManager,是一个FragmentManagerImpl对象
2. 调用dialogFragment的show方法
3. 通过FragmentManagerImpl调用beginTransaction方法new了一个BackStackRecord对象
4. 通过BackStackRecord将Fragment添加到布局中的控件中
5. 通过事务提交这次操作。


crash就出现在了第五步中,我们来看下第四步的逻辑:

```
    @Override
    public int commit() {
        return commitInternal(false);
    }
   int commitInternal(boolean allowStateLoss) {
        if (mCommitted) throw new IllegalStateException("commit already called");
        if (FragmentManagerImpl.DEBUG) {
            Log.v(TAG, "Commit: " + this);
            LogWriter logw = new LogWriter(TAG);
            PrintWriter pw = new PrintWriter(logw);
            dump("  ", null, pw, null);
            pw.close();
        }
        mCommitted = true;
        if (mAddToBackStack) {
            mIndex = mManager.allocBackStackIndex(this);
        } else {
            mIndex = -1;
        }
        mManager.enqueueAction(this, allowStateLoss);
        return mIndex;
    }
```

逻辑非常简单，先判断当前Fragment添加的事务是否已经被提交过一次(防止复提交出现页面错乱的check)。然后直接调用FragmentManager的enqueueAction方法:

```
    public void enqueueAction(OpGenerator action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                if (allowStateLoss) {
                    // This FragmentManager isn't attached, so drop the entire transaction.
                    return;
                }
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<>();
            }
            mPendingActions.add(action);
            scheduleCommit();
        }
    }
    private void checkStateLoss() {
        if (isStateSaved()) {
            throw new IllegalStateException(
                    "Can not perform this action after onSaveInstanceState");
        }
        if (mNoTransactionsBecause != null) {
            throw new IllegalStateException(
                    "Can not perform this action inside of " + mNoTransactionsBecause);
        }
    }
```

关键就处在了 checkStateLoss();这里，isStateSaved是true的时候就会抛出这个异常，那么什么时候抛出呢？

```
    @Override
    public boolean isStateSaved() {
        // See saveAllState() for the explanation of this.  We do this for
        // all platform versions, to keep our behavior more consistent between
        // them.
        return mStateSaved || mStopped;
    }
```

是在当前宿主页面onStop的时候会是true(回到home，或者是被其他窗口遮挡)，这个时候如果再去执行commit的话那就直接crash了。


这个时候就需要想办法解决这个问题了，所以以下列出四种方法:

1. 在每个DialogFragment的show方法调用之前，都判断当前宿主Activity是否已经被其他窗口覆盖或者进入home了。可以写一个boolean变量，在onSaveInstance执行的时候就置为true，然后根据这个变量判断。
不过这个方案浸入性太强，需要修改的地方太多了。这不是个好方案

2. 既然commit直接crash，那么怎么做才能阻止crash呢。我们再看下enqueueAction方法:

```
        if (!allowStateLoss) {
            checkStateLoss();
        }
```

只有在allowStateLoss为false的时候才需要去做checkStateLoss操作，所以我们只要设置alloStateLoss为true即可。不过我们没法修改DialogFragment的show方法，所以我们可以在BaseDialogFragment里面重写show方法

```
    public void show(FragmentManager manager, String tag) {
        mDismissed = false;
        mShownByMe = true;
        FragmentTransaction ft = manager.beginTransaction();
        ft.add(this, tag);
        //将ft.commit改成ft.commitAllowingStateLoss()。
        ft.commit();
    }
```

将ft.commit改成ft.commitAllowingStateLoss()。这样就解决了浸入性强的问题，但是这里又有一个问题了，因为mDismissed和mShownByMe是默认访问权限的变量，我们自己的APP无法直接获取这个参数,所以也就没法直接修改这个参数，因此这两个参数未被修改可能会引发其他的问题。所以这个方案也不行。


3. 与方案二类似，既然我们无法获取这两个参数，那我们干脆直接将DialogFragment的源码拷贝出来，自己写一个DialogFragment，这样就可以修改show方法了。这种方案虽然没有什么弊端，但是个人还是不太赞成这个方案的。

4. 接下来就是我比较推荐的方案了，我们重新捋下逻辑:
为什么DialogFragment会crash? 是因为我们在checkStateLoss();发现当前宿主Activity已经不在屏幕上显示了，所以才会抛出这个异常，那么我们能否在show的时候判断呢？自然是可以的，我们的第一种方案就是，不过第一种方案是在show外面判断，接下来我要介绍的方案是重写show方法:

```
    /**
     * 检查当前页面是否处于活跃状态
     * @param manager 当前Activity对应的fragment管理者
     * @return
     */
    protected boolean checkActivityIsActive(FragmentManager manager) {
        if(manager.isStateSaved()){
            return false;
        }
        return true;
    }

    @Override
    public void show(FragmentManager manager, String tag) {
        if(!checkActivityIsActive(manager)){
            return;
        }
        super.show(manager, tag);
    }
```


既然checkStateLoss是在FragmentManager里面判断的，所以我们也可以直接通过FragmentManager的isStateSaved进行判断。不过如果我们使用的是show(FragmentTransaction transaction, String tag)方法，那么就稍微麻烦点，下面直接给出

```
    @Override
    public int show(FragmentTransaction transaction, String tag) {
        try {
            Field field = transaction.getClass().getDeclaredField("mManager");
            field.setAccessible(true);
            if (!checkActivityIsActive((FragmentManager) field.get(transaction))) return STATUS_COMMIT_FAILED;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return super.show(transaction, tag);
    }
```

通过事务反射获取对应的FragmentManager，然后在调用之前的判断方法来判断是否需要show出来。这样浸入性不高，另外对源码的逻辑改动也不多。