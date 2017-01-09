
我们常常会在面试中被问及 xx 知识点用法，然后面试官会接着问是否了解其工作原理。无可厚非，我们不能仅仅满足于会用，难道你就不想知道它是如何工作的，不想了解它的源码吗？

之前我们在 [Android AsyncTask 基本用法，参数和步骤理解，开启异步任务之旅][article] 中提到
> AsyncTask 是经过 Android 封装、简化的异步任务实现方式，内部实现也是由 Thread 和 Handler 来实现异步任务和切换线程的。

并且还在小结中提到几项规则和一点拓展，那些都是结论，现在我们就来了解一下 AsyncTask 的相关源码，知晓它的工作原理吧。

这篇文章我根据的是 Android 7.1.1 源码，不同版本可能略有差别。

# 源码解析
上次说到 AsyncTask 基本使用开启异步任务的方式一般是这样的

```java

//至少实现 doInBackground(Params...)
private class MyTask extends AsyncTask<Params, Progress, Result> { ... }

MyTask task = new MyTask();
task.execute(Params...) ;

```

现在就直接跟入 execute 方法中的实现
```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

```

进入后发现调用是另外一个方法 executeOnExecutor，并且多了一个 sDefaultExecutor 参数传入，具体表示什么稍后再看，先看下 executeOnExecutor 方法实现。

```java

@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}

```
先只需关注 17-22行
这边先判断了 mStatus的状态，如果值为 Status.PENDING 的初始等待执行的状态，先将状态改为 Status.RUNNING（正在执行状态），然后就执行 onPreExecute() 做一些初始准备工作，这个方法就是我们之前提到的四个步骤的第一个步骤，此时都是在 UI 线程中的。
然后将 params 赋值给 mWorker.mParams ，到了真正执行任务的地方 exec.execute(mFuture)，这边的 exec 就是刚刚传入的 sDefaultExecutor，让我们回头看下 sDefaultExecutor 究竟是什么。


```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

...
/**
 * An {@link Executor} that executes tasks one at a time in serial
 * order.  This serialization is global to a particular process.
 */
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

...
private static class SerialExecutor implements Executor {
	//用来存储任务的队列
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
	//表示当前正在执行的任务
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
		//将新来的任务加入队列中
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
					//任务执行完毕，才接着执行下一个任务
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}

```

一直跟随 sDefaultExecutor, 发现它最终表示的是实现了 Executor 的 SerialExecutor。还记得刚刚的 exec.execute(mFuture) 其实就是这边的 execute 方法。 一旦 execute 方法被调用起来，然后判断是否有正在执行的任务，没有的话就调用 scheduleNext 方法。scheduleNext 方法从队列中取出处于队列头部的任务，任务存在的话，就交给 THREAD_POOL_EXECUTOR 线程池去执行。
注意！虽然这边确实是用了线程池去执行任务，但是 SerialExecutor 实现会使得 AsyncTask 会串行执行任务，而不是并发的。

这就好像虽然有 **可以容纳许多人** 的泳池，但是还是遵循排队队列机制，从队列第一个开始丢进泳池，此时这个人在游泳。尽管继续有人排队，但是得等他游完了上岸，这个时候才又从此时队列的第一个放进去游泳。

现在我们再看下 exec.execute(mFuture)中的 mFuture 这个 Runnable 对象，这个是在 AsyncTask 构造函数中初始化好的。

```java
/**
 * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
 */
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
				//执行真正我们的想要执行的耗时操作任务，并传递返回值
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
				//利用 handler 消息机制传递返回结果
                postResult(result);
            }
            return result;
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}

```
仅需关注以下几行即可：
第7行，当 call 方法被调用，标识任务被执行过
第13行，熟悉的 doInBackground(mParams) 就是在这被调用
第20行，将 任务返回结果，通过 postResult 方法传递

这里 mWorker 实现了 Callable 接口中的 call 方法，mFuture 重写 FutureTask 的 done 方法，这边初始化并不会有什么逻辑执行。直到运行 execute 方法，被放进线程池中，会执行一个 Runnable 对象的 run 方法，此时的对象就是 mFuture。进入 FutureTask 类看下 run 方法，
```java
public void run() {
    if (state != NEW ||
        !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
        return;
    try {
		//这边的 callable 就是外面初始化传入的 mWorker
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
				//调用 mWorker call()
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}

```
第7行、第13行 表示：
线程中实际调用 mFuture 的 run 方法，run 方法又调用 mWorker 的 call 方法，call 中执行耗时操作 doInBackground(mParams) 这就是四个步骤中的第二个步骤 (必须实现)，并利用 postResult 传递返回结果 result。代码如下：

```java

private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}

```

这边 getHandler() 得到的是一个静态变量 sHandler，它拥有 Looper.getMainLooper()，然后这边利用 result 包装好给 message.obj，然后发送消息给 handleMessage 中处理，至此就切换回了 UI 线程。

```java

private void finish(Result result) {
	//判断是否是取消状态
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
	//改变当前状态为 Status.FINISHED 
	//所以，外界不能再尝试调用 execute 方法，状态检查会抛出异常
    mStatus = Status.FINISHED;
}

private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
	    // 这边 handler 是在 UI 线程中，因此这边处理都已切换回 UI 线程了
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}

```
刚刚发送过来的消息 msg.what 为 MESSAGE_POST_RESULT，result.mTask.finish(result.mData[0]) 指的调用自身 AsyncTask 对象的 finish 方法。
finish 方法就说明了 onCancelled 方法和 onPostExecute 互斥，另外，mStatus 状态的改变也说明同一个 AsycnTask 对象不能被调用多次 execute 方法。这些验证了之前文章 [Android AsyncTask 基本用法，参数和步骤理解，开启异步任务之旅][article] 中取消任务时候调用的是 onCancelled 方法的结论和注意规则！
因此，阅读源码是不是会让你有一种 「知其然 知其所以然」 的感觉，棒极了，感觉世界都亮了。

另外，这边还有一个 msg.what 为 MESSAGE_POST_PROGRESS 的情况，这个是调用 publishProgress 方法，返回任务过程的进度单位时，也走的是发送 Message 消息给 handler 处理，切换回 UI 线程的路。

至此，完整的 AsyncTask 异步任务执行流程，已经基本解析完毕。

# AsyncTask 细节

这边结论说了 AsyncTask 对象执行完 execute 方法不能执行多次，那 SerialExecutor 中怎么又提到什么新来的任务加入队列和串行执行的？
请注意这边 SerialExecutor 是被 static 修饰，因此整个应用中的所有的 AsyncTask 实例用的都是一个 SerialExecutor ，当然加入的同一个任务队列，所以才有了串行执行任务之说，仅用的是单一线程而已。

```java

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}

```

它的实现，就如我所举的泳池排队游泳的例子。即使你创建了许多 AsyncTask 的实例，都是按顺序加入队列，然后逐一取出执行，执行完一个任务，才能继续执行下一个。每次 run 方法执行完，都会走 finally 中的 scheduleNext 方法，所以不要担心如果当前有正在执行的任务了，我新加入的这个任务怎么办，尽管加入队列排队等待就好了。

接下来代码验证一下想法：
```java
@Override
public void onClick(View view) {
    int id = view.getId();
    if (id == R.id.btn_download) {
        //点击下载，执行后台任务
        //修改过的代码 开始
        new ImageAsyncTask().execute(IMAGE_URL, IMAGE_URL_TWO, IMAGE_URL_THREE);
        new ImageAsyncTask().execute(IMAGE_URL, IMAGE_URL_TWO, IMAGE_URL_THREE);
        new AsyncTask<Void, Void, Void>() {

            @Override
            protected Void doInBackground(Void... params) {
                try {
                    Thread.sleep(2000);
                    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                    Log.w(TAG, "asynctask thread name = " + Thread.currentThread().getName()
                                + " at " + sdf.format(new Date()));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return null;
            }
        }.execute();
        // 结束
    }

}

//String  是图片地址参数；
//Bitmap  是整个异步任务过程中，返回的进度单位，这里是图片
//Integer 是最终返回结果，返回结果为3，说明都下载成功，
class ImageAsyncTask extends AsyncTask<String, Bitmap, Integer> {


    @Override
    protected Integer doInBackground(String... params) {
        //记录成功下载的图片个数
        int downloadSuccess = 0;

        try {
            for (int i = 0; i < params.length; i++) {

                //睡眠2秒，制造耗时操作效果
                Thread.sleep(2000);

                if (isCancelled()) {
                    break;
                }

                //循环取出可变参数中图片地址
                String url = params[i];
                //Bitmap下载
                Bitmap bitmap = null;
                URLConnection connection;
                InputStream is;
                connection = new URL(url).openConnection();
                is = connection.getInputStream();
                BufferedInputStream bis = new BufferedInputStream(is);
                bitmap = BitmapFactory.decodeStream(bis);
                //
                bis.close();
                is.close();
                //将每一次下载好的图片，作为阶段性结果发回给 onProgressUpdate()
                publishProgress(bitmap);
                //成功下载一张则累加
                downloadSuccess++;
                //下载到3张图片，就即将要结束任务
                //修改过的代码 开始
                if (downloadSuccess == 3) {
                    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                    Log.w(TAG, "asynctask thread name = " + Thread.currentThread().getName() 
                                + " at " + sdf.format(new Date()));
                }
                // 结束
            }

        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return downloadSuccess;
    }

    @Override
    protected void onProgressUpdate(Bitmap... values) {
        super.onProgressUpdate(values);
        if (isCancelled()) {
            return;
        }
        //接收到的值-图片，可以直接进行更新UI，因为运行在主线程上
        mImageView.setImageBitmap(values[0]);
    }

}

```

```
//log 如下
 W/AsynctaskActivity: asynctask thread name = AsyncTask #1 at 2017-01-08 14:29:15
 W/AsynctaskActivity: asynctask thread name = AsyncTask #2 at 2017-01-08 14:29:22
 W/AsynctaskActivity: asynctask thread name = AsyncTask #3 at 2017-01-08 14:29:24
```

以上代码基本用的还是上篇文章的，略微修改（已标出）。
特意用了两个相同的 ImageAsyncTask 和 一个 匿名 AsyncTask，查看 log 时间之差可以验证说的结论。
小结一下：
整个应用中的所有的 AsyncTask 实例 **默认** 用的都是一个 SerialExecutor，加入的是同一个队列之中。
因此，我在上篇也说到过，特别耗时的后台任务不适合用 AsyncTask，串行执行任务显然会使得排队等待时间很长，导致体验不佳，推荐特别耗时任务使用线程池。

# AsyncTask 配置并发执行

上篇结尾提到的一点拓展中
>在 Android 1.6 以前，AsyncTask 是串行处理任务，在 Android 1.6 开始是用线程池里处理并发任务。然后从 Android 3.0 开始又默认是一个线程串行处理任务，为的是避免 AsyncTask 带来并发错误，从这个版本及以后，我们可以通过 executeOnExecutor 方法来灵活传入配置的线程池处理并发，否则默认是以串行方式。

接下来，我们就来简单配置线程池，实现并发执行效果。

```java
    @Override
    public void onClick(View view) {
        int id = view.getId();
        if (id == R.id.btn_download) {
            //点击下载，执行后台任务
            //修改的代码 开始
            new AsyncTask<Void, Void, Void>() {

                @Override
                protected Void doInBackground(Void... params) {
                    try {
                        Thread.sleep(2000);
                        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                        Log.w(TAG, "asynctask thread name = " + Thread.currentThread().getName()
                                + " at " + sdf.format(new Date()));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    return null;
                }
            }.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
            new AsyncTask<Void, Void, Void>() {

                @Override
                protected Void doInBackground(Void... params) {
                    try {
                        Thread.sleep(2000);
                        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                        Log.w(TAG, "asynctask thread name = " + Thread.currentThread().getName()
                                + " at " + sdf.format(new Date()));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    return null;
                }
            }.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
            new AsyncTask<Void, Void, Void>() {

                @Override
                protected Void doInBackground(Void... params) {
                    try {
                        Thread.sleep(2000);
                        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                        Log.w(TAG, "asynctask thread name = " + Thread.currentThread().getName()
                                + " at " + sdf.format(new Date()));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    return null;
                }
            }.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
            //修改的代码 结束
        }

    }

```

```
 W/AsynctaskActivity: asynctask thread name = AsyncTask #2 at 2017-01-08 14:52:48
 W/AsynctaskActivity: asynctask thread name = AsyncTask #1 at 2017-01-08 14:52:48
 W/AsynctaskActivity: asynctask thread name = AsyncTask #3 at 2017-01-08 14:52:48
```

这边修改为三个 匿名 AsyncTask 实现，仅仅以睡眠 2秒 模拟耗时操作，以排除真正网络请求的网络因素干扰。这样效果会更加明显，log 打印出来也几乎是同一时间。传入的配置的是线程池是 AsyncTask 内静态线程池。



[article]: http://blog.csdn.net/astro_gypsophila/article/details/54124759