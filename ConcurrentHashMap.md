# ConcurrentHashMap

ConcurrentHashMap被设计用于多线程的并发操作。它与HashTable最大的不同在于，HashTable只是简单地把每个方法添加synchronized,这样粗暴的方式使得HashTable在并发访问的时候性能非常低下。

ConcurrentHashMap底层实现采用一个bucket数组存储Node<K,V>。
当往一个空bin中插入第一个节点时，使用cas来用步操作。当出现hash冲突时，会以每个链表的表头作为对象锁。
这样就避免了每个操作都锁住整个map，提高了并发效率。

其插入元素实现如下：

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
            //懒加载首次初始化table
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //当在一个空bin中插入第一个节点时，通过cas操作同步
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                //
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //当第一个节点不为空时，以改节点为对象锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            //遍历寻找key
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    //更新节点value
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                //构造新节点
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    //binCount超出threashHold,将链表转换成红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        //返回被更新值
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }    
    

