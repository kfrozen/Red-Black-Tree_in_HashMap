红黑树在HashMap中的应用

在上一篇文章：[红黑树(Red-Black Tree)解析](http://blog.csdn.net/Troy_kfrozen/article/details/78890297) 中我们了解了二叉查找树以及红黑树的概念和特性，并且对查找、插入和删除操作的实现源码进行了详细的剖析。其复杂的操作流程保证了红黑树的五条特性始终能够被满足，从而使得红黑树操作的时间复杂度为O(logN)。也正因为如此，Java的很多集合框架都引入了红黑树结构以提高性能。在JDK1.8中，我们常用的HashMap也成功傍身红黑树策马奔腾，下面就让我们一起看看HashMap是如何入手红黑树的。

这里我们依然从查找，插入和删除三个常用操作来进行分析。除去这三个操作之外还有一个地方与红黑树结构密切相关--**resize扩容操作**，关于HashMap的扩容我们在另一篇文章中有详述，这里就不再重复，有兴趣的童鞋[请戳这里（Java中集合的扩容策略及实现）](http://blog.csdn.net/Troy_kfrozen/article/details/78889947)。

- **相关成员变量**

	首先，先介绍一下相关的成员变量

	```
		//哈希表中的数组，JDK 1.8之前存放各个链表的表头。1.8中由于引入了红黑树，则也有可能存的是树的根
		transient Node<K,V>[] table;
		
		//树化阈值。JDK 1.8后HashMap对冲突处理做了优化，引入了红黑树。
		//当桶中元素个数大于TREEIFY_THRESHOLD时，就需要用红黑树代替链表，以提高操作效率。此值必须大于2，并建议大于8
		static final int TREEIFY_THRESHOLD = 8;
		
		//非树化阈值。在进行扩容操作时，桶中的元素可能会减少，这很好理解，因为在JDK1.7中，
		//每一个元素的位置需要通过key.hash和新的数组长度取模来重新计算，而1.8中则会直接将其分为两部分。
		//并且在1.8中，对于已经是树形的桶，会做一个split操作(具体实现下面会说)，在此过程中，
		//若剩下的树元素个数少于UNTREEIFY_THRESHOLD，则需要将其非树化，重新变回链表结构。
		//此值应小于TREEIFY_THRESHOLD，且规定最大值为6
		static final int UNTREEIFY_THRESHOLD = 6;
		
     	//最小树化容量。当一个桶中的元素数量大于树化阈值并请求treeifyBin操作时，
     	//table的容量不得小于4 * TREEIFY_THRESHOLD，否则的话在扩容过程中易发生冲突
    	static final int MIN_TREEIFY_CAPACITY = 64;
		
	```

- **查找**

	HashMap中的查找是最常用到的API之一，调用方法为map.get(key)，我们就从这个get方法看起：
	
	```
	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
	```
	
	可以看到这个方法调用了两个内部方法：**hash和getNode**，下面依次来看这两个方法：
	
	```
	//hash方法对传入的key进行了哈希值优化，具体做法为将key的哈希值h无符号右移16位之后与h本身按位异或，
	//相当于将h的高16位于低16位按位异或。这样做的原因在于一个对象的哈希值即使分布再松散，其低几位发生冲突的概率也较高，
	//而HashMap在计算index时正是用该方法的返回值与(length-1)按位与，结果就是哈希值的高位全归零，只保留低几位。
	//这样一来，此处的散列值优化就显得尤为重要，它混合了原始哈希值的高位与低位，以此来加大低位的松散性。
	static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    
    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key  //此处传入的就是上面hash方法的返回值，是经过优化的哈希值
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //上文提到的计算index的方法：(n - 1) & hash，first是这个数组table中index下标处存放的对象
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
            	//如果first对象匹配成功，则直接返回
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
            	//否则就要在index指向的链表或红黑树（如果有的话）中进行查找
                if (first instanceof TreeNode)
                	//如果first节点是TreeNode对象，则说明存在的是红黑树结构，这是我们今天要关注的重点
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key); 
                //否则的话就是一个普通的链表，则从头节点开始遍历查找
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
	
	重点来了，这里关注下**TreeNode.getTreeNode(hash, key)**方法，这是1.8中引入红黑树后新增的操作，它对于HashMap在哈希冲突多发，产生长链表的情况下的查找效率有着极大的提升：
	
	```
	/**
    * Calls find for root node.
    */
    //定位到树的根节点，并调用其find方法
    final TreeNode<K,V> getTreeNode(int h, Object k) {
        return ((parent != null) ? root() : this).find(h, k, null);
    }
    
    final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
        TreeNode<K,V> p = this; //p赋值为根节点，并从根节点开始遍历
        do {
            int ph, dir; K pk;
            TreeNode<K,V> pl = p.left, pr = p.right, q;
            if ((ph = p.hash) > h) //查找的hash值h比当前节点p的hash值ph小
                p = pl; //在p的左子树中继续查找
            else if (ph < h)
                p = pr; //反之在p的右子树中继续查找
            else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                return p; //若两节点hash值相等，且节点的key也相等，则匹配成功，返回p
                
        /****---- 下面的情况是节点p的hash值和h相等，但key不匹配，需继续在p的子树中寻找 ----****/
        
            else if (pl == null)
                p = pr; //若p的左子树为空，则直接在右子树寻找。若右子树也为空，则会不满足循环条件，返回null，即未找到
            else if (pr == null)
                p = pl; //反之若左子树不为空，同时右子树为空，则继续在左子树中寻找
            else if ((kc != null || (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                //若k的比较函数kc不为空，且k是可比较的，则根据k和pk的比较结果来决定继续在p的哪个子树中寻找
                p = (dir < 0) ? pl : pr;
            //若k不可比，则只能分别去p的左右子树中碰运气了，先在p的右子树pr中寻找，结果为q
            else if ((q = pr.find(h, k, kc)) != null)
                return q; //若q不为空，代表匹配成功，则返回q，结束
            else
                p = pl; //到这里表示未能在p的右子树中匹配成功，则在左子树中继续
        } while (p != null);
        //各种寻找均无果，返回null，表示查找失败。
        return null;
    }
	```
	对于HashMap的查找操作来说，如果数据足够分散，则查找效率是非常高的（时间复杂度O(1))。但是再优秀的hash算法也没法保证不发生哈希冲突，而且随着数据量的增大冲突会越发严重。由于在1.8之前是将冲突节点连成一个链表，所以在最坏情况下查找的时间复杂度会变为O(N)，红黑树的引入就是为了优化这一部分，当一个桶中的元素个数大于TREEIFY_THRESHOLD时，HashMap会将链表转变为红黑树结构，从而将操作的复杂度降为O(logN)。
	
- **插入**

	HashMap的插入操作，API为map.put(key, value)。上面我们看到了getTreeNode方法对查找操作性能的提升，这个提升得益于构建红黑树时的各种平衡操作，而构建红黑树便是在向HashMap插入节点时完成的。
	
	首先我们来看一下HashMap插入操作的流程，其实这里也是分两步走：先进行查找，如果key已经存在，则覆盖value；否则的话通过第一步查找定位到合适的插入位置，新建节点并完成插入。下面就来看put方法的源码：
	
	```
	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
	```
	
	可以看到其实是调用了内部的putVal方法，这里的hash(key)跟上文所述的是同一个方法，下面继续看putVal**（注：下文中有关resize扩容的源码在[这篇文章](http://blog.csdn.net/Troy_kfrozen/article/details/78889947)中已有详述，在此就不再重复）**：
	
	```
	/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length; //若当前哈希数组table的长度为0，则进行扩容
        //确定输入的hash在哈希数组中对应的下标i
        if ((p = tab[i = (n - 1) & hash]) == null)
        	//若数组该位置之前没有被占用，则新建一个节点放入，插入完成。
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                //若该位置上存放的第一个节点p能与输入的节点信息匹配，则将p记录为e并结束查找
                e = p;
            else if (p instanceof TreeNode)
            	//若该位置的第一个节点p为TreeNode类型，说明这里存放的是一棵红黑树，p为根节点。
            	//于是交给putTreeVal方法来完成后续操作，该方法下文会有详述
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
            	//走到这里，说明p不匹配且是一个链表的头结点，该遍历链表了
                for (int binCount = 0; ; ++binCount) {
                	//e指向p的下一个节点
                    if ((e = p.next) == null) {
                    	//若e为空，则说明已经到表尾了还未能匹配，则在表尾处插入新节点
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        	//若插入后，该桶中的节点个数已达到了树化阈值
                        	//则对该桶进行树化。该部分源码下文会有详述
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    	//匹配成功，我们需要用新的value来覆盖e节点
                        break;
                    
                    p = e; //循环继续
                }
            }
            //若执行到此时e不为空，则说明在map中找到了与key相匹配的节点e
            if (e != null) { // existing mapping for key
                V oldValue = e.value; //暂存e节点当前的值为oldValue
                if (!onlyIfAbsent || oldValue == null)
                    //若onlyIfAbsent==true，则已存在节点的value不能被覆盖，除非其value为null
                    //否则的话，用输入的value覆盖e.value
                    e.value = value;
                //钩子方法，这在HashMap中是个空方法，但是在其子类LinkedHashMap中会被Override
                //通知子类：节点e被访问过了
                afterNodeAccess(e);
                //返回已被覆盖的节点e的oldValue
                return oldValue;
            }
        }
        
        /****--执行到此处说明没有匹配到已存在节点，一定是有新节点插入--****/
        
        ++modCount; //结构操作数加一
        if (++size > threshold)
            resize(); //插入后，map中的节点数加一，若此时已达阈值，则扩容
        afterNodeInsertion(evict); //同样的钩子方法，通知子类有新节点插入
        return null; //由于是新节点插入，没有节点被覆盖，故返回null
    }
	```
	
	通过上面的代码可以清楚的看到插入操作的整体流程：
	
	a. 先通过key的hash定位到table数组中的一个桶位;
	
	b. 若此桶没有被占用，则新建节点，占坑，记录，考虑扩容，结束。若已被占用，则总是先与第一个节点进行一次匹配，若成功则无需后续的遍历操作，直接覆盖；否则的话需进行遍历；
	
	c. 若桶中的第一个节点p是TreeNode类型，则表示桶中存在的是一棵红黑树，于是后续操作将由**putTreeVal**方法来完成。否则的话说明桶中的是一个链表，则对该链表进行遍历；
	
	d. 若遍历过程中匹配到了节点e，则进行覆盖。否则的话通过遍历定位到合适的插入位置，新建节点插入，对于链表结构需考虑是否**树化**。最后进行操作记录，考虑扩容，结束。
	
	JDK1.8在上述流程对红黑树的应用体现在两个地方，**treeifyBin和putTreeValue**，分别是树化操作和向树中插入节点，对照两个方法的源码可以发现二者的实现十分相似，其实构建树的过程就是由多个插入操作组成的，此处我们通过treeifyBin方法的源码来分析下HashMap中对于红黑树插入和树化操作的实现：
	
	```
	/**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.
     */
     //这是HashMap类的实例方法，作用是将一个链表结构转化为红黑树结构
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize(); //若table数组为空或其容量小于最小树化值，则用扩容取代树化
        else if ((e = tab[index = (n - 1) & hash]) != null) { //定位到hash对应的桶位，头结点记为e
            TreeNode<K,V> hd = null, tl = null; //声明两个指针分别指向链表头尾节点
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null); //将Node类型的节点e替换为TreeNode类型的p
                if (tl == null)
                    hd = p; //若当前链表为空，则赋值头指针为p
                else {
                    p.prev = tl; //否则将p添加到链表尾部
                    tl.next = p;
                }
                tl = p; //后移尾指针
            } while ((e = e.next) != null); //循环继续
            
            if ((tab[index] = hd) != null) //将链表头节点放入table的index位置
                hd.treeify(tab); //通过treeify方法将链表树化
        }
    }
    
    
    /**
    * Forms tree of the nodes linked from this node.
    * @return root of tree
    */
    //这是TreeNode类的实例方法，以调用节点this为根节点，将链表树化
    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null; //声明root变量以记录根节点
        for (TreeNode<K,V> x = this, next; x != null; x = next) { //从调用节点this开始遍历
            next = (TreeNode<K,V>)x.next; //暂存链表中的下一个节点，记为next
            x.left = x.right = null; //当前节点x的左右子树置空
            if (root == null) {
                x.parent = null; //若root仍为空，则将x节点作为根节点
                x.red = false; //红黑树特性之一：根节点为黑色
                root = x; //赋值root
            }
            else { //否则的话需将当前节点x插入到已有的树中
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                //第二层循环，从根节点开始寻找适合x插入的位置，并完成插入操作。
                //putTreeVal方法的实现跟这里十分相似。
                for (TreeNode<K,V> p = root;;) { 
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h) //若x的hash值小于节点p的，则往p的左子树中继续寻找
                        dir = -1;
                    else if (ph < h) //反之在右子树中继续
                        dir = 1;
                    //若两节点hash值相等，且key不可比，则利用System.identityHashCode方法来决定一个方向
                    else if ((kc == null && (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                       dir = tieBreakOrder(k, pk); 

                    TreeNode<K,V> xp = p; //将当前节点p暂存为xp
                    //根据上面算出的dir值将p向下移向其左子树或右子树，若为空，则说明找到了合适的插入位置，否则继续循环
                    if ((p = (dir <= 0) ? p.left : p.right) == null) { 
                        //执行到这里说明找到了合适x的插入位置
                        x.parent = xp; //将x的parent指针指向xp
                        if (dir <= 0) //根据dir决定x是作为xp的左孩子还是右孩子
                            xp.left = x;
                        else
                            xp.right = x;
                        //由于需要维持红黑树的平衡，即始终满足其5条性质，每一次插入新节点后都需要做平衡操作
                        //这个方法的源码我们在<<红黑树(Red-Black Tree)解析>>一文中已有详细分析，此处不再重复
                        root = balanceInsertion(root, x);
                        break; //插入完成，跳出循环
                    }
                }
            }
        }
        //由于插入后的平衡调整可能会更换整棵树的根节点，
        //这里需要通过moveRootToFront方法确保table[index]中的节点与插入前相同
        moveRootToFront(tab, root);
   }
	```
	以上就是对桶中链表的树化操作treeifyBin的源码分析，其中内部的第二层循环其实就是对一个节点的插入操作，putTreeVal方法的实现与这部分十分相似，有兴趣的同学可以自行对比一下，这里就不再累述。
	
- **删除**

	HashMap的删除操作，API为map.remove(key)，其方法内部会直接调用removeNode方法。与插入操作类似，删除也是分两步，先查找定位，然后进行节点删除，若为红黑树结构，则需看情况进行删除后平衡操作或非树化操作。下面先来过一下删除节点的流程：
	
	```
	 /**
     * Implements Map.remove and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {//合法性判断，p赋值为输入的hash对应的桶中的第一个节点
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p; //若p能与输入参数匹配，即p就是要删除的节点，则记录为node，不需要循环查找
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                	//若p为TreeNode，表示桶中存放的是一棵红黑树，则调用getTreeNode方法进行匹配定位
                	//该方法在上文中已有分析，其实就是红黑树查找的过程
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                	//否则的话遍历链表查找
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
            //node不为空说明找到了与输入的key相匹配的节点
            //第二个条件是：要么matchValue==false，即不要求value也匹配，只要key匹配就进行删除；
            //否则就需要在key匹配的前提下value也匹配才能删除。
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                	//若node是红黑树中的一个节点，则交由removeTreeNode方法来完成删除操作
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                	//否则，若node是链表的头结点，则tab[index]位置存放的节点需变为node.next指向的节点
                    tab[index] = node.next;
                else
                	//否则，将node从链表中删除
                    p.next = node.next;
                ++modCount; //map的结构操作数加一
                --size; //map中的节点个数减一
                afterNodeRemoval(node); //钩子方法，抛出节点node被移除的这个事件
                return node; //返回已删除的节点node
            }
        }
        return null; //未能匹配到待删除节点，返回null
    }
	```
	以上就是HashMap删除操作的流程，先查找定位，若匹配成功，则根据找到的节点类型进行删除操作，并记录。下面重点来看下红黑树在删除操作的角色--**TreeNode.removeTreeNode**方法：
	
	```
		//这是TreeNode类的实例方法，方法被调用说明该节点本身为待删除节点
		//TreeNode类继承自Node类，它的连接指针在Node原有的next的基础上额外增加了prev,parent,left,right
		//所以除了二叉树的连接方式外，TreeNode仍维护了前驱后继两个指针，这意味着TreeNode节点可以像链表节点那样遍历，这一点从treeifyBin或putTreeVal中能够看出。
		//removeTreeNode方法中，先以链表的方式处理了next和prev指针，之后才是对于parent,left,right指针的处理
		final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable) {
            int n;
            if (tab == null || (n = tab.length) == 0)
                return;
            int index = (n - 1) & hash; //根据自身的hash值计算出自己所在的桶在table中的index
            //first和root均置为table[index]这个桶中的第一个节点
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            //succ为当前节点的后继节点，pred为前驱节点
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
            if (pred == null)
            	//若pred为空，则当前节点为桶中的第一个节点，删除后，其后继节点succ应当接替它的位置
                tab[index] = first = succ; 
            else
            	//否则的话，将其后继节点赋值给其前驱节点的后继节点，即将当前节点剔除
                pred.next = succ;
            if (succ != null)
                succ.prev = pred; //若succ不为空，置其前驱节点为pred
            if (first == null) 
                return; //上面几步都是针对next和prev指针的链表操作，若得到first为空，意味着当前节点是该桶中的唯一节点，则直接删除就好，无需后续调整
                
            //这里开始进行树的操作
            if (root.parent != null)
                root = root.root(); //若当前的root节点不是树的根节点，则通过root()方法找到树的根节点
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                //上面if中一堆条件，满足就说明桶中的这棵树太小了，没有存在的必要
                //这时进行非树化操作，对于TreeNode.untreeify方法后面会有详述
                tab[index] = first.untreeify(map);  // too small
                return;
            }
            //桶中的树还有存在的必要, p赋值为当前节点
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;
            //下面需要找到一个节点replacement来接替待删除节点p的位置
            //在红黑树解析的那篇文章中我们给出了策略，即若p为叶子节点，则直接删除；
            //否则若只有左孩子或右孩子，则其左孩子或右孩子上位；
            //否则若同时拥有左右子树，则选择左子树中最大的节点或右子树中最小的节点上位。
            if (pl != null && pr != null) { //若p的左右孩子都存在
                TreeNode<K,V> s = pr, sl; //s为p的右孩子，这里选择了右子树中最小的节点作为继承节点
                while ((sl = s.left) != null) // find successor
                    s = sl; //右子树中最左的孩子，即最小的节点
                boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                TreeNode<K,V> sr = s.right; //p的右子树中最小节点s的右孩子，记为sr
                TreeNode<K,V> pp = p.parent; //当前节点p的父节点，记为pp
                if (s == pr) { // p was s's direct parent
                	//若执行至此，说明p的右子树pr没有左孩子，即pr节点就是p的右子树中最小的，则pr直接上位即可
                    p.parent = s;
                    s.right = p;
                }
                else {
                	//否则的话，pr中的最左孩子s上位，s的右子树成为其父节点sp的左子树
                    TreeNode<K,V> sp = s.parent; 
                    if ((p.parent = sp) != null) {
                        if (s == sp.left)
                            sp.left = p;
                        else
                            sp.right = p;
                    }
                    if ((s.right = pr) != null)
                        pr.parent = s;
                }
                p.left = null;
                if ((p.right = sr) != null)
                    sr.parent = p;
                if ((s.left = pl) != null)
                    pl.parent = s; //s上位后，pl成为其左子树
                if ((s.parent = pp) == null) 
                    root = s; //s认p之前的父节点pp为自己新的父节点，若pp为空，则s为根节点
                else if (p == pp.left) //若pp不为空，则根据实际情况认s为自己新的左子树或右子树，替代p
                    pp.left = s;
                else
                    pp.right = s;
                    
                //这里需要解释一下，此处记录的replacement并非对于原先待删除节点p的继承者（即上文中的s）
                //而是指对于p右子树pr中的最左节点，也即上文中s的位置的继承者（因为s被上移去继承p，自然要有人来继承s）
                //按照我们一开始给出的调整策略，如果s有右子树sr，则应该由sr来继承s的位置；否则无人继承，则暂记为p
                //可以看出这个replacement变量是为了之后红黑树的删除调整准备的，它记录的是树经过上述调整后最终变化的地方
                if (sr != null) 
                    replacement = sr;
                else
                    replacement = p;
            }
            else if (pl != null)
                replacement = pl; //若p只有左子树，则左子树直接上位，replacement置为pl
            else if (pr != null)
                replacement = pr; //若p只有右子树，则右子树直接上位，replacement置为pr
            else
                replacement = p; //若p本身就是叶子节点，则无人继承其位置，暂记replacement为p
            if (replacement != p) {
            	//若replacement不为p，则让继承者跟新的父节点相认
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)
                    root = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null; //清空p的指针
            }
            
            //若p为红色，则删除后不会破坏红黑树的平衡，无需调整；否则的话需通过balanceDeletion方法进行删除后调整
            //该方法的具体源码在<<红黑树(Red-Black-Tree)解析>>一文中已有详述，请戳本文开头处的链接哈
            TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);
            
            //如果p为叶子节点，则在此处做删除工作，断开指针连接
            if (replacement == p) {  // detach
                TreeNode<K,V> pp = p.parent;
                p.parent = null;
                if (pp != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                }
            }
          
            if (movable)
                moveRootToFront(tab, r);
        }

	```
	以上便是HashMap删除节点的整个流程，以及对于红黑树的应用。最后我们再来看一下上面提到的**非树化操作untreeify**方法，这个方法代码很简单，作用在于删除操作之后当桶中节点数小于非树化阈值时，将红黑树结构变回链表结构：
	
	```
	/**
    * Returns a list of non-TreeNodes replacing those linked from
    * this node.
    */
    final Node<K,V> untreeify(HashMap<K,V> map) {
        Node<K,V> hd = null, tl = null;
        for (Node<K,V> q = this; q != null; q = q.next) {
           Node<K,V> p = map.replacementNode(q, null); //将每一个节点重新替换为Node类型
            if (tl == null) //头尾两个指针，将链表重新串起来
               hd = p;
            else
               tl.next = p;
            tl = p;
        }
        return hd; //返回链表头结点
    }
	```

- **最后**

	本文我们通过HashMap查找，插入和删除的源代码分析了红黑树在HashMap中的应用。当哈希冲突发生较少时，HashMap桶中的元素结构依旧是链表；而在冲突较多时，在没有引入红黑树的情况下，遍历长链表将大大降低HashMap的操作性能(时间复杂度变为O(N))，而红黑树在极端情况下的优秀性能表现在很大程度上弥补了这个短板，将时间复杂度保持在O(logN)。
	
	感谢阅读！
	
**版权声明：原创不易，转载前请留言获得作者许可，转载后标明作者 Troy.Tang 与 原文链接。**


	
