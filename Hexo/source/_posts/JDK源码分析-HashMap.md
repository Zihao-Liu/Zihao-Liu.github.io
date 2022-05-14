---
title: JDK源码分析-HashMap
date: 2022-05-14 16:37:50
tags: 
 - 源码分析
categoties:
 - 源码分析
 - JDK
---


![HashMap与HashTable类图](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/29/171c4f7451e61530~tplv-t2oaga2asx-image.image)

# 面试题

1. HashMap与HashTable的关系
2. 是否线程安全
3. 元素是否有序
4. 怎样解决哈希冲突
5. 键应该如何选择
6. 扩容时机和过程
7. 容量为什么必须是2的幂
8. 负载因子的作用
9. 什么时候会用红黑树代替链表
10. key的hashCode计算方式


不想看源码解析的同学，可以直接去最下方[查看答案](#面试题解答)

<!-- more -->

# 源码解析(JDK 1.8)

Map是一种常用的数据结构，以键值对(key-value)的形式进行存储。在Java中Map的常用实现有HashMap、HashTable、TreeMap、LinkedHashMap和ConcurrentHashMap等。本篇文章只介绍最常使用的HashMap，其他实现的分析将在其他文章中介绍。

HashMap虽然经常被称为集合类，但是并没有像List或者Set一样实现Collection接口。而只是实现了Map接口。

在JDK 1.8版本中，HashMap的底层有两种方式实现，默认情况下是基于数组与单向链表链表实现的，当超过一定阈值时，会将链表转化为红黑树，来减少时间消耗。


![HashMap默认底层实现](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/29/171c50d14e1bf1e2~tplv-t2oaga2asx-image.image)

## 底层结构

```Java
    /**
     * HashMap默认的初始容量，如果构造时没有指定容量，将会使用 1 << 4 = 16
     * 作为容量。容量必须是2的幂数，原因见面试题第7题解答
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // = 16

    /**
     *
     * 默认的最大容量，必须是2的幂
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 默认的负载因子负载因子
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 当某个槽中node的数量大于这个阈值时，这个槽中的链表会被转化为红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * 当某个槽中node的数量小于这个阈值时，这个槽中的红黑树会被转化为链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * 与TREEIFY_THRESHOLD配合使用。当Map中槽的数量大于该阈值，且TREEIFY_THRESHOLD
     * 阈值同样满足时，才会进行链表到红黑树的转化。否则会用resize替代树化
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * 底层数组。length必须是2的幂数，length即为map的capacity(不是size)
     */
    transient Node<K,V>[] table;

    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * map中键值对(key-value)的数量
     */
    transient int size;

    /**
     * 相当于map的版本号。每次对map进行结构性改造(例如，rehash)的时候，会更新版本号
     * 用来在遍历时实现fail-fast机制
     */
    transient int modCount;

    /**
     * 下一次扩容时，要增加到的容量 threshold = oldCapacity * loadFactor
     */
    int threshold;

    /**
     * 复杂因子
     */
    final float loadFactor;
```

## Node结点

```Java
    /**
     * Map默认情况下使用链表解决哈希冲突，Node为实现链表结点的数据结构
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        // Node经过计算后对应的hash值，通过key计算求出，用单独字段保存，
        // 防止每次都要计算，节省时间
        final int hash;
        final K key;
        V value;
        // 指向链表下一个结点
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

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

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
```

Map默认情况下使用链表解决哈希冲突，Node为实现链表结点的数据结构。红黑树使用TreeNode实现，在文章最后介绍。

## 工具方法

```Java
    /**
     * 扰动函数，重新计算key的hash值。
     *
     * 由于Map底层数组 N 的长度为2的幂数。计算key存放的位置时，计算公式为hash ^ (N.length - 1)，
     * 如果直接使用key的hash值，那么对于一些key来说如果只在高位有区分，由于高位
     * 不参与计算算，那么这些key都会发生碰撞，还会产生周期性重复。
     *
     * 例如，当数组长度为16时，N.length - 1为15，二进制为 1111。
     * key_1 -> 5 二进制为 101， key_2 -> 21 二进制为 10101。
     * key_1 ^ 15 = 101, key_2 ^ 15 = 101
     *
     * 所以HashMap计算key的hash值，采用了扰动函数，将key自身的hashCode的高16位与低16位
     * 异或，这样保证高位和低位都会对元素的位置产生影响
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    /**
     * 返回大于输入参数cap，且最近的2的整数次幂的数
     */
    static final int tableSizeFor(int cap) {
        // 防止cap本身就为2的幂数
        int n = cap - 1;

        // 假设n不为0，那么n的某位必为1，假设n = 0000100000，
        // 那么 n>>>1 = 0000010000，n|n>>>1 则为 0000110000，
        // 可见这一步的目的是让n中每位1的后一位也为1
        n |= n >>> 1;
        // 因为int只有32位，后面几步的步骤执行完之后，就
        // 相当使最高位1之后的所有位数字都变为1，这样n就一定为
        // 2的某次幂减1
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        
        // 假如n为0，那么就返回1，
        // 假如n不为0，那么n+1就一定是2的某次幂
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    
     // 返回map的capacity
    final int capacity() {
        // 如果table不为null，说明map已经被初始化，并且已经添加过元素，capacity为table.length
        // 如果table为null，说明没有添加过元素
        // 如果threshold > 0，说明构造map时，指定了容量，threshold就代表了容量
        // 如果threshold = 0，说明构造map时，未指定容量，就返回默认容量
        return (table != null) ? table.length : (threshold > 0) ? threshold : DEFAULT_INITIAL_CAPACITY;
    }
    
    final Node<K, V>[] resize() {
        Node<K, V>[] oldTab = table;
        // 如果table为null，说明table还未初始化，capacity为0
        int oldCap = (oldTab == null) ? 0 : oldTab.length;

        // 当table为null时，threshold为map所需的容量
        // 当table不为null时，threshold为需要扩容时的阈值
        int oldThr = threshold;
        int newCap, newThr = 0;

        if (oldCap > 0) {
            // 之前已经添加过元素，无须初始化
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 如果oldCap大于等于最大容量，无法进行扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            } else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 因为oldCap一定是2的幂数，所以oldCap左移一位相当于扩容到之前的2
                // 倍，并且newCap也一定是2的幂数。并且newCap小于最大容量时，新的阈值
                // 也为之前的2倍
                newThr = oldThr << 1;
        } else if (oldThr > 0)
            // 如果oldCap等于0，说明table并未初始化oldThr大于0，说明构造时指定了容量，
            // 新的容量即为指定的容量
            newCap = oldThr;
        else {
            // table未初始化，且未指定容量，那么就使用默认容量
            newCap = DEFAULT_INITIAL_CAPACITY;
            // 新的阈值为capacity * loadFactor
            newThr = (int) (DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }

        if (newThr == 0) {
            // 如果newThr为0，可能为两种情况：
            // 1. map之前添加过元素，且旧的容量扩容2倍后会超出最大容量
            // 2. map未添加过元素，且构造时指定了容量

            // 计算新的阈值
            float ft = (float) newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ?
                    (int) ft : Integer.MAX_VALUE);
        }
        threshold = newThr;

        // 重新哈希过程
        @SuppressWarnings({"rawtypes", "unchecked"})
        // 使用新的容量创建数组
        Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
        table = newTab;

        if (oldTab != null) {
            // 如果oldTab不为null，说明之前有元素，需要重新放置元素

            // 遍历之前的哈希槽
            for (int j = 0; j < oldCap; ++j) {
                Node<K, V> e;

                // 如果哈希槽头结点为null，说明该槽之前没有元素
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;

                    if (e.next == null)
                        // 如果之前的槽中只有一个元素，那么直接计算在newTab中应该放置
                        // 的位置，并放入新位置
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 如果哈希槽对应的是红黑树，以树的方式进行处理
                        ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                    else {
                        // 遍历链表，将之前链表的中的结点分为两个部分，分别用low和head链表保存
                        Node<K, V> loHead = null, loTail = null;
                        Node<K, V> hiHead = null, hiTail = null;
                        Node<K, V> next;

                        // 遍历链表
                        do {
                            next = e.next;

                            // 将每个结点e的hash值与oldCap相与，等于0的结点放置在low链表中
                            // 不等于0的结点放置在high链表中
                            if ((e.hash & oldCap) == 0) {
                                //
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            } else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);

                        // 低位链表中的结点还放置在之前的槽中
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }

                        // 高位链表中的结点放置在新的槽中
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

## 构造方法

```Java
    /**
     * 根据给定的容量和负载因子构造一个空Map
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                    initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                    loadFactor);
        this.loadFactor = loadFactor;
        // initialCapacity并不是真实的容量，真实的容量必须是
        // 2的幂数，所以需要通过tableSizeFor方法计算大于
        // initialCapacity的最小2的幂数
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
     * 根据给定的容量和默认的负载因子构造一个空Map
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 根据默认容量和默认的负载因子构造一个空Map
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * 创建一个HashMap，并将map m中的键值对添加到新的map中
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

## 插入/替换

```Java
    /**
     * 添加新的key-value到map中，如果key之前存在，旧的value会被覆盖
     * <p>
     * 如果key之前存在，会返回oldValue；
     * 如果key之前不存在，则返回null
     * <p>
     * 需要注意，由于HashMap允许null值的存在，所以如果key之前的value
     * 为null也会返回null，所以不能通过返回值为null，判断key之前是否存在
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }


    /**
     * @param hash          key经过扰动函数计算过的hash值
     * @param key           key
     * @param value         value
     * @param onlyIfAbsent  true-只有key之前不存在的时候，才会添加，否则不做任何改动
     *                      false-如果key之前存在，旧的value值会被覆盖
     * @param evict         false-table处于创建模式(即通过构造方法调用)
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            // 如果table为null，或者tab.length为0，则说明
            // map刚刚进行初始化，还未添加过元素，需要调用
            // resize进行初始化
            n = (tab = resize()).length;


        // (n - 1) & hash 计算哈希值(相当于 hash % n )，计算
        // 结果为key对应的槽的位置。
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 如果槽为空，这说明并未发生哈希碰撞，创建一个Node，
            // 并将对应槽的头结点置为Node
            tab[i] = newNode(hash, key, value, null);
        else {
            // 如果槽不为空，则需要处理哈希碰撞
            Node<K, V> e;
            K k;
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                // 如果槽的头结点的key与给定的key相等，直接结束。后面的代码会
                // 进行值的替换
                e = p;
            else if (p instanceof TreeNode)
                // 如果槽对应的链表已经转化为二叉树了，就使用二叉树的方法进行处理
                e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
            else {
                // 否则，遍历整个链表。
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        // 如果遍历完整个链表，没有发现相等的key，说明key之前不存在
                        // 则新创建一个Node，并插入到链表尾
                        p.next = newNode(hash, key, value, null);

                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 如果链表结点的数量超过了阈值，则转化为树
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        // key之前存在
                        break;
                    p = e;
                }
            }
            if (e != null) {
                // e != null 说明key之前存在，则将key对应的value值覆盖
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    // 查看是否可以覆盖
                    e.value = value;
                // 给LinkedHashMap提供的钩子方法，在HashMap中只是空实现
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;

        if (++size > threshold)
            // 如果size超过了阈值，就进行再哈希
            resize();

        // 给LinkedHashMap提供的钩子方法，在HashMap中只是空实现
        afterNodeInsertion(evict);
        return null;
    }

    /**
     * 将给定m中的所有元素添加到map中，如果m中的key之前存在，
     * 之前的value会被覆盖
     */
    public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }

    /**
     * @param evict false-table处于创建模式(即通过构造方法调用)
     */
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) {
                // 如果table为null，代表table刚被构造，还未添加键值对

                // table为null的时候，threshold保存的是map capacity的大小
                // 假设m的大小s为threshold，因为threshold = capacity * loadFactor，
                // 所以 threshold  / loadFactor 就求出来所需的capacity大小，
                // +1 防止因向下取整导致的容量不足
                float ft = ((float) s / loadFactor) + 1.0F;

                // 对ft向下取整，并与HashMap允许最大容量进行比较，即为所需的容量
                int t = ((ft < (float) MAXIMUM_CAPACITY) ?
                        (int) ft : MAXIMUM_CAPACITY);

                if (t > threshold)
                    // table为null的时候，threshold保存的是map capacity的大小
                    // 而不是扩容时机的阈值，所以这里t大于threshold时，说明
                    // 之前计算的capacity容量不够大，需要重新计算容量
                    threshold = tableSizeFor(t);
            } else if (s > threshold)
                // 如果map中已经存在键值对了，且m的size大于阈值，则进行再哈希
                resize();

            // 遍历添加key-value到map中
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```

## 删除

```Java
    /**
     * 删除对应的key，并返回之前的值，如果key不存在，则返回null
     */
    public V remove(Object key) {
        Node<K, V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
                null : e.value;
    }

    /**
     * @param hash       经过扰动函数计算的key的hash值
     * @param key        the key
     * @param matchValue true-只有当key与value都相等时才删除，false-只要key相等就删除
     * @param value      如果matchValue为true，那么value与这个值进行比较，matchValue为false值，该值
     *                   可以忽略
     * @param movable    false-删除时不移动其他node
     */
    final Node<K, V> removeNode(int hash, Object key, Object value,
                                boolean matchValue, boolean movable) {
        Node<K, V>[] tab;
        Node<K, V> p;
        int n, index;
        // 如果hash对应的槽存在元素才需要判断，否则说明key不存在直接返回null
        if ((tab = table) != null && (n = tab.length) > 0 &&
                (p = tab[index = (n - 1) & hash]) != null) {
            Node<K, V> node = null, e;
            K k;
            V v;
            
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                // 如果结点p的key与给定的key相等
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    // 如果是红黑树，就按红黑树处理
                    node = ((TreeNode<K, V>) p).getTreeNode(hash, key);
                else {
                    // 遍历链表，寻找与给定的key相等的结点
                    do {
                        if (e.hash == hash &&
                                ((k = e.key) == key ||
                                        (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            // node不等于null，说明key之前存在。然后判断是否需要满足matchValue的条件
            if (node != null && (!matchValue || (v = node.value) == value ||
                    (value != null && value.equals(v)))) {
                
                if (node instanceof TreeNode)
                    ((TreeNode<K, V>) node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    // 如果node是哈希槽头结点，直接把槽结点指向next
                    tab[index] = node.next;
                else
                    // 删除node结点
                    p.next = node.next;
                ++modCount;
                --size;
                // 提供给LinkedHashMap的钩子方法，在HashMap中为空实现
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }

    /**
     * 删除map中的所有键值对
     */
    public void clear() {
        Node<K, V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
```

## 查找

```Java
    public V get(Object key) {
        Node<K, V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
    final Node<K, V> getNode(int hash, Object key) {
        Node<K, V>[] tab;
        Node<K, V> first, e;
        int n;
        K k;
        
        // 如果hash对应的槽不为空才需要判断，否则说明key不存在直接返回null
        if ((tab = table) != null && (n = tab.length) > 0 &&
                (first = tab[(n - 1) & hash]) != null) {
            
            // 先检查头结点，如果头结点相等，直接返回
            if (first.hash == hash && 
                    ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            
            
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    // 按红黑树处理
                    return ((TreeNode<K, V>) first).getTreeNode(hash, key);
                // 遍历链表，查找key相等的结点
                do {
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
    
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }
    
    public boolean containsValue(Object value) {
        Node<K, V>[] tab;
        V v;
        if ((tab = table) != null && size > 0) {
            // 遍历table所有槽
            for (int i = 0; i < tab.length; ++i) {
                
                // 遍历每个槽中的每个结点
                for (Node<K, V> e = tab[i]; e != null; e = e.next) {
                    // 如果value相等，则直接返回，因为TreeNode继承了Node
                    // 所以不需要单独处理
                    if ((v = e.value) == value ||
                            (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```

## JDK 1.8 新增方法

```Java
    @Override
    public V getOrDefault(Object key, V defaultValue) {
        Node<K, V> e;
        // 返回key对应值，并用defaultValue替换null
        return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
    }

    @Override
    public V putIfAbsent(K key, V value) {
        // 1. 如果key之前不存在，就将key-value添加到map中
        // 2. 如果key之前对应的值为null，就用value替换null
        // 3. 如果key之前对应的值不为null，则什么也不做
        return putVal(hash(key), key, value, true, true);
    }

    @Override
    public boolean remove(Object key, Object value) {
        // 如果key之前存在，且值为value，则删除
        // 否则不做任何操作
        return removeNode(hash(key), key, value, true, true) != null;
    }

    /**
     * 如果key之前存在，且key之前对应的值为oldValue，则用newValue替换
     */
    @Override
    public boolean replace(K key, V oldValue, V newValue) {
        Node<K, V> e;
        V v;
        if ((e = getNode(hash(key), key)) != null &&
                ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
            e.value = newValue;
            afterNodeAccess(e);
            return true;
        }
        return false;
    }

    /**
     * 如果key之前存在，且key的值不为null。则用value替换
     */
    @Override
    public V replace(K key, V value) {
        Node<K, V> e;
        if ((e = getNode(hash(key), key)) != null) {
            V oldValue = e.value;
            e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
        return null;
    }

    @Override
    public V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction) {
        if (mappingFunction == null)
            throw new NullPointerException();
        // 通过扰动函数计算key的hash值
        int hash = hash(key);
        Node<K, V>[] tab;
        Node<K, V> first;
        int n, i;
        int binCount = 0;
        TreeNode<K, V> t = null;
        Node<K, V> old = null;

        if (size > threshold || (tab = table) == null || (n = tab.length) == 0)
            // 如果table还没有初始化，则先将table初始化
            n = (tab = resize()).length;


        if ((first = tab[i = (n - 1) & hash]) != null) {
            // 如果key对应的槽不为空

            if (first instanceof TreeNode)
                // 按红黑树处理
                old = (t = (TreeNode<K, V>) first).getTreeNode(hash, key);
            else {
                Node<K, V> e = first;
                K k;
                // 遍历链表，找到key相等结点并返回
                do {
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k)))) {
                        old = e;
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
            V oldValue;
            // 如果key之前存在，且值不为null，则直接返回oldValue
            if (old != null && (oldValue = old.value) != null) {
                afterNodeAccess(old);
                return oldValue;
            }
        }

        // 对key执行计算，得到value
        V v = mappingFunction.apply(key);

        if (v == null) {
            return null;
        } else if (old != null) {
            // key之前存在，但是oldValue为null，则替换新值
            old.value = v;
            afterNodeAccess(old);
            return v;
        } else if (t != null)
            // key之前存在且oldValue不为null，且对应槽已经转化
            // 为红黑树，则添加TreeNode
            t.putTreeVal(this, tab, hash, key, v);
        else {
            // 按正常链表处理，添加一个新结点
            tab[i] = newNode(hash, key, v, first);
            // 是否需要转化为红黑树
            if (binCount >= TREEIFY_THRESHOLD - 1)
                treeifyBin(tab, hash);
        }
        ++modCount;
        ++size;
        afterNodeInsertion(true);
        return v;
    }

    public V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (remappingFunction == null)
            throw new NullPointerException();
        Node<K, V> e;
        V oldValue;
        int hash = hash(key);
        if ((e = getNode(hash, key)) != null && (oldValue = e.value) != null) {
            // 如果key之前存在，且对应值不为null

            // 对key和oldValue进行计算，得到newValue
            V v = remappingFunction.apply(key, oldValue);

            if (v != null) {
                // 如果newValue不为null，则直接替换值
                e.value = v;
                afterNodeAccess(e);
                return v;
            } else
                // 如果newValue为null，则删除key对应结点
                removeNode(hash, key, null, false, true);
        }
        return null;
    }

    @Override
    public V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (remappingFunction == null)
            throw new NullPointerException();
        int hash = hash(key);
        Node<K, V>[] tab;
        Node<K, V> first;
        int n, i;
        int binCount = 0;
        TreeNode<K, V> t = null;
        Node<K, V> old = null;
        if (size > threshold || (tab = table) == null || (n = tab.length) == 0)
            // 如果table还没有初始化，则先将table初始化
            n = (tab = resize()).length;

        if ((first = tab[i = (n - 1) & hash]) != null) {
            // 如果key对应的槽不为空

            if (first instanceof TreeNode)
                // 按红黑树处理
                old = (t = (TreeNode<K, V>) first).getTreeNode(hash, key);
            else {
                Node<K, V> e = first;
                K k;
                // 遍历链表，找到key相等结点并返回
                do {
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        old = e;
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
        }

        V oldValue = (old == null) ? null : old.value;
        // 对key和oldValue进行计算，得到newValue
        V v = remappingFunction.apply(key, oldValue);

        if (old != null) {
            // 如果key之前存在

            if (v != null) {
                // newValue不为null，则直接替换新值
                old.value = v;
                afterNodeAccess(old);
            } else
                // 如果newValue为null，则删除旧结点
                removeNode(hash, key, null, false, true);
        } else if (v != null) {
            // 如果key之前不存在，且newValue不为null

            if (t != null)
                // 如果槽中结点已经转化为红黑树，则添加一个TreeNode
                t.putTreeVal(this, tab, hash, key, v);
            else {
                // 按正常链表处理，添加一个Node到链表结尾
                tab[i] = newNode(hash, key, v, first);
                // 是否需要转化为红黑树
                if (binCount >= TREEIFY_THRESHOLD - 1)
                    treeifyBin(tab, hash);
            }
            ++modCount;
            ++size;
            afterNodeInsertion(true);
        }
        return v;
    }

    @Override
    public V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        if (value == null)
            throw new NullPointerException();
        if (remappingFunction == null)
            throw new NullPointerException();
        int hash = hash(key);
        Node<K, V>[] tab;
        Node<K, V> first;
        int n, i;
        int binCount = 0;
        TreeNode<K, V> t = null;
        Node<K, V> old = null;
        if (size > threshold || (tab = table) == null || (n = tab.length) == 0)
            // 如果table还没有初始化，则先将table初始化
            n = (tab = resize()).length;

        if ((first = tab[i = (n - 1) & hash]) != null) {
            // 如果key对应的槽不为空

            if (first instanceof TreeNode)
                // 按红黑树处理
                old = (t = (TreeNode<K, V>) first).getTreeNode(hash, key);
            else {
                Node<K, V> e = first;
                K k;
                // 遍历链表，找到key相等结点并返回
                do {
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
                        old = e;
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
        }
        
        if (old != null) {
            // 如果key之前存在
            V v;
            if (old.value != null)
                // 如果key之前的value不为null，则通过oldValue和给定的value计算newValue
                v = remappingFunction.apply(old.value, value);
            else
                // key之前的value为null，则newValue就是给定的value
                v = value;
            
            if (v != null) {
                // 如果newValue不为null，则直接替换value
                old.value = v;
                afterNodeAccess(old);
            } else
                // 否则删除key对应结点
                removeNode(hash, key, null, false, true);
            return v;
        }
        
        // 这里value不可能为null，因为第一步判断过value为null时直接抛出异常
        // 代码运行到这里说明key之前不存在
        if (value != null) {
            if (t != null)
                // 如果槽对应的是红黑树，则添加一个TreeNode
                t.putTreeVal(this, tab, hash, key, value);
            else {
                // 按链表处理，添加一个Node到map中
                tab[i] = newNode(hash, key, value, first);
                // 是否需要转化为红黑树
                if (binCount >= TREEIFY_THRESHOLD - 1)
                    treeifyBin(tab, hash);
            }
            ++modCount;
            ++size;
            afterNodeInsertion(true);
        }
        return value;
    }

    @Override
    public void forEach(BiConsumer<? super K, ? super V> action) {
        Node<K, V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            // 遍历前先保存版本号
            int mc = modCount;
            // 遍历table所有槽
            for (int i = 0; i < tab.length; ++i) {
                // 遍历槽中链表
                for (Node<K, V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key, e.value);
            }
            // 如果版本号和之前不一致，说明遍历过程中，对map进行了结构性修改
            // 抛出异常
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }

    @Override
    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        Node<K, V>[] tab;
        if (function == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            // 遍历前先保存版本号
            int mc = modCount;
            // 遍历table所有槽
            for (int i = 0; i < tab.length; ++i) {
                // 遍历槽中链表
                for (Node<K, V> e = tab[i]; e != null; e = e.next) {
                    // 替换新值
                    e.value = function.apply(e.key, e.value);
                }
            }
            // 如果版本号和之前不一致，说明遍历过程中，对map进行了结构性修改
            // 抛出异常
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
```

## 迭代器

```Java
    abstract class HashIterator {
        // 下一个要遍历的Node
        Node<K, V> next;
        // 当前遍历的Node
        Node<K, V> current;
        // 期待版本号，实现fail-fast机制使用
        int expectedModCount;
        // current Node对应的槽的下标
        int index;

        HashIterator() {
            expectedModCount = modCount;
            Node<K, V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) {
                // 移动index到第一个非空的槽
                do {
                } while (index < t.length && (next = t[index++]) == null);
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Node<K, V> nextNode() {
            Node<K, V>[] t;
            Node<K, V> e = next;
            // 如果遍历过程中，hashMap发生过结构性更改，则快速失败，抛出异常
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            
            // 返回链表中下一个结点，如果当前槽的结点已经遍历完，就移动到下
            // 一个有结点的槽
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {
                } while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }

        public final void remove() {
            Node<K, V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }
```


## 红黑树

> 待补充

# 面试题解答

1. HashMap与HashTable的关系
    >   HashMap是非线程安全的。HashTable在get，put等方法前添加了synchronized关键字，所以是线程安全的。但是，由于在JDK1.5之后增加ConcurrentHashMap，所以HashTable不再推荐使用

2. 是否线程安全
    >   非线程安全。HashMap中的modCount字段类似于版本号的概念，在操作过程中，如果有其他线程进行结构性的操作(如put等)，会修改modCount值。触发fail-fast机制，抛出ConcurrentModificationException

3. 元素是否有序
    >   不是有序的。因为遍历时，要遍历table每个槽中的每个结点。因为resize等操作，结点所在的槽可能发生变化，所以不是有序的。
    >
    >   可以使用LinkedHashMap或者TreeMap实现有序的目的

4. 怎样解决哈希冲突
    >   HashMap采用拉链法解决哈希冲突，slot计算方式为 hash % length。当hash落在同一个槽中时，会使用链表解决冲突。同时HashMap为了防止链表过长导致的性能问题，在满足条件时，将单向链表转化为红黑树，减少查询时间

5. 键应该如何选择
    >   尽量选择Integer或者String这样的不可变类，或者重写类的hashCode和equals方法。否则，当使用可变类对象作为key放入map中时，如果对对象进行了修改，再使用该对象作为key进行操作时，就无法找到之前的key-value对了。

6. 扩容时机和过程
    >   扩容的时机：
    >   1. 第一次向map中添加元素时，需要对table进行初始化，要调用resize
    >   2. 当map中key-value键值对的数量大于threshold时，调用resize过程进行扩容
    >
    >   扩容的过程：
    >   1. 首先，将之前的容量扩充2倍(会与最大容量等比较进行修剪)
    >   2. 然后，根据新的容量计算新的阈值，并创建新的数组
    >   3. 遍历之前每个槽 j 中的结点，将每个槽中的节点通过 hash & oldCap分出高位和低位的结点。将属于低位的结点放置在原槽 j 中，将属于高位的结点放置在 oldCap+j 的槽中。

7. 容量为什么必须是2的幂
    >   拉链法计算key所在槽的位置时，通常采用 hash % length的方法(length是数组长度)。
    >
    >   但对于计算机来说，取余操作开销很大。所以Java设计师们使用了位运算。
    >
    >   当length为2的幂数时，hash ^ (length - 1) = hash % length
    >
    >   那么，既然hash要与length-1做异或操作，为什么数组大小不采用 2的幂数-1，而是用2的幂数呢。
    >
    >   因为，这样再扩容时，只需要将原来的length向左移动1位就可以了。不需要多余的操作。
    >   
    >   同时，因为2的幂数的二进制中只有1位为1，在resize过程中，只需要通过hash & oldCap是否为1，便可以将原链表中的结点分为高位和低位，解决hash冲突问题。

8. 负载因子的作用
    >   threshold的计算方式为 oldCapacity * loadFactor，loadFactor影响了扩容阈值，从而对HashMap的空间、时间性能造成影响
    >   
    >   1. 当loadFactor较小时，HashMap会偏向于扩容，容量变大后，key-value分布的就会比较稀疏，发生哈希冲突的机会会变小。因此查询性能会更优，但会耗费更多空间。同时，频繁扩容调用resize方法，会影响插入性能
    >   2. 当loadFactor较大时，特别是大于1时，HashMap需要存储更多的key-value对才能扩容，分布比较集中，容易发生哈希冲突。一个槽中的链表可能很长，影响查询时的性能，但空间耗费较少。
    >   
    >   HashMap默认采用0.75作为负载因子是基于空间、时间两个方面性能的考虑

9. 什么时候会用红黑树代替链表
    >   在put时，当槽中链表的长度超过TREEIFY_THRESHOLD时，会调用treeifyBin方法。
    >   treeifyBin会比较table长度(即map的capacity)与MIN_TREEIFY_CAPACITY。
    >   当table长度大于等于MIN_TREEIFY_CAPACITY时会转化成红黑树，否则只会resize

10. key的hashCode计算方式

    >   key.hashCode() ^ (key.hashCode() >>> 16)   
    >   
    >   通过hash扰动函数，重新计算key的hash值。
    >   由于Map底层数组 N 的长度为2的幂数。如果直接使用key的hash值，那么对于一些key来说如果只在高位有区分，那么这些key都会发生碰撞，还会产生周期性重复。
    >   
    >   所以HashMap计算key的hash值，采用了扰动函数，将key自身的hashCode的高16位与低16位向异或，这样保证高位和低位都会对元素的位置产生影响
    
    假设N.length = 16
    
![hash方法](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/30/171ca5e415d130a3~tplv-t2oaga2asx-image.image)
>   [HashMap源码注解之静态工具方法hash](https://blog.csdn.net/fan2012huan/article/details/51097331) 推荐阅读这篇文章，介绍的比较详细