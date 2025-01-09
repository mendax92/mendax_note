### 使用Handler为什么会内存泄漏
>来源：[Android 非静态内部类导致的内存泄露（非static内部类）](https://www.jianshu.com/p/6a362ea4dfd8)
>

定义一个普通内部类 和 一个静态内部类
```
public class OuterClass {
 
    public class NormalInnerClass {
        public void call() {
            fun();
        }
    }
    public static class StaticInnerClass {
        public void ask() {
//            fun(); //compile error
        }
    }
 
    public void fun() {
 
    }
}
```

反编译
- 普通内部类 OuterClass$NormalInnerClass.class
```
wangxin@wangxin:~/src/browser_6.9.7_forcoopad$ javap -c ./out/production/browser/com/qihoo/browser/OuterClass\$NormallInnerClass.class
Compiled from "OuterClass.java"
public class com.qihoo.browser.OuterClass$NormallInnerClass {
  final com.qihoo.browser.OuterClass this$0;
 
  public com.qihoo.browser.OuterClass$NormallInnerClass(com.qihoo.browser.OuterClass);
    Code:
       0: aload_0       
       1: aload_1       
       2: putfield      #1                  // Field this$0:Lcom/qihoo/browser/OuterClass;
       5: aload_0       
       6: invokespecial #2                  // Method java/lang/Object."<init>":()V
       9: return        
 
  public void call();
    Code:
       0: aload_0       
       1: getfield      #1                  // Field this$0:Lcom/qihoo/browser/OuterClass;
       4: invokevirtual #3                  // Method com/qihoo/browser/OuterClass.fun:()V
       7: return        
}
```
- 静态内部类 OuterClass$StaticInnerClass.class
```
wangxin@wangxin:~/src/browser_6.9.7_forcoopad$ javap -c ./out/production/browser/com/qihoo/browser/OuterClass\$StaticInnerClass.class
Compiled from "OuterClass.java"
public class com.qihoo.browser.OuterClass$StaticInnerClass {
  public com.qihoo.browser.OuterClass$StaticInnerClass();
    Code:
       0: aload_0       
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return        
 
  public void ask();
    Code:
       0: return        
}
```

普通的非static内部类比static内部类多了一个field: final com.qihoo.browser.OuterClass this$0; 在默认的构造方法中， 用外部类的对象对这个filed赋值.这也是为什么静态内部类调用外部方法报错原因。
```
package com.qihoo.browser;
 
import com.qihoo.browser.OuterClass;
 
public class OuterClass$NormallInnerClass {
    public OuterClass$NormallInnerClass(OuterClass var1) {
        this.this$0 = var1;
    }
 
    public void call() {
        this.this$0.fun();
    }
}
```

```
public class SampleActivity extends Activity {
 
  private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ...
    }
  }
 
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
 
    // 延时10分钟发送一个消息
    mLeakyHandler.postDelayed(new Runnable() {
      @Override
      public void run() { }
    }, 60 * 10 * 1000);
 
    // 返回前一个Activity
    finish();
  }
}
```
```
public final class Message implements Parcelable {
    ...
    Handler target;
    ...
}
```
**总结**：handler 实例化之后会和Meaasge 关联，而非静态内部类handler 内部有隐式持有外部类的引用（activity）。上面例子message持有handler对象，这个handler对象又隐式持有着SampleActivity对象.直到消息被处理前，这个handler对象都不会被释放, 因此SampleActivity也不会被释放。注意，这个匿名Runnable类对象也一样。匿名类的非静态实例持有一个隐式的外部类引用,因此SampleActivity将被泄露。
**解决方案**：
1. 界面销毁的时候，将handler的消息给移除
```
@Override
protected void onDestroy() {
    super.onDestroy();
    if (mHandler != null)  {
        mHandler.removeCallbacksAndMessages(null);
    }
}
```
2. handler改成静态内部类 或者 外部类，然后activity 使用弱引用包裹
```
public class SampleActivity extends Activity {
    /**
    * 匿名类的静态实例不会隐式持有他们外部类的引用
    */
    private static final Runnable sRunnable = new Runnable() {
            @Override
            public void run() {
            }
        };

    private final MyHandler mHandler = new MyHandler(this);

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 延时10分钟发送一个消息.
        mHandler.postDelayed(sRunnable, 60 * 10 * 1000);

        // 返回前一个Activity
        finish();
    }

    /**
    * 静态内部类的实例不会隐式持有他们外部类的引用。
    */
    private static class MyHandler extends Handler {
        private final WeakReference<SampleActivity> mActivity;

        public MyHandler(SampleActivity activity) {
            mActivity = new WeakReference<SampleActivity>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            SampleActivity activity = mActivity.get();

            if (activity != null) {
                // ...
            }
        }
    }
}
```