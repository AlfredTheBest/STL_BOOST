#STL之Deque容器
在介绍STL的deque的容器之前，我们先来总结一下vector和list的优缺点。vector在内存中是分配一段连续的内存空间进行存储，其迭代器采用原生指针即可，因此其支持随机访问和存储，支持下标操作符，节省空间。但是其在分配的内存不够的情况下，需要对容器整体进行重新分配、拷贝和释放等操作，而且在vector中间插入或删除元素效率很低。

而list是以节点形式来存放数据，使用的是非连续的内存空间来存放数据，因此，在其内部插入和删除元素的时间复杂度都是O(1),但是其不支持随机访问和存取，不支持下标，而且比vector占用的内存要多。

综合上述的优缺点，我们貌似需要一个支持随机访问和存取，支持下标访问，而且插入和删除的效率高的容器。于是，STL的deque诞生了，下面就跟着我一起去看看deque的设计和源码实现吧！

##Deque概述
vector是一个单向开口的容器，deque则是一个双向开口的容器，所谓双向开口就是再头尾两端均可以做元素的插入和删除操作。deque容器给我们的直观感觉大概是下面这样的(配图来自STL源码剖析)：

![](./media/deque_001.jpg
)

deque相比于vector最大的差异就在于支持常熟时间内对首尾两端进行插入和删除操作，而且deque没有容量的概念，其内部采用分段连续内存空间来存储元素，在插入元素的时候随时都可以重新增加一段新的空间并链接起来。


deque提供了Ramdon Access Iterator，同时也支持随机访问和存取，但是它也为此付出了昂贵的代价，其复杂度不能跟vector的原生指针迭代器相提并论。在下面的讲解中会一一为大家介绍STL是怎样”辛苦地”维持一个随机访问迭代器的。


##deque的中控器

deque为了维持整体连续的假象，设计一个中控器，其用来记录deque内部每一段连续空间的地址。大体上可以理解为deque中的每一段连续空间分布在内存的不连续空间上，然后用一个所谓的map作为主控，记录每一段内存空间的入口，从而做到整体连续的假象。其布局大概如下(配图来自STL源码剖析)

![](./media/deque_002.jpg
)

看完图解，再来看看源码会很好理解的。

```
template <class T, class Alloc = alloc, size_t BufSiz = 0>
class deque {
public:
  typedef T value_type;
  typedef value_type* pointer;
  typedef const value_type* const_pointer;
  typedef value_type& reference;
  typedef const value_type& const_reference;
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;
protected:
  typedef pointer* map_pointer;
  // 指向map, map是一个连续的空间, 其每个元素都是一个指向缓冲区的指针
  map_pointer map;
  size_type map_size;   // map容量
}
```

抛弃型别定义，我们可以看到map实际上就是一个指向指针的指针(T**)，map所指向的是一个指针，该指针指向型别为T的一块内存空间。理解到这里，大概就清楚了deque的实现原理，不过，这些都不是重点！重点是deque的各种运算符的实现。做好心理准备，咱们继续往下看！！！

##deque的迭代器

deque提供的是一个随机访问迭代器，由于是分段连续空间，其必须记录当前元素所在段的信息，从而在该段连续空间的边缘进行前进或者后退的时候能知道跳跃到的上一个或下一个缓冲区。deque必须完完全全地掌握和控制这些信息，以达到正确地跳跃！

```
/**
 * 注意deque的迭代器没有重载STL的Iterator
 */
template <class T, class Ref, class Ptr, size_t BufSiz>
struct __deque_iterator {
  typedef __deque_iterator<T, T&, T*, BufSiz>             iterator;
  typedef __deque_iterator<T, const T&, const T*, BufSiz> const_iterator;
  // 以下为支持Iterator_traits而定义的一些类型
  typedef random_access_iterator_tag iterator_category;      
  typedef T value_type;                                      
  typedef Ptr pointer;                                       
  typedef Ref reference;                                     
  typedef size_t size_type;
  typedef ptrdiff_t difference_type;                        
  typedef T** map_pointer;
  typedef __deque_iterator self;
  // 保存容器中的结点
  T* cur;       // 指向当前缓冲区中的元素
  T* first;     // 当前缓冲区的起点
  T* last;      // 当前缓冲区的终点
  map_pointer node;   // 指向管控中心
}

template <class T, class Ref, class Ptr, size_t BufSiz>
inline random_access_iterator_tag
iterator_category(const __deque_iterator<T, Ref, Ptr, BufSiz>&) {
  return random_access_iterator_tag();
}
template <class T, class Ref, class Ptr, size_t BufSiz>
inline T* value_type(const __deque_iterator<T, Ref, Ptr, BufSiz>&) {
  return 0;
}
template <class T, class Ref, class Ptr, size_t BufSiz>
inline ptrdiff_t* distance_type(const __deque_iterator<T, Ref, Ptr, BufSiz>&) {
  return 0;
}
```

从源码中可以看出，deque的迭代器中有cur,first,last和node四个指针，前三个记录了迭代器与缓冲区的联系，最后一个记录了迭代器于中控器的关系。从下面这张图可以很好的看出其关系：

![](./media/deque_003.jpg
)

仅仅定义了迭代器结构还只是开始，迭代器是一个随机访问迭代器，所以其必须提供++，–，下标操作符等运算符。下面就来一一剖析吧！


###buffer_size函数

```
/**
 * 返回deque的buffer_size大小
 */
static size_t buffer_size() {
	return __deque_buf_size(BufSiz, sizeof(T)); 
}
/**
 * 如果n不为0，传回n，表示buffer size由用户自己定义
 * 如果n为0，表示buffer_size采用默认值，
 * 				  那么如果sz(元素大小)小于512，传回512/sz
 * 					如果sz不小于512，传回1
 */
inline size_t __deque_buf_size(size_t n, size_t sz)
{
  return n != 0 ? n : (sz < 512 ? size_t(512 / sz) : size_t(1));
}
```


###set_node函数
当迭代器处在当前缓冲区的边缘时，一旦前进或者后退，就要考虑超过当前缓冲区的情况，此时需要跳转到下一个缓冲区，这时候set_node就派上用场了。

```
void set_node(map_pointer new_node)
{
  node = new_node;		// 跳转到相应缓冲区
  first = *new_node;	// 更新跳转后缓冲区first信息
  last = first + difference_type(buffer_size());  // 更新跳转后缓冲区last的信息
}
```

###各种运算子
以下源码都是deque迭代器重载的运算子，以满足随机访问迭代器的要求。

```
reference operator*() const { return *cur; }
pointer operator->() const { return &(operator*()); }

/**
 * 判断两个迭代器之间的距离，重载了‘-’运算子
 */
difference_type operator-(const self& x) const
{
  return difference_type(buffer_size()) * (node - x.node - 1) +
    (cur - first) + (x.last - x.cur);
}

/**
 * 前缀自增，注意前缀自增返回自身引用
 */
self& operator++()
{
  ++cur;	// 先自增当前元素的指针
  if (cur == last) {	// 判断是否为当前缓冲区最后一个
    set_node(node + 1);	// 如果为当前缓冲区最后一个，则跳转到下一个缓冲区
    cur = first;	// 更新为下一缓冲区的起始点
  }
  return *this;
}

/**
 *	后缀自增
 *	返回当前迭代器的一个副本, 并调用前缀自增运算符实现迭代器自身的自增
 */
self operator++(int)  {
  self tmp = *this;
  ++*this;
  return tmp;
}

/**
 *	前缀自减, 处理方式类似于前缀自增
 *	如果当前迭代器指向元素是当前缓冲区的第一个元素
 *	则将迭代器状态调整为前一个缓冲区的最后一个元素
 */
self& operator--()
{
  if (cur == first) {
    set_node(node - 1);
    cur = last;
  }
  --cur;
  return *this;
}
// 处理方法同后缀自增
self operator--(int)
{
  self tmp = *this;
  --*this;
  return tmp;
}

/**
 *	实现p+=n的功能
 *	迭代器向前移动n个元素，其中n可能为负。实现步骤如下：
 *	1、计算相对于该缓冲区起始位置的偏移量offset
 *	2、如果offset没有超出缓冲区，则直接cur+=n
 *	3、如果offset超过了缓冲区空间
 *			-- 如果offset大于0，计算向前移动多少个缓冲区，offset / difference_type(buffer_size())
 *			-- 如果offset小于0，计算向后移动多少个缓冲区,-difference_type((-offset - 1) / buffer_size()) - 1;
 *	4、调整到移动后的位置。
 */
 self& operator+=(difference_type n)
{
  difference_type offset = n + (cur - first);
  if (offset >= 0 && offset < difference_type(buffer_size()))
    cur += n;
  else {
    difference_type node_offset =
      offset > 0 ? offset / difference_type(buffer_size())
                 : -difference_type((-offset - 1) / buffer_size()) - 1;
    set_node(node + node_offset);
    cur = first + (offset - node_offset * difference_type(buffer_size()));
  }
  return *this;
}

/**
 * 实现诸如p+n的功能
 * 此函数中直接调用operator +=的函数
 */
self operator+(difference_type n) const
{
  self tmp = *this;
  // 这里调用了operator +=()可以自动调整指针状态
  return tmp += n;
}
/**
 * 实现p-=n的功能
 * 此处直接利用operator += ，改变一下n的正负即可
 */
self& operator-=(difference_type n) { return *this += -n; }
/**
 * 实现p-n的功能
 * 直接调用operator -=的函数
 */
self operator-(difference_type n) const {
  self tmp = *this;
  return tmp -= n;
}
/**
 * 下标运算子，支持随机存取的功能
 */
reference operator[](difference_type n) const { return *(*this + n); }
/**
 * 下述都是一些判断运算的实现
 */
bool operator==(const self& x) const { return cur == x.cur; }
bool operator!=(const self& x) const { return !(*this == x); }
bool operator<(const self& x) const {
  return (node == x.node) ? (cur < x.cur) : (node < x.node);
}
```

##deque的数据结构
先前在deque的中控器中讲到，deque维护着一个map，用来记录每个缓冲区的位置。除了map外，deque的数据结构中还维护着start和finish两个迭代器，分别指向deque的首尾。此外，它还必须知道map的大小，一旦map所提供的节点不足，就需要配置一块更大的map。

接下来，我们来看看deque的数据结构源代码：

```
template <class T, class Alloc = alloc, size_t BufSiz = 0>
class deque {
public:
  typedef T value_type;
  typedef value_type* pointer;
  typedef size_t size_type;
	// 这里省略一堆支持iterator_traits型别定义
public:
  typedef __deque_iterator<T, T&, T*, BufSiz>  iterator;	// deque的迭代器
protected:
  typedef pointer* map_pointer;
protected:
  iterator start;               // 表中第一个节点
  iterator finish;              // 表中最后一个节点
  
  // 这是前面讲到map指针，用来记录每一个缓冲区的地址
  map_pointer map;
  size_type map_size;   // map容量 
  //	deque专属空间配置器，每次配置一个元素大小
  typedef simple_alloc<value_type, Alloc> data_allocator;
  //	deque专属空间配置器，每次配置一个指针大小
  typedef simple_alloc<pointer, Alloc> map_allocator;
	// 分配内存, 不进行构造
	pointer allocate_node() { return data_allocator::allocate(buffer_size()); }
	// 释放内存, 不进行析构
	void deallocate_node(pointer n)
	{
		data_allocator::deallocate(n, buffer_size());
	}
}
```

在上述结构体下，可以很轻松地实现“连续”容器的各种机能函数，如下：

```
public:
  iterator begin() { return start; }	// 返回第一个节点的迭代器
  iterator end() { return finish; }		// 返回最后一个节点的迭代器
  const_iterator begin() const { return start; }	// const版本
  const_iterator end() const { return finish; }		// const版本

  /**
   *	提供随机访问的下标运算子
   *	这里计算实际地址的时候是经过一系列的计算得到的，效率上有缺失
   */
  reference operator[](size_type n) { return start[difference_type(n)]; }
  const_reference operator[](size_type n) const {
  	return start[difference_type(n)];
  }
  
  /**
   * 以下函数分别返回首尾元素的引用
   */
  reference front() { return *start; }
  reference back() {
    iterator tmp = finish;
    --tmp;
    return *tmp;
  }
  const_reference front() const { return *start; }
  const_reference back() const {
    const_iterator tmp = finish;
    --tmp;
    return *tmp;
  }
  //	返回deque的大小，这里直接调用迭代器重载的‘-’运算符 
  size_type size() const { return finish - start;; }
  //	返回deque最大容量
  size_type max_size() const { return size_type(-1); }
  // deque为空的时, 只有一个缓冲区
  bool empty() const { return finish == start; }
```

##deque的构造函数

deque和vector、list一样，提供了多种构造函数。我们先来看看默认构造函数：

```
deque() : start(), finish(), map(0), map_size(0)
{
  create_map_and_nodes(0);	// 直接调用create_map_and_nodes函数
}
// map最少为8个
static size_type initial_map_size() { return 8; }
// 创建内部使用的map，并配置每一个缓冲区
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::create_map_and_nodes(size_type num_elements)
{
	// 需要的结点数, 元素个数 / 每个缓冲区能容纳的元素数 + 1
  // 这里如果能整除，会多分配一个
  size_type num_nodes = num_elements / buffer_size() + 1;
  // map要维护的结点, 这里最小的值为8，最多为所需节点数+1，前后各留一个以便扩充
  map_size = max(initial_map_size(), num_nodes + 2);
  // 调用deque专属空间配置器，配置map空间
  map = map_allocator::allocate(map_size);
  // 将[nstart, nfinish)区间设置在map的中间,
  // 这样就能保证前后增长而尽可能减少map的重新分配次数
  map_pointer nstart = map + (map_size - num_nodes) / 2;
  map_pointer nfinish = nstart + num_nodes - 1;
  // 分配结点空间
  map_pointer cur;
  __STL_TRY {
    for (cur = nstart; cur <= nfinish; ++cur)
    	// 为每一个map指针指向的缓冲区的每一个元素分配内存空间 
      *cur = allocate_node();
  }
  // 维护指针状态，为deque的两个迭代器start和finish赋初值
  start.set_node(nstart);
  finish.set_node(nfinish);
  start.cur = start.first;
  finish.cur = finish.first + num_elements % buffer_size();
}
```

除了默认构造函数，deque还提供了一系列的构造函数：

```
/**
 * 拷贝构造函数
 */
deque(const deque& x)
  : start(), finish(), map(0), map_size(0)
{
	// 配置map和元素
  create_map_and_nodes(x.size());
  // 将x的元素拷贝到本deque内
  uninitialized_copy(x.begin(), x.end(), start);
}
/**
 * 构造一个deque，含有n个值为value的元素
 */
deque(size_type n, const value_type& value)
  : start(), finish(), map(0), map_size(0)
{
  fill_initialize(n, value);	// 调用fill_initialize函数
}
// 分配n个结点, 并以value为元素值初始化
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::fill_initialize(size_type n,
    const value_type& value)
{
  create_map_and_nodes(n);	// 配置map和缓冲区
  map_pointer cur;
  __STL_TRY {
  	// 为每一个缓冲区设定初值
    for (cur = start.node; cur < finish.node; ++cur)
      uninitialized_fill(*cur, *cur + buffer_size(), value);
    // 尾端可能留有备用空间，不必设初值
    uninitialized_fill(finish.first, finish.cur, value);
  }
  catch (...) {
    for (map_pointer n = start.node; n < cur; ++n)
      destroy(*n, *n + buffer_size());
    destroy_map_and_nodes();
    throw;
  }
}
/**
 * 以区间值来构造deque
 */
template <class InputIterator>
deque(InputIterator first, InputIterator last)
  : start(), finish(), map(0), map_size(0)
{
  range_initialize(first, last, iterator_category(first));
}
template <class T, class Alloc, size_t BufSize>
template <class ForwardIterator>
void deque<T, Alloc, BufSize>::range_initialize(ForwardIterator first,
    ForwardIterator last,
    forward_iterator_tag) {
  size_type n = 0;
  distance(first, last, n);	// 计算有多少个元素
  create_map_and_nodes(n);	// 配置map和缓冲区
  uninitialized_copy(first, last, start);	// 调用全局函数，将[first,last)拷贝到新配置的空间上
}
```
当然，deque还提供了很多种构造函数，基本上都调用上述函数来构造map和缓冲区，这里就不在赘述！

##deque的析构函数
```
~deque()
{
  destroy(start, finish);     // 调用全局函数
  destroy_map_and_nodes();		// 释放map和缓冲区
}
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::destroy_map_and_nodes()
{
  for (map_pointer cur = start.node; cur <= finish.node; ++cur)
    deallocate_node(*cur);	// 释放每一个节点
  map_allocator::deallocate(map, map_size);	// 释放map空间
}
```

##deque支持的操作函数
###push_back
push_back完成在尾部插入一个元素，根绝上述的deque的结构特点，里面有很多情况需要考虑。

* 如果备用空间足够，就直接push进去
* 如果备用空间不足，就要考虑配置一个新的缓冲区

配置新缓冲区的时候，还需要考虑map空间是否足够

* 如果map空间足够，就直接配置一块新的缓冲区，链接到map中
* 如果map空间不足，就需要考虑重新配置一块map

可见，为了维持整体连续的假象，确确实实，deque的操作函数需要考虑各个方面。下面来看看源代码。

```
/**
 * 在deque的尾部压入一个元素
 */
void push_back(const value_type& t)
{
  // 注意这里采用STL的前闭后开原则
  // 所以last要-1
  // 如果deque里面还有备用空间，则直接压入
  if (finish.cur != finish.last - 1) {
    construct(finish.cur, t);
    ++finish.cur;
  }
  // 容量已满就要新申请内存了
  else
    push_back_aux(t);
}

/**
 * 仅当finish.cur == finish.last - 1才调用
 * 即最后一个缓冲区没有空间才调用
 */
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::push_back_aux(const value_type& t)
{
  value_type t_copy = t;
  // 判断是否需要调整map空间
  reserve_map_at_back();
  *(finish.node + 1) = allocate_node();	// 配置一块新的缓冲区
  __STL_TRY {
    construct(finish.cur, t_copy);	// 构造新加入的元素
    finish.set_node(finish.node + 1);	// 调整finish
    finish.cur = finish.first;
  }
  __STL_UNWIND(deallocate_node(*(finish.node + 1)));
}

/**
 * map空间不足，需要调整
 */
void reserve_map_at_back (size_type nodes_to_add = 1)
{
	if (nodes_to_add + 1 > map_size - (finish.node - map))
		// 此时，需要调整map，更换一个更大的map
		reallocate_map(nodes_to_add, false);
}

/**
 * 重新配置map, 不会对缓冲区进行操作, map维护的是指向缓冲区的指针
 */
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::reallocate_map(size_type nodes_to_add,
    bool add_at_front)
{
  size_type old_num_nodes = finish.node - start.node + 1;
  size_type new_num_nodes = old_num_nodes + nodes_to_add;
  map_pointer new_nstart;
  // 此处为了防止出现一端已经用完，另一端却还有很多剩余的情况
  if (map_size > 2 * new_num_nodes) {
  	// 调整新的map中的起始点
    new_nstart = map + (map_size - new_num_nodes) / 2
                 + (add_at_front ? nodes_to_add : 0);
    // 如果前端剩余很多
    if (new_nstart < start.node)
      copy(start.node, finish.node + 1, new_nstart);
    else   // 尾端剩余很多
      copy_backward(start.node, finish.node + 1, new_nstart + old_num_nodes);
  }
  else {	// map不够用了，就需要配置一块更大的map
    size_type new_map_size = map_size + max(map_size, nodes_to_add) + 2;
    // 配置一块大的map
    map_pointer new_map = map_allocator::allocate(new_map_size);
    // 始终要使start和finish处在map空间的中间
    new_nstart = new_map + (new_map_size - new_num_nodes) / 2
                 + (add_at_front ? nodes_to_add : 0);
    // 拷贝到新的map空间中去
    copy(start.node, finish.node + 1, new_nstart);
    // 释放旧的空间
    map_allocator::deallocate(map, map_size);
    // 改变map和size参数
    map = new_map;
    map_size = new_map_size;
  }
  // 调整新的start和finish
  start.set_node(new_nstart);
  finish.set_node(new_nstart + old_num_nodes - 1);
}
```

###pop_back
pop_back是将deque的尾部元素弹出，即拿掉该元素并释放空间。

```
void pop_back()
{
	// 如果尾端不是该缓冲区最开始的那个元素
  if (finish.cur != finish.first) {
    --finish.cur;
    destroy(finish.cur);  // 直接拿掉并释放空间
  }
  else
    pop_back_aux();   // 需要调整map的情况
}
/**
 * 在pop_back中，如果碰到为首元素的情况，调用此函数
 */
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>:: pop_back_aux()
{
  deallocate_node(finish.first);	// 释放节点
  finish.set_node(finish.node - 1);	// 重新设定finish
  finish.cur = finish.last - 1;
  destroy(finish.cur);	
}
```

###push_front
此函数用来在deque的头部压入一个元素。

```
void push_front(const value_type& t)
{
	// 还是一样，不需要调整map的情况，直接压入
  if (start.cur != start.first) {
    construct(start.cur - 1, t);
    --start.cur;
  }
  else
    push_front_aux(t);
}
/**
 * 只有再start.cur== start.first的情况下调用
 */
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::push_front_aux(const value_type& t)
{
  value_type t_copy = t;
  reserve_map_at_front();	// 同push_back()，检查是否需要调整map
  *(start.node - 1) = allocate_node();	// 配置一块新的缓冲区
  __STL_TRY {
    start.set_node(start.node - 1);	// 调整start
    start.cur = start.last - 1;
    construct(start.cur, t_copy);
  }
  catch (...) {
    start.set_node(start.node + 1);
    start.cur = start.first;
    deallocate_node(*(start.node - 1));
    throw;
  }
}
```

###pop_front
此函数实现从头部弹出一个元素，同pop_back()。

```
void pop_front() {
  if (start.cur != start.last - 1)
  {
    destroy(start.cur);
    ++start.cur;
  }
  else
    pop_front_aux();
}
/**
 * 只有在start.cur == start.last - 1的时候调用
 * 此时需要调整map
 */
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::pop_front_aux()
{
  destroy(start.cur);
  deallocate_node(start.first);
  start.set_node(start.node + 1);
  start.cur = start.first;
}
```

###clear
擦除deque中的每一个元素

```
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::clear()
{
  // 首先析构除起点和终点的所有元素, 并释放相应空间
  for (map_pointer node = start.node + 1; node < finish.node; ++node) {
    destroy(*node, *node + buffer_size());
    data_allocator::deallocate(*node, buffer_size());
  }
  // 如果deque本身不为空, 析构所有对象, 并释放掉结尾的内存
  if (start.node != finish.node) {
    destroy(start.cur, start.last);  // 将头缓冲区的元素清除
    destroy(finish.first, finish.cur);  //将尾缓冲区的元素清除
    data_allocator::deallocate(finish.first, buffer_size()); // 头缓冲区保留，释放尾缓冲区
  }
  // 析构所有元素, 但是不释放空间, 因为deque要满足这个前置条件
  else
    destroy(start.cur, finish.cur);
  finish = start; // 调整finish
}
```

###erase

erase实现了擦除单个指定元素和擦出区间两个版本，源代码分析如下：

```
/**
 * 此函数实现擦除单个指定元素
 */
iterator erase(iterator pos)
{
  iterator next = pos;
  ++next;
  // 计算待擦除点前的元素个数
  difference_type index = pos - start;
  // 判断待擦除结点前后元素的个数, 哪部分少就移动哪部分
  if (index < (size() >> 1))
  {
    // 前面部分的元素少
    copy_backward(start, pos, next);  
    pop_front();
  }
  // 后面部分的元素少
  else {
    copy(next, finish, pos); 
    pop_back();
  }
  return start + index;
}
```

擦除[first,last)区间的元素。此函数按下列步骤来擦除区间。

* 需要擦除整个空间，直接调用clear()
* 需要擦出中间指定区间

擦除中间指定区间，需要考虑一下两种情况


* 区间前面的元素少，就移动前面的元素
* 区间后面的元素少，就移动后面的元素


```
template <class T, class Alloc, size_t BufSize>
deque<T, Alloc, BufSize>::iterator
deque<T, Alloc, BufSize>::erase(iterator first, iterator last)
{
  if (first == start && last == finish) {	// 需要擦除整个deque
    clear();
    return finish;
  }
  else {
    difference_type n = last - first;	// 清除区间的长度
    difference_type elems_before = first - start;  // 待清除区间前方的元素个数
    if (elems_before < (size() - n) / 2) {	// 如果前方的元素个数较少
      copy_backward(start, first, last);	// 向后移动前方元素
      iterator new_start = start + n;     // 调整新的起始点
      destroy(start, new_start);          // 全局函数，析构节点元素
      for (map_pointer cur = start.node; cur < new_start.node; ++cur)
        data_allocator::deallocate(*cur, buffer_size());   // 释放缓冲区空间
      start = new_start;
    }
    else {    // 后方元素比较少的情况
      copy(last, finish, first);	// 向前移动后方元素
      iterator new_finish = finish - n; // 调整新的finish迭代器
      destroy(new_finish, finish);      // 全局函数，析构节点元素
      for (map_pointer cur = new_finish.node + 1; cur <= finish.node; ++cur)
        data_allocator::deallocate(*cur, buffer_size());  // 释放缓冲区空间
      finish = new_finish;
    }
    return start + elems_before;
  }
}
```

###insert
在指定位置前插入元素，deque的源码中，为insert提供了多个版本，这里列举插入一个元素和n和元素的版本。

在指定位置插入一个元素

```
iterator insert(iterator position, const value_type& x)
{
  // 如果是在deque的最前端插入, 那么直接push_front()即可
  if (position.cur == start.cur) {
    push_front(x);
    return start;
  }
  // 如果是在deque的末尾插入, 直接调用push_back()
  else if (position.cur == finish.cur) {
    push_back(x);
    iterator tmp = finish;
    --tmp;
    return tmp;
  }
  else {
    return insert_aux(position, x);
  }
}

/**
 * 不在首尾插入元素的时候调用此函数
 */
template <class T, class Alloc, size_t BufSize>
typename deque<T, Alloc, BufSize>::iterator
deque<T, Alloc, BufSize>::insert_aux(iterator pos, const value_type& x)
{
  difference_type index = pos - start;  // 插入元素前面的元素个数
  value_type x_copy = x;
  if (index < size() / 2) {  // 如果前端的元素比较少
    push_front(front());  // 在最前面插入一个与第一个元素一样的数
    iterator front1 = start;  // 记录起始点
    ++front1;
    iterator front2 = front1; 
    ++front2;
    pos = start + index;
    iterator pos1 = pos;
    ++pos1;
    copy(front2, pos1, front1); // 拷贝空间，将[front2,pos1)拷贝到front1以后
  }
  else {   // 后端的元素比较少，原理用上
    push_back(back());
    iterator back1 = finish;
    --back1;
    iterator back2 = back1;
    --back2;
    pos = start + index;
    copy_backward(pos, back2, back1);
  }
  *pos = x_copy;
  return pos;
}
```

在指定位置插入n个元素的情况

```
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert(iterator pos,
                                      size_type n, const value_type& x)
{
  if (pos.cur == start.cur) {   // 如果插入点再最前端
    iterator new_start = reserve_elements_at_front(n); // 调整新的start位置
    uninitialized_fill(new_start, start, x);	//直接在前端构造n个元素
    start = new_start;  // 调整新的start
  }
  else if (pos.cur == finish.cur) {
  	// 与reserve_elements_at_front相同
  	// 考虑篇幅，这里不列出源代码
    iterator new_finish = reserve_elements_at_back(n); 
    uninitialized_fill(finish, new_finish, x);
    finish = new_finish;
  }
  else
    insert_aux(pos, n, x);
}
/**
 * 插入区间前方备用空间能否容纳n个元素
 */
iterator reserve_elements_at_front(size_type n)
{
  size_type vacancies = start.cur - start.first;
  if (n > vacancies)   // 如果容纳不了，就需要重新配置map
    new_elements_at_front(n - vacancies);
  return start - difference_type(n);
}
/**
 * 只有在前方备用空间容纳不了待插入的n个元素的情况下调用
 */
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::new_elements_at_front(size_type new_elements)
{
  size_type new_nodes = (new_elements + buffer_size() - 1) / buffer_size();
  reserve_map_at_front(new_nodes);  // 调整map
  size_type i;
  __STL_TRY {
    for (i = 1; i <= new_nodes; ++i)
      *(start.node - i) = allocate_node(); // 为每一个map指针配置空间
  }
  catch (...) {
    for (size_type j = 1; j < i; ++j)
      deallocate_node(*(start.node - j));
    throw;
  }
}
/**
 * 调整map的前端，以在前端能连接更多缓冲区
 */
void reserve_map_at_front (size_type nodes_to_add = 1)
{
  if (nodes_to_add > start.node - map)
    reallocate_map(nodes_to_add, true);  // 此函数上面有说明
}
/**
 * 好吧，这里才是最重要的insert_aux函数，实现在中间某个位置插入n个元素
 */
template <class T, class Alloc, size_t BufSize>
void deque<T, Alloc, BufSize>::insert_aux(iterator pos,
    size_type n, const value_type& x)
{
  const difference_type elems_before = pos - start;  // 计算该位置前面的元素个数
  size_type length = size();
  value_type x_copy = x;
  if (elems_before < length / 2) {  // 如果位置前面的元素比较少
    iterator new_start = reserve_elements_at_front(n); // 同上
    iterator old_start = start;
    pos = start + elems_before;
    __STL_TRY {
      if (elems_before >= difference_type(n)) { 
        iterator start_n = start + difference_type(n);
        uninitialized_copy(start, start_n, new_start);
        start = new_start;
        copy(start_n, pos, old_start);
        fill(pos - difference_type(n), pos, x_copy);
      }
      else {
        __uninitialized_copy_fill(start, pos, new_start, start, x_copy);
        start = new_start;
        fill(old_start, pos, x_copy);
      }
    }
    __STL_UNWIND(destroy_nodes_at_front(new_start));
  }
  else {   // 该位置后面的元素比较少
    iterator new_finish = reserve_elements_at_back(n);
    iterator old_finish = finish;
    const difference_type elems_after = difference_type(length) - elems_before;
    pos = finish - elems_after;
    __STL_TRY {
      if (elems_after > difference_type(n)) {
        iterator finish_n = finish - difference_type(n);
        uninitialized_copy(finish_n, finish, finish);
        finish = new_finish;
        copy_backward(pos, finish_n, old_finish);
        fill(pos, pos + difference_type(n), x_copy);
      }
      else {
        __uninitialized_fill_copy(finish, pos + difference_type(n),
        x_copy,
        pos, finish);
        finish = new_finish;
        fill(pos, old_finish, x_copy);
      }
    }
    __STL_UNWIND(destroy_nodes_at_back(new_finish));
  }
}
```




