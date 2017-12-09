title: Exception to String
date: 2015-01-27 20:42:20
categories: [Java]
tags: Exception

---
### 需求概要
最近设计平台的异常信息处理功能，由于平台前台使用的是Flex，所以没有像传统的Java Web项目那样搞一些500、404之类的跳转页面，而是准备设计成类似于桌面应用那样的弹出框，里面显示一些简单的错误提示信息，然后点击“查看详细>>”按钮展开下面的详细异常堆栈信息。然而这样就需要获取后台异常的详细堆栈信息，以前就知道异常的堆栈信息搞一个*e.printStackTrace()*就显示在控制台中了，或者用log4j的logger.error("msg",e)也就显示在日志中了，还真没想过怎么把异常的堆栈信息搞到字符串中。
### 解决方法
那好，既然需求确定了，那就来看看Throwable类中几个获取堆栈信息的方法
```
e.printStackTrace();
printStackTrace(PrintStream s);
printStackTrace(PrintWriter s);
```
第一个方法不用多说，IDE通常自动生成的，在控制台打印的堆栈信息，看源代码的话会发现，这个方法中其实调用的还是第二个方法，只不过参数传入的系统控制台的err打印输出流，所以这玩意一般在IDE里面看是一片飘红
``` Java
public void printStackTrace() {
    printStackTrace(System.err);
}
```
也就是说，我们可以忽略第一方法，直接看第二个和第三个
``` Java
public void printStackTrace(PrintStream s) {
    printStackTrace(new WrappedPrintStream(s));
}
public void printStackTrace(PrintWriter s) {
    printStackTrace(new WrappedPrintWriter(s));
}
```
那无非就是一个是字节流，一个是字符流，而且他们最终调用的还是同一个方法
``` Java
private void printStackTrace(PrintStreamOrWriter s);
```
好，到此我们就不向下深究了，那接下来的问题就是，怎么把输出流转换成字符串呢？
好了，不买关子，直接看代码吧，一目了然
``` Java
public static String getStackTrace1(Throwable e){
	ByteArrayOutputStream os = new ByteArrayOutputStream();
	PrintStream ps = new PrintStream(os, true);
	e.printStackTrace(ps);
	return os.toString();
}

public static String getStackTrace2(Throwable e){
	StringWriter sw = new StringWriter();
	PrintWriter pw = new PrintWriter(sw, true);
	e.printStackTrace(pw);
	return sw.toString();
}
```
由此，我们就得到了我们想要的异常堆栈信息。
当然，我们还有其他的方式，比如：
``` Java
public static String getStackTrace3(Throwable e){
	StringBuilder msg = new StringBuilder(e.toString());
	msg.append("\n");
	StackTraceElement[] elements = e.getStackTrace();
	for(StackTraceElement element : elements){
		msg.append("\t")
		.append(element.getClassName())
		.append(".")
		.append(element.getMethodName())
		.append("(")
		.append(element.getFileName())
		.append(":")
		.append(element.getLineNumber())
		.append(")")
		.append("\n");
	}
	return msg.toString();
}
```
只是这种方式想必于前面两种还有些不同，因为这种方式获取的异常堆栈信息只是当前异常的，而不能打印到根异常信息去，如果想得到更多的异常信息，难免要做递归的调用去处理，这样，就不如直接使用前面两个更为方便的方法。
### 代码示例
下面为源码例子，可以对这三种方式做下对比看看：
``` Java
package lang.exceptions;

import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.io.PrintWriter;
import java.io.StringWriter;

public class ExceptionUtilTest {

	public static void main(String[] args) {
		try{
			divide(1, 0);
		}catch(Exception e){
			e.printStackTrace(new PrintStream(new ByteArrayOutputStream()));
			System.out.println(getStackTrace1(e));
			System.out.println(getStackTrace2(e));
			System.out.println(getStackTrace3(e));
		}
	}
	
	public static int divide(int a, int b){
		int c = 0;
		//去掉注释，可以查看三种方式输出的差异
//		try{
			c = a / b;
//		}catch(Exception e){
//			throw new RuntimeException("除法异常", e);
//		}
		return c;
	}
	
	public static String getStackTrace1(Throwable e){
		ByteArrayOutputStream os = new ByteArrayOutputStream();
		PrintStream ps = new PrintStream(os, true);
		e.printStackTrace(ps);
		return os.toString();
	}
	
	public static String getStackTrace2(Throwable e){
		StringWriter sw = new StringWriter();
		PrintWriter pw = new PrintWriter(sw, true);
		e.printStackTrace(pw);
		return sw.toString();
	}
		
	public static String getStackTrace3(Throwable e){
		StringBuilder msg = new StringBuilder(e.toString());
		msg.append("\n");
		StackTraceElement[] elements = e.getStackTrace();
		for(StackTraceElement element : elements){
			msg.append("\t")
			.append(element.getClassName())
			.append(".")
			.append(element.getMethodName())
			.append("(")
			.append(element.getFileName())
			.append(":")
			.append(element.getLineNumber())
			.append(")")
			.append("\n");
		}
		return msg.toString();
	}
}

```