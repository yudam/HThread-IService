# HThread-IService
Android开发艺术探索阅读笔记之：HandlerThread+IntentService源码阅读
Time：2017年11月7日 
context：HandlerThread+IntentService
HandlerThread：Thread的子类，内部实现了Handler和Looper，有自己的消息循环机制，主要使用场景IntentService

    @Override
    public void run() {
        mTid = Process.myTid();
        //创建Looper
        Looper.prepare();
        synchronized (this) {
            //获取当前Looper
            mLooper = Looper.myLooper();
            //幻想当前对象监视器上的所有线程
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        //循环读取消息
        Looper.loop();
        mTid = -1;
    }
    
IntentService：Service的一个特殊子类，属于抽象类，因此使用时必须继承并实现该类的抽象方法，内部可以执行后台耗时任务，任务结束自动停止，IntentService是Service的子类，优先级比单纯的线程要高，比较适合执行一些高优先级的任务。内部封装了HandlerThread和ServiceHandler。
 @Override
    public void onCreate() {

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
    
可以看到在OnCreate方法中初始化了HandlerThread和创建了ServiceHandler，根据Service的启动方式的声明周期：onCreate方法只会执行一次，之后的每一次调用都会执行onStartCommand方法，onStartCommand方法中有调onStart方法。
@Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
     @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
    
    
   在onStart方法中进行消息的发送，交由ServiceHandler的handleMessage方法处理，注意这里的intent就是我们启动Service时的Intent，通过该Intent可以解析出启动Service的参数。
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
    
ServiceHandler继承自Handler，传递过来的Intent交由onHandleIntent抽象方法处理，因此该方法的处理须有我们继承IntentService自行处理。接下来会调用stopSelf方法停止服务。这里调用的是stopSelf(int startId)，主要是因为stopSelf(int startId)会等到所有消息处理完毕才会结束服务，而stopSelf会立刻结束服务。

 public final void stopSelf() {
        stopSelf(-1);
    }

    public final void stopSelf(int startId) {
        if (mActivityManager == null) {
            return;
        }
        try {
            mActivityManager.stopServiceToken(
                    new ComponentName(this, mClassName), mToken, startId);
        } catch (RemoteException ex) {
        }
    }
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
