## HTML与CSS拾遗
---

1、文件系统的导航

.. 符号类似与linux的..符号，可以用在文件路径中用于向上导航一级，例如:../../test.html

2、页面的锚点

a标签可以通过href属性通过页面其他元素的id属性来定位到其他元素的位置，格式为:地址#id。例如:<a href="index.html#bottom">go bottom</a>

3、CSS的使用方式

三种：内嵌、内联、通过link标签链接外部css文件,例如

```
<link rel="stylesheet" type="text/css" href="style.css">

<style type="text/css">
	body{
		background-color: #F0F
	}
</style>

<button onclick="toast()" style="color:blue;font-size:100px">Android Toast</button>
<p style="color:yellow;font-size:100px">I am g pragraph</p>

```
4、HTML 布局 与 CSS的float

流式布局从上到下、从左到右，所有的元素排队等待被被放到制定的位置，而对于float属性，该元素会从队列中被拎出来然后其他的元素回去填补它的位置，而该元素则会尽力浮动到指定的方向（但是值得注意的就是它仍然会在是在它正常应该存在的位置进行浮动）实际上其他的块元素会位于他的下面(块元素不知道它的存在)，而块元素中的内联元素则会绕过该位置。

```
//从下面这里例子可以看出来p2抢占了p1的位置，而div2却无法抢占div1的位置(由于块元素的换行,它们根本不在同一行),如果需要
//它们在同一行显示，则可以一个向右浮动，一个向左浮动
<body>
<style type="text/css">
    #div1{
        width: 100px;
        height: 100px;
        background-color: blue;
    }
    #div2{
        float: left;
        width: 100px;
        height: 100px;
        background-color: aquamarine;
    }
    #q1{
        background-color: blue;
    }
    #q2{
        float: left;
        background-color: aquamarine;
    }
</style>
<div id="div1">
    <p>I am div1,I am here</p>
</div>
<div id="div2">
    <p>I am div2,I am here</p>
</div>
<q id="q1">quote one</q>
<q id="q2">quote two</q>
</body>

```

5、CSS的position属性
	
	1、static:表示元素处于正常的流中，默认值
	2、absotion：绝对布局，该绝对指的是相对与父元素的绝对
	3、fixed:绝对布局，指的是相对与浏览器窗口的绝对，或者或是固定布局
	4、relative:相对布局，该元素仍然位于流中，但是它会相对与正常应该在的位置有一个偏移

此外对于absolution、fixed此类的固定布局，它与float相似的地方在于其对应的元素同样也是从流式布局的被单独拎出来，但是对于页面中的任何元素都不知道其存在，而不像float属性的元素，内联元素知道它的存在会绕过该元素围绕在周围。例如：

```

<body>
<style type="text/css">
    #div1{
        float: left;
        width: 300px;
        height: 400px;
        background-color: blue;
    }
    #div2{
        float: right;
        width: 300px;
        height: 400px;
        background-color: aquamarine;
    }
    #q1{
        position: absolute;
        top: 50px;
        left: 100px;
        background-color: yellow;
    }
    #q2{
        position: fixed;
        top: 90px;
        left: 100px;
        float: left;
        background-color: greenyellow;
    }
    #p1{
        position: relative;
        top: 50px;
        left: 100px;
    }
</style>
<div id="div1">
    <p id="p1">I am div1,I am here</p>
    <q id="q1">quote one</q>
</div>
<div id="div2">
    <p id="p2">I am div2,I am here</p>
    <q id="q2">quote two</q>
</div>
</body>

```

6、CSS的选择器：

	1、标签选择器:直接选择HTML标签
	2、类选择器:**.**+class名
	3、id选择器:#+id名
	4、子选择器:使用**>**符号连接选择器(第一层)
	5、后代选择器:使用空格连接选择器
	6、通用选择器: * 
	7、伪类选择器:例如 a:hover
	8、可用使用,分隔同时链接多个选择器，从而设置共同样式