#理解STL之List容器
上一课介绍的vector和数组类似，它拥有一段连续的内存空间，并且起始地址不变，很好的支持了随机存取，但由于是连续空间，所以在中间进行插入、删除等操作时都造成了内存块的拷贝和移动，另外在内存空间不足时还需要重新申请一块大内存来进行内存的拷贝。为了克服这些缺陷，STL定义了另一种容器List，它对于数据插入和删除的时间复杂度均为O(1)，而且再内存方面不用频繁的拷贝转移。下面，就一起来看看List的源码实现吧！

##List概述
List和Vector都是STL的序列式容器，唯一不同的地方就在于：Vector是一段连续的内存空间，List则是一段不连续的内存空间，相比于Vector来说，List在每次插入和删除的时候，只需要配置或释放一个元素空间，对于任何位置的插入和删除操作，List永远能做到常数时间。但是，List由于不连续的内存空间，导致不支持随机寻址，所以尺有所长寸有所短，在程序中选择使用那种容器还要视元素的构造复杂度和存取行为而定。

##List的节点
List的节点结构如下：

```
template <class T>
struct __list_node
{
  typedef void* void_pointer; 
  void_pointer next;	//型别为void*，也可以设为__list_node<T>*
  void_pointer prev;
  T data;
};
```

从节点结构可以看出，List就是一个双向链表，其结构如下图所示：


![](./media/list_001.jpg
)

##List的迭代器

在Vector中，由于是连续的存储空间，支持随机存取，所以其迭代器可以直接用普通指针代替。但是，在List中行不通。List必须有能力指向List的节点，并有能力进行正确的递增、递减、取值和成员存取等操作。

List是一个双向链表，迭代器必须具备前移、后退的能力，所以List的迭代器是一个Bidirectional Iterator！在Vector中如果进行插入和删除操作后迭代器会失效，List有一个重要的性质就是插入和接合操作都不会造成原有的List迭代器失效。而且，再删除一个节点时，也仅有指向被删除元素的那个迭代器失效，其他迭代器不受任何影响。下面来看看List迭代器的源码。

```
template<class T, class Ref, class Ptr>
struct __list_iterator
{
  typedef __list_iterator<T, T&, T*>             iterator;   // 支持Iterator_traits
  typedef __list_iterator<T, const T&, const T*> const_iterator;
  typedef __list_iterator<T, Ref, Ptr>           self;
	// 以下为支持Iterator_traits而定义的一些类型
  typedef bidirectional_iterator_tag iterator_category;	//List的迭代器类型为双向迭代器
  typedef T value_type;                                
  typedef Ptr pointer;    
  typedef Ref reference;                                
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
  typedef __list_node<T>* link_type;                     
  // 这个是迭代器实际管理的资源指针
  link_type node;
  // 迭代器构造函数
  __list_iterator(link_type x) : node(x) {}
  __list_iterator() {}
  __list_iterator(const iterator& x) : node(x.node) {}
  // 在STL算法中需要迭代器提供支持
  bool operator==(const self& x) const { return node == x.node; }
  bool operator!=(const self& x) const { return node != x.node; }
  // 重载operator *, 返回实际维护的数据
  reference operator*() const { return (*node).data; }
  // 成员调用操作符
  pointer operator->() const { return &(operator*()); }
  // 前缀自加
  self& operator++()
  {
    node = (link_type)((*node).next);
    return *this;
  }
  // 后缀自加, 需要先产生自身的一个副本, 然会再对自身操作, 最后返回副本
  self operator++(int)
  {
    self tmp = *this;
    ++*this;
    return tmp;
  }
  self& operator--()
  {
    node = (link_type)((*node).prev);
    return *this;
  }
  self operator--(int)
  {
    self tmp = *this;
    --*this;
    return tmp;
  }
}
```

List的迭代器实现了==,!=,++,–,取值和成员调用等操作，由于是存放在不连续的内存空间，所以并不支持vector那样的p+n的操作。

##List的数据结构
List的数据结构个List的节点数据结构是分开定义的，SGI的List不仅是一个双向链表，而且还是一个环状双向链表，所以它只需要一个指针，就能完整表现一个链表。

```
template <class T, class Alloc = alloc>
class list
{
protected:
  typedef void* void_pointer;
  typedef __list_node<T> list_node;
  // 这个提供STL标准的allocator接口
  typedef simple_alloc<list_node, Alloc> list_node_allocator;
  // 链表的头结点，并不存放数据
  link_type node;
  //....以下还有一堆List的操作函数
}
```

![](./media/list_002.jpg
)


##List构造函数
List提供了一个空构造函数，如下：

```
list() { empty_initialize(); }
// 用于空链表的建立
void empty_initialize()
{
  node = get_node();
  node->next = node;  // 前置节点指向自己
  node->prev = node;  // 后置节点指向自己
}
```

另外，List还提供了带参的构造函数，支持如下初始化操作：

```
List<int> myList(5,1); // 初始化5个1的链表，{1,1,1,1,1}
```

其构造函数源码如下：

```
// 带参构造函数
list(size_type n, const T& value) { fill_initialize(n, value); }

//创建值为value共n个节点的链表
void fill_initialize(size_type n, const T& value)
{
    empty_initialize(); //先创建一个空链表
    insert(begin(), n, value); //插入n个值为value的节点
}

//在指定位置插入n个值为x的节点
void insert (iterator pos, int n, const T& x)
{
    insert(pos, (size_type)n, x);
}

// 在position前插入n个值为x的元素
template <class T, class Alloc>
void list<T, Alloc>::insert(iterator position, size_type n, const T& x)
{
  for ( ; n > 0; --n)
    insert(position, x);
}

// 好吧，到这里才是真正的插入操作
// 很简单的双向链表插入操作
iterator insert(iterator position, const T& x)
{
  link_type tmp = create_node(x);
  tmp->next = position.node;
  tmp->prev = position.node->prev;
  (link_type(position.node->prev))->next = tmp;
  position.node->prev = tmp;
  return tmp;
}

```

STL的List提供了很多种构造函数，此处我列举了其中一个，以此为例。

##List的其他操作函数
###get_node
此函数用来配置一个节点。

```
/ 配置一个节点并返回
link_type get_node() { 
	return list_node_allocator::allocate(); 
}
```

###put_node
此函数用来释放一个节点。

```
// 释放指定结点, 不进行析构, 析构交给全局的destroy,
void put_node(link_type p) {
	list_node_allocator::deallocate(p); 
}
```

###create_node
此函数用来配置并构造一个节点，并初始化其值

```
// 配置一个节点，并初始化其值为x
link_type create_node(const T& x)
{
  link_type p = get_node();
  construct(&p->data, x);	//全局函数
  return p;
}
```

###destory_node
此函数用来析构一个节点。

```
void destroy_node(link_type p)
{
	destroy(&p->data);	//全局函数
	put_node(p);
}
```

###insert
此函数用来在制定位置插入一个节点(上面提到过这个函数，这里重复一下，List的主要插入工作都交给这个函数)，该函数是一个重载函数，其有多种形式。

```
// 好吧，到这里才是真正的插入操作
// 很简单的双向链表插入操作
iterator insert(iterator position, const T& x)
{
  link_type tmp = create_node(x);
  tmp->next = position.node;
  tmp->prev = position.node->prev;
  (link_type(position.node->prev))->next = tmp;
  position.node->prev = tmp;
  return tmp;
}
// 其还有如下多种形式的重载函数
// 在[first,last]区间内插入元素
template <class T, class Alloc> template <class InputIterator>
void list<T, Alloc>::insert(iterator position,
                            InputIterator first, InputIterator last)
{
  for ( ; first != last; ++first)
    insert(position, *first);
}
// 在position位置插入元素，元素调用该型别默认构造函数
iterator insert(iterator position) { return insert(position, T()); }
```

###push_back
在尾部插入元素，有了上面的insert函数之后，push_back就比较容易实现了。

```
// 在链表最后插入结点
void push_back(const T& x) { insert(end(), x); }
```

###push_front

```
// 在链表前端插入结点
void push_front(const T& x) { insert(begin(), x); }
```

###earse
移除迭代器所指的元素

```
// 擦除指定结点
iterator erase(iterator position)
{
	// 双向链表移除节点的操作
	link_type next_node = link_type(position.node->next);
	link_type prev_node = link_type(position.node->prev);
	prev_node->next = next_node;
	next_node->prev = prev_node;
	destroy_node(position.node);
	return iterator(next_node);
}
// 上述函数还有一个重载版本，移除区间内所有的节点
// 擦除[first, last)间的结点
template <class T, class Alloc>
list<T, Alloc>::iterator list<T, Alloc>::erase(iterator first, iterator last)
{
	while (first != last) erase(first++);
	return last;
}
```

###pop_front

移除头节点元素，有了上述的erase函数，就很方便的实现了。

```
// 删除链表第一个结点
void pop_front() { erase(begin()); }
```

###pop_back

移除链表中最后一个元素

```
// 删除链表最后一个结点
void pop_back()
{
	iterator tmp = end();
	erase(--tmp);
}
```

###clear
清除链表中的所有节点，也就是一个一个的清除

```
// 销毁所有结点, 将链表置空
template <class T, class Alloc>
void list<T, Alloc>::clear()
{
  link_type cur = (link_type) node->next;
  while (cur != node) {	//遍历每一个节点
    link_type tmp = cur;
    cur = (link_type) cur->next;
    destroy_node(tmp);
  }
  node->next = node;// 移除后注意要保持链表是一个循环链表
  node->prev = node;
}
```

###remove
将链表中值为value的节点移除

```
// 移除特定值的所有结点
// 时间复杂度O(n)
template <class T, class Alloc>
void list<T, Alloc>::remove(const T& value)
{
	iterator first = begin();
	iterator last = end();
	while (first != last) {	//保证链表非空
		iterator next = first;
		++next;
		if (*first == value) erase(first);	//擦除该节点
		first = next;
	}
}
```

###transfer
将某段连续范围内的元素迁移到指定位置。（非公开接口）

```
void transfer(iterator position, iterator first, iterator last)
{
	if (position != last)
	{
		(*(link_type((*last.node).prev))).next = position.node;
		(*(link_type((*first.node).prev))).next = last.node;
		(*(link_type((*position.node).prev))).next = first.node;
		link_type tmp = link_type((*position.node).prev);
		(*position.node).prev = (*last.node).prev;
		(*last.node).prev = (*first.node).prev;
		(*first.node).prev = tmp;
	}
}
```

![](./media/list_003.jpg
)

###splice
List提供的接合函数是Splice，上述transfer是非公开的函数。splice函数有如下几个版本:

```
// 将链表x移动到position之前
void splice(iterator position, list& x)
{
	if (!x.empty())
		transfer(position, x.begin(), x.end());	//仅仅调用了transfer函数
}
// 将链表中i指向的内容移动到position之前
void splice(iterator position, list&, iterator i)
{
	iterator j = i;
	++j;
	if (position == i || position == j) return;
	transfer(position, i, j);
}
// 将[first, last}元素移动到position之前
void splice(iterator position, list&, iterator first, iterator last)
{
	if (first != last)
		transfer(position, first, last);
}
```

###merge

此函数用来合并两个链表，这里两个链表必须是已拍好序的。

```
// 假设当前容器和x都已序, 保证两容器合并后仍然有序
template <class T, class Alloc>
void list<T, Alloc>::merge(list<T, Alloc>& x)
{
  iterator first1 = begin();
  iterator last1 = end();
  iterator first2 = x.begin();
  iterator last2 = x.end();
  while (first1 != last1 && first2 != last2)
    if (*first2 < *first1) {
      iterator next = first2;
      transfer(first1, first2, ++next);	//将first2节点迁移到first1之后
      first2 = next;
    }
    else
      ++first1;
  if (first2 != last2) transfer(last1, first2, last2);	//如果first2还有剩余的，直接接合再链表1尾部
}
```

###reverse
此函数用来反转链表，其具体实现如下：

```
// 将链表倒置
template <class T, class Alloc>
void list<T, Alloc>::reverse()
{
  if (node->next == node || link_type(node->next)->next == node) return;
  iterator first = begin();
  ++first;
  while (first != end()) {
    iterator old = first;	// 取出一个节点
    ++first;
    transfer(begin(), old, first);	// 插入到begin()之后
  }
}
```

###sort
此函数对链表进行升序排序，具体实现如下：

```
// 按照升序排序
template <class T, class Alloc>
void list<T, Alloc>::sort()
{
  if (node->next == node || link_type(node->next)->next == node) return;
  list<T, Alloc> carry;
  list<T, Alloc> counter[64];
  int fill = 0;
  while (!empty()) {
  	// 从链表中取出一个节点
    carry.splice(carry.begin(), *this, begin());
    int i = 0;
    // 把carry中的新元素和counter中的结果逐一进行归并
    while (i < fill && !counter[i].empty()) {
      counter[i].merge(carry);
      carry.swap(counter[i++]);
    }
    // 把归并后的结果存放在counter[i]中
    carry.swap(counter[i]);
    // 已经达到2*fill，fill自增1
    if (i == fill) ++fill;
  }
  // 将counter中的所有元素进行归并
  for (int i = 1; i < fill; ++i) counter[i].merge(counter[i - 1]);
  // 将counter链表和本链表进行交换
  swap(counter[fill - 1]);
}
// 交换本链表和链表x
void swap(list<T, Alloc>& x) { 
	swap(node, x.node); 
}
``` 

