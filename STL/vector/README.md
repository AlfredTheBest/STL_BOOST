#深入理解STL之Vector
C++内置了数组的类型，在使用数组的时候，必须指定数组的长度，一旦配置了就不能改变了，通常我们的做法是：尽量配置一个大的空间，以免不够用，这样做的缺点是比较浪费空间，预估空间不当会引起很多不便。

STL实现了一个Vector容器，该容器就是来改善数组的缺点。vector是一个动态空间，随着元素的加入，它的内部机制会自行扩充以容纳新元素。因此，vector的运用对于内存的合理利用与运用的灵活性有很大的帮助，再也不必因为害怕空间不足而一开始就配置一个大容量数组了，vector是用多少就分配多少。

要想实现动态分配数组，Vector内部就需要对空间控制做到有效率的掌控，这些机制要如何运作才能高效地实现动态分配呢？本篇博客就从源代码的角度带你领略一下Vector容器内部的构造艺术。

##Vector概述
大家知道，初始化一个数组的时候，需要给数组分配一块内存，数组中的数据都是按序存放的。vector也是如此，再初始化的时候给vector容器分配一块内存，用来存放容器中的数据，一旦分配的内存不足以存放新加入的数据时，就需要扩充空间。STLVector的做法是：重新开辟一段新的空间，将原空间的数据迁移过去，然后新加入的数据存放在新空间之后并释放掉原有空间。

在这个过程中，配置新空间->数据移动->释放旧空间会带来一定的时间成本，所以必须尽可能高效的实现，STL的Vector设计中对这一部分做了相当大的优化，使得时间成本尽可能的小。下面就一起去看看这些优秀的设计吧。

###Vector的数据结构
Vector的数据结构相当简单，由于需要判断内存是否够用，所以要用到三个指针，分别指向头，目前使用空间的尾，目前可用空间的尾。其源代码如下：

```
template <class T, class Alloc = alloc> //alloc是STL的空间配置器
class vector
{
    //这里提供STL标准的allocator接口
    typedef simple_alloc<value_type, Alloc> data_allocator
    
    iterator start;             //内存空间起始点
    iterator finish;            //当前使用的内存空间结束点
    iterator end_of_storage;    //实际分配内存空间的结束点
    
}
```
每当初始化一个vector的时候，先分配一段内存，称为目前可用空间，大小为end_of_storage - start + 1，当往vector里面加入数据的时候，finish就往后移，代表目前已使用的空间，这样做的好处是，不用频繁的扩充空间和转移数据，使得时间成本下降。在上述代码中，我们看到vector采用了STL标准的空间配置其接口。

vector提供了如下函数来支持获取其数据结构中的相关参数。

```
//获取指向vector首元素的迭代器
iterator begin() { return start; }

//获取指向vector尾元素的迭代器
iterator end() { return finish; }

// 返回当前对象个数，即已使用空间的大小
size_type size() const { return size_type(end() - begin()); }

// 返回重新分配内存前最多能存储的对象数，即目前可用空间的大小
size_type capacity() const {return size_type(end_of_storage - begin());}
```

###Vector的迭代器
vector维护的是一段连续的内存空间，所以不论容器中元素的型别为何，普通指针都可以作为vector的迭代器而满足所有必要的条件。vector支持随机存取，所以vector提供的是Random Access Iterator。

下面来看看vector关于迭代器的源码：

```
template <class T, class Alloc = alloc>
class vector
{
public:
	// vector内部是连续内存空间，所以迭代器采用原生指针即可
  typedef value_type* iterator;  								
  //以下为满足Traits功能定义的内嵌型别
  typedef T value_type;
  typedef value_type* pointer;                  
  typedef const value_type* const_pointer;
  typedef const value_type* const_iterator;
  typedef value_type& reference;                
  typedef const value_type& const_reference;
  typedef ptrdiff_t difference_type;  
  typedef size_t size_type;         
}
```


##vector的构造函数
###默认构造函数
在使用vector的时候，我们通常会有如下定义：

```
#include <vector>
vector<int> vec;
```

在上述定义中，调用了vector的默认构造函数，其默认不分配内存空间，如下：

```
// vector的默认构造函数默认不分配内存空间
vector() : start(0), finish(0), end_of_storage(0) {}
```

###带参构造函数

通常，vector的初始化可以指定元素个数和初始化类型。如下：

```
vector<int> vec(10,1); // 将vec初始化为10个1
```

![](./media/vector_001.jpg
)


up vote
437
down vote
accepted
POD stands for Plain Old Data - that is, a class (whether defined with the keyword struct or the keyword class) without constructors, destructors and virtual members functions. Wikipedia's article on POD goes into a bit more detail and defines it as:

```
A Plain Old Data Structure in C++ is an aggregate class that contains only PODS as members, has no user-defined destructor, no user-defined copy assignment operator, and no nonstatic members of pointer-to-member type.
```

带参构造函数

```
// 构造函数，允许指定vector的元素个数和初值
vector(size_type n, const T& value) { fill_initialize(n, value); }
vector(int n, const T& value) { fill_initialize(n, value); }
vector(long n, const T& value) { fill_initialize(n, value); }

// 需要对象提供默认构造函数
explicit vector(size_type n) { fill_initialize(n, T()); }

/**
 * 填充并予以初始化
 */
void fill_initialize(size_type n, const T& value)
{
  start = allocate_and_fill(n, value);
  finish = start + n;                         // 设置当前使用内存空间的结束点
  //这里不过多的分配内存
  end_of_storage = finish;
}

/**
 * 配置一块大小为n的内存空间，并予以填充
 */
iterator allocate_and_fill(size_type n, const T& x)
{
	// 调用STL的空间配置器配置一块大小为n的内存空间
  iterator result = data_allocator::allocate(n); 
  // 调用底层函数uninitialized_fill_n予以填充
  uninitialized_fill_n(result, n, x);
  return result;
}

```


这里面调用了uninitialized_fill_n函数，这个函数是STL的内存基本处理函数，存放在stl_uninitialized.h中，下面来看看它的源码：

```
//如果copy construction 和operator = 等效，并且destructor is trivial
//那么就可以使用本函数
template <class ForwardIterator, class Size, class T>
inline ForwardIterator
__uninitialized_fill_n_aux(ForwardIterator first, Size n,
                           const T& x, __true_type)
{
  return fill_n(first, n, x);
}

// 不是POD类型使用以下函数
template <class ForwardIterator, class Size, class T>
ForwardIterator
__uninitialized_fill_n_aux(ForwardIterator first, Size n,
                           const T& x, __false_type)
{
  ForwardIterator cur = first;
  for ( ; n > 0; --n, ++cur)
    construct(&*cur, x);
  return cur;
}

// 利用type_traits来判断是否是POD类型
template <class ForwardIterator, class Size, class T, class T1>
inline ForwardIterator __uninitialized_fill_n(ForwardIterator first, Size n,
    const T& x, T1*)
{
  typedef typename __type_traits<T1>::is_POD_type is_POD;
  return __uninitialized_fill_n_aux(first, n, x, is_POD());
}

```

##vector的元素操作函数

###push_back()
push_back()函数将新元素插入于vector的尾部，该函数再完成这一操作的时候，先检查是否还有备用空间，如果有直接再备用空间上构造函数；如果没有就扩充空间，通过重新配置一块大空间，移动数据，释放原空间的操作来完成push_back操作。其源代码如下：

```
////////////////////////////////////////////////////////////////////////////////
// 向容器尾追加一个元素, 可能导致内存重新分配
////////////////////////////////////////////////////////////////////////////////
//                          push_back(const T& x)
//                                   |
//                                   |---------------- 容量已满?
//                                   |
//               ----------------------------
//           No  |                          |  Yes
//               |                          |
//               ↓                          ↓
//      construct(finish, x);       insert_aux(end(), x);
//      ++finish;                           |
//                                          |------ 内存不足, 重新分配
//                                          |       大小为原来的2倍
//      new_finish = data_allocator::allocate(len);       <stl_alloc.h>
//      uninitialized_copy(start, position, new_start);   <stl_uninitialized.h>
//      construct(new_finish, x);                         <stl_construct.h>
//      ++new_finish;
//      uninitialized_copy(position, finish, new_finish); <stl_uninitialized.h>
////////////////////////////////////////////////////////////////////////////////
void push_back(const T& x)
{
  // 内存满足条件则直接追加元素, 否则需要重新分配内存空间
  if (finish != end_of_storage) {
    construct(finish, x);
    ++finish;
  }
  else
    insert_aux(end(), x);
}

////////////////////////////////////////////////////////////////////////////////
// 提供插入操作
////////////////////////////////////////////////////////////////////////////////
//                 insert_aux(iterator position, const T& x)
//                                   |
//                                   |---------------- 容量是否足够?
//                                   ↓
//              -----------------------------------------
//        Yes   |                                       | No
//              |                                       |
//              ↓                                       |
// 从opsition开始, 整体向后移动一个位置                     |
// construct(finish, *(finish - 1));                    |
// ++finish;                                            |
// T x_copy = x;                                        |
// copy_backward(position, finish - 2, finish - 1);     |
// *position = x_copy;                                  |
//                                                      ↓
//                            data_allocator::allocate(len);
//                            uninitialized_copy(start, position, new_start);
//                            construct(new_finish, x);
//                            ++new_finish;
//                            uninitialized_copy(position, finish, new_finish);
//                            destroy(begin(), end());
//                            deallocate();
////////////////////////////////////////////////////////////////////////////////

template <class T, class Alloc>
void vector<T, Alloc>::insert_aux(iterator position, const T& x)
{
  if (finish != end_of_storage) {       // 还有剩余内存
    construct(finish, *(finish - 1));
    ++finish;
    T x_copy = x;
    copy_backward(position, finish - 2, finish - 1);
    *position = x_copy;
  }
  else {        
    // 内存不足, 需要重新分配
    const size_type old_size = size();
    //配置原则：如果原大小为0，就配置1个元素大小
    //        如果原大小不为0，就配置原大小的两倍
    //				前半段用来放置原数据，后半段用来放置新数据
    const size_type len = old_size != 0 ? 2 * old_size : 1;
    iterator new_start = data_allocator::allocate(len);
    iterator new_finish = new_start;
    // 将内存重新配置
    __STL_TRY {
    	// 将原vector的内容拷贝到新vector
      new_finish = uninitialized_copy(start, position, new_start);
      // 构造新元素并赋值为x
      construct(new_finish, x);
      // 调整finish的位置
      ++new_finish;
      // 将安插点的原内容也拷贝过来
      new_finish = uninitialized_copy(position, finish, new_finish);
    }
		// 分配失败则抛出异常
    catch (...) {
      destroy(new_start, new_finish);
      data_allocator::deallocate(new_start, len);
      throw;
    }
    // 析构原容器中的对象
    destroy(begin(), end());
    // 释放原容器分配的内存
    deallocate();
    // 调整内存指针状态
    start = new_start;
    finish = new_finish;
    end_of_storage = new_start + len;
  }
}

```

###pop_back()函数
pop_back函数弹出当前尾端元素。其源代码比较简单，如下：

```
void pop_back()
{
	//调整finish
  --finish;
  //释放调弹出的元素
  destroy(finish);
}
```

###erase()函数
erase函数支持两个版本：

* 清除某个位置上的元素

```
iterator erase(iterator position)
{
  if (position + 1 != end())
    copy(position + 1, finish, position); //将[position+1,finish]移到[position,finish]
  --finish;
  destroy(finish);
  return position;//返回删除点的迭代器
}
```

* 清除某个区间上的所有函数

```
iterator erase(iterator first, iterator last)
{
  iterator i = copy(last, finish, first);//关于copy函数的源码分析在以后的博文中会提到
  // 析构掉需要析构的元素
  destroy(i, finish);
  finish = finish - (last - first);
  return first;
}
```

这里放上两张《STL源码剖析》中的图，便于理解这一过程：

![](./media/vector_002.jpg
)

有上述erase函数，可以衍生出一个函数，用来清除迭代器中所有的元素

```
void clear() { erase(begin(), end()); }
```

###insert()函数
insert函数实现的功能是：从position开始，插入n个元素，元素的初值均为x。其源码如下：

```
////////////////////////////////////////////////////////////////////////////////
// 在指定位置插入n个元素
////////////////////////////////////////////////////////////////////////////////
//             insert(iterator position, size_type n, const T& x)
//                                   |
//                                   |---------------- 插入元素个数是否为0?
//                                   ↓
//              -----------------------------------------
//        No    |                                       | Yes
//              |                                       |
//              |                                       ↓
//              |                                    return;
//              |----------- 内存是否足够?
//              |
//      -------------------------------------------------
//  Yes |                                               | No
//      |                                               |
//      |------ (finish - position) > n?                |
//      |       分别调整指针                              |
//      ↓                                               |
//    ----------------------------                      |
// No |                          | Yes                  |
//    |                          |                      |
//    ↓                          ↓                      |
// 插入操作, 调整指针           插入操作, 调整指针           |
//                                                      ↓
//            data_allocator::allocate(len);
//            new_finish = uninitialized_copy(start, position, new_start);
//            new_finish = uninitialized_fill_n(new_finish, n, x);
//            new_finish = uninitialized_copy(position, finish, new_finish);
//            destroy(start, finish);
//            deallocate();
////////////////////////////////////////////////////////////////////////////////
template <class T, class Alloc>
void vector<T, Alloc>::insert(iterator position, size_type n, const T& x)
{
  // 如果n为0则不进行任何操作
  if (n != 0) {
    if (size_type(end_of_storage - finish) >= n) {      // 剩下的内存够分配
      T x_copy = x;
      const size_type elems_after = finish - position; // 计算插入点之后的现有元素个数
      iterator old_finish = finish;
      if (elems_after > n) {  // 插入点之后的现有元素个数大于新增元素个数，见下图1
      	// 先复制尾部n个元素到尾部
        uninitialized_copy(finish - n, finish, finish);
        finish += n; // 调整新的finish
        // 从后往前复制剩余的旧元素
        copy_backward(position, old_finish - n, old_finish);
        // 从position开始填充新元素
        fill(position, position + n, x_copy);
      }
      else {
      	// 插入点之后的现有元素个数小于新增元素个数，见下图2
      	// 先在尾部填充n - elems_after个新增元素
        uninitialized_fill_n(finish, n - elems_after, x_copy);
        // 调整新的finish
        finish += n - elems_after;
        // 复制[position,old_finish]区间的数到新的finish之后
        uninitialized_copy(position, old_finish, finish);
        // 调整finish
        finish += elems_after;
        // 从position开始填充新增元素
        fill(position, old_finish, x_copy);
      }
    }
    else {      // 剩下的内存不够分配, 需要重新分配
      const size_type old_size = size();
      const size_type len = old_size + max(old_size, n);
      iterator new_start = data_allocator::allocate(len);
      iterator new_finish = new_start;
      __STL_TRY {
      	// 将旧的vector中插入点之前的元素复制到新空间，见下图3
        new_finish = uninitialized_copy(start, position, new_start);
        // 将新增元素复制到新空间
        new_finish = uninitialized_fill_n(new_finish, n, x);
        // 将插入点之后的元素复制到新空间
        new_finish = uninitialized_copy(position, finish, new_finish);
      }
      catch (...) {
        destroy(new_start, new_finish);
        data_allocator::deallocate(new_start, len);
        throw;
      }
      // 清除并释放原有vector
      destroy(start, finish);
      deallocate();
      // 调整新的start和finish
      start = new_start;
      finish = new_finish;
      end_of_storage = new_start + len;
    }
  }
}
```

* 插入点之后的现有元素个数大于新增元素个数的情况

![](./media/vector_003.jpg
)

* 插入点之后的现有元素个数小于新增元素个数的情况

![](./media/vector_004.jpg
)

* 剩下的内存不够分配，重新配置的情况

![](./media/vector_005.jpg
)

##Reference
[带你深入理解STL之Vector容器](http://zcheng.ren/2016/08/24/STLVector/)

