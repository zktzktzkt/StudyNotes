#Android中Drawable的使用及自定义Drawable#

##一、Drawable概述##

###1、官方文档中关于drawable的描述###

Android提供了一个定制化的用于绘制形状、图像，这些类位于android.graphics.drawable包下用于绘制2D图形。

Drawable是对于那种可绘制的东西的抽象。

一共有三种方法来定义或者初始化Drawable:使用工程中的图像、使用定义了Drawable属性的XML、使用构造函数

最后文档中提到关于.9图的黑线的意思，左边及上面的黑线表示可拉伸的区域,附带一张官网的图

![](https://github.com/getletCodes/StudyNotes/blob/master/part2/ninepatch_raw.png)

###2、部分子类的作用###

* AnimatedVectorDrawable:
* BitmapDrawable:一个包装了bitmap的drawable,图像可以是平铺、拉伸、对齐的，在xml中可以使用bitmap节点来创建
* ColorDrawable:一种是用特定颜色填充画布的Drawable，该Drawable忽略ColorFilter,在xml中可以使用color节点来创建
* DrawableContainer:如其名，一个包含多个Drawable而且可以选择显示的帮助类，不过该类本身不能直接用，应该继承它或者直接使用它的子类
* GradientDrawable:一种颜色渐变的Drawable,在xml中可以使用shape节点来创建
* LayerDrawable:一个管理Drawable数组的Drawable，它仅仅按照数组顺序绘制Drawable，在xml中使用layer-list节点创建，子item标签使用item
* NinePatchDrawable:使用.9图的可自适应的Drawable
* PictureDrawable:一个包装了Picture的Drawable
* RoundedBitmapDrawable(v4):一个包装了bitmap且将其绘制成圆角矩形的Drawable
* ShapeDrawable:一个可以绘制基本形状的Drawable,该类持有一个shape对象并控制其在屏幕上的显示，在xml中使用shape节点创建
* VectorDrawable:该类允许用户基于xml创建矢量图形，在xml中使用vector标签创建
* AnimationDrawable:帧动画相关
* ClipDrawable：一种根据其上含有的Drawable的level来进行显示的Drawable
* InsetDrawable：嵌入一个drawable,典型的使用情况是需要一个drawable作为背景，但是不要填满View
* LevelListDrawable:根据level值来显示drawable,每个子item都需要一个范围
* PaintDrawable：也算是一个绘制圆角矩形的办法吧
* RippleDrawable：API21以上，水波纹效果
* RotateDrawable：旋转其内容，补间动画中常用
* ScaleDrawable:缩放其内容，与level相关
* StateListDrawable：根据View的状态显示不同的drawable，注意匹配模式即可
* TransitionDrawable:允许子两个drawable之间淡入淡出的drawable，仅支持两个之间，使用start、revert方法进行淡入淡出

##二、各个子类的简单使用##

首先先从Drawable的部分set方法来看公共的基本属性

1、BitmapDrawable
	android:tileMode启用时android:gravity属性被忽略，另外alpha之从0~255
2、ColorDrawable
	只有一个属性Color值，纯色
3、GradientDrawable
	android:shape属性(定义形状)：rectangle(填满View的矩形)、oval(适应View尺寸的椭圆)、line(等于View的宽度的直线，需要搭配stroke属性)、ring(圆形)

	只与ring标签使用的属性

		android:innerRadius：内圆的半径，ring默认由内圆和外圆组成

		android:innerRadiusRatio：内圆半径=圆的宽度/值
		
		android:thickness：圆环的厚度
		
		android:thicknessRatio：圆环的厚度=圆的宽度/值
	
	最后ring需要定义stroke属性定义绘制的宽度与颜色

	corner属性：只对shape为rectangle有效，其下的属性就是指各个直角的弯曲度

	gradient属性:设置形状的渐变色

		android:angle:颜色渐变的方向,0表示从左到右，90表示从下到上，必须是45的倍数

		android:centerX、android:centerY：渐变中心的相对坐标，从0到1

		android:startColor、android:centerColor、android:endColor：三个过渡色

		android:gradientRadius：渐变半径，仅当，android:type=radial时生效

		android:type：渐变样式，linear(线性渐变)、radial(辐射式，由中心)、sweep(扫描式)

	padding属性：与想象中不同的时它是作用于View，就像View的padding一样，并不是设置shape的padding

	size属性：设置大小，在View的布局未warp_content时生效，其他情况拉伸以适合于View

	solid属性：填充shape的颜色，与gradient存在冲突

	stroke属性：与绘制的线条相关

		android:dashGap、android:dashWidth:一起使用才有效，虚线之间的距离与一条实线的长度

4、ShapeDrawable
	标签意义同上
	
	从上面及文档中看二者都可以使用shape标签在xml中创建，那么shape节点到底是指的哪一个呢？在Drawable中有一个这样的方法，也是View的background标签初始化使用的方法
	```
		public static Drawable createFromXmlInner(Resources r, XmlPullParser parser, AttributeSet attrs,
	            Theme theme) throws XmlPullParserException, IOException {
	        final Drawable drawable;

	        final String name = parser.getName();
	        switch (name) {
	            case "selector":
	                drawable = new StateListDrawable();
	                break;
	            case "animated-selector":
	                drawable = new AnimatedStateListDrawable();
	                break;
	            case "level-list":
	                drawable = new LevelListDrawable();
	                break;
	            case "layer-list":
	                drawable = new LayerDrawable();
	                break;
	            case "transition":
	                drawable = new TransitionDrawable();
	                break;
	            case "ripple":
	                drawable = new RippleDrawable();
	                break;
	            case "color":
	                drawable = new ColorDrawable();
	                break;
	            case "shape"://注意，这里只创建GradientDrawable，没有根据情况来区分是Gradient、Shape
	                drawable = new GradientDrawable();
	                break;
	            case "vector":
	                drawable = new VectorDrawable();
	                break;
	            case "animated-vector":
	                drawable = new AnimatedVectorDrawable();
	                break;
	            case "scale":
	                drawable = new ScaleDrawable();
	                break;
	            case "clip":
	                drawable = new ClipDrawable();
	                break;
	            case "rotate":
	                drawable = new RotateDrawable();
	                break;
	            case "animated-rotate":
	                drawable = new AnimatedRotateDrawable();
	                break;
	            case "animation-list":
	                drawable = new AnimationDrawable();
	                break;
	            case "inset":
	                drawable = new InsetDrawable();
	                break;
	            case "bitmap":
	                drawable = new BitmapDrawable();
	                break;
	            case "nine-patch":
	                drawable = new NinePatchDrawable();
	                break;
	            default:
	                throw new XmlPullParserException(parser.getPositionDescription() +
	                        ": invalid drawable tag " + name);

	        }
	        drawable.inflate(r, parser, attrs, theme);
	        return drawable;
	    }

	```
	可以看来android是根据根标签来创建相应的Drawable，而上面对shape标签只会创建GradientDrawable，而不是根据情况创建ShapeDrawable或者只是创建ShapeDrawable，但是文档中只对ShapeDrawable进行了xml创建的说明与解释，不明觉厉。

5、LayerDrawable

	每个item属性包含一个drawable以及各自的padding

6、NinePatchDrawable

	直接使用.9图即可或者使用src属性引用

7、RoundedBitmapDrawable

	快速生成圆角矩形图片、圆形图片的一种方法，可以使用RoundedBitmapDrawableFactory来创建

8、AnimatedVectorDrawable、VectorDrawable
	
	矢量图相关的，并且有相应的compat兼容类，以后在svg用到再说

9、ClipDrawable:level从0~10000,从显示到不显示，在某种程度上可以当作进度条使用

	android:clipOrientation:裁剪方向或者说显示方向

##三、如何自定义Drawable##

首先在View的draw方法中有这样的流程说明

>
 1. Draw the background
 2. If necessary, save the canvas' layers to prepare for fading
 3. Draw view's content
 4. Draw children
 5. If necessary, draw the fading edges and restore layers
 6. Draw decorations (scrollbars for instance)

在源码中可以看到一个View的绘制的canvas为同一个canvas,而且在该方法中调用了drawBackground方法从而调用了Drawable的draw方法，而自定义Drawable需要复写的方法的关键也就在这里了，所以Drawable的自定义和View的自定义有点像，另外关于getIntrinsicWidth、height的实现，如果只需要和View一样大可以根据getBounds参数来做(View自己将尺寸信息设置了，ColorDrawable就是这么来绘制纯色背景的)或者自定义

自定义一个简单圆角的Drawable代码如下

```
class RoundBitmapDrawable extends Drawable{

        private Paint mPaint;
        private Bitmap mBitmap;
        public RoundBitmapDrawable(Resources res,int id){
            mBitmap=BitmapFactory.decodeResource(res,id);
            mPaint=new Paint(Paint.ANTI_ALIAS_FLAG);
        }

        @Override
        public void draw(Canvas canvas) {
            mPaint.setXfermode(null);
            int layerId=canvas.saveLayer(0,0,getIntrinsicWidth(),getIntrinsicHeight(),null,Canvas.ALL_SAVE_FLAG);
            int radius=Math.min(getIntrinsicWidth(),getIntrinsicHeight())/2;
            int cx=getIntrinsicWidth()/2;
            int cy=getIntrinsicHeight()/2;
            canvas.drawCircle(cx,cy,radius,mPaint);
            mPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
            //尺寸的设置只是为了看起来效果明显一点
            canvas.drawBitmap(mBitmap,new Rect(0,0,mBitmap.getWidth(),mBitmap.getHeight())
            ,new Rect(cx-radius-30,cy-radius-30,cx+radius+30,cy+radius+30),mPaint);
            canvas.restoreToCount(layerId);

        }

        @Override
        public void setAlpha(int alpha) {

        }

        @Override
        public int getIntrinsicHeight() {
            return getBounds().height();
        }

        @Override
        public int getIntrinsicWidth() {
            return getBounds().width();
        }

        @Override
        public void setColorFilter(ColorFilter colorFilter) {

        }

        @Override
        public int getOpacity() {
            return 0;
        }
    }

```

效果图如下

![](https://github.com/getletCodes/StudyNotes/blob/master/part2/round_circle_drawable.png)
    
