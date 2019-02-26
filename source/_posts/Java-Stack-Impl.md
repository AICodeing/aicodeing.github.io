---
title: Java-Stack Impl
date: 2019-02-26 22:35:37
tags:
	- Java
category: Java
comments: ture
---

前几天给一个亲戚学生讲了下 计算机中栈的相关知识，记得在 `JDK`中也有 `Stack`的实现`java.util.Stack`它继承于 `Vector`类，是线程安全的。通过翻看源码发现，它是基于 `线性表`来实现的`Stack`, 而`Stack`最为我们熟知的还是使用`单链表`来实现。  

### 基于 单链表实现 Stack
下面我们来基于`单链表`来实现一个`Stack`  

先定义 `Stack`需要的必要方法  

```java
public interface Stack<E> {

    void push(E item);

    E pop();

    E peek();

    int size();

    boolean empty();
}
```

既然基于单链表来实现，就需要定义一个链表的结构  

```java
private static class Node<E> {
        Node<E> next;
        E value;
    }
```

完整的 `StackWithLinkList`实现  

```java
public class StackWithLinkList<E> implements Stack<E> {

    private static class Node<E> {
        Node<E> next;
        E value;
    }

    Node<E> head;

    public StackWithLinkList() {
        init();
    }

    void init() {
        head = new Node<>();
    }

    /**
     * 头插法 实现 入栈操作
     *
     * @param item
     */
    public synchronized void push(E item) {
        Node<E> node = new Node<>();
        node.value = item;
        if (head.next == null) {
            head.next = node;
        } else {
            node.next = head.next;
            head.next = node;
        }
    }

    /**
     * 弹出栈顶元素
     */
    public synchronized E pop() {
        E res = null;
        if (head.next == null) {
            return null;
        } else {
            Node<E> first = head.next;
            res = first.value;
            head.next = first.next;
        }
        return res;
    }

    /**
     * 返回栈顶元素,不执行出栈操作
     */
    public synchronized E peek() {
        if (head.next == null) {
            return null;
        } else {
            return head.next.value;
        }
    }

    public boolean empty() {
        return head.next == null;
    }

    public synchronized int size() {
        int count = 0;
        Node tmp = head;
        while (tmp.next != null) {
            count++;
            tmp = tmp.next;
        }
        return count;
    }
```

很轻松的就基于 `单链表`实现了一个`栈`, 但在实现的过程中有一些小小的不爽，我们都知道，计算单链表中有多少个元素，就需要遍历一遍 链表，时间复杂度是`O(n)`，为了实现复杂度`O(1)`，我们只需加上计数器即可，其实 `JDK`中的`LinkedList`这种链表结构也是使用了计数器来实现`O(1)`复杂度的`size`计算。  

现在只需要在类中增加一个成员变量,并且在需要计数的地方，修改 计数器  

```java
int size = 0;
...
 public synchronized void push(E item) {
        Node<E> node = new Node<>();
        node.value = item;
        if (head.next == null) {
            head.next = node;
        } else {
            node.next = head.next;
            head.next = node;
        }
        size++;
    }
...
 public synchronized E pop() {
        E res = null;
        if (head.next == null) {
            return null;
        } else {
            Node<E> first = head.next;
            res = first.value;
            head.next = first.next;
            size--;
        }
        return res;
    }
...
   public synchronized int size() {
        return size;
    }
```

在 `JDK`中集合都采用了 `迭代器设计模式` 有一个 `iterator()`方法来隔离集合的遍历功能。我希望 `Stack`也拥有这个功能，于是乎开始动手  

```java
private class Itr implements Iterator<E> {

        Node<E> cursor = head.next;
        Node<E> cur = head;

        @Override
        public boolean hasNext() {
            return cursor != null;
        }

        @Override
        public E next() {
            E res = null;
            if (cursor != null) {
                res = cursor.value;
                cur = cursor;
                cursor = cursor.next;
            }
            return res;
        }

        @Override
        public void remove() {
            if (cur != null) {
                // 从head 开始查找
                Node tmp = head;
                while (tmp.next != null) {
                    if (cur == tmp.next) {
                        tmp.next = cur.next;
                        size--;
                        return;
                    }
                    tmp = tmp.next;
                }
            }
        }
    }
```
利用 `单链表`的特性，写出一个 `Iterator`还是比较顺畅的。
现在可以很Happy的添加`iterator()`方法啦  

```java
public synchronized Iterator<E> iterator() {
        return new Itr();
    }
```

### 基于 线性表 实现 Stack

前面说到`JDK`是利用`线性表`实现的`Stack`,但是 `java.util.Stack`不能满足我们的要求，我首先查看了`JDK`中`java.util.Stack`的实现，发现它没有实现 `Iterator`,但可以使用父类`Vector`的`iterator()`方法，但不幸的是，它还是顺序的，没有实现`Stack`的`先进后出`。如果能实现一个 带反转功能的 `Iterator`就能完美解决问题啦。

那下面我们来实现一个基于`线性表`的`Stack`
我们利用 `JDK`中已经完善的`ArrayList`来实现`Stack`,避免做太多与主题无关的实现。    

```java
public class StackWithList<E> implements Stack<E> {

    List<E> list = new ArrayList<>();

    @Override
    public void push(E item) {
        list.add(item);
    }

    @Override
    public E pop() {
        E obj;
        int len = size();
        obj = peek();
        list.remove(len - 1);
        return obj;
    }

    @Override
    public E peek() {
        int len = size();
        return list.get(len - 1);
    }

    @Override
    public int size() {
        return list.size();
    }

   @Override
    public Iterator<E> iterator() {
        //需要反转 iterator
        return new ReverseJavaListIterator<>(list);
        	//顺序 iterator
//        return new JavaListIterator<>(list);
			//使用 ArrayList中自带的 iterator
//        return list.iterator();
    }
    
    @Override
    public boolean empty() {
        return list.isEmpty();
    }
}
```

利用`线性表`的`顺序性`和`连续性`，实现 `Stack`比较高效，直观。  为了避免 像`java.util.Stack`没有`Iterator`的尴尬，我们自己实现一个 具有反转功能的`Iterator`。  

为了同时兼容顺序的`Iterator`,首先写一个适配 `线性表`的 `Iterator`,以后可以扩展非 `List`实现的线性表。

```java
public abstract class ArrayLikeIterator<T> implements Iterator<T> {
    protected int index;

    ArrayLikeIterator() {
        this.index = 0;
    }

    /**
     * 是否反转遍历
     */
    public boolean isReverse() {
        return false;
    }

    protected int bumpIndex() {
        return this.index++;
    }

    public int nextIndex() {
        return this.index;
    }

    /**
     * 如果 {@link #isReverse()}为 true， 需要重写该方法
     */
    public int lastIndex() {
        return index - 1;
    }

    public void remove() {
        throw new UnsupportedOperationException("remove");
    }

    public abstract long getLength();
}
```

我们的 线性表实现的`Stack`使用的是 `java.util.List`,所有 继承`ArrayLikeIterator ` 实现一个`JavaListIterator`  

```java
public class JavaListIterator<E> extends ArrayLikeIterator<E> {

    protected final List<E> list;
    protected int length;

    public JavaListIterator(List<E> list) {
        this.list = list;
        this.length = list.size();
    }

    protected void resetLength() {
        this.length = list.size();
    }

    protected void reviseIndex() {
        if (!isReverse()) {
            index--;
        }
    }

    protected boolean indexInArray() {
        return index < length && index >= 0;
    }

    @Override
    public long getLength() {
        return length;
    }

    /**
     * Returns {@code true} if the iteration has more elements.
     * (In other words, returns {@code true} if {@link #next} would
     * return an element rather than throwing an exception.)
     *
     * @return {@code true} if the iteration has more elements
     */
    @Override
    public boolean hasNext() {
        return indexInArray();
    }

    /**
     * Returns the next element in the iteration.
     *
     * @return the next element in the iteration
     */
    @Override
    public E next() {
        return list.get(this.bumpIndex());
    }

    @Override
    public void remove() {
        list.remove(lastIndex());
        resetLength();
        reviseIndex();
    }
}
```

实现的`JavaListIterator` 其实还是`顺序迭代`的，为了实现我们带__反转功能__的`Iterator`，我们还得实现一个`ReverseJavaListIterator`，原理很简单，就是倒序遍历 `List` 

```java
public class ReverseJavaListIterator<E> extends JavaListIterator<E> {

    public ReverseJavaListIterator(List<E> list) {
        super(list);
        this.index = list.size() - 1;
    }

    @Override
    public boolean isReverse() {
        return true;
    }

    protected boolean indexInArray() {
        return index >= 0;
    }

    @Override
    public int lastIndex() {
        return nextIndex() + 1;
    }

    protected int bumpIndex() {
        return index--;
    }
}
```

### 测试 Stack

```java
    public static void main(String[] args) {
//        Stack<Integer> stack = new StackWithLinkList<>();
        Stack<Integer> stack = new StackWithList<>();
        stack.push(0);
        stack.push(1);
        stack.push(2);
        stack.push(3);

        System.out.println("size=" + stack.size());

        Iterator<Integer> iterator = stack.iterator();
        while (iterator.hasNext()) {
            Integer next = iterator.next();
            System.out.println(next);
            if (next == 2) {
                iterator.remove();
                System.out.println("remove " + next);
            }
        }

        System.out.println("---- print all ----");
        System.out.println("size=" + stack.size());
        Iterator<Integer> iterator1 = stack.iterator();
        while (iterator1.hasNext()) {
            Integer next = iterator1.next();
            System.out.println(next);
        }
    }
```

输出:  

```
size=4
3
2
remove 2
1
0
---- print all ----
size=3
3
1
0
```

至此，我们基于 `单链表`和`线性表`的两种`Stack`实现方案测试通过。  
其实还有很多需要完善的地方，比如多线程冲突修改问题，我们可以模仿 `JDK`,每次增删修改元素都使用`modCount`来记录，如果 `modCount != expectedModCount` 我们也可以抛出`ConcurrentModificationException`异常。




