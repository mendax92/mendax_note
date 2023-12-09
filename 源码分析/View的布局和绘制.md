View的布局和绘制

https://www.jianshu.com/p/ee5d3bb5ab90

1.measure

测量:---------如果想改变大小:onmeasure();

​	是否是View的子类(1.View的子类  2.ViewGroup的子类)

​	 是:setMeasuredDimension()

​	 否:测量孩子  child.measure

2.layout

布局:---------onLayout

​	是否是ViewGroup的子类(View没有子控件)

​	 是:child.layout

​	 否:覆盖此方法无意义

3.draw

绘制:---------onDraw()

​	不管是View还是ViewGroup都需要绘制

​	如果是ViewGroup的子类

​	 是:child.canvas绘制(绘制儿子)

​	 否:直接canvas绘制

![image-20220717195227934](https://raw.githubusercontent.com/mendax92/pic/main/img/9zdMtjmBUoWyn35.png)

实例代码

标签控件

```
public class LineBreakLayout extends ViewGroup {

   private final static String TAG = "LineBreakLayout";
   private int VIEW_MARGIN_LEFT = 20;   //dip
   private int VIEW_MARGIN_TOP = 10;

   public LineBreakLayout(Context context) {
      this(context, null);
   }

   public LineBreakLayout(Context context, AttributeSet attrs) {
      this(context, attrs, 0);
   }

   public LineBreakLayout(Context context, AttributeSet attrs, int defStyleAttr) {
      super(context, attrs, defStyleAttr);
//    TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.LineBreakLayout);
//    VIEW_MARGIN_LEFT = ta.getDimensionPixelSize(R.styleable.LineBreakLayout_mLeft, 60);
//    VIEW_MARGIN_TOP = ta.getDimensionPixelSize(R.styleable.LineBreakLayout_mTop, 40);
   }

   @Override
   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      int widthMode = MeasureSpec.getMode(widthMeasureSpec);   //获取宽的模式
      int heightMode = MeasureSpec.getMode(heightMeasureSpec); //获取高的模式
      int widthSize = MeasureSpec.getSize(widthMeasureSpec);   //获取宽的尺寸
      int heightSize = MeasureSpec.getSize(heightMeasureSpec); //获取高的尺寸
      int row = 0;
      int shengyuW = widthSize;
      int oneHeight = 0;
      int height ;
      if (heightMode == MeasureSpec.EXACTLY) {
         height = heightSize;
      } else {
         int childCount = getChildCount();
         if(childCount<=0){
            height = 0;
         }else{
            row = 1;
            for(int i = 0;i<childCount; i++){
               View view = getChildAt(i);
               view.measure(0,0);
               int childW = view.getMeasuredWidth();
               oneHeight = view.getMeasuredHeight();
               if(shengyuW >= childW ){
                  shengyuW = shengyuW-childW;
               }else{
                  row ++;
//                L.e(TAG, "增加一行");
                  shengyuW = widthSize-childW;
               }
               shengyuW = shengyuW -VIEW_MARGIN_LEFT;
//             L.v(TAG, "childW="+childW+"   VIEW_MARGIN_LEFT="+VIEW_MARGIN_LEFT +"  剩余宽度:"+shengyuW);

            }
            height = VIEW_MARGIN_TOP * (row-1) + (row * oneHeight);
//          L.v(TAG, "height="+height+"   VIEW_MARGIN_TOP="+VIEW_MARGIN_TOP +"  row:"+row);
         }
//       L.v(TAG , "总高度:"+height);
      }
      //保存测量宽度和测量高度
      setMeasuredDimension(widthSize, height);
   }

   @Override
   protected void onLayout(boolean arg0, int arg1, int arg2, int arg3, int arg4) {
      final int count = getChildCount();
      int row = 0;
      int right = 0;
      int botom = 0;
      for (int i = 0; i < count; i++) {
         final View child = this.getChildAt(i);
         child.measure(0,0);
         int width = child.getMeasuredWidth();
         int height = child.getMeasuredHeight();
         right += width;
         botom = row * (height + VIEW_MARGIN_TOP) + height;
         if (right > (arg3- VIEW_MARGIN_LEFT)){
            right = width;
            row++;
            botom = row * (height + VIEW_MARGIN_TOP) + height;
         }
         child.layout(right - width, botom - height,right,botom);
         TextView tv = (TextView)child;
      /* L.i(TAG, "摆放"+tv.getText().toString() +
               "left:"+(right - width)+" top:"+(botom - height)
               +" right:"+right+" botom:"+botom);*/
         right += VIEW_MARGIN_LEFT;
      }
   }
}
```