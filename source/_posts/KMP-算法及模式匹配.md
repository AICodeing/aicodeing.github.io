---
title: KMP 算法及模式匹配
date: 2018-08-31 18:39:25
tags:
	- 算法
	- KMP
category: 算法
comments: true
---
## 字符串的模式匹配
字符串的模式匹配 通常是 定位一个子串在一个字符串中的位置。例如 `String.indexOf()`方法。  

#### 约定
子串称为 `模式串`

### 常规算法
思想:从左往右一个字符一个字符的匹配，如果匹配的过程中有某一个字符不匹配，主串回退到本次匹配开始位置的下一个位置，模式串回退到第一位，然后重新开始匹配。  
过程如下:  
![](/img/kmp/kmp_1.jpg "")  
指针 i 指向主串第一个位置，j指向模式串第一个位置  
![](/img/kmp/kmp_2.jpg "")   
当出现不一致情况时:  
![](/img/kmp/kmp_3.jpg "")  

将指针 `i` 回到 `B` 的位置 ;将指针 `j` 回退到 `A`的位置。
![](/img/kmp/kmp_4.jpg "") 

`Java JDK` 中 `String.indexOf`实现就是该思想。该匹配过程易于理解，且在某些应用场合，如文本编辑等，效率也较高。但是时间复杂度比较高，达到了 `O(n^2)`。

__JDK实现__:  

```java
private static int indexOf(char[] source, int sourceOffset, int sourceCount, char[] target, int targetOffset, int targetCount, int fromIndex) {
        if (fromIndex >= sourceCount) {
            return targetCount == 0 ? sourceCount : -1;
        }
        if (fromIndex < 0) {
            fromIndex = 0;
        }
        if (targetCount == 0) {
            return fromIndex;
        }

        char first = target[targetOffset];
        int max = sourceOffset + (sourceCount - targetCount);
        for (int i = sourceOffset + fromIndex; i <= max; i++) {
            //查找第一个字母
            if (source[i] != first) {
                while (++i <= max && source[i] != first) ;
            }
            if (i <= max) {
                //找到第一个字母匹配的 i,开始 匹配第2个字符，并以此循环匹配剩余字符
                int j = i + 1;
                int end = j + targetCount - 1;
                for (int k = targetOffset + 1; j < end && source[j] == target[k]; j++, k++) ;
                if (j == end) {
                    return i - sourceOffset;
                }
            }
        }
        return -1;
    }
```

相同思想的其他方式:  

```java
private static int indexOf(char[] source, int sourceOffset, int sourceCount, char[] target, int targetOffset, int targetCount, int fromIndex) {
        if (fromIndex >= sourceCount) {
            return targetCount == 0 ? sourceCount : -1;
        }
        if (fromIndex < 0) {
            fromIndex = 0;
        }
        if (targetCount == 0) {
            return fromIndex;
        }

        int i = sourceOffset + fromIndex;
        int j = targetOffset;
        while (i < sourceCount && j < targetCount) {
            if (source[i] == target[j]) {//继续比较后续字符
                ++i;
                ++j;
            } else {//指针回溯，重新开始匹配
                i = i - j + 1;
                j = targetOffset;
            }
        }
        if (j >= targetCount) {//匹配完成
            return i - j;
        }

        return -1;
    }
```


