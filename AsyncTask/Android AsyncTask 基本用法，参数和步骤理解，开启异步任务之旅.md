转载请标明出处：【Gypsophila 的博客】 http://blog.csdn.net/astro_gypsophila/article/details/54124759 

## 介绍：

AsyncTask 是轻量级的异步任务类，轻松地在 UI 线程控制后台操作和后台操作所返回结果，无需使用 Thread 和 Handler 这样的组合来进行切换。实际上 AsyncTask 是为我们所设计的关于 Thread 和 Handler 的帮助类。

AsyncTask 是经过 Android 封装、简化的异步任务实现方式，内部实现也是由 Thread 和 Handler 来实现异步任务和切换线程的。


## 使用：

AsyncTask 的一般使用代码写法形式如下：
```java
private class MyTask extends AsyncTask<Params, Progress, Result> { ... }
MyTask task = new MyTask();
task.execute(Params...) ;

```

如果我们想要快速弄懂 AsyncTask ，只需明白三个参数和四个步骤即可。

在异步任务中的三个参数如下：

1. Params: 执行任务时传入的参数类型
2. Progress: 后台操作执行过程中发回主线程中的阶段性结果返回类型
3. Result: 后台操作完全结束时返回给主线程的期待的返回类型

在异步任务中被调用四个步骤的顺序如下：

1. onPreExecute(): 在后台任务即 doInBackground(Params...) 执行前被调用，一般用于初始化某些值，例如可以是在交互页面上显示一个 ProgressBar 来提示将要进行后台任务。

2. doInBackground(Params...): 它将在 onPreExecute() 被调用后马上调用，后台耗时操作真正是被放在这里执行。这边的可变参数 Params 正是 三个参数 中的第一个参数传入到这边的，另外会在任务执行结束后 return 第三个泛型参数 Result。过程中还可以调用 publishProgress(Progress...)  来发送返回阶段性结果，给处于主线程中调用的 onProgressUpdate(Progress...) 中使用。

3. onProgressUpdate(Progress...)： 它会在执行 publishProgress(Progress...) 后被主线程调用起，这时仍在后台执行的任务会发回一个或多个阶段性进度结果，这个是可以用来去更新交互页面。

4. onPostExecute(Result)：当后台任务完全结束时会在主线程调用，这里的 Result 正是 doInBackground() 的返回值传入。

这些方法中参数包含 ... 的，是在 Java 中使用 ... 表示参数数量不固定，是一种数组型参数。

如果我的文字没有表述清楚，那就进入具体代码，加以相应注释帮助理解。

### 下载一张图片

目标1 是实现下载一张网络图片，看下 AsyncTask 是如何来操作的。


```java
package com.gypsophila.testdemo.asynctask;

import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.os.AsyncTask;
import android.os.Bundle;
import android.support.annotation.Nullable;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Button;
import android.widget.ImageView;
import android.widget.TextView;

import com.gypsophila.testdemo.R;

import java.io.BufferedInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.URL;
import java.net.URLConnection;

/**
 * Description：
 * Author：AstroGypsophila
 * GitHub：https://github.com/AstroGypsophila
 */
public class AsynctaskActivity extends AppCompatActivity implements View.OnClickListener {

    private TextView mResponseTv;
    private Button mDownloadBtn;
    private static String IMAGE_URL = "http://noavatar.csdn.net/7/F/C/1_astro_gypsophila.jpg";
    private ImageView mImageView;
    private ImageAsyncTask imageTask;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_asynctask);
        mResponseTv = (TextView) findViewById(R.id.response);
        mDownloadBtn = (Button) findViewById(R.id.btn_download);
        mImageView = (ImageView) findViewById(R.id.image);
        mDownloadBtn.setOnClickListener(this);
    }

    @Override
    public void onClick(View view) {
        int id = view.getId();
        if (id == R.id.btn_download) {
            //点击下载，执行后台任务
            imageTask = new ImageAsyncTask();
            imageTask.execute(IMAGE_URL);
        }

    }

    //String是图片地址参数；
    //Void是进度结果类型，这里用不到
    //Bitmap是最终返回结果，这里希望下载到一张图片
    class ImageAsyncTask extends AsyncTask<String, Void, Bitmap> {

        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            //页面提示
            mResponseTv.setText("下载中...");
        }

        @Override
        protected Bitmap doInBackground(String... params) {
            Bitmap bitmap = null;
            String url = params[0];
            URLConnection connection;
            InputStream is;
            try {
                //睡眠2秒，制造耗时操作效果
                Thread.sleep(2000);
                connection = new URL(url).openConnection();
                is = connection.getInputStream();
                BufferedInputStream bis = new BufferedInputStream(is);
                bitmap = BitmapFactory.decodeStream(bis);
                bis.close();
                is.close();

            } catch (IOException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return bitmap;
        }

        @Override
        protected void onPostExecute(Bitmap bitmap) {
            super.onPostExecute(bitmap);
            if (bitmap != null) {
                mImageView.setImageBitmap(bitmap);
                mResponseTv.setText("下载完成");
            }
        }
    }
}
```
以上代码，仅仅写了简单的布局，以及实例化了 AsyncTask 子类，由于 AsyncTask 是抽象类，且至少需要实现抽象方法 doInBackground(Params... params)。
在进行下载图片前，onPreExecute() 内做一些提示性或者初始化的操作；然后在真正下载图片代码则是在处于 doInBackground(String... params) 中，它是在子线程中被调用的；最后在onPostExecute(Bitmap) 中拿到图片，更新 UI。

效果查看：

![使用一](http://img.blog.csdn.net/20170106020827450?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYXN0cm9fR3lwc29waGlsYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


目标1 中代码并没有都使用上所有三个参数和四个步骤，为符合情景现在稍作修改，将它们都使用上。

### 下载多张图片
目标2 是总共下载三张图片，过程就每下载一张图片就直接先返回更新当前的 ImageView 。

修改的代码如下：

```java 
...
//下载三张图片
private static String IMAGE_URL = "http://noavatar.csdn.net/7/F/C/1_astro_gypsophila.jpg";
private static String IMAGE_URL_TWO = "http://tva1.sinaimg.cn/crop.0.5.309.309.180/69252704jw8erra828plgj208l0ekaaj.jpg";
private static String IMAGE_URL_THREE = "http://noavatar.csdn.net/7/F/C/1_astro_gypsophila.jpg";
...

```

```java 
...
 @Override
 public void onClick(View view) {
     int id = view.getId();
     if (id == R.id.btn_download) {
         //点击下载，执行后台任务
         imageTask = new ImageAsyncTask();
         imageTask.execute(IMAGE_URL, IMAGE_URL_TWO, IMAGE_URL_THREE);
     } 
 }
...
```
```java 
//String  是图片地址参数；
//Bitmap  是整个异步任务过程中，返回的进度单位，这里是图片
//Integer 是最终返回结果，返回结果为3，说明都下载成功，
class ImageAsyncTask extends AsyncTask<String, Bitmap, Integer> {

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
        //页面提示
        mResponseTv.setText("下载中...");
    }

    @Override
    protected Integer doInBackground(String... params) {
        //记录成功下载的图片个数
        int downloadSuccess = 0;
        try {
            for (int i = 0; i < params.length; i++) {
                //睡眠2秒，制造耗时操作效果
                Thread.sleep(2000);
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
        //接收到的值-图片，可以直接进行更新UI，因为运行在主线程上
        mImageView.setImageBitmap(values[0]);
    }

    @Override
    protected void onPostExecute(Integer integer) {
        super.onPostExecute(integer);
        if (integer == 3) {
            mResponseTv.setText("全部成功下载");
        }
    }
}

```
以上代码，在下载三张图片的过程中，每次下载完成都会返回一张图片来设置给 ImageView 。特别说明下 publishProgress(Progress) 和 onProgressUpdate(Progress... values) 里面的参数是你期望在后台任务执行过程中，想返回的什么类型，就在这边体现。
看了很多人举了加载进度条的例子，这边返回的是 Integer 数值，并且又因为泛型参数名字为 Progress，担心初学者会误认为只指任务执行的进度数值。其实不是，它应该是你期望在后台任务执行过程中想要返回的参数类型或者是进度单位。

效果图：

![使用二](http://img.blog.csdn.net/20170106020418944?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYXN0cm9fR3lwc29waGlsYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 取消异步任务
目标3 在下载三张图片过程中，取消这一异步任务，并且给与我们响应。
修改的代码如下：

点击取消下载按钮，设置取消标志
```java
...
@Override
public void onClick(View view) {
    int id = view.getId();
    if (id == R.id.btn_download) {
        //点击下载，执行后台任务
        imageTask = new ImageAsyncTask();
        imageTask.execute(IMAGE_URL, IMAGE_URL_TWO, IMAGE_URL_THREE);
    } else if (id == R.id.btn_cancel) {
        if (imageTask != null && imageTask.getStatus() == AsyncTask.Status.RUNNING) {
            //仅仅将 AsyncTask 的状态标记为 cancel 状态，并不是真正取消
            imageTask.cancel(true);
        }
    }

}
...
```
真正的取消，是判断是否取消的标志，然后不继续执行接下来代码
```java
@Override
protected Integer doInBackground(String... params) {
    //记录成功下载的图片个数
    int downloadSuccess = 0;
    try {
        for (int i = 0; i < params.length; i++) {

            //睡眠2秒，制造耗时操作效果
            Thread.sleep(2000);
			//若为取消状态，则跳出循环
            if (isCancelled()) {
                break;
            }
            ...
        }

    } catch (IOException e) {
        e.printStackTrace();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    return downloadSuccess;
}

@Override
protected void onPostExecute(Integer integer) {
    super.onPostExecute(integer);
    if (integer == 3) {
        mResponseTv.setText("全部成功下载");
    }
    //注意!取消下载后，这边并不会响应返回
    //因为 AsyncTask 源码中判断当前是取消状态时，调用的是onCancelled(),而不是onPostExecute()

}
```

新增重写 onCancelled(）方法

```java
@Override
protected void onCancelled(Integer integer) {
    super.onCancelled(integer);
    //取消后的响应在这边写
    mResponseTv.setText("取消下载");
}
```
直接看效果：
![使用三](http://img.blog.csdn.net/20170107002522952?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYXN0cm9fR3lwc29waGlsYQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 小结和注意规则

网络请求或者其他不确定的耗时操作，当然要开启子线程来执行，涉及到异步任务使用 AsyncTask 能较为简易地满足需求。其实 AsyncTask 并不适合处理特别耗时的后台任务，对于特别耗时任务建议使用线程池，但普通的异步任务，有它就大胆放心使用吧。
有了 AsyncTask，我们不必使用 Thread 和Handler 组合来进行线程中发送消息和处理消息的方式来处理异步任务，但是我们需要注意以下 AsyncTask 使用过程中的线程规则。

1. AsyncTask 的类必须在 UI 线程中加载。这个过程在 JELLY_BEAN(Android 4.1) 及以上版本被系统自动完成。
2. AsyncTask 的实例必须在 UI 线程中创建。
3. execute 方法必须在 UI 线程中调用。
4. AsyncTask 的 onPreExecute(), onPostExecute(Result), doInBackground(Params...), onProgressUpdate(Progress...) 不能手动调用。
5. 一个 AsyncTask 实例只能执行一次，即只能调用 execute 方法一次，否则将抛出运行时异常。

一点拓展是：
在 Android 1.6 以前，AsyncTask 是串行处理任务，在 Android 1.6 开始是用线程池里处理并发任务。然后从 Android 3.0 开始又默认是一个线程串行处理任务，为的是避免  AsyncTask 带来并发错误，从这个版本及以后，我们可以通过 executeOnExecutor 方法来灵活传入配置的线程池处理并发，否则默认是以串行方式。


本文若有描述不清或者错误之处，还请指出，多谢。