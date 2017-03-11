#STL之RBTree
##概述
红黑树是平衡二叉搜索树的一种，其通过特定的操作来保持二叉查找树的平衡。首先，我们来复习一下二叉查找树的知识，建议如果对二叉查找树不理解的先去搜一下相关博客来了解一下。

二叉搜索树是指一个空树或者具有以下性质的二叉树：

* 任意节点的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
* 任意节点的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
* 任意节点的左、右子树也分别为二叉查找树；
* 没有键值相等的节点


我们知道，一颗由n个节点随机构造的二叉搜索树的高度为logn，但是，由于输入值往往不够随机，导致二叉搜索树可能失去平衡，造成搜索效率低下的情况。从而，引出了平衡二叉搜索树的概念。对于“平衡”这个约束不同的结构有不同的规定，如AVL树要求任何节点的两个子树的高度最大差别为1，可谓是高度平衡啊；而红黑树仅仅确保没有一条路径会比其他路径长出两倍，因而达到接近平衡的目的。红黑数不仅是一个平衡二叉搜索树，而且还定义了相当多的约束来确保插入和删除等操作后能达到平衡。


![](./media/rbTree_001.jpg
)

那么，红黑树究竟是怎么定义，来使得能够达到平衡的目的呢？我们接着看下去。

##红黑树的定义
红黑树既然属于二叉搜索树的一种，当然需要满足上述二叉搜索树的性质，除此之外，红黑树还为每一个节点增加了一个存储位来表示节点的颜色属性，它可以为red或者black，通过对任何一条从根到叶子节点的路径上每个点进行着色方式的限制，来确保没有一条路径会比其他路径长出两倍，因而达到接近平衡的目的。

那么，红黑树是如何进行着色的呢？下面引出了红黑树的五条性质：

* 每个节点或者是黑色，或者是红色。
* 根节点是黑色。
* 每个叶子节点（NIL）是黑色。 [注意：这里叶子节点，是指为空(NIL或NULL)的叶子节点！]
* 如果一个节点是红色的，则它的子节点必须是黑色的。
* 从一个节点到该节点的子孙节点的所有路径上包含相同数目的黑节点。

```
typedef bool __rb_tree_color_type;
const __rb_tree_color_type __rb_tree_red = false;  // 紅色為 0
const __rb_tree_color_type __rb_tree_black = true; // 黑色為 1
struct __rb_tree_node_base
{
  typedef __rb_tree_color_type color_type;
  typedef __rb_tree_node_base* base_ptr;
  color_type color;	// 节点颜色
  base_ptr parent;	// 指向父节点
  base_ptr left;		// 指向左子节点
  base_ptr right;		// 指向右子节点
  // 一直往左走，就能找到红黑树的最小值节点
  // 二叉搜索树的性质
  static base_ptr minimum(base_ptr x)
  {
    while (x->left != 0) x = x->left;
    return x;
  }
  // 一直往右走，就能找到红黑树的最大值节点
  // 二叉搜索树的性质
  static base_ptr maximum(base_ptr x)
  {
    while (x->right != 0) x = x->right;
    return x;
  }
};
// 真正的节点定义，采用双层节点结构
// 基类中不包含模板参数
template <class Value>
struct __rb_tree_node : public __rb_tree_node_base
{
	typedef __rb_tree_node<Value>* link_type;
	Value value_field;    // 節點實值
};
```

###红黑树的迭代器

为了将RBtree实现为一个泛型容器，迭代器的设计很关键。我们要考虑它的型别，以及前进(increment)、后退(devrement)、提领(dereference)和成员访问(member access)等操作。

迭代器和节点一样，采用双层设计，STL红黑树的节点rb_tree_node继承于rb_tree_node_base；STL的迭代器结构rb_tree_iterator继承于rb_tree_base_iterator，我们可以用一张图来解释这样的设计目的。

![](./media/rbTree_002.jpg
)


将这些分开设计，可以保证对节点和迭代器的操作更具有弹性。下面来看迭代器的源码:

```
struct __rb_tree_base_iterator
{
  typedef __rb_tree_node_base::base_ptr base_ptr;
  typedef bidirectional_iterator_tag iterator_category;
  typedef ptrdiff_t difference_type;
  base_ptr node;	// 用来连接红黑树的节点
  // 寻找该节点的后继节点上
  void increment()
  {
    if (node->right != 0) {	// 如果存在右子节点
      node = node->right;		// 直接跳到右子节点上
      while (node->left != 0) // 然后一直往左子树走，直到左子树为空
        node = node->left;
    }
    else {                    // 没有右子节点
      base_ptr y = node->parent;    // 找出父节点
      while (node == y->right) {    // 如果该节点一直为它的父节点的右子节点
        node = y;                		// 就一直往上找，直到不为右子节点为止
        y = y->parent;
      }
      if (node->right != y)      // 若此时该节点不为它的父节点的右子节点
        node = y;                // 此时的父节点即为要找的后继节点
                                 // 否则此时的node即为要找的后继节点，此为特殊情况，如下
                                 // 我们要寻找根节点的下一个节点，而根节点没有右子节点
                                 // 此种情况需要配合rbtree的header节点的特殊设计，后面会讲到
    }                        
  }
  // 寻找该节点你的前置节点
  void decrement()
  {
    if (node->color == __rb_tree_red && // 如果此节点是红节点
        node->parent->parent == node)		// 且父节点的父节点等于自己
      node = node->right;								// 则其右子节点即为其前置节点
    // 以上情况发生在node为header时，即node为end()时
    // 注意：header的右子节点为mostright，指向整棵树的max节点，后面会有解释
    else if (node->left != 0) {					// 如果存在左子节点
      base_ptr y = node->left;					// 跳到左子节点上
      while (y->right != 0)							// 然后一直往右找，知道右子树为空
        y = y->right;			
      node = y;													// 则找到前置节点
    }
    else {															// 如果该节点不存在左子节点
      base_ptr y = node->parent;				// 跳到它的父节点上
      while (node == y->left) {					// 如果它等于它的父子节点的左子节点
        node = y;												// 则一直往上查找
        y = y->parent;									
      }																	// 直到它不为父节点的左子节点未知
      node = y;													// 此时他的父节点即为要找的前置节点
    }
  }
}
template <class Value, class Ref, class Ptr>
struct __rb_tree_iterator : public __rb_tree_base_iterator
{
	// 配合迭代器萃取机制的一些声明
  typedef Value value_type;
  typedef Ref reference;
  typedef Ptr pointer;
  typedef __rb_tree_iterator<Value, Value&, Value*>     iterator;
  typedef __rb_tree_iterator<Value, const Value&, const Value*> const_iterator;
  typedef __rb_tree_iterator<Value, Ref, Ptr>   self;
  typedef __rb_tree_node<Value>* link_type;
  // 迭代器的构造函数
  __rb_tree_iterator() {}
  __rb_tree_iterator(link_type x) { node = x; }
  __rb_tree_iterator(const iterator& it) { node = it.node; }
  // 提领和成员访问函数，重载了*和->操作符
  reference operator*() const { return link_type(node)->value_field; }
  pointer operator->() const { return &(operator*()); }
  // 前置++和后置++
  self& operator++() { increment(); return *this; }
  self operator++(int) {
    self tmp = *this;
    increment();		// 直接调用increment函数
    return tmp;
  }
  // 前置--和后置--
  self& operator--() { decrement(); return *this; }
  self operator--(int) {
    self tmp = *this;
    decrement();		// 直接调用decrement函数
    return tmp;
  }
};
```

在上述源代码中，一直提到STL RBTree特殊节点header的设计，这个会在RBTree结构中讲到，下面跟着我一起继续往下看吧。

###红黑树的数据结构

有了上面的节点和迭代器设计，就能很好的定义出一颗RBTree了。废话不多说，一步一步来剖析源代码吧。

```
template <class Key, class Value, class KeyOfValue, class Compare,
          class Alloc = alloc>
class rb_tree {
protected:
  typedef void* void_pointer;
  typedef __rb_tree_node_base* base_ptr;
  typedef __rb_tree_node<Value> rb_tree_node;		
  typedef simple_alloc<rb_tree_node, Alloc> rb_tree_node_allocator; // 专属配置器
  typedef __rb_tree_color_type color_type;
public:
	// 一些类型声明
  typedef Key key_type;
  typedef Value value_type;
  typedef value_type* pointer;
  typedef const value_type* const_pointer;
  typedef value_type& reference;
  typedef const value_type& const_reference;
  typedef rb_tree_node* link_type;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
protected:
  // RB-tree的数据结构
  size_type node_count; // 记录树的节点个数
  link_type header;  		// header节点设计
  Compare key_compare;	// 节点间的键值大小比较准则
  // 以下三个函数用来取得header的成员
  link_type& root() const { return (link_type&) header->parent; }
  link_type& leftmost() const { return (link_type&) header->left; }
  link_type& rightmost() const { return (link_type&) header->right; }
  // 以下六个函数用来取得节点的成员
  static link_type& left(link_type x) { return (link_type&)(x->left); }
  static link_type& right(link_type x) { return (link_type&)(x->right); }
  static link_type& parent(link_type x) { return (link_type&)(x->parent); }
  static reference value(link_type x) { return x->value_field; }
  static const Key& key(link_type x) { return KeyOfValue()(value(x)); }
  static color_type& color(link_type x) { return (color_type&)(x->color); }
  // 以下六个函数用来取得节点的成员，由于双层设计，导致这里需要两个定义
  static link_type& left(base_ptr x) { return (link_type&)(x->left); }
  static link_type& right(base_ptr x) { return (link_type&)(x->right); }
  static link_type& parent(base_ptr x) { return (link_type&)(x->parent); }
  static reference value(base_ptr x) { return ((link_type)x)->value_field; }
  static const Key& key(base_ptr x) { return KeyOfValue()(value(link_type(x)));} 
  static color_type& color(base_ptr x) { return (color_type&)(link_type(x)->color); }
  // 求取极大值和极小值，这里直接调用节点结构的函数极可
  static link_type minimum(link_type x) { 
    return (link_type)  __rb_tree_node_base::minimum(x);
  }
  static link_type maximum(link_type x) {
    return (link_type) __rb_tree_node_base::maximum(x);
  }
public:
	// RBTree的迭代器定义
  typedef __rb_tree_iterator<value_type, reference, pointer> iterator;
  typedef __rb_tree_iterator<value_type, const_reference, const_pointer> 
          const_iterator;
public:
  Compare key_comp() const { return key_compare; }	// 由于红黑树自带排序功能，所以必须传入一个比较器函数
  iterator begin() { return leftmost(); }        // RBTree的起始节点为左边最小值节点
  const_iterator begin() const { return leftmost(); }
  iterator end() { return header; }							// RBTree的终止节点为右边最大值节点
  const_iterator end() const { return header; }
  bool empty() const { return node_count == 0; }	// 判断红黑树是否为空	
  size_type size() const { return node_count; }		// 获取红黑树的节点个数
  size_type max_size() const { return size_type(-1); }	// 获取红黑树的最大节点个数，
																												// 没有容量的概念，故为sizetype最大值
};
```

我们看到，在RBTree的数据结构中，定义了RBTree节点和迭代器，然后添加了header节点，以及node_count参数，其他都是一下简单的函数声明和类型声明。除此之外，并没有过多的增加东西。这里理解起来还是比较简单，至于header有什么作用，请继续往下看。

#红黑树的构造与内存管理

##红黑树的构造函数

红黑树的空构造函数将创建一个空树，此”空树“非彼二叉树的空树也。空构造函数首先配置一个节点空间，使header指向该节点空间，然后将header的leftmost和rightmost指向自己，父节点指向0。非空的STL RBTree中，header和root之间互为父节点，然后header的leftmost始终指向该树的最小值节点，rightmost始终指向该树的最大值节点，其示例图如下（左图为空树，右图为非空树）：

![](./media/rbTree_003.jpg
)

下面来看看它的构造函数代码吧：

```
template <class Key, class Value, class KeyOfValue, class Compare,
          class Alloc = alloc>
class rb_tree {
	// 这部分代码是从红黑树的结构定义中提取出来的
protected:
	typedef simple_alloc<rb_tree_node, Alloc> rb_tree_node_allocator; // 专属配置器
  link_type get_node() { return rb_tree_node_allocator::allocate(); } // 配置空间
  link_type create_node(const value_type& x) {
    link_type tmp = get_node();            // 配置空間
    __STL_TRY {
      construct(&tmp->value_field, x);    // 构造内容
    }
    __STL_UNWIND(put_node(tmp));
    return tmp;
  }
  // 初始化函数，用来初始化一棵RBTree
  void init() {
    header = get_node();    // 产生一个节点空间，令header指向它
    color(header) = __rb_tree_red; // 令header为红色，用来区分header和root
    root() = 0;
    leftmost() = header;    // 令header的左子节点为其自己
    rightmost() = header;    // 令header的右子节点为其自己
  }
  // 真正的默认构造函数
  rb_tree(const Compare& comp = Compare())
  : node_count(0), key_compare(comp) { init(); }	// 直接调用初始化函数
  // 带参构造函数，以另一棵RBTree为初值来初始化
  rb_tree(const rb_tree<Key, Value, KeyOfValue, Compare, Alloc>& x) 
    : node_count(0), key_compare(x.key_compare)
  { 
    header = get_node();    // 產生一個節點空間，令 header 指向它
    color(header) = __rb_tree_red;    // 令 header 為紅色
    if (x.root() == 0) {    //  如果 x 是個空白樹
      root() = 0;
      leftmost() = header;     // 令 header 的左子節點為自己。
      rightmost() = header; // 令 header 的右子節點為自己。
    }
    else {    //  x 不是一個空白樹
      __STL_TRY {
        root() = __copy(x.root(), header);        //调用copy函数
      }
      __STL_UNWIND(put_node(header));
      leftmost() = minimum(root());    // 令 header 的左子節點為最小節點
      rightmost() = maximum(root());    // 令 header 的右子節點為最大節點
    }
    node_count = x.node_count;
  }
  // copy函数定义如下
  template <class K, class V, class KeyOfValue, class Compare, class Alloc>
	typename rb_tree<K, V, KeyOfValue, Compare, Alloc>::link_type 
	rb_tree<K, V, KeyOfValue, Compare, Alloc>::__copy(link_type x, link_type p) {
	  link_type top = clone_node(x); // 克隆root节点
	  top->parent = p;	// 将root节点父节点指向p
	 	// 以下为非递归的二叉树复制过程
	  __STL_TRY {
	    if (x->right)
	      top->right = __copy(right(x), top); // 一直copy右子节点
	    p = top;	
	    x = left(x);	// 取左节点
	    while (x != 0) {	// 左子节点不为空
	      link_type y = clone_node(x);	// 克隆左子节点
	      p->left = y;	// p的左子节点设为y
	      y->parent = p;	// y的父节点设为p
	      if (x->right)		// 如果左子节点还有右子节点，继续复制
	        y->right = __copy(right(x), y);
	      p = y;	// 直到没有左子节点
	      x = left(x);
	    }
	  }
	  __STL_UNWIND(__erase(top));
	  return top;
	}
	// clone一个节点函数
  link_type clone_node(link_type x) {    // 複製一個節點（的值和色）
	  link_type tmp = create_node(x->value_field);
	  tmp->color = x->color;
	  tmp->left = 0;
	  tmp->right = 0;
	  return tmp;
	}
}
```

###红黑树的析构函数

析构函数负责清除和释放红黑树上的每一个节点，并且初始化该红黑树为空树，即恢复header为初始状态

```
template <class Key, class Value, class KeyOfValue, class Compare,
          class Alloc = alloc>
class rb_tree {
	// 这部分代码是从红黑树的结构定义中提取出来的
protected:
	void put_node(link_type p) { rb_tree_node_allocator::deallocate(p); }
	// 析构一个节点
	void destroy_node(link_type p) {
    destroy(&p->value_field);        // 析构内容
    put_node(p);                    // 释放空间 
  }
  // 析构函数
  ~rb_tree() {
    clear();
    put_node(header);
  }
  // 清除整棵树并初始化header节点
  void clear() {
    if (node_count != 0) {
      __erase(root());
      // 初始化Header节点
      leftmost() = header;
      root() = 0;
      rightmost() = header;
      node_count = 0;
    }
  }
  // 清除RBTree的每个节点
	template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
	void rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::__erase(link_type x) {
	  // 直接清，不用调平衡
	  while (x != 0) {
	    __erase(right(x));
	    link_type y = left(x);
	    destroy_node(x);
	    x = y;
	  }
	}
}
```

##红黑树的插入

红黑树在插入节点的时候，会破坏其平衡，这时候需要旋转和变色来使红黑树重新达到平衡。这才是红黑树的精髓所在啊！所以这部分很长，大家耐心看，最好拿纸笔画一下。

###平衡二叉树的插入

我们先来看看平衡二叉树的旋转问题。平衡二叉树中，插入一个节点后，有可能会破坏二叉树的平衡，这时候可以通过左旋和右旋函数来使其重新恢复平衡。

通常破坏平衡二叉树平衡的插入情况有如下四种：

* 插入点位于X节点的左子节点的左子树，左左
* 插入点位于X节点的右子节点的右子树，右右
* 插入点位于X节点的左子节点的右子树，左右
* 插入点位于X节点的右子节点的左子树，右左

其中，情况1和2彼此对称，称为外侧插入，可以采用单旋转操作来调整；情况3和4彼此对称，称为内侧插入，需要采用双旋转操作来调整。

####外侧插入

对于外侧插入，我们通常采用一次旋转就能解决问题，请看下面两种外侧插入的示意图，其中左图对应上述情况1，右图对应上述情况2。

![](./media/rbTree_004.jpg
)


对于左图的情况，采用一次右旋操作，可以使二叉树重新恢复平衡，操作示意图如下所示：

![](./media/rbTree_005.jpg
)

####内侧插入

对于内侧插入，通常比较复杂，需要采用两次旋转操作来调整平衡。请看下面的内侧插入的示意图，其中左图对于上述情况3，右图对应上述情况4。

![](./media/rbTree_006.jpg
)

对于左图的情况，需要先进行依次左旋操作，再进行一次右旋操作即可调整平衡，操作示意图如下：

![](./media/rbTree_007.jpg
)

对于右图的情况，需要先进行依次右旋操作，再进行一次左旋操作即可调整平衡，操作示意图如下：

![](./media/rbTree_008.jpg
)


###红黑树的插入
有了上面的单旋转和双旋转知识，应该就很好理解红黑数的平衡调整过程。红黑树在插入节点的时候，不仅需要考虑插入导致的不平衡，还要考虑颜色属性是否满足其性质要求。为了更清楚的表达整个过程，我们先来定义一下几个标示量，使用X来表示一个新插入的节点，使用P来表示新插入节点的父节点，使用U来表示P节点的兄弟节点，使用G来表示P节点的父亲节点，使用N代表NIL节点。于是，可以将插入情况分为一下几类：

####树为空
新插入的节点为红色，因为此时树为空，那么插入该节点后只需要把节点颜色调整为黑色即可。

####父节点为黑
如果插入的节点的父节点为黑色，那么插入一个红节点将不会影响树的平衡，直接插入即可。这里就体现了黑红树的优势，O(1)时间内就能判断是否破坏了平衡，如果这里是AVL树的话就需要进行一次O(logn)判断是否平衡。

####父节点为红色
当父节点为红色的时候，就不满足条件4，即父节点为红色的时候，子节点必须为黑色，而新加入的节点为红色。这个时候需要考虑如下两种情况

* 叔父节点为红色

这种情况下，需要将父节点和叔父节点变为黑色，将祖父节点变为红色即可，不过如果祖父节点为红色的话，还是违反了红黑树的性质，此时必须执行一个继续向上迭代的程序来对红黑树的颜色进行调整，最后需要将根节点设置为黑色，如下图所示：

![](./media/rbTree_009.jpg
)

* 叔父节点为黑色(Nil节点为黑)

这时候，需要对其进行旋转操作，和上面AVL树的旋转一样，分为4种情况，下面一一举例来说明这四种情况：

* 左左，外侧插入

![](./media/rbTree_010.jpg
)

* 左右，内侧插入

![](./media/rbTree_011.jpg
)

* 右左，内侧插入

![](./media/rbTree_012.jpg
)

* 右右，外侧插入

![](./media/rbTree_013.jpg
)

###红黑树的插入源代码分析

红黑树的插入主要靠以下三个函数来实现：

```
// 左旋函数
inline void
__rb_tree_rotate_left(__rb_tree_node_base* x, __rb_tree_node_base*& root)
// 右旋函数
inline void
__rb_tree_rotate_right(__rb_tree_node_base* x, __rb_tree_node_base*& root)
// 插入后调节平衡函数
inline void
__rb_tree_rebalance(__rb_tree_node_base* x, __rb_tree_node_base*& root)
// 插入的核心函数
iterator __insert(base_ptr x, base_ptr y, const value_type& v);
```

上述三个函数就代表了上一小节示例图中的左旋，右旋和变色的源代码接口。

另外，STL还提供了两个个接口函数，如下：

```
// 允许出现相同的值
iterator insert_equal(const value_type& x);
// 不允许出现相同的值
iterator insert_unique(iterator position, const value_type& x);
```

下面就针对这五个函数一一来分析一下他们的源代码(以insert_unique为例)：

```
// 此插入函数不允许重复
// 返回的是一个pair，第一个元素为红黑树的迭代器，指向新增节点
// 第二个元素表示插入操作是否成功的
template<class Key , class Value , class KeyOfValue , class Compare , class Alloc>  
pair<typename rb_tree<Key , Value , KeyOfValue , Compare , Alloc>::iterator , bool>  
rb_tree<Key , Value , KeyOfValue , Compare , Alloc>::insert_unique(const Value &v)  
{  
    rb_tree_node* y = header;    // 根节点root的父节点  
    rb_tree_node* x = root();    // 从根节点开始  
    bool comp = true;  
    while(x != 0)  
    {  
        y = x;  
        comp = key_compare(KeyOfValue()(v) , key(x));    // v键值小于目前节点之键值？  
        x = comp ? left(x) : right(x);   // 遇“大”则往左，遇“小于或等于”则往右  
    }  
    // 离开while循环之后，y所指即插入点之父节点（此时的它必为叶节点）  
    iterator j = iterator(y);     // 令迭代器j指向插入点之父节点y  
    if(comp)     // 如果离开while循环时comp为真（表示遇“大”，将插入于左侧）  
    {  
        if(j == begin())    // 如果插入点之父节点为最左节点  
            return pair<iterator , bool>(_insert(x , y , z) , true);// 调用_insert函数
        else     // 否则（插入点之父节点不为最左节点）  
            --j;   // 调整j，回头准备测试  
    }  
    if(key_compare(key(j.node) , KeyOfValue()(v) ))  
        // 新键值不与既有节点之键值重复，于是以下执行安插操作  
        return pair<iterator , bool>(_insert(x , y , z) , true);  
    // 以上，x为新值插入点，y为插入点之父节点，v为新值  
  
    // 进行至此，表示新值一定与树中键值重复，那么就不应该插入新值  
    return pair<iterator , bool>(j , false);  
} 
// 真正地插入执行程序 _insert()  
// 返回新插入节点的迭代器
template<class Key , class Value , class KeyOfValue , class Compare , class Alloc>  
typename<Key , Value , KeyOfValue , Compare , Alloc>::_insert(base_ptr x_ , base_ptr y_ , const Value &v)  
{  
    // 参数x_ 为新值插入点，参数y_为插入点之父节点，参数v为新值  
    link_type x = (link_type) x_;  
    link_type y = (link_type) y_;  
    link_type z;  
    // key_compare 是键值大小比较准则。应该会是个function object  
    if(y == header || x != 0 || key_compare(KeyOfValue()(v) , key(y) ))  
    {  
        z = create_node(v);    // 产生一个新节点  
        left(y) = z;           // 这使得当y即为header时，leftmost() = z  
        if(y == header)  
        {  
            root() = z;  
            rightmost() = z;  
        }  
        else if(y == leftmost())     // 如果y为最左节点  
            leftmost() = z;          // 维护leftmost()，使它永远指向最左节点  
    }  
    else  
    {  
        z = create_node(v);        // 产生一个新节点  
        right(y) = z;              // 令新节点成为插入点之父节点y的右子节点  
        if(y == rightmost())  
            rightmost() = z;       // 维护rightmost()，使它永远指向最右节点  
    }  
    parent(z) = y;      // 设定新节点的父节点  
    left(z) = 0;        // 设定新节点的左子节点  
    right(z) = 0;       // 设定新节点的右子节点  
    // 新节点的颜色将在_rb_tree_rebalance()设定（并调整）  
    _rb_tree_rebalance(z , header->parent);      // 参数一为新增节点，参数二为根节点root  
    ++node_count;       // 节点数累加  
    return iterator(z);  // 返回一个迭代器，指向新增节点  
}  
// 全局函数  
// 重新令树形平衡（改变颜色及旋转树形）  
// 参数一为新增节点，参数二为根节点root  
inline void _rb_tree_rebalance(_rb_tree_node_base* x , _rb_tree_node_base*& root)  
{  
    x->color = _rb_tree_red;    //新节点必为红  
    while(x != root && x->parent->color == _rb_tree_red)    // 父节点为红  
    {  
        if(x->parent == x->parent->parent->left)      // 父节点为祖父节点之左子节点  
        {  
            _rb_tree_node_base* y = x->parent->parent->right;    // 令y为伯父节点  
            if(y && y->color == _rb_tree_red)    // 伯父节点存在，且为红  
            {  
                x->parent->color = _rb_tree_black;           // 更改父节点为黑色  
                y->color = _rb_tree_black;                   // 更改伯父节点为黑色  
                x->parent->parent->color = _rb_tree_red;     // 更改祖父节点为红色  
                x = x->parent->parent;  
            }  
            else    // 无伯父节点，或伯父节点为黑色  
            {  
                if(x == x->parent->right)   // 如果新节点为父节点之右子节点  
                {  
                    x = x->parent;  
                    _rb_tree_rotate_left(x , root);    // 第一个参数为左旋点  
                }  
                x->parent->color = _rb_tree_black;     // 改变颜色  
                x->parent->parent->color = _rb_tree_red;  
                _rb_tree_rotate_right(x->parent->parent , root);    // 第一个参数为右旋点  
            }  
        }  
        else          // 父节点为祖父节点之右子节点  
        {  
            _rb_tree_node_base* y = x->parent->parent->left;    // 令y为伯父节点  
            if(y && y->color == _rb_tree_red)    // 有伯父节点，且为红  
            {  
                x->parent->color = _rb_tree_black;           // 更改父节点为黑色  
                y->color = _rb_tree_black;                   // 更改伯父节点为黑色  
                x->parent->parent->color = _rb_tree_red;     // 更改祖父节点为红色  
                x = x->parent->parent;          // 准备继续往上层检查  
            }  
            else    // 无伯父节点，或伯父节点为黑色  
            {  
                if(x == x->parent->left)        // 如果新节点为父节点之左子节点  
                {  
                    x = x->parent;  
                    _rb_tree_rotate_right(x , root);    // 第一个参数为右旋点  
                }  
                x->parent->color = _rb_tree_black;     // 改变颜色  
                x->parent->parent->color = _rb_tree_red;  
                _rb_tree_rotate_left(x->parent->parent , root);    // 第一个参数为左旋点  
            }  
        }  
    }//while  
    root->color = _rb_tree_black;    // 根节点永远为黑色  
}  
// 左旋函数  
inline void _rb_tree_rotate_left(_rb_tree_node_base* x , _rb_tree_node_base*& root)  
{  
    // x 为旋转点  
    _rb_tree_node_base* y = x->right;          // 令y为旋转点的右子节点  
    x->right = y->left;  
    if(y->left != 0)  
        y->left->parent = x;           // 别忘了回马枪设定父节点  
    y->parent = x->parent;  
  
    // 令y完全顶替x的地位（必须将x对其父节点的关系完全接收过来）  
    if(x == root)    // x为根节点  
        root = y;  
    else if(x == x->parent->left)         // x为其父节点的左子节点  
        x->parent->left = y;  
    else                                  // x为其父节点的右子节点  
        x->parent->right = y;  
    y->left = x;  
    x->parent = y;  
}  
// 右旋函数  
inline void _rb_tree_rotate_right(_rb_tree_node_base* x , _rb_tree_node_base*& root)  
{  
    // x 为旋转点  
    _rb_tree_node_base* y = x->left;          // 令y为旋转点的左子节点  
    x->left = y->right;  
    if(y->right != 0)  
        y->right->parent = x;           // 别忘了回马枪设定父节点  
    y->parent = x->parent;  
  
    // 令y完全顶替x的地位（必须将x对其父节点的关系完全接收过来）  
    if(x == root)  
        root = y;  
    else if(x == x->parent->right)         // x为其父节点的右子节点  
        x->parent->right = y;  
    else                                  // x为其父节点的左子节点  
        x->parent->left = y;  
    y->right = x;  
    x->parent = y;  
}
```

###红黑树的删除
红黑树在删除节点后，需要调整以使得红黑树保持平衡，由于删除后调节平衡实在太复杂，本文就不做分析，只提供其接口函数。

如果对其感兴趣的话，可以参考一下这篇博文：[(图解）红黑树的插入和删除](http://www.cnblogs.com/deliver/p/5392768.html)

```
// 删除节点后调节平衡
inline __rb_tree_node_base*
__rb_tree_rebalance_for_erase(__rb_tree_node_base* z,
                              __rb_tree_node_base*& root,
                              __rb_tree_node_base*& leftmost,
                              __rb_tree_node_base*& rightmost);
```

STL为红黑树提供了以下删除操作的节点函数：

```
// 删除指定位置的节点
void erase(iterator position);
// 删除迭代器区间位置内的节点
void erase(iterator first, iterator last);
// 清除所有的节点
void clear();
```

下面来看看它们的源码实现：


```

// 删除指定位置的节点
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
inline void
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::erase(iterator position) {
  // 删除节点后需要使其重新恢复平衡
  link_type y = (link_type) __rb_tree_rebalance_for_erase(position.node,
                                                          header->parent,
                                                          header->left,
                                                          header->right);
  // 清除掉被删除的节点，释放内存
  destroy_node(y);
  --node_count;
}
// 清除掉区间内的节点
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
void rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::erase(iterator first, 
                                                            iterator last) {
  if (first == begin() && last == end())
    clear();
  else
    while (first != last) erase(first++);
}
// 清除所有的节点
void clear() {
  if (node_count != 0) {
    // 这里不需要调用上面的__rb_tree_rebalance_for_erase
    // 而是直接调用不需要调节平衡的删除节点函数，见下面
    __erase(root()); 
    leftmost() = header;
    root() = 0;
    rightmost() = header;
    node_count = 0;
  }
} 
// 删除红黑树的节点，删除过程中不需要调节平衡
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
void rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::__erase(link_type x) {
  while (x != 0) {
    __erase(right(x));
    link_type y = left(x);
    destroy_node(x);
    x = y;
  }
}
```

##红黑树的元素搜寻

find函数用于查找是否存在键值为k的节点。

```
// 寻找RBTree中是否存在键值为k的节点
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::find(const Key& k) {
  link_type y = header;        // Last node which is not less than k. 
  link_type x = root();        // Current node. 
  while (x != 0) 
    // key_compare是节点键值大小比较函数
    if (!key_compare(key(x), k)) 
      // 如果节点x的键值大于k，则继续往左子树查找
      y = x, x = left(x);    // 
    else
      // 如果节点x的键值小于k，则继续往右子树查找
      x = right(x);
  iterator j = iterator(y); 
  // y的键值不小于k，返回的时候需要判断与k是相等还是小于  
  return (j == end() || key_compare(k, key(j.node))) ? end() : j;
}
```

另外，STL的红黑树还针对multiset和multimap提供了几个搜寻函数，分别如下：

```

2
3
4
5
6
7
8
// 计算键值为x的节点的个数
size_type count(const key_type& x)
// 提供了查询与某个键值相等的节点迭代器范围
pair<iterator,iterator> equal_range(const key_type& x);
//返回不小于k的第一个节点迭代器
iterator lower_bound(const key_type& x);
//返回大于k的第一个节点迭代器
iterator upper_bound(const key_type& x);
```

其实现函数也一并贴出来，让大家好理解一下吧。

```
// 计算RBTree的节点值为k的节点个数
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::size_type 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::count(const Key& k) {
  pair<const_iterator, const_iterator> p = equal_range(k);
  size_type n = 0;
  distance(p.first, p.second, n);
  return n;
}
// 查询与键值k相等的节点迭代器范围
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
inline pair<typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator,
            typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator>
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::equal_range(const Key& k) {
  return pair<iterator, iterator>(lower_bound(k), upper_bound(k));
}
//返回不小于k的第一个节点迭代器
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::lower_bound(const Key& k) {
  link_type y = header; /* Last node which is not less than k. */
  link_type x = root(); /* Current node. */
  while (x != 0) 
    if (!key_compare(key(x), k))
      y = x, x = left(x);
    else
      x = right(x);
  return iterator(y);
}
//返回大于k的第一个节点迭代器
template <class Key, class Value, class KeyOfValue, class Compare, class Alloc>
typename rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::iterator 
rb_tree<Key, Value, KeyOfValue, Compare, Alloc>::upper_bound(const Key& k) {
  link_type y = header; /* Last node which is greater than k. */
  link_type x = root(); /* Current node. */
   while (x != 0) 
     if (key_compare(k, key(x)))
       y = x, x = left(x);
     else
       x = right(x);
   return iterator(y);
}
```

###Reference
[带你深入理解STL之RBTree](http://zcheng.ren/2016/09/03/STLRBTree1/)

