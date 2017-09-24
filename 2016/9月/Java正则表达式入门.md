#Java正则表达式#
---

##一、正则表达式中几种括号的用法##

* []:定义了一组字符集合即一类字符，只要目标字符匹配其中一个即匹配成功,同时还可以使用一些逻辑运算符
	* [abc]:匹配a,b,c,即待匹配的字符必须为a,b,c之一
	* [^abc]:匹配除a,b,c以外的所有字符,可以得出取反与并运算的优先级 
	* [a-zA-Z]:匹配小写字母和大写字母,可以使用**-**来表示范围
	* [a-d[m-p]]:匹配a-d以及m-p，类似[a-zA-Z],[]可以嵌套
	* [a-z&&[def]]:匹配d、e、f,交运算
	* [a-z&&[^bc]]:匹配小写字母除去b、c
	* [a-z&&[^m-p]]:匹配小写字母除去m~p

* {}：一般可用来决定同时匹配的字符数量
	* X{n}：X匹配n次，即匹配n个连续的X的字符串
	* X{n,}:X匹配至少n此,即X至少连续n次
	* X{n,m}:X匹配n~m次

* ():表示一个捕获组，一个捕获组将一组字符当作一个整体看待，每个捕获组都有一个编号，按照**(**出现的顺序进行编号,例如对于正则式 ((abb(a)c)),一共有3个组(实际上还有组0)
	* ((abb(a)c)) 捕获组1
	* (abb(a)c) 捕获组2
	* (a) 捕获组3

	```

	//一个使用的例子
	public static void main(String[] args) {
		String ps="((abb(a)c))";
		Pattern pattern=Pattern.compile(ps);
		Matcher matcher=pattern.matcher("abbac12abbac");
		System.out.println("Group Count="+matcher.groupCount());
		if(matcher.find()){//字符串是否存在匹配项
			System.out.println("Group One="+matcher.group(1));//组1匹配到的项
			System.out.println("Group two="+matcher.group(2));
			System.out.println("Group three="+matcher.group(3));
			//打印匹配的第一项
			System.out.println("Result="+"abbac12abbac".substring(matcher.start(), matcher.end()));
		}
	}

	```

	捕获组另一个值得注意的是back引用,即我们可以在正则式中引用捕获组,通过反斜杠加捕获号的方式,如例

	```

	//虽然我们的捕获组一是一个正则式，m2中的字符串在形式上符合要求，如果是"(\\d*@\\d)---\\d+\\d*@\\d"的话
	//但是文档中说到在匹配时会保存捕获组匹配到的序列，所以捕获组引用要求匹配的字符一模一样
	public static void main(String[] args) {
		String ps="(\\d*@\\d)---\\d+\\1";//我们希望匹配的形式是结尾和开头的字符是一样的
		Pattern pattern=Pattern.compile(ps);
		Matcher m1=pattern.matcher("@1---12@1");
		Matcher m2=pattern.matcher("@1---12@2");
		if(m1.find()){//字符串匹配
			System.out.println("Result="+"@1---12@1".substring(m1.start(), m1.end()));
		}
		if(m2.find()){//字符串不匹配
			System.out.println("Result="+"@1---12@2".substring(m2.start(), m2.end()));
		}
	}

	```

##二、量词以及通配符##

* . :匹配任意字符
* \d:digit匹配数字
* \D:非数字
* \s:空白字符
* \S:非空白字符
* \w:字母和数字
* \W:非\w
* *:量词表示>=0次
* ?:量词表示<=1次
* +:量词表示>=1次

对于\d之类的通配符在正则式中一般需要双反斜杠:\\d

##三、边界符##

* ^:行的开头，**注意:用在[]中表示取反**
* $:行的结尾
* \b:单词边界
* \B:非单词边界
* \A:输入的开头
* \G:上一个匹配的结尾
* \Z:输入的结尾，仅用于最后的结束符
* \z:输入的结尾

```

public static void main(String[] args) {
		String ps="[a-zA-Z]{1,}$";
//		String ps="^[a-zA-Z]{1,}";
		Pattern pattern=Pattern.compile(ps);
		String str="a123fde";
		String str2="1ae23";
		Matcher m1=pattern.matcher(str);
		Matcher m2=pattern.matcher(str2);
		
		while(m1.find()){
			System.out.println("Result="+str.substring(m1.start(), m1.end()));
		}
		while(m2.find()){
			System.out.println("Result="+str2.substring(m2.start(), m2.end()));
		}
	}

```

##四、简单的例子##

* 匹配网址:String ps="http[s]?://[^\\s]+";
* 匹配电话号码:String ps="\\d{11}";
* 匹配小数:String ps="[+-]?\\d+.\\d+";
