转载请标明出处：【Gypsophila 的博客】 http://blog.csdn.net/astro_gypsophila/article/details/54588293

## Handler 介绍

Android 消息机制主要指的就是 Handler 运行机制和 Handler 附带的 Looper 和 MessageQueue 的工作过程，其重要性不言而喻。平时我们大多都只需要接触到 Handler，它的主要使用是将封装的 Message 和 Runnable 对象加入消息队列中，并在循环队列消息取出时执行相应任务。

另外，有一点需要明确，Handler 其实并不是专门用来更新 UI 的，而是我们通过常常这种消息机制来进行更新 UI 而已。

## Handler 在各场景使用

接下来看具体几种 Handler 使用：

### 子线程向主线程发送消息

代码一

```java

private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (resultTv != null) {
                resultTv.setText("下载完成");
            }
        }
    };
    
...
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_handler);

    downloadBtn = (Button) findViewById(R.id.btn_download);
    resultTv = (TextView) findViewById(R.id.tv_result);
    //子线程向主线程发送消息，常被用于更新 UI
    downloadBtn.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            resultTv.setText("下载中...");
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        //模拟耗时操作
                        Thread.sleep(3000);
                        //下载完成通知 UI 线程更新
                        Message message = Message.obtain();
                        message.what = 1;
                        mHandler.sendMessage(message);
//                            mHandler.post(new Runnable() {
//                                @Override
//                                public void run() {
//                                    if (resultTv != null) {
//                                        resultTv.setText("下载完成");
//                                    }
//                                }
//                            });
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    });
}
```

以上代码，我们常常是创建 Hanlder 子类并且重写 handleMessage 方法，然后在子线程中处理 IO 或网络请求等耗时操作，然后利用 Hanlder 的 sendMessage 方法，发送相应 message 到 UI 线程中，最后在 handleMessage 方法继续处理相应逻辑，比如更新 UI。

另外注释的部分，post 方法也可以发送一个 Runnable 对象到消息队列之中，区别之处是不必去重写 handleMessage 方法，而是直接可以在 run 方法中即可处理相应逻辑。具体原因会在接下来的 Handler 源码解析的文章中说明。

### 主线程向子线程发送消息

代码二

```java

@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_handler);
    mThread = new MyThread();
    mThread.start();

    mCount = 0;

    sendToChildBtn.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            Message message = Message.obtain();
            message.what = 1;
            message.arg1 = ++mCount;
            mThread.mHandler.sendMessage(message);
        }
    });
}



class MyThread extends Thread {
    public Handler mHandler;
    public Looper mLooper;
    public static final String TAG = "MyThread";

    @Override
    public void run() {
        //子线程中初始化 Looper 实例和 MessageQueue 消息队列
        Looper.prepare();

        mLooper = Looper.myLooper();
        mHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);

                //消息对应的耗时操作
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                Log.w(TAG, "handle message from main thread, cnt = " + msg.arg1 +
                        ", current thread = " + Thread.currentThread().getName());
                

                //可用于定时、自动循环等场景
//                    Message message = Message.obtain();
//                    message.arg1 = ++mCount;
//                    mHandler.sendMessageDelayed(message, 1000);
            }
        };
        
		//其他耗时操作
		...
        //开启消息循环
        Looper.loop();

        //之后的代码不会被执行，loop 方法为无限循环
    }
}



```

```java
 W/MyThread: handle message from main thread, cnt = 1, current thread = Thread-12865

```
以上代码，可以应用于主线程“干预”或者“控制”子线程处理，这时候可以通过这样 Handler 发送消息来实现。

前面我们所写的两个示例，其实均是使用与 Handler 所在线程的 Looper。第一个中是使用主线程 ActivityThread 的 main 方法中初始化的 Looper，是自动实现初始化。因此我们在主线程中无须自己去做初始化 Looper 操作，并且本身运行应用后即进入 ActivityThread 的 main 入口，就做了初始化 Looper 和 Looper.loop() 开启消息循环操作，由此可见，其实我们一直是处于消息机制工作之中的。第二个中是我们创建的子线程就需要手动初始化和开启消息循环，同时我们也在子线程实例化了 Handler 。

代码三

```java
public static void main(String[] args) {

    ...

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    // End of event ActivityThreadMain.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}

```
以上代码就是主线程中帮我们实现初始化和开启消息循环


### 非默认 Looper 所构造的 Handler

之前的两个示例中代码所创建的 Handler 实例，都是与当前线程中所构造的 Looper 实例进行关联，现在就使用外部传入的 Looper 实例进行构造 Handler 对象。
因为我曾经疑惑于 Handler 能够切换在指定线程中执行任务，是由于 Handler 被创建的所在线程决定的还是由于 Handler 所关联的 Looper 实例被创建的所在线程决定的。至少在经过代码测试后了解，后者才是正确的。

代码四 

```java

// Looper 来自 MyThread 线程, Handler 在主线程中实例化
Handler mHanlderForThread = new Handler(mThread.mLooper) {
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        Log.w(MyThread.TAG, "current thread = " + Thread.currentThread() + " message!");
    }
};

mHanlderForThread.sendEmptyMessage(1);


```
虽然以上代码在传入 Looper 实例构造 Handler 时可能存在问题，稍后再说。
以上代码，创建的 mHanlderForThread 所传入的 Looper 实例是之前 MyThread 中初始化好的，也就是说 MyThread 中的 mHandler 与这里 mHanlderForThread 是共用一个 Looper 来做消息的分发处理，但是他们完全不会影响对方。因为哪个 Handler 发出的消息，最终都会被指定为该 Handler 来处理对应自己发出的消息的。


```java
 W/MyThread: current thread = Thread[Thread-13058,5,main] message!
 W/MyThread: handle message from main thread, cnt = 1, current thread = Thread-13058

```
log 上打印出的第一条是属于 mHanlderForThread 中 handleMessage 方法打印，第二条是属于 MyThread 中的 mHandler 内 handleMessage 方法打印出。虽然两个不同 Handler 在不同线程中被实例化，但可以看出它们执行的任务是属于同一线程，因为它们共用做消息循环的 Looper，但他们都在各自的 handleMessage 方法中处理自己的逻辑。

接下来回头看下在传入 Looper 实例构造 Handler 时可能存在问题：
在为 Handler 传入指定线程的 Looper 实例时，有可能出现 Looper 实例还未创建完成，导致的空指针问题。

为了避免空指针问题，系统已经为我们封装了 HandlerThread 类来为我们解决了这个问题，因此我们接下来看下如何去使用 HandlerThread 即可。因为它的内部实现几乎与以上示例代码一样，只不过增加了一些为了避免空指针增加的线程等待和唤醒判断。

代码五

```java

//为线程指定名字
mHandlerThead = new HandlerThread("handler thread");
mHandlerThead.start();

//将 HandlerThread 中初始化的 Looper 实例与 Handler 关联
mHandlerForHandlerThread = new Handler(mHandlerThead.getLooper()) {
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        Log.w("HandlerThread", "current thread = " + Thread.currentThread() + " message!");
    }
};
handlerTheadBtn.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        mHandlerForHandlerThread.sendEmptyMessage(1);
    }
});

}

```

对应打印出的 log

```java
 W/HandlerThread: current thread = Thread[handler thread,5,main] message!

```

以上代码，当 mHandlerThead.start() 执行后，其实线程内 run 方法中就是做的初始化 Looper 和 loop 方法。

```java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}

```

因此这个线程会一直在运行，Looper.loop() 为无限循环，所以在必要时候我们需要手动退出 Looper，运行 HandlerThread 内的 quit 方法就好。该类就是帮忙保证了传给 Handler 的 Looper 实例一定要被先初始化完成，内部 quit、quitSafely 方法都是调用 Looper 中的 quit、quitSafely。

### 小结

以上的几种情况，我们可以知道了 Handler 消息运行机制重要性。我们需要注意的是：

1. Handler 能够起到在指定线程中执行任务的作用，是由于与之关联的 Looper 实例决定。
2. 由第1点可得，与主线程 Looper 实例关联的 Handler 不可用于做非常耗时的操作。
3. 我们可以明确 Handler 的作用绝不仅仅是在子线程中完成耗时操作后去更新 UI 的，只是其中的一种使用场景而已。


## 几种常见的更新 UI 方式

- handler.sendMessage 方法 与 handler handleMessage 方法配合
- handler.post(Runnable r) 
- view.post(Runnable r)
- Activity 中 runOnUiThread(Runnable action)

```java

private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        if (resultTv != null) {
            resultTv.setText("下载完成");
        }
    }
};

//子线程向主线程发送消息，常被用于更新 UI
downloadBtn.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        resultTv.setText("下载中...");
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    //模拟耗时操作
                    Thread.sleep(3000);
                    //下载完成通知 UI 线程更新
                    //第一种方式
                    Message message = Message.obtain();
                    message.what = 1;
                    mHandler.sendMessage(message);
                    //第二种方式
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            if (resultTv != null) {
                                resultTv.setText("下载完成");
                            }
                        }
                    });
                    //第三种方式
                    resultTv.post(new Runnable() {
                        @Override
                        public void run() {
                            resultTv.setText("下载完成");
                        }
                    });
                    //第四种方式
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            resultTv.setText("下载完成");
                        }
                    });
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
});

```

其实这四种方式，只要我们稍微去查看下它的源码实现，其实都能发现 Handler 的身影。它们的本质还是通过 Handler 消息机制去实现的。
