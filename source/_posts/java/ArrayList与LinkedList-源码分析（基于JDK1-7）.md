---
title: ArrayList与LinkedList 源码分析（基于JDK1.7）
toc: true
date: 2017-08-29 11:25:00
tags: 
    - 源码
---
### List接口

List接口中的方法有很多，但最重要的无非是增删查改，我们从ArrayList与LinkedList的实现上来讨论他们的增删查改性能问题。先列出这几个重要的方法:
```
public interface List<E> extends Collection<E> {
    boolean add(E e); 
    void add(int index, E element);
    E get(int index);
    E set(int index, E element);
    E remove(int index);
    boolean remove(Object o);
}
```
<!-- more -->
### ArrayList

ArrayList的默认容量大小是 10 

![image.png](http://upload-images.jianshu.io/upload_images/3353177-31e9bddfa37838c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

构造函数
ArrayList底层使用的是动态数组，我们常用到的构造方法一般是如下两种:

```
 /**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
    }

    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        super();
        this.elementData = EMPTY_ELEMENTDATA;
    }
```

从源码可以看出，两者的区别在于初始化数组的长度，前者给定一个空数组，后者若initialCapacity大于0即给定一个initialCapacity大小的数组

#### add一个元素
```
 /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

```
 private void ensureCapacityInternal(int minCapacity) {
        if (elementData == EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```
```
   private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

添加元素时会检测数组大小是否满足条件，不满足会去新建一个更大的数组，把原来数组中的元素都copy过来，可以看出，对于ArrayList的add操作来讲，是比较低效的(当需要扩容时)。 
另外还有个public void add(int index, E element)方法，它在指定位置add元素时，需要把指定位置后面的所有元素都往后移动一个位置，所以也是比较低效的。


#### get一个元素
```
public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

 private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

 E elementData(int index) {
        return (E) elementData[index];
    }

```
可以看到，ArrayList的Get是非常高效的，只要index没有越界，直接从底层数组中返回即可，这也是ArrayList的优势所在。同理 ，ArrayList的Set也很高效，直接往数组中写即可。

#### remove一个元素
```
public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

```

ArrayList在做删除操作时，因为需要把后面的所有元素整体前移来填空，所也也是非常耗资源的。 
另外还有个public boolean remove(Object o)方法，这个是做一次遍历查询，然后删除。

### LinkedList

#### 构造函数
通过名字就可知道，LinkedList底层使用的是链表的形式去实现的，它的构造函数什么也没干:
```
 public LinkedList() {
    }

```

#### add一个元素
```
 public boolean add(E e) {
        linkLast(e);
        return true;
    }

 /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```

add一个元素就直接把这个元素放到链表的末端即可。但需要注意的是，相对于ArrayList的add方法，LinkedList的add方法并不见得高效，而且当数据量大后还远慢于ArrayList。 
同时，LinkedList还是双向链表，所以内部同时保留了transient Node<E> last和transient Node<E> first;的引用。

#### get一个元素
```
   public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
 private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

通过查看index更靠近链表的哪一端，决定从哪一端去遍历。LinkedList在查找时需要遍历，所以相对于ArrayList的随机存取来说，会低效一些。

#### remove一个元素
```
public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }

E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```

### 总结

+ ArrayList因为是基于动态数组去实现，在随机存取时，有着良好的性能。而增删时需要扩容，整块移动元素，所以相对较慢。但在数据量很大，顺序添加时是个例外，这种情况下它的性能优于LinkedList。
+ LinkedList因为是基于链表实现，随机增删较快，而存取时需要遍历查询，相对于ArrayList会更慢。
之后会比较下两种实现的迭代器性能。