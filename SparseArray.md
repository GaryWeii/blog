# SparseArray
在满足两种条件下SparseArray用于替代HashMap能获得更好的性能：
1.  数据量较小时。
2.  key为int型。

HashMap由于使用hash表来存储数据，其容量大小默认为16，并且在数据量超出容量大小乘以加载因子时以2倍扩容。并且，在扩容过程中会不断的需要做hash运算。所以当数据量刚好到达扩容的临界点时，将有一般以上的内存是浪费的。而SparseArray只在大小超出容量时才会扩容。不存在加载因子，对内存利用率更高。

SparseArray在内部通过两个数组来存储数据：
    
    private int[] keys;
    private Object[] mValues;
    
而HashMap的数据结构要复杂的多：
    
    HashMap.Node<K, V>[] table;
    Set<Entry<K, V>> entrySet;
    int size;
    int modCount;
    int threshold;
    float loadFactor;
    
从内存利用上来比较，以数据量1000为例

SparseArray所需的内存：

    4byte * 1000 + 8byte * 1000 = 12kb
 
HashMap所需内存：
 
    table = 8byte * 1000 = 8kb
    Node object =  ((对象头)16byte + 4byte + 8byte * 3) * 1000 = 44kb
    
以上计算较为粗略，但是已经可以看出在内存利用上SparseArray有明显优势。


然而，SparseArray实际上是以时间换取空间的一种实现。在数据量增大至1000以上时，SparseArray的查找效率下降显著。其原因在于SparseArray采用二分查找的方式来查找数据。

     ContainerHelpers.binarySearch(mKeys, mSize, key)

并且在后续如果有插入操作时，SparseArray可能会伴随有大量的数组复制移动操作

    if (mSize - i != 0) {
                System.arraycopy(mKeys, i, mKeys, i + 1, mSize - i);
                System.arraycopy(mValues, i, mValues, i + 1, mSize - i);
    }
