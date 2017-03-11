#STL之Stack和Queue

* 如果需要随机访问一个容器，vector比list要好
* 如果需要经常插入和删除操作的话，list比vector要好
* 如果既要随机存取，又要关心两端数据的插入和删除，则选择deque

好了，复习完前面的知识后，开始介绍今天的两个容器stack和queue。由于stack和queue都是基于deque来实现的，所以相应的代码会比较简单，也是比较轻松易实现的，下面一起去看看吧。

##stack
如果把deque比作一个管道，两头都可进可出的话，stack就是一个桶！只能一头进一头出，而且，压在下面的东西你看不到，你要是想看，只能把上面的东西拿出来再去看。

stack是一种先进后出的数据结构，允许新增元素、移除元素和取得最顶端的元素，除了最顶端，没有任何其他方法可以存取stack中的元素，也就是说stack没有遍历行为，因此，stack是没有迭代器的！！！！！

以deque为底层容器来实现stack这种数据结构，简直不能再简单，基本的操作函数都已经定义好了，deque可以为它完成所有工作。与其说stack是一种容器，倒不如说它是一种配接器，一种容器适配器。

下面我们就来看看stack的源码，真的没骗你，超级简单。

```
template <class T, class Sequence = deque<T>  // 以deque作为缺省底层容器
class stack
{
	// #define __STL_NULL_TMPL_ARGS <>
  friend bool operator== __STL_NULL_TMPL_ARGS (const stack&, const stack&);
  friend bool operator< __STL_NULL_TMPL_ARGS (const stack&, const stack&);
public:
  typedef typename Sequence::value_type value_type;
  typedef typename Sequence::size_type size_type;
  typedef typename Sequence::reference reference;
  typedef typename Sequence::const_reference const_reference;
protected:
  Sequence c;   // 底层容器，stack全靠它来实现
public:
	// 以下函数直接调用底层容器的接口即可实现
  // 判断stack是否为空
  bool empty() const { return c.empty(); }	
  // stack中元素个数
  size_type size() const { return c.size(); }
  // 返回栈顶元素, 注意这里返回的是引用!!!
  reference top() { return c.back(); }
  const_reference top() const { return c.back(); }
  // 在栈顶追加新元素
  void push(const value_type& x) { c.push_back(x); }
  // 移除栈顶元素, 注意不返回元素的引用,
  // 很多初学者随机用此容器时经常误认为pop()操作同时会返回栈顶元素的引用
  void pop() { c.pop_back(); }
};
// 判断两个stack是否相等, 就要测试其内部维护容器是否相等
// x.c == y.c会调用容器重载的operator ==
template <class T, class Sequence>
bool operator==(const stack<T, Sequence>& x, const stack<T, Sequence>& y)
{
  return x.c == y.c;
}
// 比较两个迭代器的大小，即比较底层容器的大小
template <class T, class Sequence>
bool operator<(const stack<T, Sequence>& x, const stack<T, Sequence>& y)
{
  return x.c < y.c;
}
```

你没有看错，stack的源码就只有上面几句话，全是调用底层容器的接口。下面再来看看它的同胞queue，同样很简单。

##queue

queue是一种先进先出的数据结构，上面说道，dequeu是个双向可进可出的管道，stack是一个桶，queue就是一个单向的水管，只能一端进，一端出。

queue允许新增元素、移除元素、从最底端插入元素，从最顶端取得元素，但是，从了最底端插入，最顶端取出之外，没有任何其他方法可以存取queue里面的元素，queue和stack一样，不允许有遍历行为，因此，queue也没有迭代器！！！！

queue和stack一样，也是一种容器适配器，只需要调用底层容器的接口就能实现。下面来看看它的源码吧。

```
template <class T, class Sequence = deque<T> >
class queue
{
  friend bool operator== __STL_NULL_TMPL_ARGS (const queue& x, const queue& y);
  friend bool operator< __STL_NULL_TMPL_ARGS (const queue& x, const queue& y);
public:
  // 由于queue仅支持对队头和队尾的操作, 所以不定义STL要求的
  // pointer, iterator, difference_type
  typedef typename Sequence::value_type value_type;
  typedef typename Sequence::size_type size_type;
  typedef typename Sequence::reference reference;
  typedef typename Sequence::const_reference const_reference;
protected:
  Sequence c;   // 底层容器
public:
	// 以下操作和stack一样
  bool empty() const { return c.empty(); }
  size_type size() const { return c.size(); }
  reference front() { return c.front(); }
  const_reference front() const { return c.front(); }
  reference back() { return c.back(); }
  const_reference back() const { return c.back(); }
  void push(const value_type& x) { c.push_back(x); }
  void pop() { c.pop_front(); }
};
// 重载==操作符，比较底层容器即可
template <class T, class Sequence>
bool operator==(const queue<T, Sequence>& x, const queue<T, Sequence>& y)
{
  return x.c == y.c;
}
// 同上
template <class T, class Sequence>
bool operator<(const queue<T, Sequence>& x, const queue<T, Sequence>& y)
{
  return x.c < y.c;
}
```

