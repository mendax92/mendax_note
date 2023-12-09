全部-supper

![image-20231209153803240](https://raw.githubusercontent.com/mendax92/pic/main/img/image-20231209153803240.png?token=AEW7REFMLGWYQ7FUGHWO3K3FOQMZ2)

activity-dispathTouchEvent-true或者false

![image-20231209153834430](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20231209153834430.png)

ViewGroup-dispathTouchEvent-true

![image-20231209153850709](https://raw.githubusercontent.com/mendax92/pic/main/img/image-20231209153850709.png?token=AEW7REH5EUUROOPLVWEP7WTFOQM4U)

ViewGroup-dispathTouchEvent-false

![image-20231209153912295](https://raw.githubusercontent.com/mendax92/pic/main/img/image-20231209153912295.png?token=AEW7REEGHY4KMVGUDXW2JFLFOQM56)

View--dispathTouchEvent-true

![image-20231209154030226](https://raw.githubusercontent.com/mendax92/pic/main/img/image-20231209154030226.png?token=AEW7REHJIVHS6A2WDZ4ZN6TFOQNC4)

View--dispathTouchEvent-false

![image-20231209154111123](https://raw.githubusercontent.com/mendax92/pic/main/img/image-20231209154111123.png?token=AEW7REGNKRL5BVDC6QLPBHTFOQNFM)

### Android 事件分发流

关于Android 事件分发机制网上的博文很多，但是很多都是写个Demo然后贴一下输出的Log或者拿[源码](https://so.csdn.net/so/search?q=源码&spm=1001.2101.3001.7020)分析，然后一堆的注释和说明，如果用心的去看肯定是收获不少但是确实很难把整个流程说清和记住。曾经也是拼命想记住整个流程，但是一段时间又忘了，最后觉得分析这种问题和事件流的走向，一张图来解释和说明会清晰很多，下面我根据画的一张事件分发流程图,说明的事件从用户点击之后，在不同函数不同返回值的情况的最终走向。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtYjljYjY1YWNlZWE5MjE5Yi5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy84ODUvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

**注：**

> - 仔细看的话，图分为3层，从上往下依次是Activity、ViewGroup、View

- 事件从左上角那个白色箭头开始，由Activity的dispatchTouchEvent做分发
- 箭头的上面字代表方法返回值，（return true、return false、return super.xxxxx(),super 的意思是调用父类实现。
- dispatchTouchEvent和 onTouchEvent的框里有个【**true---->消费**】的字，表示的意思是如果方法返回true，那么代表事件就此消费，不会继续往别的地方传了，事件终止。
- 目前所有的图的事件是针对ACTION_DOWN的，对于ACTION_MOVE和ACTION_UP我们最后做分析。
- 之前图中的Activity 的dispatchTouchEvent 有误（图已修复），只有return super.dispatchTouchEvent(ev) 才是往下走，返回true 或者 false 事件就被消费了（终止传递）。

仔细看整个图，我们得出事件流 走向的几个结论（希望读者专心的看下图 1，多看几遍，脑子有比较清晰的概念。）
1、**如果事件不被中断，整个事件流向是一个类U型图**，我们来看下这张图，可能更能理解U型图的意思。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtZDAxYTU4NDVmNzQyNjA5Ny5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy84NzQvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

所以如果我们没有对控件里面的方法进行重写或更改返回值，而直接用super调用父类的默认实现，那么整个事件流向应该是从Activity---->ViewGroup--->View 从上往下调用dispatchTouchEvent方法，一直到叶子节点（View）的时候，再由View--->ViewGroup--->Activity从下往上调用onTouchEvent方法。

**2、dispatchTouchEvent 和 onTouchEvent 一旦return true,事件就停止传递了（到达终点）（没有谁能再收到这个事件）**。看下图中只要return true事件就没再继续传下去了，**对于return true我们经常说事件被消费了，消费了的意思就是事件走到这里就是终点，不会往下传，没有谁能再收到这个事件了**。

 

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtODRjMzc1NTFhZDUyMDY0MS5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy85MDgvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

**3、dispatchTouchEvent 和 onTouchEvent return false的时候事件都回传给父控件的onTouchEvent处理。**

 

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtNmJjOGUyNzk1YTMwZjk4Zi5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy84OTMvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

看上图深蓝色的线，对于返回false的情况，事件都是传给父控件onTouchEvent处理。

> - **对于dispatchTouchEvent 返回 false 的含义应该是：事件停止往子View传递和分发同时开始往父控件回溯（父控件的onTouchEvent开始从下往上回传直到某个onTouchEvent return true），事件分发机制就像递归，return false 的意义就是递归停止然后开始回溯。**

- **对于onTouchEvent return false 就比较简单了，它就是不消费事件，并让事件继续往父控件的方向从下往上流动。**

**4、dispatchTouchEvent、onTouchEvent、onInterceptTouchEvent
ViewGroup 和View的这些方法的默认实现就是会让整个事件安装U型完整走完，所以 return super.xxxxxx() 就会让事件依照U型的方向的完整走完整个事件流动路径），中间不做任何改动，不回溯、不终止，每个环节都走到。**

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtN2YzYWI5ZTdlN2ExZjBhMi5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy82NDEvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

所以如果看到方法return super.xxxxx() 那么事件的下一个流向就是走U型下一个目标，稍微记住上面这张图，你就能很快判断出下一个走向是哪个控件的哪个函数。
5、onInterceptTouchEvent 的作用

 

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtNDAzZjZlOGI4MjBmNzFmMS5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy84OTMvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

Intercept 的意思就拦截，每个ViewGroup每次在做分发的时候，问一问拦截器要不要拦截（也就是问问自己这个事件要不要自己来处理）如果要自己处理那就在onInterceptTouchEvent方法中 return true就会交给自己的onTouchEvent的处理，如果不拦截就是继续往子控件往下传。**默认是不会去拦截的，因为子View也需要这个事件，所以onInterceptTouchEvent拦截器return super.onInterceptTouchEvent()和return false是一样的，是不会拦截的，事件会继续往子View的dispatchTouchEvent传递**。

6、ViewGroup 和View 的dispatchTouchEvent方法返回super.dispatchTouchEvent()的时候事件流走向。

 

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtNGVkZjE3Yjc4OWNkMDQ3Zi5wbmc_aW1hZ2VNb2dyMi9hdXRvLW9yaWVudC9zdHJpcHxpbWFnZVZpZXcyLzIvdy84MTkvZm9ybWF0L3dlYnA?x-oss-process=image/format,png)

首先看下ViewGroup 的dispatchTouchEvent，之前说的return true是终结传递。return false 是回溯到父View的onTouchEvent，**然后ViewGroup怎样通过dispatchTouchEvent方法能把事件分发到自己的onTouchEvent处理呢，return true和false 都不行，那么只能通过Interceptor把事件拦截下来给自己的onTouchEvent，所以ViewGroup dispatchTouchEvent方法的super默认实现就是去调用onInterceptTouchEvent，记住这一点**。
那么对于**View**的dispatchTouchEvent return super.dispatchTouchEvent()的时候呢事件会传到哪里呢，很遗憾View没有拦截器。**但是同样的道理return true是终结。return false 是回溯会父类的onTouchEvent，怎样把事件分发给自己的onTouchEvent 处理呢，那只能return super.dispatchTouchEvent,View类的dispatchTouchEvent（）方法默认实现就是能帮你调用View自己的onTouchEvent方法的。**

说了这么多，不知道有说清楚没有，我这边最后总结一下：

- 对于 dispatchTouchEvent，onTouchEvent，return true是终结事件传递。return false 是回溯到父View的onTouchEvent方法。
- ViewGroup 想把自己分发给自己的onTouchEvent，需要拦截器onInterceptTouchEvent方法return true 把事件拦截下来。
- ViewGroup 的拦截器onInterceptTouchEvent 默认是不拦截的，所以return super.onInterceptTouchEvent()=return false；
- View 没有拦截器，为了让View可以把事件分发给自己的onTouchEvent，View的dispatchTouchEvent默认实现（super）就是把事件分发给自己的onTouchEvent。

ViewGroup和View 的dispatchTouchEvent 是做事件分发，那么这个事件可能分发出去的四个目标

> 注：------> 后面代表事件目标需要怎么做。
> 1、 自己消费，终结传递。------->return true ；
> 2、 给自己的onTouchEvent处理-------> 调用super.dispatchTouchEvent()系统默认会去调用 onInterceptTouchEvent，在onInterceptTouchEvent return true就会去把事件分给自己的onTouchEvent处理。
> 3、 传给子View------>调用super.dispatchTouchEvent()默认实现会去调用 onInterceptTouchEvent 在onInterceptTouchEvent return false，就会把事件传给子类。
> 4、 不传给子View，事件终止往下传递，事件开始回溯，从父View的onTouchEvent开始事件从下到上回归执行每个控件的onTouchEvent------->return false；
> **注：** 由于View没有子View所以不需要onInterceptTouchEvent 来控件是否把事件传递给子View还是拦截，所以View的事件分发调用super.dispatchTouchEvent()的时候默认把事件传给自己的onTouchEvent处理（相当于拦截），对比ViewGroup的dispatchTouchEvent 事件分发，View的事件分发没有上面提到的4个目标的第3点。

ViewGroup和View的onTouchEvent方法是做事件处理的，那么这个事件只能有两个处理方式：

> 1、自己消费掉，事件终结，不再传给谁----->return true;
> 2、继续从下往上传，不消费事件，让父View也能收到到这个事件----->return false;View的默认实现是不消费的。所以super==false。

ViewGroup的onInterceptTouchEvent方法对于事件有两种情况：

> 1、拦截下来，给自己的onTouchEvent处理--->return true;
> 2、不拦截，把事件往下传给子View---->return false,ViewGroup默认是不拦截的，所以super==false；

### 关于ACTION_MOVE 和 ACTION_UP####

上面讲解的都是针对ACTION_DOWN的事件传递，ACTION_MOVE和ACTION_UP在传递的过程中并不是和ACTION_DOWN 一样，你在执行ACTION_DOWN的时候返回了false，后面一系列其它的action就不会再得到执行了。简单的说，就是当dispatchTouchEvent在进行事件分发的时候，只有前一个事件（如ACTION_DOWN）返回true，才会收到ACTION_MOVE和ACTION_UP的事件。具体这句话很多博客都说了，但是具体含义是什么呢？我们来看一下下面的具体分析。

上面提到过了，事件如果不被打断的话是会不断往下传到叶子层（View），然后又不断回传到Activity，dispatchTouchEvent 和 onTouchEvent 可以通过return true 消费事件，终结事件传递，而onInterceptTouchEvent 并不能消费事件，它相当于是一个分叉口起到分流导流的作用，ACTION_MOVE和ACTION_UP 会在哪些函数被调用，之前说了并不是哪个函数收到了ACTION_DOWN，就会收到 ACTION_MOVE 等后续的事件的。
下面通过几张图看看不同场景下，ACTION_MOVE事件和ACTION_UP事件的具体走向并总结一下规律。

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtMTc4NGJlMjM2ZDIwNjBmYi5wbmc?x-oss-process=image/format,png)

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtZjFkOWVkYmMyMWU5NTVjOC5wbmc?x-oss-process=image/format,png)

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtZDUzZTNmNjgwYmExZmRkMS5wbmc?x-oss-process=image/format,png)

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtZTFlOTczOWMwODI2ZjlmMy5wbmc?x-oss-process=image/format,png)

![img](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy85NjYyODMtYzE4NDA5MTlmMWM5NTQ0Ny5wbmc?x-oss-process=image/format,png)

对于在onTouchEvent消费事件的情况：**在哪个View的onTouchEvent 返回true，那么ACTION_MOVE和ACTION_UP的事件从上往下传到这个View后就不再往下传递了，而直接传给自己的onTouchEvent 并结束本次事件传递过程。**

对于ACTION_MOVE、ACTION_UP总结：**ACTION_DOWN事件在哪个控件消费了（return true）， 那么ACTION_MOVE和ACTION_UP就会从上往下（通过dispatchTouchEvent）做事件分发往下传，就只会传到这个控件，不会继续往下传，如果ACTION_DOWN事件是在dispatchTouchEvent消费，那么事件到此为止停止传递，如果ACTION_DOWN事件是在onTouchEvent消费的，那么会把ACTION_MOVE或ACTION_UP事件传给该控件的onTouchEvent处理并结束传递。**