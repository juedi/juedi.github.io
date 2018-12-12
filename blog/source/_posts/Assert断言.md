---
title: Assert断言
date: 2014-12-21 16:22:29
categories: java
tags: 
- Assertion
- Exception
---
最近，遇到一个计算生态补偿金额的业务，具体的计算公式是一个比较麻烦的东西，里面各种数据符号，什么Σ、Φ、α、β之类的玩意，对于我来说，好久不见这些东西，符号该怎么读都差不多忘记了，很是苦恼。最后终于跟客户弄清楚了，公式的意义，就开始编码计算。

公式的数据来源是数据库中查出来的，但数据库中的数据并非就是完整的、干净的，就是说我再处理α*β这类玩意的时候很有可能从数据库中查出来的是一个非数字字符串，或者null数据。那么问题来了，这样子去计算公式程序必然会报各种异常，然后跟客户反应后客户提出要求：运行公式前，先检查公式需要数据的完整性，如果公式需要的数据不符合要求，就在页面上给客户提示，哪些数据有问题。

## 条件判断方式
将数据从数据库中查出，依次按照各个数据的校验条件进行校验，使用返回字符串的形式返回校验结果。

```
	public String validate(){
	String errorInfo="";
	//dataA from database
	if(dataA == null){//或者其他条件要求，例如dataA必须为数字，数字范围在[1,20]等
		errorInfo = "dataA 数据有误";
		return;
	}
	//dataB from database
	if(dataB == null){//或者其他条件要求，例如dataA必须为数字，数字范围在[1,20]等
		errorInfo = "dataB 数据有误";
		return;
	}
	...
	//校验完毕，开始执行公式计算
	}
```

这种方式就会有大量的检查代码，重复的逻辑判断*a==null*,*a is number*等，使代码看起来冗长切不易理解，校验代码和执行代码实际上就可以是两个不同的模块。
## ibatis方式
最近在看ibatis源代码时看到它在处理异常信息时使用了一种非常有趣的方式。首先，有一个保存错误信息的类ErrorContext:

```

	public class ErrorContext {

 	 private String resource;
 	 private String activity;
 	 private String objectId;
 	 private String moreInfo;
 	 private Throwable cause;
	 //setter
	 //getter
 	 public String toString() {
   	 StringBuffer message = new StringBuffer();
	    // resource
	    if (resource != null) {
	      message.append("  \n--- The error occurred in ");
	      message.append(resource);
	      message.append(".");
	    }	
	    // activity
	    if (activity != null) {
	      message.append("  \n--- The error occurred while ");
	      message.append(activity);
	      message.append(".");
	    }
	    // object
	    if (objectId != null) {
	      message.append("  \n--- Check the ");
	      message.append(objectId);
	      message.append(".");
	    }
	    // more info
	    if (moreInfo != null) {
	      message.append("  \n--- ");
	      message.append(moreInfo);
	    }
	    // cause
	    if (cause != null) {
	      message.append("  \n--- Cause: ");
	      message.append(cause.toString());
	    }
	    return message.toString();
	  }
	  public void reset() {
	    resource = null;
	    activity = null;
	    objectId = null;
	    moreInfo = null;
	    cause = null;
	  }	
	}
```

然后在程序中使用时，直接在执行每一步关键代码的前面set校验信息，如果程序的下一步出现异常，那就跳到catch块中，将具体的异常信息保存到ErrorContext对象中，最终在处理异常时，就可以将详细的异常信息展示出来：

```
	protected void executeQueryWithCallback(StatementScope statementScope, Connection conn, Object parameterObject, Object resultObject, RowHandler rowHandler, int skipResults, int maxResults)
      throws SQLException {
	//开始设置异常信息
    ErrorContext errorContext = statementScope.getErrorContext();
    errorContext.setActivity("preparing the mapped statement for execution");
    errorContext.setObjectId(this.getId());
    errorContext.setResource(this.getResource());

    try {
      parameterObject = validateParameter(parameterObject);

      Sql sql = getSql();
	  //执行代码前设置异常信息
      errorContext.setMoreInfo("Check the parameter map.");
      ParameterMap parameterMap = sql.getParameterMap(statementScope, parameterObject);

      errorContext.setMoreInfo("Check the result map.");
      ResultMap resultMap = sql.getResultMap(statementScope, parameterObject);

      statementScope.setResultMap(resultMap);
      statementScope.setParameterMap(parameterMap);

      errorContext.setMoreInfo("Check the parameter map.");
      Object[] parameters = parameterMap.getParameterObjectValues(statementScope, parameterObject);

      errorContext.setMoreInfo("Check the SQL statement.");
      String sqlString = sql.getSql(statementScope, parameterObject);

      errorContext.setActivity("executing mapped statement");
      errorContext.setMoreInfo("Check the SQL statement or the result map.");
      RowHandlerCallback callback = new RowHandlerCallback(resultMap, resultObject, rowHandler);
      sqlExecuteQuery(statementScope, conn, sqlString, parameters, skipResults, maxResults, callback);

      errorContext.setMoreInfo("Check the output parameters.");
      if (parameterObject != null) {
        postProcessParameterObject(statementScope, parameterObject, parameters);
      }

      errorContext.reset();
      sql.cleanup(statementScope);
      notifyListeners();
    } catch (SQLException e) {
	  //异常信息保存近errorContext对象中
      errorContext.setCause(e);
      throw new NestedSQLException(errorContext.toString(), e.getSQLState(), e.getErrorCode(), e);
    } catch (Exception e) {
      errorContext.setCause(e);
      throw new NestedSQLException(errorContext.toString(), e);
    }
	}

```

*这种方式的好处就是不必挨个的去使用if语句判断，只需要在合适的地方加上错误信息的保存，程序的正常执行逻辑不变，当正常执行逻辑出现异常时，自然就会中断，然后此时保存的异常信息就是最近的信息。
当然，这种方式处理较为复杂的条件判断是就会比较麻烦，比如:a的数字范围是[10,20]，如果a的数值为30，带入到公式中，公式也是可以计算的，不会出现异常。*
## spring方式

使用过Spring的同学想必就见过这样的代码:

```
	public void addCallback(ListenableFutureCallback<? super T> callback) {
		Assert.notNull(callback, "'callback' must not be null");
		synchronized (mutex) {
			switch (state) {
				case NEW:
					callbacks.add(callback);
					break;
				case SUCCESS:
					callback.onSuccess((T)result);
					break;
				case FAILURE:
					callback.onFailure((Throwable) result);
					break;
			}
		}
	}

```

Spring中的很多操作的一开始都有这么一个断言Assert.notNull()，那么这个notNull()方法是什么呢？

```
	public static void notNull(Object object, String message) {
		if (object == null) {
			throw new IllegalArgumentException(message);
		}
	}

	public static void notNull(Object object) {
		notNull(object, "[Assertion failed] - this argument is required; it must not be null");
	}
```

可以看出，如果断言失败，那么程序就会抛出异常，这是非常严格的校验方式了。看到这里，有人也行会问，这根我们的校验有什么关系，我们是要返回具体的校验信息的。到这里，我们其实可以看出，假如我们想要返回具体的校验信息，只需要综合ibatis和spring的异常处理方式，在Assert之前把错误信息set近Context中就行了:

```
	public String validate(){
	try{
		//dataA from database
		errorContext.setErrorInfo("数据A不能为空");
		Assert.notNull(dataA);
		errorContext.setErrorInfo("数据范围为[10,20]");
		Assert.range(10,20);
	}catch(Exception e){
		//返回异常信息
		return errorContext.toString();
	}
	...
	//校验完毕，开始执行公式计算
	}
```

这样，既避免的代码重复，逻辑混乱，也可以更好的返回详细信息。
