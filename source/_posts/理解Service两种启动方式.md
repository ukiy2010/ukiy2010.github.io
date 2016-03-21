---
title: 理解Service两种启动方式
date: 2016-03-19 22:54:33
tags:
  - Android
  - Service
---
Service作为四大组件之一，在Android中扮演着处理后台任务的角色，比如后台下载啦，后台轮询啦，后台放音乐啦，反正一切和UI无关的耗时操作都可以丢给它来做，需要注意的是，Service和Activity一样，也是运行在主线程的哦，所以如果直接在其`onCreate()`，`onStartCommand()`等函数中直接进行耗时操作，可是会导致ANR错误的。More generally，四大组件都是运行在主线程里的呢~

有人可能要说，本来还以为你Service可以做些耗时的后台操作，现在搞得和Activity一样也运行在主线程，那我还要你作甚，还不如我直接在Activity里直接开个线程来处理咧。那么现在请考虑这样一个场景，我在Activity中点击了一个按钮，开启了一个线程进行下载操作，由于这个文件实在太大，用户不想一直在这个页面等着，于是点了返回按钮，这下好了，进行下载的Thread变成了一个孤儿了，由于它所属的Activity被从任务栈中弹出，它变得完全不可控，而且还会造成内存泄露！可能又有人要说，我在Activity的`onPause()`方法里把这个Thread结束掉不就完啦。可这样的话，当用户再回来看这个页面的时候该抱怨了：等了这么久，你丫竟然偷懒，没在给我下载啊？Android问：谁能担此大任？这时，只见小朋友们四大组件中，钻出来一个Service，说道：我来！

## 两种启动方式的生命周期

Service也有自己的一套独立于Activity的生命周期，这使得它能够在Activity结束后仍能够继续下载。Service的生命周期与它的两种启动方式有关：

![](http://7xs2xe.com1.z0.glb.clouddn.com/blog_service_lifecycle.png)

既然两种方法都能够启动Service，那么这两种方式有啥区别呢，举一个很通俗的例子

> 如果说Service是一个老师，现在你想让你的儿子（参数）接受老师的教育（后台操作），有两种方法，一种是直接把儿子送到老师那儿上课，一种是把老师请到家里来教。前者就相当于`startService(mIntent)`，你需要做的就是把儿子送上校车（Intent），好处就是把儿子送走后你就不用管了，该上班上班，该买菜买菜，缺点是校车只能接送int，double，String等基本类型的孩子，如果是自定义类型，那就只有办个vip（继承Parcelable或者Serializable接口）才能上车；后者就像`bindService`，你需要做的就是把老师请到家里来（`bindService()`），然后等老师教完儿子后再把老师送走（`unBindService()`），好处就是儿子不用出门了（废话），缺点就是得接待老师。

说完这些是不是理解一些了呢，下面我们来看看它们的具体操作，就拿后台下载这个例子来说吧。

## 使用startService

我们首先定义一个MyService类

``` Java
public class MyService extends Service {

    public void download(final Callback callback) {
        new Thread() {
            @Override
            public void run() {
                try {
                    sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                callback.onDone();
            }
        }.start();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Callback callback = (Callback) intent.getSerializableExtra("callback");//获取回调
        download(callback);
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    public interface Callback extends Serializable{
        void onDone();//下载完成的回调
    }
}
```

我们定义了一个下载的函数`download()`，里面开启线程模拟下载操作，在下载完成后执行回调`callback.onDone()`，回调是一个自定义的接口`Callback`继承了`Serializable`，为啥咧？上面说过，因为Intent能传递的参数种类十分有限，这也算是一种妥协吧。Service定义好了，接下来在Activity中启动它。

``` Java
Intent intent = new Intent(this, MyService.class);
intent.putExtra("callback", new MyService.Callback(){
    @Override
    public void onDone() {
		Log.d("MainActivity", "done in start!");
    }
});
startService(intent);
```

点击运行，咦，怎么报错了，平时传回调接口不就是这么传的吗？需要注意，`Serializable`接口在jvm看来就是个标记，说明这个类可以序列化，啥是序列化？我的理解就是把内存中的数据以某种形式保存在文件中，在将来某个时刻又可以把这个文件再还原到内存中，这是反序列化。`Serializable`有个规定，就是实现了这个接口的类，里面所有的成员变量都要是基本类型，或者实现`Serializable`接口。可是在这个匿名内部类中我们没有成员变量啊，只是重写了个方法嘛。别忘了，在Java中，非静态内部类是持有外部类引用的，而这个外部类Activity可没有实现`Serializable`接口哦。哈哈，知道这个后就好改啦，将这个内部类静态化！

``` Java
    static class MyCallback implements MyService.Callback{
        @Override
        public void onDone() {
            Log.d("MainActivity", "done in start!");
        }
    }
```

要注意，千万不要把非基本类型且非继承`Serializable`或`Parcelable`的类传进来当做成员变量使用，不然又绕回去了，真是如履薄冰呢。把原来启动Service的代码改成下面这样，点击运行按钮，不出所料的话2秒后就会打印出“done in start!”。

``` Java
Intent intent = new Intent(this, MyService.class);
intent.putExtra("callback", new MyCallback());
startService(intent);
```

是不是感觉做件这么简单的事竟然这么啰嗦，这就是`startService`的不足之处，传递自定义类型的时候束缚很多。但是如果仅仅传递基本类型的话还是很方便哒。

## 使用bindService

下面来介绍一下使用`bindService`方式来做同样一件事，对比它们的异同。直接在MyService类中添加一个类`MyBinder`，还拿上面送小孩上学的例子来说，这个类就相当于是一个专门送老师上门的车，在里面我们声明一个public函数，返回当前`MyService`实例（其实应该返回一个代理，这样可以隐藏Service里的一些细节，减少耦合度，这里只是为了方便），并且修改`onBind`函数的返回值，返回一个`Mybinder`的实例：

``` Java
    @Override
    public IBinder onBind(Intent intent) {
        return new MyBinder();
    }

    public class MyBinder extends Binder {
        public MyService getService() {
            return MyService.this;
        }
    }
```

接下来回到Activity中，我们定义一个变量实现`ServiceConnection`接口，这个接口就相当于是`bindService`的回调：

``` Java
    private ServiceConnection conn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            MyService myService = ((MyService.MyBinder) service).getService();
            myService.download(new MyService.Callback() {
                @Override
                public void onDone() {
                    Log.d("MainActivity", "done in bind!");
                }
            });
            unbindService(this);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
```

`onServiceConnected`回调参数中的`IBinder`就是刚才`MyService`里`onBind`的返回值，强制转换一下，就可以调用Service里边的方法啦，老师请到了家里，想让他咋教就咋教，内部类，自定义类型啥的都不在话下。于是我直接上了一个`Callback`匿名内部类，最后解绑。启动Service也很简单，喏：

``` Java
Intent intent = new Intent(this, MyService.class);
bindService(intent, conn, BIND_AUTO_CREATE);
```

点击运行，2秒后就看到“done in bind!”啦。

## 总结：

Service的两种启动方式各有千秋，`startService`在调用端的代码更简洁一些，把参数送出去后具体的处理逻辑就完全交给Service了，但是对传递的参数类型有限制，只能是一些基本类型或者是继承了`Serializable`或者`Parcelable`接口的类型；`bindService`方式启动的话，好处就是对参数类型没有限制，控制也更精细一些，也正由于此，调用端承载了一部分处理逻辑，所以代码耦合度会更高一些。在项目中如何取舍就得具体情况具体分析了。