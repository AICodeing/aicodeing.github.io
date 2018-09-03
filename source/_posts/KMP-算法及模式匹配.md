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

### 常规实现
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
**实现1:**  

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
**实现2:**  

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

__分析:__  
按照上图中的示例，主串 A **B** **C** D E F A B D J K。 模式串 A B D  
常规算法中 除了 主串中 粗体字母 B 和 C 比较了两次以外，其他字母均只和模式串比较一次。在这种情况下，此算法的时间复杂度为O(n+m)。其中n 和m 分别是主串和模式串的长度。  

然而，在有些情况下，该算法的效率却很低。例如，当模式串为 ‘00000001’，主串为‘0000 0000 0000 0000 0001’时，由于模式中前7个字符均为‘0’，，又，主串中前 19 个字符均为 ‘0’，每趟比较都在模式的最后一个字符出现不等，此时需将指针 i 回溯到 i -6 的位置上，并从模式的第一个字符开始重新比较，整个匹配过程中指针i 需要回溯 12次,则在算法 ——__实现2__ 中 while 循环的次数为 13 * 8 _(index * m)_。  
	可见，算法实现2 在最坏情况下时间复杂度为O(n*m)。这种情况在只有 0，1 两种字符的文本处理中经常出现，因为在主串中可能存在多个和模式串“部分匹配”的子串，因为引起指针 `i`的多次回溯。 `01`串可以用在许多应用之中。比如，一些计算机的图形显示就是把画面表示为一个01串，一页书就是一个几百万个0和1组成的串。在二进制计算机上实际上处理的都是01串。一个字符的ASCII码也可以看成是8位二进制的01串。  

### KMP算法  
`D.E.Knuth`、`J.H.Morris`和`V.R.Pratt` 发明了`KMP`，该算法主要解决的问题就是 前面提到的`常规算法`需要多次回溯指针。  
![](/img/kmp/kmp_5.jpg "")   
如图所示，当 `i=2；j=2`模式匹配失败时,会经历 `i = 1,i = 2...`直到 指针 `i`指向`i = 6;`而`j`需要由`j = 2`回溯到`j = 0`。示例中这种情况还是比较理想的，我们也就是多比较了几次而已。但 对与模式串为 ‘00000001’，主串是‘0000 0000 0000 0000 0001’，每次 `i`指针更新一次，`j`指针都是到最后一个位置才知道不匹配，然后 `i和j`回溯，这个效率显然很低。那么`KMP`的思想呢?  

__`KMP`__希望利用 **前面已经匹配的信息，来避免 `i`指针的回溯，即`i`指针不回溯，修改`j`指针，让模式串尽量地移动到有效的位置**  

__`KMP`__想解决的问题是 **当某个字符与主串不匹配时，我们应该知道`j`指针要移动到哪？**   

我们看下规律:  
当出现不匹配时:
![](/img/kmp/kmp_6.jpg "")   
如图，C 和D 不匹配了，我们要把 `j`移动到哪？显然是 第 0 个位置，因为前面 i = 2 的位置是一个A，模式串的第一个字符也是A。  
![](/img/kmp/kmp_7.jpg "")  

再看看另外一个示例:  
![](/img/kmp/kmp_8.jpg "")  
把指针 j 移到 j = 2的位置，因为前面两个字母一样的:
![](/img/kmp/kmp_9.jpg "")

我们可以大概看出一些规律，**当匹配失败的时候，`j`要移动的下一个位置 `k`，它前面的`k`个字符和`i`之前的最后k个字符是一样的。**  
有:  
**'p(0)p(1) ...p(k-1)' = 's(i-k)s(i-k+1)...s(i-1)'**  
_p 是模式串，s是主串_  
而已经得到的“部分匹配”的结果是:    
**'p(j-k)p(j-k+1)...p(j-1)' = 's(i-k)s(i-k+1)...s(i-1)'**  
 ==>  
**'p(0)p(1) ...p(k-1)' = 'p(j-k)p(j-k+1)...p(j-1)'**  

即 **P[0 ~ k-1] = P[j-k ~ j-1]**
 
 那么如何求解 `k`就是 `KMP算法`的核心问题了。也即是计算每一个位置 `j`对应的`k`。所以用一个` 数组next ` 保持 `k`。  
 
 	next[j] = k
 
表示当S(i) != P(j)时，`j指针`需要回溯的位置。 
算法表示:   
 
```java
private static int[] getNext(char[] target) {
        char[] p = target;
        int[] next = new int[p.length];
        next[0] = -1;
        int j = 0;
        int k = -1;
        while (j < p.length - 1) {
            if (k == -1 || p[j] == p[k]) {
                next[++j] = ++k;
            } else {
                k = next[k];
            }
        }
        return next;
    }
```

这段代码在数据结构中直接给出来，那么为什么这样处理呢？  
当`j =0`时 如果出现不匹配:  

 ![](/img/kmp/kmp_11.jpg "")  
 这种情况下 j 已经在最左边了，不能再移动了，这时候需要i指针后移，所以代码中有 
 
 	next[0] = -1;初始化。  
 当j = 1时:  
  ![](/img/kmp/kmp_12.jpg "")  
  指针 j 一定会回溯到 0 的位置，因为它前面也只有这个位置了。  
  再看看非边界情况:  
  ![](/img/kmp/kmp_13.jpg "")
  
  出现匹配不通过的位置 j和 下一个回溯位置 k 的字符相同时， j+1 位置对应的k 和 j 位置对应的k 的关系是  `next[j+1] = next[j]+1`  
即**当 `P[k] = P[j]`**时有:  
  `next[j+1] = next[j]+1`  
  
  这个结论也可以通过数学证明:  
  因为 `p(j)`之前已经有 `P[0 ~ k-1] = P[j-k ~ j-1]`。
  如果 `P[k] = P[j]` 则 `P[0 ~ k-1] +P[k] = P[j-k ~ j-1] + P[j]`
即有`P[0 ~ k] = P[j-k ~ j]` => `next[j+1] = k+1 = next[j]+1`

**当 `P[k] != P[j]`时:**  
如图:  
![](/img/kmp/kmp_14.jpg "")  
代码中 对于 不相等的情况 `k = next[k];`  
![](/img/kmp/kmp_15.jpg "")  

所以有 `k = next[k];` 

至此，我们获得了 `next数组`  

以下是`KMP`算法的实现:  

```java
 private static int indexOf_KMP(char[] source, int sourceOffset, int sourceCount, char[] target, int targetOffset, int targetCount, int fromIndex) {
        int i = sourceOffset + fromIndex;
        int j = targetOffset;
        int[] next = getNext_old(target);
        while (i < sourceCount && j < targetCount) {
            if (j == -1 || source[i] == target[j]) {
                ++i;
                ++j;
            } else {
                j = next[j];
            }
        }
        if (j >= targetCount) {
            return i - j;
        }
        return -1;
    }
```
可以看出相对于 `常规算法` `i`就不需要回溯了。

该算法还可以进一步优化:  
![](/img/kmp/kmp_16.jpg "")  
显然，当通过 next算法获取 到的next 数组为[-1,0,0,1]
所以可以把j 移动到第1个元素:
![](/img/kmp/kmp_17.jpg "") 
其实这一步是完全没有意义的，因为 后面的B 已经不匹配了，那前面的B 也一定不匹配了，同样的情况还发生在第 2 个元素A上。
显然当`p[j] = P[next[j]]`时，我们可以进一步处理:  

```java
private static int[] getNext(char[] target) {
        char[] p = target;
        int[] next = new int[p.length];
        next[0] = -1;
        int j = 0;
        int k = -1;
        while (j < p.length - 1) {
            if (k == -1 || p[j] == p[k]) {
                if (p[++j] == p[++k]) { // 当两个字符相等时要跳过
                    next[j] = next[k];
                } else {
                    next[j] = k;
                }
            } else {
                k = next[k];
            }
        }
        return next;
    }
```
即当 两个字符相等时，跳过比较。

以上就是 KMP 算法的分析。