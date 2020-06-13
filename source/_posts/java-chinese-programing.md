---
title: Java练习-中文编程
categories: Java练习
date: 2020-02-05 21:47:28
---
## 需求
请编写一个程序，制作一个简易的中文语言编译器，即使用中文语法进行编程，输入为逐行输入，每行为一个语句，一个语句代变一个操作，满足以下语法要求（括号内代变格式类型，具体参考样例）：

- 变量定义：整数 （变量名） 等于 （数字）
- 运算（加法）：（变量名） 增加 （数字）
- 运算（减法）：（变量名） 减少 （数字）
- 输出：看看 （变量名） or 看看 “（字符串内容）”

附加要求：
选择：如果 （判断语句） 则 （操作语句） 否则 （操作语句）
若否则后没有任何操作使用（无）来进行填充（参考样例2）。

## 测试样例
输入
```
整数 气温 等于 十
气温 减少 三
气温 增加 二
看看 气温
如果 气温 大于 八 则 看看 “你好，世界” 否则 看看 “冻死我了”
```
输出
```
九
你好，世界
```

输入
```
整数 小杨年龄 等于 八
整数 小杨零花钱 等于 二
小杨年龄 增加 一
如果 小杨年龄 大于 八 则 小杨零花钱 增加 一 否则 无
看看 小杨零花钱
```
输出
```
三
```


## 分析
需求分析
- 整数变量定义、赋值
- 加减运算
- 输出变量或字符串
- 三目运算：如果 则 否则

需要解决的问题
- 字符串截取
- 字符串语义判断
- 异常处理
- 负数，多位数

## 遇到的一些坑
- 字符串split的时候左右边的空格会被加入到split返回的数组

## 功能实现情况
- 数字输入输出（支持中文习惯和机械式的转化）
  - 符合中国人习惯的输入输出，支持-999到999（例如：九百九十九、九百一、九百一十，九百零一，九百，九十，十九，十，零，负九百九十九）
  - 机械式的转化（例如：一零零）支持的范围：-2147483647到2147483647
  - 传奇地加入了负零(效用等同于零)

- 支持去除语句中多余的空格

- 变量定义：整数 钱包 等于 零
  - 支持定义多个变量

- 运算操作：钱包 增加 十
  - 支持增加、减少、乘以、除以、模除

- 输出：看看 钱包  、 看看 “字符串”
  - 支持输出字符串或者变量

- 三目运算符：如果 钱包 大于 十 则 看看 “钱太多了” 否则 看看 “我穷死了”
  - 条件判断两边都可以是整数变量或者整数常量

- 异常处理
  - 关键字不存在
  - 运算时变量未定义
  - 输出时变量未定义
  - 除数不能为0
  - 非法输入


## 完整代码
GitHub仓库地址：[learn-programming](https://github.com/aaronlinv/learn-programming)

主函数
```java
public class DemoMain {

	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		while (true) {
			Utils.runMain(sc.nextLine());
		}
	}

}

```


Utils类
```java

public class Utils {
	private static String numWords = "零一二三四五六七八九十";
	private static Map<String, Integer> map = new HashMap<String, Integer>();

	/**
	 * 运行语句并处理异常
	 * @param str
	 */
	public static void runMain(String str) {
		try {
			Utils.callFunction(str);
		} catch (Exception e) {
			System.out.println(e.getMessage());
		}
	}

	/**
	 * 调用方法
	 * @param str
	 */
	public static void callFunction(String str) {
		String[] split = str.trim().split("\\s+");
		String keyword = split[0];
		if (keyword.equals("无")) {
			return;
		}
		if (isManipulate(str) || isVar(keyword)) {
			Utils.manipulateNum(str);
		} else {
			// isManipulate 已经判断了len == 1情况 不会越界
			switch (keyword) {
			case "整数":
				// System.out.println(map.get("钱包"));// 不存在返回null
				Utils.assignInt(str);
				break;
			case "看看":
				Utils.printOut(str);
				break;
			case "如果":
				Utils.ternaryOperator(str);
				break;
			default:
				throw new IllegalArgumentException("没有关键字：" + keyword + " 请使用关键字：整数、看看、如果");
			}
		}
	}

	/**
	 *  赋值整数变量
	 * @param array
	 * @return
	 */
	public static void assignInt(String str) {
		String[] strArr = str.trim().split("\\s+");
		// 短路 所以不会越界
		if (strArr.length != 4 || !strArr[2].equals("等于")) {
			throw new DemoException("语法有错，请检查语法");
		}

		setVar(strArr[1], toNum(strArr[3]));
	}

	/**
	 * 输出字符串或者变量
	 * @param array
	 */
	public static void printOut(String str) {
		String[] strArr = str.trim().split("\\s+");

		// 输出变量
		if (strArr.length == 2) {
			String varStr = strArr[1];
			if (isVar(varStr)) {
				System.out.println(toChStr(getVar(varStr)));
				return;
			} else if (!varStr.contains("“") && !varStr.contains("”")) {
				throw new DemoException("变量：" + varStr + " 未定义，请定义变量");
			}
			
		}
		// 去除字符串中的“看看”
		String newStr = str.trim().substring(2).trim();		
	
		if (newStr.indexOf("“") == 0 && newStr.lastIndexOf("”") == (newStr.length() - 1)) {
			System.out.println(newStr.substring(1, ( newStr.length()-1)  ));
		} else {
			throw new DemoException("语法有错，请检查语法");
		}

//		if (strArr.length != 2) {
//			throw new DemoException("语法有错，请检查语法");
//		}
//		String printStr = strArr[1];
//		if (printStr.contains("“") && printStr.contains("”")) {
//			System.out.println(printStr.replace("“", "").replace("”", "")); // 看看 “字符串”
//		} else {
//			System.out.println(toChStr(getVar(printStr)));
//		}
	}


	/**
	 * 三目运算
	 * @param str
	 */
	public static void ternaryOperator(String str) {
		// 如果 钱包 大于 十 则 看看 “钱太多了” 否则 看看 “我穷死了”
		// 先不考虑三目嵌套三目的情况
		String statement1 = null;
		String statement2 = null;
		String statement3 = null;
		try {
			statement1 = str.substring(0, str.indexOf("则")).replace("如果", "");
			statement2 = str.substring(str.indexOf("则"), str.indexOf("否则")).replace("则", "");
			statement3 = str.substring(str.indexOf("否则")).replace("否则", "");
		} catch (Exception e) {
			throw new DemoException("语法有错，请检查语法");
		}

		boolean judge = judgeOperator(statement1);
		// System.out.println(judge);

		if (judge) {
			callFunction(statement2);
		} else {
			callFunction(statement3);
		}
	}

	/**
	 * 判断语句，传入如果语句 例如：零 等于 零
	 * @param str
	 * @return 
	 */
	public static boolean judgeOperator(String str) {
		String[] strArr = str.trim().split("\\s+");// 不去除左右空格，空格会被加入到分割数组
		if (strArr.length != 3) {
			throw new DemoException("语法有错，请检查语法");
		}
		String leftStr = strArr[0];
		String rightStr = strArr[2];
		String middle = strArr[1];
		int leftInt = 0;
		int rightInt = 0;

		// 如果都不是变量，那么toNum来异常处理
		if (isVar(leftStr)) {
			leftInt = getVar(leftStr);
		} else {
			try {
				leftInt = toNum(leftStr);
			} catch (Exception e) {
				throw new DemoException("变量：" + leftStr + " 未定义，请定义变量");
			}
		}

		if (isVar(rightStr)) {
			rightInt = getVar(rightStr);
		} else {
			try {
				rightInt = toNum(rightStr);
			} catch (Exception e) {
				throw new DemoException("变量：" + rightStr + " 未定义，请定义变量");
			}

		}

		switch (middle) {
		case "等于":
			return leftInt == rightInt;
		case "大于":
			return leftInt > rightInt;
		case "小于":
			return leftInt < rightInt;

		default:
			throw new IllegalArgumentException("没有关键字：" + middle + " 请使用关键字：大于、等于、小于");
		}
	}

	/**
	 * 汉字单个转化为单个数字
	 * @param str
	 * @return
	 */

	public static int toSingleNum(String str) {
		int num = numWords.indexOf(str);
		if (num == -1) {
			throw new DemoException("语法有错，请检查语法，字符转化错误");
		}
		return num;// -1不存在
	}

	/**
	 * 单个数字转化为单个汉字
	 * @param num
	 * @return
	 */
	public static String toSingleChStr(int num) {
		if (num < 0 || num > 10) {
			throw new DemoException("语法有错，请检查语法，字符转化错误");
		}
		return numWords.substring(num, num + 1);
	}

	/**
	 * 汉字转化为数字
	 * @return
	 */
	public static int toNum(String str) {
		int flag = 1;// 负数标记
		int var = 0;
		char[] arr = str.toCharArray();
		if (arr[0] == '负') {
			str = str.replace("负", "");

			flag = -1;
		}
		if (str.length() == 1 &&str.equals("百")) {
			return flag*100;
		}
		// 数字机械式的转化
		if (str.length() > 1 && !str.contains("百") && !str.contains("十")) {
			char[] chArr = str.toCharArray();
			int temp = 1;
			for (int i = chArr.length - 1; i >= 0; i--) {
				String s = "" + chArr[i];
				var += temp * toSingleNum(s);
				temp *= 10;
			}
			return var * flag;
		}

		if (str.contains("百")) {
			var = 100 * toSingleNum(str.substring(0, str.indexOf('百')));// 百位
			str = str.substring(str.indexOf('百') + 1);
			if (str.contains("零")) {
				var += toSingleNum(str.substring(str.indexOf("零") + 1));// 几百零几
				return var * flag;
			}
		}

		if (str.length() == 1 && var >= 100) { // 判断var 不然十 可能会输出100
			var += 10 * toSingleNum(str); // 几百几：九百九
			return var * flag;
		}

		if (str.contains("十")) {
			if (str.indexOf('十') == 0) {
				var += 10; // 十几
			}
			var += 10 * toSingleNum(str.substring(0, str.indexOf('十')));// 十
			str = str.substring(str.indexOf('十') + 1);
		}

		if (str.length() != 0) {
			var += toSingleNum(str);
		}

		return var * flag;
	}

	@Test
	public void test2() {
		// Java int Max ：2147483647 二一四七四八三六四七 Min：-2147483648
		// System.out.println(toNum("负二一四七四八三六四八"));
		System.out.println(toNum("负九九九"));
	}

	/**
	 * 数字转化为汉字
	 * @param num
	 * @return
	 */
	public static String toChStr(int num) {

		if (num == -2147483648) {
			throw new DemoException("超出支持的输入输出范围(-2147483647到2147483647)");
		}
		String varStr = "";

		if (num < 0) {
			varStr += "负";
			num = num * -1; // 转化为正数
		}

		// 数字机械式的转化
		if (num > 999) {
			// 1234
			char[] CharArr = Integer.toString(num).toCharArray();
			for (char c : CharArr) {
				String s = "" + c;
				varStr += toSingleChStr(Integer.parseInt(s));
			}
			return varStr;
		}
		if (num == 0) { // 零
			return "零";
		}

		if (num / 10 == 1) { // 用十几 改变一十几的写法
			varStr += "十";
			if (num % 10 != 0) {
				varStr += toSingleChStr(num % 10);
			}
			return varStr;
		}

		if (num / 100 > 0) {
			varStr += toSingleChStr(num / 100) + "百";
			if (num % 100 == 0) {// 几百
				return varStr;
			} else if (num % 100 < 10) { // 几百零几
				return varStr += "零" + toSingleChStr(num % 100);
			}
		}

		num %= 100;
		if (num / 10 != 0) {
			varStr += toSingleChStr(num / 10) + "十";// 十
		}

		if (num % 10 != 0) {
			varStr += toSingleChStr(num % 10); // 几
		}

		return varStr;
	}

	/**
	 * 运算操作
	 * @param str
	 */
	public static void manipulateNum(String str) {
		String[] strArray = str.trim().split("\\s+");
		if (strArray.length != 3) {
			throw new DemoException("语法有错，请检查语法");
		}

		String operator = strArray[1];
		String varStr = strArray[0];
		String numStr = strArray[2];

		int var = getVar(varStr);
		int num = toNum(numStr);

		switch (operator) {
		case "减少":
			var -= num;
			break;
		case "增加":
			var += num;
			break;
		case "乘以":
			var *= num;
			break;
		case "除以":
			if (num == 0) {
				throw new DemoException("除数不能等于零哦");
			} else {
				var /= num;
			}
			break;
		case "模除":
			var %= num;
			break;
		default:
			throw new IllegalArgumentException("没有关键字：" + operator + " 请使用关键字：增加、减少、乘以、除以、模除 ");
		}

		setVar(varStr, var);
	}

	/**
	 * 判断是否为运算操作语句
	 * @param str
	 * @return
	 */
	public static boolean isManipulate(String str) {
		String[] array = str.trim().split("\\s+");

		String[] keywords = { "增加", "减少", "乘以", "除以", "模除" };

		// 下面会用到array[1]
		if (array.length == 1) {
			throw new DemoException("语法有错，请检查语法");
		}
		String symbol = array[1];

		for (String s : keywords) {
			if (symbol.equals(s)) {
				return true;
			}
		}
		return false;
	}

	@Test
	public void testSymbol() {
		System.out.println(isManipulate("钱包 模除  十"));
	}

	/**
	 * 从Map中获取变量，不存在抛出异常
	 * @param str 需要获取数据的变量名
	 * @return
	 */
	private static int getVar(String varStr) {
		if (isVar(varStr)) {
			return map.get(varStr);
		} else {
			throw new DemoException("变量：" + varStr + " 未定义，请定义变量");
		}
	}

	/**
	 * 存入变量到map中
	 * @param varStr 键
	 * @param var 值
	 */
	private static void setVar(String varStr, int var) {
		map.put(varStr, var);
	}

	/**
	 * 判断是否为已存在的变量
	 * @param varStr 键
	 * @return
	 */
	private static boolean isVar(String varStr) {
		return map.get(varStr) != null ? true : false;
	}
}



```


Exception类
```java
public class DemoException extends RuntimeException {

package Demo1;


public class DemoException extends RuntimeException {

	public DemoException() {
		super();
	}

	public DemoException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
		super(message, cause, enableSuppression, writableStackTrace);
	}

	public DemoException(String message, Throwable cause) {
		super(message, cause);
	}

	public DemoException(String message) {
		super(message);
	}

	public DemoException(Throwable cause) {
		super(cause);
	}

}


}

```

#### 思路
代码有三个类
- DemoMain 存放主函数（处理传入的字符串，分割成行）
- Utils 字符处理的工具类 
- DemoException （Demo的异常处理）

Utils类
- callFunction 传入一行字符串，判断逻辑关键字并调用对应的方法
- assignInt 整数赋值
- printOut 输出字符串或者整数
- ternaryOperator 三目操作
- judgeOperator 判断语句（举例：零 等于 零）
- toNum 把中文数字转化为数字
- toChStr 把数字转化为中文数字
- manipulateNum 增减运算
- toSingleNum 单个数字转化为单个汉字
- toSingleChStr 单个汉字转化为单个数字
- isManipulate 判断是否为运算操作语句
- getVar从Map中获取变量，不存在抛出异常

更新于2020.2.24