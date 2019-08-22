<!-- TOC -->

- [一、概述](#一概述)
    - [1.1 说明](#11-说明)
    - [1.1 Collection](#11-collection)
        - [1.1.1 List](#111-list)
        - [1.1.2 Set](#112-set)
        - [1.1.3 Queue](#113-queue)
    - [1.2 Map](#12-map)
    - [1.3 Arrays.asList()](#13-arraysaslist)
- [二、源码分析](#二源码分析)
    - [2.1 ArrayList](#21-arraylist)
        - [2.1.1 线程安全方案 Vector、CopyOnWriteArrayList、Collections.synchronizedList() 对比](#211-线程安全方案-vectorcopyonwritearraylistcollectionssynchronizedlist-对比)
    - [2.2 LinkedList](#22-linkedlist)
    - [2.3 HashMap](#23-hashmap)
        - [2.3.1 成员变量和构造函数](#231-成员变量和构造函数)
        - [2.3.2 存储结构](#232-存储结构)
        - [2.3.3 put、get、resize](#233-putgetresize)
        - [2.3.4 线程安全方案 HashTable、ConcurrentHashMap、Collections.synchronizedMap() 对比](#234-线程安全方案-hashtableconcurrenthashmapcollectionssynchronizedmap-对比)

<!-- /TOC -->

# 一、概述

容器主要包括 Collection 和 Map 两种，Collection 存储着对象的集合，而 Map 存储着键值对（两个对象）的映射表。

<div align ="center"> <img src ="../pictures//Java%20容器.webp" /> </div><br>

## 1.1 说明

**（1）RandomAccess**

RandomAccess 是一个接口，仅仅起一个标识的作用，标识实现的类支持快速随机访问功能，即可通过元素的序号快速获取元素对象 (对应于 get(int index) 方法)。

**（2）fail-fast 和 fail—safe**

- fail-fast：用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行了修改（增加、删除、修改时 modCount 会自增），遍历期间 modCount 字段不同则会抛出 ConcurrentModificationException。
- fail—safe：支持在多线程下并发使用和修改，也可以在 foreach 中删减，原理是遍历时先复制原有集合内容，在拷贝的集合上遍历。

## 1.1 Collection

### 1.1.1 List

List 接口存储一组不唯一（可以有多个元素引用相同的对象），有序的对象。

- ArrayList：基于动态数组实现，支持随机访问。
- Vector：和 ArrayList 类似，但它是线程安全的（方法加锁）。
- LinkedList：基于双向链表实现，只能顺序访问，但是可以快速地在链表中间插入和删除元素。不仅如此，LinkedList 还可以用作栈、队列和双向队列。

### 1.1.2 Set

不允许重复对象的集合。

- TreeSet：基于红黑树实现，支持有序性操作，例如根据一个范围查找元素的操作。但是查找效率不如 HashSet，HashSet 查找的时间复杂度为 O(1)，TreeSet 则为 O(logN)。
- HashSet：基于哈希表实现，支持快速查找，但不支持有序性操作。并且失去了元素的插入顺序信息，也就是说使用 Iterator 遍历 HashSet 得到的结果是不确定的。
- LinkedHashSet：具有 HashSet 的查找效率，且内部使用双向链表维护元素的插入顺序。

### 1.1.3 Queue

- LinkedList：可以用它来实现双向队列。
- PriorityQueue：基于堆结构实现，可以用它来实现优先队列。

## 1.2 Map

- TreeMap：基于红黑树实现。
- HashMap：基于哈希表实现。
- HashTable：和 HashMap 类似，但它是线程安全的，这意味着同一时刻多个线程可以同时写入 HashTable 并且不会导致数据不一致。它是遗留类，不应该去使用它。现在可以使用 ConcurrentHashMap 来支持线程安全，并且 ConcurrentHashMap 的效率会更高，因为 ConcurrentHashMap 引入了分段锁。
- LinkedHashMap：使用双向链表来维护元素的顺序，顺序为插入顺序或者最近最少使用（LRU）顺序。

## 1.3 Arrays.asList()

Arrays.asList() 可以将一个数组转换为一个 List 集合。该返回对象是一个 Arrays 内部类，并没有实现集合的修改方法，因此 add/remove/clear 等方法会抛出 UnsupportedOperationException 异常。

Arrays.asList() 是泛型方法，传入的对象必须是对象数组。若传入一个原生数据类型数组时，Arrays.asList() 的真正得到的参数就不是数组中的元素，而是数组对象本身！此时 List 的唯一元素就是这个数组。使用包装类型数组就可以解决这个问题。

```java
int[] myArray = { 1, 2, 3 };
List myList = Arrays.asList(myArray);
System.out.println(myList.size());//1
System.out.println(myList.get(0));//数组地址值
System.out.println(myList.get(1));//报错：ArrayIndexOutOfBoundsException
int [] array=(int[]) myList.get(0);
System.out.println(array[0]);//1

// 可通过以下方式将数组转为可修改的 List。
List list = new ArrayList<>(Arrays.asList("a", "b", "c"))
```

# 二、源码分析

若非单独说明，本文源码分析都基于 JDK1.8。

## 2.1 ArrayList

ArrayList 的特性：

1. 实现了 RandomAccess 接口，RandomAccess 是一个标志接口，表明实现这个这个接口的 List 集合是支持快速随机访问的。
2. 实现了 Cloneable 接口，即覆盖了函数 clone()，能被克隆。
3. 实现了 Serializable 接口，这意味着 ArrayList 支持 Serializable 序列化。


在看 ArrayList 的源码前，先看一个方法，它在 ArrayList 中经常用到。
```java
public final class System {
    /**
     * 从数组 src 的第 srcPos（包含） 开始，复制 length 个元素到数组 dest ，并在第 destPos（包含）个索引开始复制。
     *
     */
    public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
}
```

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认初始数组容量大小。
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 空数组，当 initialCapacity == 0 时使用。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 用于空参数的构造函数实例化赋值，目的是在第一次添加元素时再确定 ArrayList 的初始容量。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 元素（数据）集，ArrayList 本质上是一个数组。
     * 使用 transient 修饰，不会被序列化。
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * ArrayList 所包含的元素个数。
     */
    private int size;

    /**
     * 带初始容量参数的构造函数。（用户自己指定容量）
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            // 创建 initialCapacity 大小的数组。
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 使用静态常量空数组。
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 默认构造函数，DEFAULTCAPACITY_EMPTY_ELEMENTDATA 为 0，默认容量为 10，
     * 也就是说初始其实是空数组，当第一次添加元素的时候为 10 和添加元素（集）数量的最大值。
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 构造一个包含指定集合的元素的列表，按照它们由集合的迭代器返回的顺序。
     */
    public ArrayList(Collection<? extends E> c) {
        //
        elementData = c.toArray();
        // 如果指定集合元素个数不为 0。
        if ((size = elementData.length) != 0) {
            // c.toArray 可能返回的不是 Object 类型的数组。
            if (elementData.getClass() != Object[].class)
            // 手动创建一个新的数组并拷贝（浅拷贝）子元素到新的数组中。
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 集合元素个数为 0，则用空数组代替。
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

    /**
     * 修改数组的的容量为当前元素数量的大小。
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
    /**
     * ArrayList 的扩容机制
     *
     * 如有有必要，增加数组的容量，以确保它至少能容纳元素的数量。
     * @param   minCapacity   所需的最小容量
     */
    public void ensureCapacity(int minCapacity) {
        // 即：实例化时没指定数组大小则为默认大小 DEFAULT_CAPACITY，否则为 0。
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) ? 0
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            // 扩容
            ensureExplicitCapacity(minCapacity);
        }
    }

    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            // 获取默认的容量和传入参数的较大值作为所需的最小容量
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // 所需最小容量大于当前数组的大小则进行扩容
        if (minCapacity - elementData.length > 0)
            // 开始扩容
            grow(minCapacity);
    }

    /**
     * ArrayList 定义的最大数组大小
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 扩容的核心方法。
     */
    private void grow(int minCapacity) {
        // oldCapacity 为旧容量大小，newCapacity 为新容量，minCapacity 为所需的最小容量。
        int oldCapacity = elementData.length;
        // 将 oldCapacity 右移一位，其效果相当于 oldCapacity /2，
        // 位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的 1.5 倍。
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量。
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 再检查新容量是否超出了 ArrayList 所定义的最大容量。
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            // 若超出了，则比较 minCapacity 和 MAX_ARRAY_SIZE，若大于 MAX_ARRAY_SIZE，则返回 Integer.MAX_VALUE 作为容器的最大容量，否则返回 MAX_ARRAY_SIZE。
            newCapacity = hugeCapacity(minCapacity);
        // 创建一个大小为 newCapacity 的新数组，并将元素集复制到新数组中。
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    /**
     * 返回元素的数量。 
     */
    public int size() {
        return size;
    }

    /**
     * 空数据集，则返回 true 。
     */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 如果包含指定的元素，则返回 true。
     */
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    /**
     * 从列表中首节点开始向前驱节点遍历查找第一次找到的指定元素的索引。
     * 如果此列表不包含此元素，则为-1 
     */
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    /**
     * 从列表中尾节点开始向前驱节点遍历查找第一次找到的指定元素的索引。
     * 如果此列表不包含元素，则返回 -1。.
     */
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    /**
     * 返回此 ArrayList 实例的浅拷贝。
     */
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            // 创建一个新数组，并复制 elementData 中的元素前 size 个到新数组。
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }
    }

    /**
     * 返回一个包含了所有元素的新数组。
     */
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

    /**
     * 将元素集复制到指定数组 a 中。
     */
    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // 如果指定数组 a 的长度小于元素集的数量，
            // 则直接返回一个新数组，包含了 elementData 中前 size 个元素。
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        // 否则将元素集复制到数组 a 中。
        System.arraycopy(elementData, 0, a, 0, size);
        
        if (a.length > size)
            a[size] = null;
        return a;
    }

    /**
     * 返回此列表中指定位置的元素（不校验）。
     */
    @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }

    /**
     * 返回此列表中指定位置的元素。
     */
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

    /**
     * 替换数组中第 index 个的元素为 element，并返回被替换的元素。 
     */
    public E set(int index, E element) {
        rangeCheck(index);
        // 暂存旧值。
        E oldValue = elementData[index];
        // 赋值新值。
        elementData[index] = element;
        // 返回旧值。
        return oldValue;
    }

    /**
     * 将指定的元素追加到此列表最后一个元素之后。 
     */
    public boolean add(E e) {
        // 添加元素前，先扩容（若需要的话）。
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // ArrayList 添加元素的本质就相当于为数组赋值。
        elementData[size++] = e;
        return true;
    }

    /**
     * 在指定索引 index 添加元素 element。 
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1); 
        // 先将原索引 index 及之后的元素复制到原索引位置的后一位（索引加一）。
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        // 赋值
        elementData[index] = element;
        size++;
    }

    /**
     * 删除该列表中指定位置的元素，并返回删除的元素.
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        // 先拿到要删除的值。
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        // index+1 及之后的元素复制到原索引位置的前一位（索引减一）。
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        // 数组最后一位元素置 null，下次 GC 回收该无用元素。
        elementData[--size] = null; 
        return oldValue;
    }

    /**
     * 从列表中首节点开始遍历删除第一次找到的指定元素节点。
     * 如果此列表包含指定的元素返回 ture，否则返回 false。
     */
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        // 删除的本质是将自身元素对前一位索引中的元素赋值。
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

    /**
     * 清空数组中的所有元素。 
     */
    public void clear() {
        modCount++;

        // 把数组中所有的元素的值设为 null。
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }

    /**
     * 将指定集合 c 中的所有元素按 Iterator 返回的顺序追加到此列表的末尾。
     */
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        // 扩容（如果需要的话）
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

    /**
     * 将指定集合中的所有元素插入到此列表中，从指定的位置开始。
     */
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

    /**
     * 删除此列表中第 formIndex 个到第 toIndex-1 个元素。
     */
    protected void removeRange(int fromIndex, int toIndex) {
        modCount++;
        int numMoved = size - toIndex;
        // 将第 toIndex 个元素到最后一个元素复制到第 forIndex 个元素依次往后。
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

        // clear to let GC do its work
        int newSize = size - (toIndex-fromIndex);
        // 置空第 newSize 到 size-1 的元素（元素全都被复制到索引 newSize 之前）
        for (int i = newSize; i < size; i++) {
            elementData[i] = null;
        }
        size = newSize;
    }

    /**
     * 检查给定的索引是否在数组范围内。
     */
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    /**
     * 用于 add 和 addAll 的校验。
     */
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    /**
     * 返回 IndexOutOfBoundsException 细节信息
     */
    private String outOfBoundsMsg(int index) {
        return "Index: "+index+", Size: "+size;
    }

    /**
     * 从此列表中删除指定集合中包含的所有元素。如果 ArrayList 被修改则返回 true，反之为 false。
     */
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

    /**
     * 仅保留此列表中在指定集合 c 中的元素，其余的删除。
     */
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }

    /**
     * 实现了 writeObject() 和 readObject() 来控制只序列化数组中有元素填充那部分。
     * 序列化时需要使用 ObjectOutputStream 的 writeObject() 将对象转换为字节流并输出，
     * 而 writeObject() 方法会反射调用该对象的 writeObject() 来实现序列化。
     * 反序列化使用的是 ObjectInputStream 的 readObject() 方法，原理也是反射。
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
     * deserialize it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }

    /**
     * 从列表中的指定位置开始，返回列表中的元素（按正确顺序）的列表迭代器（fail-fast）。
     */
    public ListIterator<E> listIterator(int index) {
        if (index < 0 || index > size)
            throw new IndexOutOfBoundsException("Index: "+index);
        return new ListItr(index);
    }

    /**
     * 返回列表中的列表迭代器（fail-fast）。 
     */
    public ListIterator<E> listIterator() {
        return new ListItr(0);
    }

    /**
     * 正序返回该列表中的元素的迭代器。 
     */
    public Iterator<E> iterator() {
        return new Itr();
    }

```

### 2.1.1 线程安全方案 Vector、CopyOnWriteArrayList、Collections.synchronizedList() 对比

**（1）Vector**

```java
// 和 ArrayList 类似，只是在方法上加了 synchronized 关键字。
public synchronized boolean add(E e) {
    ++this.modCount;
    this.ensureCapacityHelper(this.elementCount + 1);
    this.elementData[this.elementCount++] = e;
    return true;
}

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 扩容为原来容量的 2 倍
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                        capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**（2）CopyOnWriteArrayList**

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;

    // 写操作需要加锁，防止并发写入时导致写入数据丢失。
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 每一次写操作在一个复制的数组上进行，读操作还是在原始数组中进行，读写分离，互不影响。
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        // 写操作结束之后需要把原始数组指向新的复制数组。
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
final void setArray(Object[] a) {
    elements = a;
}
```

**（3）Collections.synchronizedList()**

所有的 Collections.synchronizedXXX() 方法都是通过锁住和元素相关的代码块，和 Vecotor 锁方法类似，不同在于可以指定锁对象。

```java
public void add(int index, E element) {
    synchronized (mutex) {
        list.add(index, element);
    }
}
```

**（4）小结**

1. 扩容方面：Vector 扩容为原来容量的 2 倍，ArrayList 扩容为原来容量 1.5 倍，CopyOnWriteArrayList 每次都是一个新的数组。
2. 性能方面：没有并发需求的情况下，优先使用 ArrayList。有并发需求的情况下：
- Collections.synchronizedList() 和 Vector 的读写性能几乎一致，理论 Vector 比  Collections.synchronizedList() 还快一些，因为后者对方法多包了一层。
- 仅在读多写少的应用场景，推荐使用 CopyOnWriteArrayList，它在写操作的同时允许读操作，大大提高了读操作的性能，但是在写操作时需要复制一个新的数组，使得内存占用为原来的两倍左右。
- 其余情况推荐自己控制并发或使用 Collections.synchronizedList()。
3. 拓展性：Collections.synchronizedList() 支持设置锁对象，因此拓展性更好。
2. Vector 和 Collections.synchronizedList() 看似已经线程安全，但 Iterator 例外，在使用 Iterator 的时候，需要对整个迭代过程加锁，否则在迭代过程使用非迭代器修改数据会抛 ConcurrentModificationException 异常。

## 2.2 LinkedList

LinkedList 的特性：

1. 实现了 Cloneable 接口，即覆盖了函数 clone()，能被克隆。
2. 实现了 Serializable 接口，这意味着 ArrayList 支持 Serializable 序列化。
3. 实现了 Deque 接口，它拥有队列的特性。
4. 支持高效的插入和删除操作，在索引上需要遍历链表，因此相对较慢。

我们先看一下链表的节点类 Node。因此链表的本质就是围绕着节点的前驱节点和后驱节点设置引用对象。

```java
private static class Node<E> {
    E item;         // 本节点的值
    Node<E> next;   // 前驱节点
    Node<E> prev;   // 后继节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
   /**
     * 元素数量。
     */
    transient int size = 0;

   /**
     * 首节点。
     */
    transient Node<E> first;

    /**
     * 尾结点。
     */
    transient Node<E> last;

    /**
     * 使用该构造函数，生成一个空链表。
     */
    public LinkedList() {
    }

    /**
     * 按照 Collection 的 iterator 返回的顺序构造一个包含指定集合元素的链表。
     */
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    /**
     * 添加一个包含元素 e 的新节点作为首节点。
     */
    private void linkFirst(E e) {
        // 临时变量存下当前的第一个节点。
        final Node<E> f = first;
        // 生成一个新节点，且后继节点为当前的第一个节点
        final Node<E> newNode = new Node<>(null, e, f);
        // 首节点指向新节点。
        first = newNode;
        if (f == null)
            // 如果 f == null，说明链表一个节点都没有，则尾节点也指向新节点。
            last = newNode;
        else
            // f 的前驱节点指向新节点。
            f.prev = newNode;
        // 元素数量加 1。
        size++;
        modCount++;
    }

    /**
     * 添加一个包含元素 e 的新节点作为尾节点。
     */
    void linkLast(E e) {
        // 临时变量存下当前的最后一个节点。
        final Node<E> l = last;
        // 生成一个新节点。
        final Node<E> newNode = new Node<>(l, e, null);
        // 尾节点指向新节点。
        last = newNode;
        // 如果 l == null，说明列表一个节点都没有。
        if (l == null)
        // 则首节点也指向新节点。
            first = newNode;
        else
        // l 的后驱节点指向新节点。
            l.next = newNode;
        // 元素数量加 1。
        size++;
        modCount++;
    }

    /**
     * 在非 null 节点 succ 之前插入元素 e。
     */
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        // 暂存节点 succ 的前驱节点 pred。
        final Node<E> pred = succ.prev;
        // 生成一个新节点，前驱节点为 pred，后驱节点为 succ。
        final Node<E> newNode = new Node<>(pred, e, succ);
        // 将 succ 的前驱节点指向新节点。
        succ.prev = newNode;
        if (pred == null)
        // 若 pred == null，说明插入新节点前 succ 是首节点，因此现在首节点指向新节点。
            first = newNode;
        else
        // pred 的后驱节点指向新节点。
            pred.next = newNode;
        size++;
        modCount++;
    }

    /**
     * 删除首节点。该方法的参数 f 一定是首节点且不会为 null。
     */
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        // 拿到首节点 f 的后驱节点。
        final Node<E> next = f.next;
        f.item = null; // help GC
        f.next = null; // help GC
        // 首节点变为 f 的后驱节点。
        first = next;

        if (next == null)
        // 如果 next == null，说明整个链表只有一个节点，因此尾节点也指向 null。
            last = null;
        else
        // 相当于置空 f 在后驱节点的引用。
            next.prev = null;
        size--;
        modCount++;
        // 返回删除节点的元素
        return element;
    }

    /**
     * 删除尾节点。该方法的参数 f 一定是尾节点且不会为 null。
     */
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        // 拿到首节点 f 的前驱节点。
        final Node<E> prev = l.prev;
        l.item = null; // help GC
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
        // 如果 prev == null，说明只有一个节点。
            first = null;
        else
        // 相当于置空 f 在前驱节点的引用。
            prev.next = null;
        size--;
        modCount++;
        // 返回删除节点的元素。
        return element;
    }

    /**
     * 删除节点 x。
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        // 拿到首节点 x 的后驱节点。
        final Node<E> next = x.next;
        // 拿到首节点 x 的前驱节点。
        final Node<E> prev = x.prev;

        if (prev == null) {
        // prev == null，说明 x 是首节点，删除 x 后，它的后驱节点便是首节点。
            first = next;
        } else {
        // x 的前驱节点指向 x 的后驱节点。
            prev.next = next;
            x.prev = null; // help gc
        }

        if (next == null) {
        // next == null，说明 x 是尾节点，删除 x 后，它的前驱节点便为尾节点。
            last = prev;
        } else {
        // x 的后驱节点指向 x 的前驱节点。
            next.prev = prev;
            x.next = null; // help gc
        }

        x.item = null;
        size--;
        modCount++;
        // 返回删除节点的元素
        return element;
    }

    /**
     * 返回首节点的元素，若为 null，则抛出 NoSuchElementException。
     */
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }

    /**
     * 返回尾节点的元素，若为 null，则抛出 NoSuchElementException。
     */
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

    /**
     * 删除首节点，若首结点为 null，而抛出 NoSuchElementException。
     */
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        // 返回被删除的元素。
        return unlinkFirst(f);
    }

    /**
     * 删除尾节点，若尾结点为 null，而抛出 NoSuchElementException。
     */
    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        // 返回被删除的元素。
        return unlinkLast(l);
    }

    /**
     * 插入一个包含元素 e 的新节点作为首节点。
     */
    public void addFirst(E e) {
        linkFirst(e);
    }

    /**
     * 插入一个包含元素 e 的新节点作为尾节点。
     */
    public void addLast(E e) {
        linkLast(e);
    }

    /**
     * 链表是否包含指定元素。
     */
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    /**
     * 返回元素个数。
     */
    public int size() {
        return size;
    }

    /**
     * 在链表尾部插入一个包含元素 e 的新节点。
     */
    public boolean add(E e) {
        linkLast(e);
        return true;
    }

    /**
     * 遍历链表，删除第一次包含指定元素 o 的节点。
     * 若删除成功返回 true，反之返回 false。
     */
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    /**
     * 按照 Collection 的 iterator 返回的顺序将所有元素插入到链表的尾节点之后。
     */
    public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }

    /**
     *按照 Collection 的 iterator 返回的顺序在此链表第 index 个节点前插入所有元素，第 index 个节点链上 c 的最后一个元素的节点。
     */
    public boolean addAll(int index, Collection<? extends E> c) {
        // 检查位置索引是否在有效范围内。0 ~ size 之间。
        checkPositionIndex(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        // 容器 c 元素个数为 0，直接 return false 表示添加失败。
        if (numNew == 0)
            return false;

        // pred 用于指向不断新增的集合 c 元素的节点。 
        // succ 指向链表第 index 个节点（若存在的话）。
        Node<E> pred, succ;
        if (index == size) {
            // index == size，说明在表尾追加元素。
            succ = null;
            pred = last;
        } else {
            // 遍历拿到第 index 个节点。
            succ = node(index);
            // 拿到第 index 个节点的前驱节点（可能为 null）。
            pred = succ.prev;
        }
        
        // 遍历集合 a，按 iterator 顺序插入链表。
        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            // 新节点的前驱节点为 pred。
            Node<E> newNode = new Node<>(pred, e, null);
            if (pred == null)
                first = newNode;
            else
                // pred 的后驱节点为新节点。
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            // succ == null，说明在表尾追加元素。因此尾节点是 c 的最后一个元素节点。
            last = pred;
        } else {
            // c 集合的最后一个节点的后驱节点指向原第 index 个节点。
            pred.next = succ;
            // 原第 index 个的前驱节点指向 c 集合的最后一个节点。
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }

    /**
     * 删除链表中所有节点。
     */
    public void clear() {
        // 从首节点开始，遍历置空。
        for (Node<E> x = first; x != null; ) {
            // 暂存当前节点的后继节点，用于遍历。
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }
        first = last = null;
        size = 0;
        modCount++;
    }

    /**
     * 返回此链表中第 index 个节点的元素。
     */
    public E get(int index) {
        // 判断参数是否是现有元素的索引（0 ~ size-1）
        checkElementIndex(index);
        // none(index) 返回第 index 个节点。
        return node(index).item;
    }

    /**
     * 替换此链表中第 index 个节点中的元素为 e，并返回被替换的元素。
     */
    public E set(int index, E element) {
        // 校验元素索引是否在现有元素范围内（0 ~ size-1）
        checkElementIndex(index);
        // 拿到第 index 个节点 x。
        Node<E> x = node(index);
        // x 赋值新值
        E oldVal = x.item;
        x.item = element;
        // 返回旧元素
        return oldVal;
    }

    /**
     * 将指定元素 element 插入链表第 index 个节点前。
     */
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    /**
     * 删除链表中第 index 个节点并返回被删除节点的元素。
     */
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }

    /**
     * 判断参数是否是现有元素的索引
     */
    private boolean isElementIndex(int index) {
        return index >= 0 && index < size;
    }

    /**
     * 判断参数是否是迭代器或添加操作的有效位置的索引。
     */
    private boolean isPositionIndex(int index) {
        return index >= 0 && index <= size;
    }

    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private void checkPositionIndex(int index) {
        if (!isPositionIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    /**
     * 返回第 index 个节点。
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);

        // 索引节点需要通过遍历的形式，所以先判断索引的位置，若在前半部分则从首开始遍历，反之从尾结点遍历，算是一个小优化。
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

    /**
     * 返回链表中第一个节点元素等于 o 的索引，如果链表不包含该元素，则返回 -1。
     */
    public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }

    /**
     * 返回链表中倒序遍历第一个节点元素等于 o 的索引，如果链表不包含该元素，则返回 -1。
     */
    public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }

    /**
     * 返回首节点的元素，与 element() 和 getFirst() 的不同在于，首节点为 null 时返回 null，不抛异常。
     */
    public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }

    /**
     * 返回首节点的元素，同 getFirst()。
     *
     */
    public E element() {
        return getFirst();
    }

    /**
     * 删除首节点并返回节点中的元素，若首节点为 null，则返回 null。
     */
    public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }

    /**
     * 删除首节点并返回节点中的元素，若首节点为 null，则抛异常。
     */
    public E remove() {
        return removeFirst();
    }

    /**
     * 插入包含元素 e 的新节点作为原尾结点的后驱节点。
     */
    public boolean offer(E e) {
        return add(e);
    }

    /**
     * 插入包含元素 e 的新节点作为原首结点的前驱节点。
     */
    public boolean offerFirst(E e) {
        addFirst(e);
        return true;
    }

    /**
     * 添加包含元素 e 的新节点至当前尾结点。等价于 offer()。
     */
    public boolean offerLast(E e) {
        addLast(e);
        return true;
    }

    /**
     * 返回首节点的元素，若节点不存在则返回 null。
     */
    public E peekFirst() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
     }

    /**
     * 返回尾节点的元素，若节点不存在则返回 null。
     */
    public E peekLast() {
        final Node<E> l = last;
        return (l == null) ? null : l.item;
    }

    /**
     * 删除首节点，并返回删除节点的元素，若节点不存在则返回 null。
     */
    public E pollFirst() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }

    /**
     * 删除尾节点，并返回删除节点的元素，若节点不存在则返回 null。
     */
    public E pollLast() {
        final Node<E> l = last;
        return (l == null) ? null : unlinkLast(l);
    }

    /**
     * 插入包含元素 e 的新节点作为原首结点的前驱节点。
     */
    public void push(E e) {
        addFirst(e);
    }

    /**
     * 删除首节点并返回节点中的元素，若首节点为 null，则抛异常。
     */
    public E pop() {
        return removeFirst();
    }

    /**
     *  正序遍历链表，删除第一个节点元素等于 o 的节点，若删除成功则返回 true，反之返回 false。
     */
    public boolean removeFirstOccurrence(Object o) {
        return remove(o);
    }

    /**
     * 倒序遍历链表，删除第一个节点元素等于 o 的节点，若删除成功则返回 true，反之返回 false。
     */
    public boolean removeLastOccurrence(Object o) {
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    @SuppressWarnings("unchecked")
    private LinkedList<E> superClone() {
        try {
            return (LinkedList<E>) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }
    }

    /**
     * 返回当前实例的浅拷贝。
     */
    public Object clone() {
        LinkedList<E> clone = superClone();

        // Put clone into "virgin" state
        clone.first = clone.last = null;
        clone.size = 0;
        clone.modCount = 0;

        // Initialize clone with our elements
        for (Node<E> x = first; x != null; x = x.next)
            clone.add(x.item);

        return clone;
    }

    /**
     * 返回一个包含了所有元素的新数组。
     */
    public Object[] toArray() {
        Object[] result = new Object[size];
        int i = 0;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        return result;
    }

    /**
     * 将所有元素赋值到新数组 a 中。
     */
    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        // 如果数组大小小于元素个数，则重新实例化一个大小为元素数量的新数组。
        if (a.length < size)
            a = (T[])java.lang.reflect.Array.newInstance(
                                a.getClass().getComponentType(), size);
        int i = 0;
        Object[] result = a;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;

        if (a.length > size)
            a[size] = null;

        return a;
    }

}
```

## 2.3 HashMap

由于 HashMap 的源码相对复杂了一些，因此拆分为几个部分阅读。

### 2.3.1 成员变量和构造函数

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    // 序列号
    private static final long serialVersionUID = 362498820763181265L;    
    // 默认的初始容量是 16。
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   
    // 最大容量。
    static final int MAXIMUM_CAPACITY = 1 << 30; 
    // 默认的填充因子。
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 当桶 (bucket) 上的结点数大于这个值时会转成红黑树。
    static final int TREEIFY_THRESHOLD = 8; 
    // 当桶 (bucket) 上的结点数小于这个值时树转为链表。
    static final int UNTREEIFY_THRESHOLD = 6;
    // 桶中结构转化为红黑树对应的 table 的最小大小。
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 存储元素的数组，也是 HashMap 的原理所在。
    // 数组容量任何时候总是 2 的幂次倍。
    // 原因在于（n 代表 table 的容量）：使用 (n - 1) & hash 确定 hash 值在数组中的索引时，(n-1) 的二进制一定是 1111111***111 形式的，从而在按位与的时候，能够充分的散列（利用到每一位），使得添加的节点均匀分布在每个位置上。
    transient Node<k,v>[] table; 
    // 存放具体节点的 Set。
    transient Set<map.entry<k,v>> entrySet;
    // 存放元素的个数，注意这个不等于数组的长度。
    transient int size;
    // 每次扩容和更改 map 结构的计数器。
    transient int modCount;   
    // 临界值（capacity * loadFactor）。当实际大小 (容量*填充因子) 超过临界值时，会进行扩容。
    int threshold;
    // 加载因子。
    final float loadFactor;

     public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; 
    }

     public HashMap(Map<? extends K, ? extends V> m) {
         this.loadFactor = DEFAULT_LOAD_FACTOR;
         putMapEntries(m, false);
    }

    /**
     * 指定默认容量大小的构造函数，若要设置刚好包括 N 个元素的容量大小，可通过 N/ loadFactor 计算得到初始容量。
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 指定 初始容量大小 和 加载因子 的构造函数。
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

   final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            // 判断 table 是否已经初始化
            if (table == null) { // pre-size
                // 计算刚好容纳 Map 元素数量的数组容器大小。
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            // 已初始化，并且 m 元素个数大于阈值，则扩容。
            else if (s > threshold)
                resize();
            // 将 m 中的所有元素添加至 HashMap 中。
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
}
```

- loadFactor：控制数组存放数据的疏密程度，loadFactor 越趋近于 1，那么数组中存放的数据 (entry) 也就越多，也就越密，也就是会让链表的长度增加，loadFactor 越小，越趋近于 0，数组中存放的数据 (entry) 也就越少，也就越稀疏。

loadFactor 太大导致查找元素效率低，太小导致数组的利用率低，存放的数据会很分散。loadFactor 的默认值为 0.75f 是官方给出的一个比较好的临界值。

给定的默认容量为 16，负载因子为 0.75。Map 在使用过程中不断的往里面存放数据，当数量达到了 16 * 0.75 = 12 就需要将当前 16 的容量进行扩容，而扩容这个过程涉及到 rehash、复制数据等操作，所以非常消耗性能。

### 2.3.2 存储结构

JDK1.8 之前 HashMap 由 **数组+链表** 组成，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的，遇到哈希冲突，则将冲突的元素链到链表中即可。JDK1.8 及以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）时，将链表转化为 [红黑树](https://www.jianshu.com/p/e136ec79235c)，以减少搜索时间。

```java
// 链表结构
static class Node<K,V> implements Map.Entry<K,V> {
       final int hash;  // 哈希值，存放元素到 hashmap 中时用来与其他元素 hash 值比较。
       final K key;  // 键
       V value;  // 值
       // 指向下一个节点
       Node<K,V> next;
       Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
        // 重写了 hashCode() 方法
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
        // 重写了 equals() 方法
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
}
// 红黑树结构
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // 父
        TreeNode<K,V> left;    // 左
        TreeNode<K,V> right;   // 右
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;           // 判断颜色
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
```

### 2.3.3 put、get、resize

**（1）put**

```java
public V put(K key, V value) {
    // 调用 hash(key) 得到 key 的 hash 值。
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    // 通过 key 的 hashCode() 计算 hash 值。
    // 当 key == null 时，hash 值 为 0，在 putVal() 可由 (n - 1) & hash 可得值为 0，因此 null key 一定在数组的第一个桶内（索引为 0）。
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

/**
 * @param onlyIfAbsent true：不改变已经存在的节点（以 key 作为标识是否存在）。
 * @return 返回被替换的旧值
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // tab 即为 table ，若未初始化或者长度为 0，进行扩容。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 通过 (n - 1) & hash 计算元素存放在哪个桶中。
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 若桶为空，生成一个新节点放入桶中。
        tab[i] = newNode(hash, key, value, null);
    else {
        // 桶中已经存在元素。
        Node<K,V> e; K k;
        // 比较桶中第一个元素的 key 值和 key 的 hash 值是否与新元素相等。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 用 e 暂存当前桶的第一个节点 p。
            e = p;
        else if (p instanceof TreeNode)
            // 第一个节点的 key 与新增的 key 不相等，且节点是 TreeNode（红黑树结构）。
            // 插入一个新的 TreeNode 节点并赋值给 e。
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 第一个节点的 key 与新增的 key 不相等，且节点是 Node（链表结构）。
            // 遍历至链表表尾后插入结点
            for (int binCount = 0; ; ++binCount) {
                // 到达链表的尾部
                if ((e = p.next) == null) {
                    // 在尾部插入新结点
                    p.next = newNode(hash, key, value, null);
                    // 节点数量达到阈值，转化为红黑树。-1 是因为从 0 开始计数。
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    // 跳出循环。
                    break;
                }
                // 当前节点的 key 值与插入的元素的 key 值是否相等。即遍历查找 key 相等的节点。
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的 e = p.next 组合遍历链表。
                p = e;
            }
        }
        // 若 e != null 则表示在桶中找到 key 值、hash 值与插入元素的相等的节点。
        if (e != null) { 
            // 记录 e 的 value
            V oldValue = e.value;
            // onlyIfAbsent 传入时为 false。
            if (!onlyIfAbsent || oldValue == null)
                // 用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 元素结构修改
    ++modCount;
    // 大于阈值则扩容
    if (++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
} 
```

**（2）get**

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 数组不为 null 且数组长度不为 0 且相应的桶存在节点。
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 首节点是要找的 value 则直接返回。
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 从红黑树中查找相应的节点。
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 遍历链表根据 key 获取相应的节点。
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

**（3）resize**

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    // 旧的数组容量。
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 旧的数组容量阈值。
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 容量超过最大值就不再扩充了，（结果就是 hash 碰撞会变多）。
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新容量等于旧容量乘以 2（为什么是 2 的倍数，在成员变量介绍有讲）。
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
        // 新容量小于最大值且旧容量大于等于默认容量大小，则设置新的容量阈值为旧阈值的两倍。
            newThr = oldThr << 1; 
    }
    else if (oldThr > 0)
        // 说明首次添加元素且实例化时指定了初始元素个数。
        newCap = oldThr;
    else { 
        // 说明首次添加元素且使用了无参构造函数实例化。
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算新的数组容量阈值（从上面可得有 2 种情况会 newThr == 0）
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    // 赋值新的数组容量阈值。
    threshold = newThr;

    // 每次扩容都会使用新的数组并复制数据到新数组，部分节点还需要重新计算 hash，因此应该尽量减少扩容的次数。
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 遍历旧的数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 提取桶中的节点。
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // help gc
                if (e.next == null)
                // 说明桶里只有一个节点。
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                     // 按原始链表顺序，过滤出来扩容后位置不变的元素（低位=0），放在一起。
                     // loHead 是过滤后的头节点，loTail 用于不断链接节点。
                    Node<K,V> loHead = null, loTail = null;
                    // 按原始链表顺序，过滤出来扩容后位置改变到（index+oldCap）的元素（高位=0），放在一起。
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        // 暂存下一个节点，用于遍历。
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 当前桶中位置不变的节点继续放在当前桶中。
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 当前桶中位置变化的节点放在索引为（index+oldCap）的桶中。
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

### 2.3.4 线程安全方案 HashTable、ConcurrentHashMap、Collections.synchronizedMap() 对比

**（1）HashTable**

HashTable 也是使用的 **数组+链表** 的数据结构，使用 synchronized 来保证线程安全，效率相对低下。当一个线程访问同步方法时，其他线程也访问同步方法时，可能会进入阻塞或轮询状态，例如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，会导致锁竞争会越来越激烈从而导致效率越低。

```java
public synchronized V put(K key, V value) {
        // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    HashtableEntry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    HashtableEntry<K,V> entry = (HashtableEntry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}

public synchronized V get(Object key) {
    HashtableEntry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (HashtableEntry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

**（2）ConcurrentHashMap**

在 JDK 1.7 的时候，ConcurrentHashMap（分段锁）对整个桶数组进行了分割分段 (Segment)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，会大大减少锁竞争的可能性，提高并发访问率。 到了 JDK1.8 的时候已经摒弃了 Segment 的概念，而是直接使用 **数组+ Node 链表+红黑树** 的数据结构实现，并发使用 synchronized 和 CAS 来控制。（JDK1.6 以后对 synchronized 做了很多优化，从而降低了 synchronized 的性能开销），虽然在 JDK1.8 中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本的方法。

**（3）Collections.synchronizedMap()**

所有的 Collections.synchronizedXXX() 方法都是锁住和数据相关的 API 代码块，和 Vecotor 锁方法类似，不同在于可以指定锁对象。

```java
// 重写相应的方法，并锁住相应的代码块，和 Vecotor 锁方法类似，不同在于可以指定锁对象。
public V put(K key, V value) {
    synchronized (mutex) {
        return m.put(key, value);
    }
}
```

**（4）小结**

1. 对 Null key 和 Null value 的支持：HashTable、ConcurrentHashMap 中的键值不能为 null，若为 null 会直接抛出 NullPointerException。而 Collections.synchronizedMap() 取决于传入的 Map。
2. 扩容方面：为了使 Hash 值能够更均匀的分布，数组的扩容皆为原来容量的 2 倍。
3. 性能方面：没有并发需求的情况下，优先使用 HashMap。低并发需求的情况下使用 Collections.synchronizedList() 即可，高并发需求则使用 ConcurrentHashMap。
4. 拓展性：Collections.synchronizedList() 支持设置锁对象，因此拓展性更好。
5. 3 个并发集合看似已经线程安全，但 Iterator 例外，在使用 Iterator 的时候，需要对整个迭代过程加锁，否则在迭代过程使用非 Iterator 修改数据会抛 ConcurrentModificationException 异常。
