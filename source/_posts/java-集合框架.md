---
title: java-集合框架
date: 2020-09-10 16:54:28
tags:
    [集合]
categories: Java
---

# 1. 集合
集合可以看作是一个容器，用来储存**对象**信息。我们都知道数组可以用来保存**多个数据或者对象**的，那为什么java中还有着比如**List，Map，Set**这样的集合类呢？说明数组有一定的局限性，比如：
*  数组长度是固定的,不能自动增长
` String[] array3=new String[5];`  `String[] array={"hello","world"}`;
无论是动态初始化还是静态初始化，在创建的时候，我们就需要声明数组的大小，这样虚拟机才会分配一个连续的内存空间来存储数组。假如此时我又有了新的数据，但是由于我们的数组已经满了，因此新的数据无法保存，我们只能创建一个更大的数组来保存数组，而无法在原有基础上进行扩展。
* 其次，我们在数组里保存了很多数据，但是只是单一的数据而已，如果我们需要保存映射的关系，比如**Key-value**关系，那么数组就无法满足我们了。

针对这些，java在`java.util`里提供了集合类。

<!--more-->

# 2.集合类的基本框架
* 集合类主要有四种Set,Queue,List,Map
* Set表示无序且元素互异的集合，有点像数学里面学到的集合
* List表示有序的且集合内元素可重复的集合
* Map表示有映射关系的集合
* Queue表示队列
（队列：一种只允许在**表的前端进行删除操作**且在**表的后端进行插入操作**的线性表。队列遵循先进先出原则（first in first out））
![队列单向](https://img-blog.csdnimg.cn/20200626192628335.png)
他们的基本关系有以下两个图表示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200627161605728.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoYXJtaW5nY2o=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/202006271621555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoYXJtaW5nY2o=,size_16,color_FFFFFF,t_70)
# 3.Colloction接口
从上面的图可以看到Set，Queue，List都实现了Collection的接口
* `int size();` 返回集合中元素的个数
* ` boolean isEmpty() `判断集合是否为空
* `boolean contains（object o）`判断是否存在指定元素
* `Iterator<E> iterator();`返回一个Iterator对象，用来遍历集合中的元素
* `Object[] toArray();`返回一个包含这个集合所有元素的Object类数组（若集合有序的话，返回的数组是同序的）
* `<T> T[] toArray(T[] a);`和前面一个方法作用是一样的，不过返回的是指定的泛型T的一个数组
* ` boolean add(E e);`向集合添加一个元素
* `boolean remove（Object o)`删除指定 的元素，如果包含一个或者多个元素o,则这些元素都会被删除
* `boolean containsAll(Collection<?> c);`判断是否包含集合c中的全部元素

里面方法很多不一一列举
# 4.Iterator迭代器
* 查看Collection接口源码，可以看到`public interface Collection<E> extends Iterable<E>`
* 实现iterable接口的原因就是为集合类创立一个iterator对象，可以遍历并选择序列中的对象，而开发人员不需要了解该序列的底层结构 ( 迭代器把访问逻辑从不同类型的集合类中抽取出来，从而避免向外部暴露集合的内部结构)
Iterator又称为遍历器，通常我们可以通过它来遍历Collecion集合中的元素，但是java中的iterator比较简单，只能单向遍历。
1) 使用方法iterator()要求容器返回一个Iterator。第一次调用Iterator的next()方法时，它返回序列的第一个元素。注意：iterator()方法是java.lang.Iterable接口,被Collection继承。

2) 使用next()获得序列中的下一个元素。

3) 使用hasNext()检查序列中是否还有元素。

4) 使用remove()将迭代器新返回的元素删除。
# 5.Set集合
Set是Collection的一个子接口`public interface Set<E> extends Collection<E> `

## 特点
1. 无序，没有前后顺序的分别（这里指添加的时候的顺序），所有的元素没有位置的概念，所有的元素都在集合中。（好比一个罐子里面装东西一样，不分顺序，都装在这个罐子里）
2. 无索引，每个元素没有特定的编号
3. 不可重复，元素只有值的区别，没有位置的区别，如果重复，无法区分。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200626225253579.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoYXJtaW5nY2o=,size_16,color_FFFFFF,t_70)

结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200626225345534.png)
这就很好的反映了set的无序和互异的特性
## 方法
Set本身和Collection的源码一样，没有什么特别的方法
## 实现类
### 1. HashSet
* HashSet实现元素有序不重复的机制
>HashSet会调用该对象的hashCode（）方法来得到该对象的hashCode值，
然后根据该hashCode值决定该对象在hashSet中的存储位置。如果有两个元素通过equals()方法比较返回true,
 	但它们的hashCode()返回值不相等，HashSet将会把它们存储在不同的位置，依然可以添加成功。
 	也就是说HashSet集合判断两个元素相等的标准是两个对象通过equals()方法比较相等,并且两个对象的hashCode()方法返回值也相等。

举例：

```java
public class HashSetTest {

    public static void main(String[] args){

        HashSet<Person> set=new HashSet<>();
        set.add(new Person(10,"abc"));
        set.add(new Person(10,"a"));
        set.add(new Person(10,"a"));

        System.out.println(set.toString());
    }
}
```
> [Person{age=10, name='abc'}, Person{age=10, name='a'}, Person{age=10, name='a'}]

重写方法hashCode和equals方法

```java
@Override
    public boolean equals(Object o) {
        System.out.println("调用了equals方法");
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        if(((Person) o).age==person.getAge()&&((Person) o).name==person.getName()) return true;
        else return false;
    }

    @Override
    public int hashCode() {
        System.out.println("调用了hashCode方法");
        return 55;
    }
```
结果：
>调用了hashCode方法
调用了hashCode方法
调用了equals方法
调用了hashCode方法
调用了equals方法
[Person{age=10, name='abc'}]

### 2. LinkedHashSet
* 是Hashet的子类，具有可预知迭代顺序的 Set 接口的哈希表和链接列表实现，即按照将元素插入到 set 中的顺序（插入顺序）进行迭代。
* LinkedHashSet集合同样是根据元素的hashCode值来决定元素的存储位置，但是它同时使用链表维护元素的次序。这样使得元素看起 来像是以插入顺序保存的，也就是说，当遍历该集合时候，LinkedHashSet将会以元素的添加顺序访问集合的元素。
* LinkedHashSet在迭代访问Set中的全部元素时，性能比HashSet好，但是插入时性能稍微逊色于HashSet。
### 3. TreeSet
TreeSet 是一个有序的集合，它的作用是提供有序的Set集合。它继承于AbstractSet抽象类，实现了NavigableSet, Cloneable, java.io.Serializable接口。
* TreeSet 继承于AbstractSet，所以它是一个Set集合，具有Set的属性和方法。
* TreeSet 实现了NavigableSet接口，意味着它支持一系列的导航方法。比如查找与指定目标最匹配项。
* TreeSet 实现了Cloneable接口，意味着它能被克隆。
* TreeSet 实现了java.io.Serializable接口，意味着它支持序列化。

* TreeSet是基于TreeMap实现的。TreeSet中的元素支持2种排序方式：自然排序 或者 根据创建TreeSet 时提供的 Comparator 进行排序。这取决于使用的构造方法。
* TreeSet为基本操作（add、remove 和 contains）提供受保证的 log(n) 时间开销。

简单看看源码：

```java
  // 带比较器的构造函数。
    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<E,Object>(comparator));
    }
```

```java
  // 返回TreeSet的逆序排列的迭代器。
    // 因为TreeSet时TreeMap实现的，所以这里实际上时返回TreeMap的“键集”对应的迭代器
    public Iterator<E> descendingIterator() {
        return m.descendingKeySet().iterator();
    }

```

```java
  // 返回Set的比较器
    public Comparator<? super E> comparator() {
        return m.comparator();
    }

    // 返回Set的第一个元素
    public E first() {
        return m.firstKey();
    }

    // 返回Set的最后一个元素
    public E first() {
    public E last() {
        return m.lastKey();
    }

    // 返回Set中小于e的最大元素
    public E lower(E e) {
        return m.lowerKey(e);
    }

    // 返回Set中小于/等于e的最大元素
    public E floor(E e) {
        return m.floorKey(e);
    }

    // 返回Set中大于/等于e的最小元素
    public E ceiling(E e) {
        return m.ceilingKey(e);
    }

    // 返回Set中大于e的最小元素
    public E higher(E e) {
        return m.higherKey(e);
    }
```
写了一个简单的例子：

```java
public class TreeSetTest {
    public static void main(String[] args){
//        构造TreeSet时，传入person的比较年龄的方法
        TreeSet<Person> tree=new TreeSet<>(Person::compareTo);
        tree.add(new Person(10,"a"));
        tree.add(new Person(20,"b"));
        tree.add(new Person(30,"c"));
        tree.add(new Person(40,"d"));
//        年龄较小，而TreeSet是有序列的，无法放到集合中
        tree.add(new Person(10,"e"));
        System.out.println(tree);
//        返回年龄小于30的人里年龄最大的人
        System.out.println("Lower a:"+tree.lower(new Person(30,"n")));
//        TreeSet不支持快速随机遍历，只能通过迭代器进行遍历
//        可以倒序遍历
        Iterator iterator=tree.descendingIterator();
        while (iterator.hasNext()) System.out.println(iterator.next());


    }
```
输出的结果：
>[Person{age=10, name='a'}, Person{age=20, name='b'}, Person{age=30, name='c'}, Person{age=40, name='d'}]
Lower a:Person{age=20, name='b'}
Person{age=40, name='d'}
Person{age=30, name='c'}
Person{age=20, name='b'}
Person{age=10, name='a'}

# 6.Queue队列
继承了Collection接口，方法比较简单，都是针对队列性质的操作
>add(E e):添加一个元素
remove():删除一个元素
offer(E e):添加一个元素
poll(E e):删除一个元素
peek():查看最上一个元素
element() ：查看最上一个元素,与上面不同的是会在队列为空的时候抛出一个异常

## 实现类: PriorityQueue
PriorityQueue是一个比较标准的队列实现类。之所以说它是比较标准的队列实现，而不是绝对标准的队列实现，是因为PriorityQueue保存队列元素的顺序并不是按加入队列的顺序，而是会按队列元素的大小或指定的比较器进行重新排序(像是二叉树排列)。因此当调用peek()方法或者poll()方法取出队列中的元素时，并不是取出最先进入队列的元素，而是取出队列中最小的元素。这就违反队列的最基本原则：先进先出(FIFO)。

```java
public class QueueTest {
    public static void main(String[] args){
        PriorityQueue priorityQueue1=new PriorityQueue();
        priorityQueue1.offer(-2);
        priorityQueue1.offer(10);
        priorityQueue1.offer(5);
        priorityQueue1.offer(6);
        priorityQueue1.offer(7);
        priorityQueue1.offer(8);

        System.out.println(priorityQueue1);
        priorityQueue1.poll();
        System.out.println(priorityQueue1);
    }

}

```
>[-2, 6, 5, 10, 7, 8]
[5, 6, 8, 10, 7]

# 7.List列表
几个Collection接口里没有的方法:

```java
E get（int index）：返回列表中指定位置的元素。
E set（int index，E element）：用指定的元素替换列表中指定位置的元素。
List <E> subList（int fromIndex，int toIndex）：返回指定fromIndex（包含）和toIndex（不包括）之间的此
列表部分的视图。返回的列表由此列表支持，因此返回列表中的非结构更改将反映在此列表中，反之亦然。
default void replaceAll（UnaryOperator <E>运算符）：将此列表的每个元素替换为将运算符应用于该元素的结果。
default void sort（Comparator <super E> c）：根据指定的Comparator引发的顺序对此列表进行排序。
default Spliterator <E> spliterator（）：在此列表中的元素上创建Spliterator。
```
有两种方法可以对列表进行排序。我们可以使用Collections类进行自然排序，或者我们可以使用List sort（）方法并使用我们自己的Comparator进行排序

## 实现类
### 1. ArrayList
底层是基于动态数组，根据下表随机访问数组元素的效率高，向数组尾部添加元素的效率高；但是，删除数组中的数据以及向数组中间添加数据效率低，因为需要移动数组。例如最坏的情况是删除第一个数组元素，则需要将第2至第n个数组元素各向前移动一位。而之所以称为动态数组，是因为Arraylist在数组元素超过其容量大，Arraylist可以进行扩容（针对JDK1.8  数组扩容后的容量是扩容前的1.5倍），Arraylist源码中最大的数组容量是Integer.MAX_VALUE-8

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
### 2. LinkedList


LinkedList底层的数据结构是基于双向链表的，且头结点中不存放数据,如下： 
![在这里插入图片描述](https://img-blog.csdnimg.cn/202006271619086.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoYXJtaW5nY2o=,size_16,color_FFFFFF,t_70)
既然是双向链表，那么必定存在一种数据结构——我们可以称之为节点，节点实例保存业务数据，前一个节点的位置信息和后一个节点位置信息，如下图所示： 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200627162004574.png)


* **具体源码**

```java
 transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;

```


```java
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
节点类很简单，element存放业务数据，prev与next分别存放前后节点的信息.具体的各种方法的实现不详细解释。

LinkedList 和 ArrayList 一样，都实现了 List 接口，但其内部的数据结构有本质的不同。LinkedList 是基于链表实现的（通过名字也能区分开来），所以它的插入和删除操作比 ArrayList 更加高效。但也是由于其为基于链表的，所以随机访问的效率要比 ArrayList 差。

# 8.Map图
## 特点
* 给定一个键和一个值，你可以将该值存储在一个Map对象. 之后，你可以通过键来访问对应的值。
* 当访问的值不存在的时候，方法就会抛出一个NoSuchElementException异常.
* 当对象的类型和Map里元素类型不兼容的时候，就会抛出一个 ClassCastException异常。
* 当在不允许使用Null对象的Map中使用Null对象，会抛出一个NullPointerException 异常。
* 当尝试修改一个只读的Map时，会抛出一个UnsupportedOperationException异常。
## 方法
插入元素

```java
map.put("key1", "value1");
```

获取元素

```java
map.get("key1")
```

移除元素

```java
map.remove("key1");
```

清空map

```java
map.clear();
```

## 实现类（内容有点多，下次再写完）