title: 关于AsyncTaskLoader
date: 2015-07-01 11:14:47
tags:
- Android
- Concurrency
---

Concurrency基础
===============

Executor
--------
`Executor`是一个用来执行`Runable`实例的对象，接口定义如下

```
/**
 * An object that executes submitted {@link Runnable} tasks. This
 * interface provides a way of decoupling task submission from the
 * mechanics of how each task will be run, including details of thread
 * use, scheduling, etc.  An {@code Executor} is normally used
 * instead of explicitly creating threads.
 */
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```
<!--more-->

ExecutorService
----------------
`ExecutorService`继承自`Executor`，也是用来执行`Runable`对象的，但是额外提供了方法能够返回`Future`对象用来跟踪异步任务的状态


ThreadPoolExecutor、ScheduledThreadPoolExecutor
-----------------------------------------------
都是`ExecutorService`的实现类，`ScheduledThreadPoolExecutor`继承自`ThreadPoolExecutor`，提供了重复执行异步任务的功能


Executors
---------
`ExecutorService`、`Callable`等的工厂类工具了，提供了一些方法返回各种`ExecutorService`的实例类，以及可以把`Runnable`转换成`Callable`实例


Runnable
--------
`Runnable`代表可执行线程的单元。

```
public class SimpleTask implements Runnable{

	@Override
	public void run() {
		System.out.println("SimpleTask, Runnable: Executing Logic");
	}

}

public class Client {

/**
* @param args
*/
public static void main(String[] args) {
	// Step1 : Create a Runnable
	Runnable simpleTask = new SimpleTask();
	// Step 2: Configure Executor
	// Uses FixedThreadPool executor
	ExecutorService executor = Executors.newFixedThreadPool(2);
	executor.submit(simpleTask);
	executor.shutdown();
}

}
```

Callable
--------
和`Runnable`类似，但是它可以返回值、抛出异常

```
public class CallableTask implements Callable<String>{

	@Override
		public String call() throws Exception {
		String s="Callable Task Run at "+System.currentTimeMillis();
		return s;
	}

}

public class CallableClient {
 
/**
* @param args
*/
public static void main(String[] args) {
 
	// Step1 : Create a Runnable
	Callable callableTask = new CallableTask();
	// Step 2: Configure Executor
	// Uses FixedThreadPool executor
	ExecutorService executor = Executors.newFixedThreadPool(2);
	Future<String> future = executor.submit(callableTask);
	boolean listen = true;
	while (listen) {
		if (future.isDone()) {
			String result;
			try {
			result = future.get();
			listen = false;
			System.out.println(result);
			} catch (InterruptedException | ExecutionException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
			}
		 
		}
	}
 
}
 
```

Future
------
代表一个异步执行的结果。可以用来查询最终的执行结果以及查询异步任务的状态

FutureTask
----------
实现一个可以取消的异步执行。它实现了`Runnable`和`Future`接口，所以它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值。


Android Loader
===============

AsyncTask
----------
`AsyncTask`提供一个更为简单的方式使用UI线程的机制，它允许我们在其他线程执行后台操作，最终把执行结果push回UI线程。
需要在后台执行的逻辑需要实现在`doInBackground`方法，最终执行这个`AsyncTask`需要调用execute方法。

`AsyncTask`会在初始化的时候实例化一个实现了`Runnable`接口的`WorkerRunnable`，在它的call方法的逻辑中主要调用了需要子类实现的`doInBackground`方法：

```
mWorker = new WorkerRunnable<Params, Result>() {
    public Result call() throws Exception {
        mTaskInvoked.set(true);

        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        //noinspection unchecked
        return postResult(doInBackground(mParams));
    }
};
```
同时还会初始化一个`FutureTask`对象实例：

```
mFuture = new FutureTask<Result>(mWorker) {
    @Override
    protected void done() {
        try {
            postResultIfNotInvoked(get());
        } catch (InterruptedException e) {
            android.util.Log.w(LOG_TAG, e);
        } catch (ExecutionException e) {
            throw new RuntimeException("An error occured while executing doInBackground()",
                    e.getCause());
        } catch (CancellationException e) {
            postResultIfNotInvoked(null);
        }
    }
};
```

最终调用execute方法的时候就会执行上述的`FutureTask`实例

ModernAsyncTask
---------------
`ModernAsyncTask`是`AsyncTask`的一个拷贝，它仅仅包括了用来支持`AsyncTaskLoader`的代码，因为它需要其中一部分的功能。

AsyncTaskLoader
---------------
`AsyncTaskLoader`继承子`Loader`，主要用来通过异步任务来加载数据。所以它内部有一个继承自`ModernAsyncTask`的内部类`LoadTask`的成员变量，`AsyncTaskLoader`最终的异步执行都是代理给这个成员变量来执行的。

```
final class LoadTask extends ModernAsyncTask<Void, Void, D> implements Runnable {

        D result;
        boolean waiting;

        private CountDownLatch done = new CountDownLatch(1);

        /* Runs on a worker thread */
        @Override
        protected D doInBackground(Void... params) {
            if (DEBUG) Log.v(TAG, this + " >>> doInBackground");
            result = AsyncTaskLoader.this.onLoadInBackground();
            if (DEBUG) Log.v(TAG, this + "  <<< doInBackground");
            return result;
        }

        /* Runs on the UI thread */
        @Override
        protected void onPostExecute(D data) {
            if (DEBUG) Log.v(TAG, this + " onPostExecute");
            try {
                AsyncTaskLoader.this.dispatchOnLoadComplete(this, data);
            } finally {
                done.countDown();
            }
        }

        @Override
        protected void onCancelled() {
            if (DEBUG) Log.v(TAG, this + " onCancelled");
            try {
                AsyncTaskLoader.this.dispatchOnCancelled(this, result);
            } finally {
                done.countDown();
            }
        }

        @Override
        public void run() {
            waiting = false;
            AsyncTaskLoader.this.executePendingTask();
        }
    }
```

可以看到`LoadTask`还实现了`Runnable`接口，并且在实现逻辑中调用了`AsyncTaskLoader`的`executePendingTask`方法。
`executePendingTask`最终则会执行这个`LoadTask`的异步代码:

```
void executePendingTask() {
        if (mCancellingTask == null && mTask != null) {
            if (mTask.waiting) {
                mTask.waiting = false;
                mHandler.removeCallbacks(mTask);
            }
            if (mUpdateThrottle > 0) {
                long now = SystemClock.uptimeMillis();
                if (now < (mLastLoadCompleteTime+mUpdateThrottle)) {
                    // Not yet time to do another load.
                    if (DEBUG) Log.v(TAG, "Waiting until "
                            + (mLastLoadCompleteTime+mUpdateThrottle)
                            + " to execute: " + mTask);
                    mTask.waiting = true;
                    mHandler.postAtTime(mTask, mLastLoadCompleteTime+mUpdateThrottle);
                    return;
                }
            }
            if (DEBUG) Log.v(TAG, "Executing: " + mTask);
            mTask.executeOnExecutor(ModernAsyncTask.THREAD_POOL_EXECUTOR, (Void[]) null);
        }
    }
```

除了`Runnable`接口之外，`forceLoad`方法也调用了`executePendingTask`会执行实际的异步任务。而且实际我也发现这个`Runnable`对象也没有在另外的线程中执行。

所以我们的实现了`LoaderCallbacks#onCreateLoader`的方法后会发现，Loader并没有执行，而是要调用`forceLoad`方法才会执行。当然更合理的是把调用移到`onStartLoading`中：

```
@Override
protected void onStartLoading() {
	if (data != null)
	  deliverResult(data);

	if (takeContentChanged() || data == null)
	  forceLoad();
}
```

其他的一些问题
============

forceLoad的问题
-------------------
每次Loader开始执行后，在执行完成deliverResult之前，如果再次执行forceLoad，那么上一次直接结果最终并不会执行onLoadFinished方法。原因如下：

上面的分析看到`LoadTask`的onPostExecute会调用`AsyncTaskLoader`的dispatchOnLoadComplete方法：

```
void dispatchOnLoadComplete(LoadTask task, D data) {
    if (mTask != task) {
        if (DEBUG) Log.v(TAG, "Load complete of old task, trying to cancel");
        dispatchOnCancelled(task, data);
    } else {
        if (isAbandoned()) {
            // This cursor has been abandoned; just cancel the new data.
            onCanceled(data);
        } else {
            mLastLoadCompleteTime = SystemClock.uptimeMillis();
            mTask = null;
            if (DEBUG) Log.v(TAG, "Delivering result");
            deliverResult(data);
        }
    }
}
```

而`AsyncTaskLoader`的onForceLoader方法又是会重新创建`LoadTask`的实例

```
@Override
protected void onForceLoad() {
    super.onForceLoad();
    cancelLoad();
    mTask = new LoadTask();
    if (DEBUG) Log.v(TAG, "Preparing load: mTask=" + mTask);
    executePendingTask();
}
```

所以最终的执行结果并不会传递到LoaderManager中。


