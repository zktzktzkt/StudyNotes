##Android字符的尺寸的详解##
---
#一、Canvas绘制字符的基准点在哪#

一般来说当我们需要绘制字符的时候，我们都会调用Canvas的drawText系列的重载方法来绘制字符，方法声明是这样的

```
public void drawText (String text, int start, int end, float x, float y, Paint paint) 
```
从方法声明来看，与其他绘制方法没什么差别，都是数据、位置、画笔，而且一般来说x,y都是绘制的左上角位置，但是这里的x,y却是baseline的位置，即绘制的基准点是一个字符的baseline，文档如下

>Parameters
text	The text to be drawn
x	The x-coordinate of the origin of the text being drawn
y	The y-coordinate of the baseline of the text being drawn
paint	The paint used for the text (e.g. color, size, style)

#二、关于测量字符时的几个字段的解释#
先上一张图

![](https://github.com/getletCodes/AndroidNotes/blob/master/part1/font_metrics.png)

然后从绘制看起，先用这样的代码绘制一段text

```
	@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        String mText1="安卓Android jokes";
        canvas.drawText(mText1,0,0,mTextPaint);
    }
```
结果就是模糊的看到只有一点点被绘制出来，就是baseline以下的部分

接下来再看Paint类下测量字符串的几个API获取的字段的值

```
@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        String mText1="安卓Android jokes";
        float tw1=mTextPaint.measureText(mText1);//返回文字所占的宽度
        float tascent1=mTextPaint.ascent();
        float tdescent1=mTextPaint.descent();
        float tspace1=mTextPaint.getFontSpacing();//根据当前的字体和字号返回所占的高度
        Rect tBounds1=new Rect();
        mTextPaint.getTextBounds(mText1,0,mText1.length(),tBounds1);//获取字符串的包围盒，文档指出是恰好能够包围字符串的大小
        float twidths[]=new float[mText1.length()];
        mTextPaint.getTextWidths(mText1,twidths);//返回每个字符所占的宽度，其和等于mTextPaint.measureText(mText1);
        log("宽度",tw1);
        log("ascent",tascent1);
        log("descent",tdescent1);
        log("space",tspace1);
        log("bounds",tBounds1);
        String text="";
        float sum=0;
        for(float i:twidths){
            text+=i+" ";
            sum+=i;
        }
        log("characters' widths",text);
        log("widths sum",sum);
        Paint.FontMetrics fm=mTextPaint.getFontMetrics();
        log("fm ascent",fm.ascent);
        log("fm bottom",fm.bottom);
        log("fm descent",fm.descent);
        log("fm leading",fm.leading);
        log("fm top",fm.top);
    }
```
log的截图如下,Paint的ascent、descent只是一个快捷方法和FontMetrics中的值相同。然后根据图示的来实验一下各个参数是否是正确的意思

![](https://github.com/getletCodes/AndroidNotes/blob/master/part1/font_draw_log.png)

```
	//修改一下y坐标为ascent值，果然文字就全部绘制出来了
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        String mText1="安卓Android jokes";
        canvas.drawText(mText1,0,-mTextPaint.ascent(),mTextPaint);
    }
```

```
	//修改一下y坐标为View的高度，果然文字的baseline一下的部分被遮住了
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        String mText1="安卓Android jokes";
        canvas.drawText(mText1,0,getHeight(),mTextPaint);
    }
```
bottom和top字段的意思也可如此实验，最后leading是推荐的两行文字之间的间距。

#三、一个绘制多行本的函数的实现#

```
	public static void drawTextUtil(Canvas canvas,Paint paint,int viewWidth,String text){
        int cloumns=0;
        int i=0;
        float sum=0;
        float lineHeight=paint.getFontSpacing();//一行文本的高度
        float baseline=-paint.getFontMetrics().top;//基线的位置
        int start=0;
        while(i<text.length()){
            float width=paint.measureText(text,i,i+1);
            if(sum+width<viewWidth){
                sum+=width;
                i++;
            }
            else{
                canvas.drawText(text,start,i,0,baseline+lineHeight*cloumns,paint);
                sum=0;
                start=i;
                cloumns++;
            }
        }
        if(sum!=0){
            canvas.drawText(text,start,i-1,0,baseline+lineHeight*cloumns,paint);
        }
    }

```
效果如下，其中蓝色是自己绘制的，黑色为TextView绘制的，通过对照可以看出来，完全一致，并且重合了两行

![](https://github.com/getletCodes/AndroidNotes/blob/master/part1/font_draw_demo.png)

#四、总结#
总的来说，只要抓住文本的绘制是依照baseline这个参数来的，那么问题就基本解决了