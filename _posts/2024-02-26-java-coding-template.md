---
title: "Java刷题模版"
categories:
  - algorithm
tags:
  - template
---

# Java刷题模版



```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        // 输入
        Scanner scanner = new Scanner(System.in);
      	// String
      	//截取子串（左闭右开）
				str.substring(beginIndex);
				str.substring(beginIndex, endIndex);
				//获取子串在原串中第一次出现的位置
				str.indexOf(s);
				//字符串比较（str1小于str2返回负数，大于返回正数，相等返回0）
				str1.compareTo(str2);
      
      	// int to String
      	String.valueOf(i);

        // StringBuilder
				StringBuilder sb = new StringBuilder();
				//添加字符或字符串（可叠加）
				sb.append(str);
				//获取StringBuilder长度
				sb.length();
				//获取某位字符
				sb.charAt(i);
				//插入子串
				sb.insert(index, str)
				//替换子串
				sb.replace(start, end, str);
				//删除某位字符
				sb.deleteCharAt(index);
				//删除子串（左闭右开），必须传区间
				sb.delete(start, end);

    }
}
```

