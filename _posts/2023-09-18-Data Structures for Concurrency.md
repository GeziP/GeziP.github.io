---
title: Data Structures for Concurrency
date: 2023-09-18 18:18:18 +0800
categories: c++ concurrency
tags: tbb concurrency
---



TBB提供了高度并发的容器类，适用于所有C++多线程应用程序；TBB并发容器类可以与任何线程方法一起使用，当然也包括TBB自身！

C++标准模板库最初并没有考虑并发性。通常情况下，C++ STL容器不支持并发更新，因此尝试同时修改它们可能导致容器损坏。当然，可以使用粗粒度的互斥锁将STL容器包装起来，以确保只有一个线程可以同时操作容器。然而，这种方法会消除并发性，从而限制了并行速度提升，特别是在性能关键的代码中。在前面的文章中，已经展示了使用互斥锁保护元素递增的示例，以防止并发的问题。类似的保护也可以应用于非线程安全的STL例程，以避免正确性问题。如果不在性能关键的部分进行保护，性能影响可能很小。这是一个重要的观点：将容器转换为TBB并发容器应该是出于需要的动机。

在并行应用程序中使用的数据结构应该被设计为支持并发，以实现应用程序的扩展。

TBB中的并发容器提供了与标准模板库（STL）提供的容器类似的功能，但以线程安全的方式实现。例如，tbb::concurrent_vector类似于std::vector类，但允许我们安全地并行增加向量的大小。如果只在并行中从容器中读取数据，则不需要并发容器；只有当我们有修改容器的并行代码时才需要特殊支持。

TBB提供了多个容器类，以与STL容器相兼容的方式替代它们，允许多个线程同时调用同一容器的某些方法。这些TBB容器提供了更高级别的并发性，可以通过以下一种或两种方法之一来实现：
- 细粒度锁定：多个线程只锁定它们真正需要锁定的部分。只要不同线程访问不同的部分，它们就可以同时进行。
- 无锁技术：不同线程会考虑并纠正其他干扰线程的影响。

值得注意的是，TBB并发容器确实会带来一些开销。它们通常比常规STL容器具有更高的开销，因此对它们的操作可能会略长于STL容器。当存在并发访问的可能性时，应该使用并发容器。但如果不可能进行并发访问，建议使用STL容器。也就是说，当从额外的并发性中获得的性能提升超过了它们较慢的顺序性能时，我们使用并发容器。

容器的接口与STL中的接口保持相同，除非需要更改以支持并发性。这可能需要在某些情况下提供新的功能，以确保线程安全性。有一种典型的情况是需要一种新的"pop-if-not-empty"能力（称为try_pop）来代替依赖于使用STL的test-for-empty后跟pop的代码序列。这种STL代码的危险在于，另一个线程可能在原始线程的测试之后但在pop之前将容器清空，从而创建了一个pop实际上会阻塞的竞争条件。这意味着STL代码不是线程安全的。我们可以在整个序列周围放置锁以防止在我们的测试和我们的pop之间修改队列，但是已知在应用程序的并行部分使用这样的锁会降低性能。理解这个简单的例子将有助于阐明支持并行性所需的内容。

与STL一样，TBB容器是关于分配器参数的模板化的。每个容器都使用该分配器来为用户可见的项目分配内存。TBB的默认分配器是与TBB一起提供的可扩展内存分配器。无论指定了哪个分配器，容器的实现可能还会使用不同的分配器来严格处理内部结构。

目前，TBB提供以下并发容器：
- 无序关联容器
  - Unordered map （包括unordered multimap  ）
  - Unordered set  （包括unordered multiset  ）
  - Hash table  
- Queue  （包括有界队列和优先队列）
- Vector

## 并发无序关联容器

| 类名和C++11连接说明           | 并行遍历和交叉 | key有关联value | 支持同步擦除 | 内置锁定 | lock-free接口 | 允许插入相同item | []和at 函数 |
| ----------------------------- | :------------: | -------------- | ------------ | -------- | ------------- | ---------------- | ----------- |
| concurrent_hash_map           |       √        | √              | √            | √        | ×             | ×                | ×           |
| concurrent_unordered_map      |       √        | √              | ×            | ×        | √             | ×                | √           |
| concurrent_unordered_multimap |       √        | √              | ×            | ×        | √             | √                | ×           |
| concurrent_unordered_set      |       √        | ×              | ×            | ×        | √             | ×                | ×           |
| concurrent_unordered_multiset |       √        | ×              | ×            | ×        | √             | √                | ×           |

​									表1.并发无序关联容器的比较

无序关联容器是一组实现哈希表变体的类模板。表1列出了这些容器及其关键的区分特征。并发无序关联容器可以用于存储任意元素，如整数或自定义类，因为它们是模板。TBB提供了可以在并发环境中高效执行的无序关联容器的实现。

哈希映射(hash map)（通常称为哈希表(hash table)）是一种数据结构，使用哈希函数将键映射到值。哈希函数计算出一个键的索引，然后使用该索引来访问与键关联的“桶(bucket)”，在其中存储了与该键相关的值。

选择一个好的哈希函数非常重要！完美的哈希函数会将每个键分配到一个唯一的桶中，因此不同键之间不会发生冲突。然而，在实际中，哈希函数并不完美，偶尔会为多个键生成相同的索引。这些冲突需要哈希表实现的某种形式的处理，这将引入一些开销 - 哈希函数应设计成通过将输入散列到几乎均匀分布在桶中来最小化冲突。

哈希映射的优势在于，在平均情况下，提供O(1)的搜索、插入和键操作时间。TBB哈希映射的优势在于支持并发使用，既能确保正确性，又能提高性能。这前提是要使用一个良好的哈希函数 - 一个不会为使用的键引发许多冲突的哈希函数。当存在不完美的哈希函数或哈希表未正确调整大小时，理论上的最坏情况仍然为O(n)。

通常情况下，哈希映射在实际使用中比其他查找表数据结构，包括搜索树，更高效。这使得哈希映射成为许多用途的首选数据结构，包括关联数组、数据库索引、缓存和集合。

### concurrent_hash_map

TBB提供了concurrent_hash_map，它以一种允许多个线程通过find、insert和erase方法并发访问值的方式将键映射到值。正如我们稍后将讨论的那样，tbb::concurrent_hash_map是为并行性而设计的，因此其接口是线程安全的，不同于我们将在本章后面介绍的STL map/set接口。

这些键是无序的。concurrent_hash_map中每个键最多只有一个元素。该键可能在操作中有其他元素，但不在映射中。类型HashCompare指定了如何对键进行哈希和如何比较它们的相等性。与哈希表通常预期的一样，如果两个键相等，那么它们必须哈希到相同的哈希码。这就是为什么HashCompare将比较(compare)和哈希(hash)的概念合并成一个单独的对象，而不是分开处理它们的原因。这也意味着在哈希表非空的情况下，我们不应更改键的哈希码。

concurrent_hash_map充当了std::pair<const Key,T>类型元素的容器。通常，在访问容器元素时，我们要么是对其进行更新，要么是对其进行读取。模板类concurrent_hash_map分别支持这两个目的，使用充当智能指针的accessor和const_accessor类。accessor表示更新（写入）访问。只要它指向一个元素，所有其他尝试在表中查找该键的操作都会被阻塞，直到accessor完成。const_accessor类似，只不过它表示只读访问。多个accessor可以同时指向相同的元素。在元素经常被读取而不经常被更新的情况下，这个特性可以极大地提高并发性能。

我们在下面code1中共享了一个使用concurrent_hash_map容器的简单示例代码。通过缩短元素访问的生命周期，可以提高此示例的性能。find和insert方法以accessor或const_accessor作为参数。选择告诉concurrent_hash_map我们是要请求更新访问还是只读访问。一旦方法返回，访问将持续到accessor或const_accessor被销毁。由于访问元素可能会阻塞其他线程，因此尽量缩短accessor或const_accessor的生命周期。为此，请在尽可能内层的块中声明它。要比块的结束更早释放访问权，请使用release方法。#ifdef FASTER中的循环体的重新编写，使用release而不是依赖销毁来结束线程生命周期。remove(key)方法也可以并发操作。它隐式请求写入访问权限。因此，在删除键之前，它会等待键上的任何其他现存访问。

```cpp
#include <tbb/concurrent_hash_map.h>
#include <tbb/blocked_range.h>
#include <tbb/parallel_for.h>
#include <string>
 
// Structure that defines hashing and comparison operations for user's type.
struct MyHashCompare {
  static size_t hash( const std::string& x ) {
    size_t h = 0;
    for( const char* s = x.c_str(); *s; ++s )
      h = (h*17)^*s;
    return h;
  }
  //! True if strings are equal
  static bool equal( const std::string& x, const std::string& y ) {
    return x==y;
  }
};
 
// A concurrent hash table that maps strings to ints.
typedef tbb::concurrent_hash_map<std::string,int,MyHashCompare> StringTable;
 
// Function object for counting occurrences of strings.
struct Tally {
  StringTable& table;
  Tally( StringTable& table_ ) : table(table_) {}
  void operator()( const tbb::blocked_range<std::string*> range ) const {
    // the next few lines can be improved, see Figure 6.4 (define FASTER in compilation)
    for( std::string* p=range.begin(); p!=range.end(); ++p ) {
      StringTable::accessor a;
      table.insert( a, *p );
      a->second += 1;
#ifdef FASTER
      a.release();
#endif
    }
  }
};

const size_t N = 10;
 
std::string Data[N] = { "Hello", "World", "TBB", "Hello",
			"So Long", "Thanks for all the fish", "So Long",
			"Three", "Three", "Three" };
 
int main() {
  // Construct empty table.
  StringTable table;
 
  // Put occurrences into the table
  tbb::parallel_for( tbb::blocked_range<std::string*>( Data, Data+N, 1000 ),
		     Tally(table) );
 
  // Display the occurrences using a simple walk
  // (note: concurrent_hash_map does not offer const_iterator)
  // see a problem with this code???
  // read "Iterating thorough these structures is asking for trouble"
  // coming up in a few pages
  for( StringTable::iterator i=table.begin();
       i!=table.end(); 
       ++i )
    printf("%s %d\n",i->first.c_str(),i->second);

  return 0;
}
```

​									code 1.Hash Table 例子

PERFORMANCE TIPS FOR HASH MAPS  ：

- 始终为哈希表指定初始大小。默认的大小为1，会导致性能严重下降！一个好的初始大小通常应该从几百开始。如果较小的大小看起来正确，那么在小表上使用锁定会因缓存局部性而在速度上有优势。

- 检查您的哈希函数 - 确保哈希值的低位有良好的伪随机性。特别地，您不应该使用指针作为键，因为通常指针的低位会由于对象对齐而包含一组零位。如果是这种情况，强烈建议将指针除以其指向的类型的大小，从而将总是零位替换为可变位。乘以一个质数，并且移除一些低位位数，是一种可考虑的策略。与任何形式的哈希表一样，相等的键必须具有相同的哈希码，理想的哈希函数应该在哈希码空间中均匀分布键。优化哈希函数肯定是应用程序特定的，但使用TBB提供的默认哈希函数通常效果良好。

- 如果可以避免使用accessor，请不要使用它们，并在需要accessor时尽量限制它们的生命周期。它们实际上是细粒度的锁，存在时会阻塞其他线程，因此可能会限制扩展性。

- 使用TBB内存分配器（TODO，提供个链接）。如果要强制使用内存分配器，请将scalable_allocator用作容器的模板参数（不允许回退到malloc）- 这至少在开发期间进行性能测试时是一个良好的健全性检查。

### 支持map/multimap和set/multiset的并发接口

标准C++ STL定义了unordered_set、unordered_map、unordered_multiset和unordered_multimap。这些容器之间唯一的区别在于它们对元素施加的约束不同。表1是一个方便的参考，用于比较我们在并发map/set支持中有五种选择，包括我们在代码示例中使用的tbb::concurrent_hash_map。

STL没有定义任何名为“hash”的东西，因为最初C++没有定义哈希表。对于将哈希表支持添加到STL的兴趣很广泛，因此有广泛使用的STL版本进行了扩展，包括SGI、gcc和Microsoft等版本，以包括哈希表支持。由于没有标准，关于“哈希表”或“哈希映射”在C++程序员眼中的功能和性能意味着什么出现了差异。从C++11开始，STL添加了对哈希表的支持，并选择了unordered_map作为类的名称，以防止与预标准实现产生混淆和冲突。可以说unordered_map这个名称更具描述性，因为它暗示了类的接口以及其元素的无序性。

最初的TBB哈希表支持早于C++11，称为tbb::concurrent_hash_map。这个哈希函数仍然非常有价值，不需要改变以匹配标准。TBB现在包括对unordered_map和unordered_set的支持，以反映C++11的增加，接口只在需要支持并发访问时进行了增强或调整。避免使用一些不友好于并行的接口是"引导我们"进行有效的并行编程的一部分。对于更好的并行扩展性，三个值得注意的调整如下：

• 忽略需要C++11语言特性（例如，右值引用）的方法。

• C++标准函数的erase方法前缀加上unsafe_以指示它们不是并发安全的（因为只有concurrent_hash_map支持并发删除）。这不适用于concurrent_hash_map，因为它支持并发删除。

• 桶方法（桶的数量、最大桶的数量、桶的大小以及通过桶进行迭代的支持）前缀加上unsafe_，以提醒它们在插入方面不是并发安全的。它们受到STL兼容性的支持，但如果可能的话应该避免使用。如果使用了这些接口，应该防止它们与插入操作同时进行。这些接口不适用于concurrent_hash_map，因为TBB的设计者避免了这样的功能。

### 内置锁定与无可见锁定

容器`concurrent_hash_map`和`concurrent_unordered_*`在访问元素的锁定方面存在一些差异。因此，在竞争情况下它们可能会表现出非常不同的行为。`concurrent_hash_map`的`accessor`本质上是锁定：`accessor`是独占锁，`const_accessor`是共享锁。基于锁的同步已经内置到容器的使用模型中，不仅保护容器的完整性，而且在一定程度上也保护了数据的完整性。code1的代码在向表中插入数据时使用了一个`accessor`。

### **遍历这些结构可能会引发问题**

在code1中，当我们遍历哈希表以将其输出时，我们混入了一些并发不安全的代码。如果在我们遍历表的过程中进行了插入或删除操作，这可能会有问题。为自己辩护，我们只会说“这是调试代码 - 我们不关心！”但是，经验告诉我们，这样的代码很容易渗入生产代码中。要小心！

TBB的设计者为了调试目的保留了`concurrent_hash_map`的迭代器，但他们故意没有诱惑我们将迭代器作为其他成员的返回值。

不幸的是，STL以一些我们应该学会抵制的方式引诱我们。`concurrent_unordered_*`容器与`concurrent_hash_map`不同 - API遵循C++标准的关联容器（请记住，原始的TBB `concurrent_hash_map`早于C++对并发容器的任何标准化）。添加或查找数据的操作返回一个迭代器，这引诱我们使用它进行迭代。在并行程序中，我们面临与`map/set`上的其他操作同时进行的风险。如果我们屈服于诱惑，保护数据完整性完全取决于程序员自己，容器的API不会提供帮助。可以说C++标准容器提供了额外的灵活性，但缺乏`concurrent_hash_map`提供的内置保护。STL接口足够容易同时使用，只要我们避免使用从添加或查找操作返回的迭代器来进行除了引用我们查找的项之外的任何操作。如果我们屈服于诱惑（我们不应该这样做！），那么我们需要仔细思考应用程序中的并发更新问题。当然，如果没有发生更新操作 - 只有查找操作 - 那么使用迭代器就不会引发并行编程问题。

## **并发队列：常规队列、有界队列和优先队列**

队列是有用的数据结构，其中项目通过称为push（添加）和pop（删除）的操作添加或删除队列。无界队列接口提供了“尝试弹出”操作，告诉我们队列是否为空并且没有值从队列中弹出。这使我们不必编写自己的逻辑来避免通过测试是否为空来进行阻塞式弹出 - 这是一个不是线程安全的操作(见code3)。

在多个线程之间共享队列可以是一种有效的方式，用于将工作项从一个线程传递到另一个线程 - 一个包含“工作”的队列可以添加工作项以请求将来的处理，并由希望执行处理的任务删除这些项。

通常，队列以先进先出（FIFO）的方式运行。如果我从一个空队列开始，执行push(10)，然后执行push(25)，那么第一个pop操作将返回10，第二个pop将返回25。这与堆栈的行为大不相同，堆栈通常是后进先出。但是，我们这里不讨论堆栈！

我们在code2的Simple Q展示了一个简单的示例，清楚地显示pop操作以与push操作将它们添加到队列的顺序返回值。

```cpp
#include <tbb/concurrent_queue.h>
#include <tbb/concurrent_priority_queue.h>
#include <iostream>

int myarray[10] = { 16, 64, 32, 512, 1, 2, 512, 8, 4, 128 };

void pval(int test, int val) {
  if (test) {
    std::cout << " " << val;
  } else {
    std::cout << " ***";
  }
}

void simpleQ() {
  tbb::concurrent_queue<int> queue;
  int val = 0;

  for( int i=0; i<10; ++i )
    queue.push(myarray[i]);

  std::cout << "Simple  Q   pops are";

  for( int i=0; i<10; ++i )
    pval( queue.try_pop(val), val );

  std::cout << std::endl;
}

void prioQ() {
  tbb::concurrent_priority_queue<int> queue;
  int val = 0;

  for( int i=0; i<10; ++i )
    queue.push(myarray[i]);

  std::cout << "Prio    Q   pops are";

  for( int i=0; i<10; ++i )
    pval( queue.try_pop(val), val );

  std::cout << std::endl;
}

void prioQgt() {
  tbb::concurrent_priority_queue<int,std::greater<int>> queue;
  int val = 0;

  for( int i=0; i<10; ++i )
    queue.push(myarray[i]);

  std::cout << "Prio    Qgt pops are";

  for( int i=0; i<10; ++i )
    pval( queue.try_pop(val), val );

  std::cout << std::endl;
}

void boundedQ() {
  tbb::concurrent_bounded_queue<int> queue;
  int val = 0;

  queue.set_capacity(6);

  for( int i=0; i<10; ++i )
    queue.try_push(myarray[i]);

  std::cout << "Bounded Q   pops are";

  for( int i=0; i<10; ++i )
    pval( queue.try_pop(val), val );

  std::cout << std::endl;
}

int main() {
  simpleQ();
  boundedQ();
  prioQ();
  prioQgt();
  return 0;
}
```

​									code 2.并发队列例子

Simple Q的输出为：

```cmd
Simple Q pops are 16 64 32 512 1 2 512 8 4 128
```

队列有两个特殊之处：有界和优先级。有界队列引入了限制队列大小的概念。这意味着如果队列已满，可能无法进行push操作。为了处理这种情况，有界队列接口提供了等待直到可以添加到队列的方法，或者提供了“尝试推送”的操作，如果可以进行推送，它将执行推送，否则通知我们队列已满。默认情况下，有界队列是无限的！如果我们需要有界队列，我们需要使用`concurrent_bounded_queue`并调用`set_capacity`方法来设置队列的大小。我们在code 2的bounded Q中展示了有界队列的简单用法，其中只有前六个推送的项目进入了队列。我们可以在`try_push`上添加测试并执行某些操作。在这种情况下，当pop操作发现队列为空时，程序将打印***。

bounded Q的输出为：

```cmd
Simple 	Q pops are 16 64 32 512 1 2 512 8 4 128
Bounded Q pops are 16 64 32 512 1 2 *** *** *** ***
```

优先级队列为先进先出队列增加了一点独特性，通过有效地对队列中的项目进行排序。

如果我们在代码中没有指定优先级，默认的优先级是`std::less<T>`。这意味着pop操作将返回队列中具有最高值的项目。code2的prio Q展示了两个优先级使用的示例，一个默认使用`std::less<int>`，而另一个明确指定了`std::greater<int>`。

prio Q的输出为：

```cmd
Simple 	Q 	pops are 16 64 32 512 1 2 512 8 4 128
Bounded Q 	pops are 16 64 32 512 1 2 *** *** *** ***
Prio 	Q 	pops are 512 512 128 64 32 16 8 4 2 1
Prio 	Qgt pops are 1 2 4 8 16 32 64 128 512 512
```

正如我们在前面的三个示例中所展示的，为了实现这三种队列变体，TBB提供了三个容器类：concurrent_queue、concurrent_bounded_queue和concurrent_priority_queue。所有并发队列都允许多个线程同时推送(push)和弹出(pop)项目。接口类似于STL的std::queue或std::priority_queue，除非必须有所不同以确保安全地进行队列的并发修改。

队列上的基本方法是push和try_pop。push方法的工作方式与std::queue相同。需要注意的是，不支持front或back方法，因为在并发环境中它们不安全，因为这些方法返回队列中项目的引用。在并行程序中，队列的front或back可能会被另一个线程并行更改，从而使front或back的使用毫无意义。

类似地，对于无界队列，不支持pop和测试是否为空的方法 - 取而代之的是定义了try_pop方法，如果项目可用，则弹出并返回true状态；否则，它不返回任何项目，并返回false状态。测试是否为空和pop方法被合并成一个单一方法，以鼓励线程安全编码。对于有界队列，除了可能会阻塞的push方法外，还有一个非阻塞的try_push方法。这有助于我们避免使用size方法来查询队列的大小。通常情况下，应该避免使用size方法，特别是如果它们是从顺序程序的遗留下来的。由于队列的大小可以在并行程序中同时更改，因此如果使用size方法，需要仔细考虑。其中一件事是，当队列为空且有未处理的pop方法时，TBB可以为size方法返回负值。empty方法在size为零或更小时返回true。

**大小限制**

对于`concurrent_queue`和`concurrent_priority_queue`，容量是无限的，受目标计算机上的内存限制。`concurrent_bounded_queue`提供了对界限的控制 - 一个关键特性是push方法将阻塞，直到队列有空间。有界队列在减缓生产者以匹配消费者方面非常有用，而不是允许队列不受限制地增长。

`concurrent_bounded_queue`是唯一提供pop方法的`concurrent_queue_*`容器。pop方法将阻塞，直到有项目可用。只有在`concurrent_bounded_queue`中，push方法才可能是阻塞的，因此这种容器类型还提供了一个非阻塞方法称为`try_push`。

这种将界限与速率匹配的概念，以避免内存溢出或超额分配核心，也存在于流图（请参见第3章），通过使用`limiter_node`来实现。

**优先级排序**

优先级队列基于排队项目的优先级维护队列中的排序。正如我们之前提到的，普通队列具有先进先出的策略，而优先级队列对其项目进行排序。我们可以提供自己的`Compare`函数来更改默认的排序方式，例如，使用`std::greater<T>`将导致最小的元素在`pop`方法中被检索。在code2的prio Q示例代码中，我们正是这样做的。

**保持线程安全：尽量避免使用top、size、empty、front、back**

重要的要注意，没有`top`方法，我们应该避免使用`size`和`empty`方法。并发使用意味着所有三个方法的值都可以由其他线程中的push/pop方法而改变。此外，尽管支持`clear`和`swap`方法，但它们不是线程安全的。当将`std::priority_queue`用法转换为`tbb::concurrent_priority_queue`时，TBB强制我们重新编写使用`top`的代码，因为返回的元素可能会被并发`pop`使之无效。由于返回值不会受到并发的影响，TBB确实支持`std::priority_queue`的`size`、`empty`和`swap`方法。然而，我们建议在并发应用程序中仔细审查使用这两个函数的wisdom，因为对任何一个的依赖很可能意味着需要为并发重新编写代码。

![image-1](/assets/image/posts/2023-09-18-Data Structures for Concurrency/1.png)

code 3 .Motivation for try_pop instead of top and pop shown in a side-by-side comparison of STL and TBB priority queue code. Both will total 50005000 in this example without parallelism, but the TBB scales and is thread-safe.  

**迭代器**

仅用于调试目的，所有三个并发队列都提供有限的迭代器支持（具有`iterator`和`const_iterator`类型）。此支持仅旨在允许我们在调试期间检查队列。`iterator`和`const_iterator`类型都遵循前向迭代器的常规STL约定。迭代顺序是从最近推入到最近推入。修改队列会使引用它的任何迭代器无效。这些迭代器相对较慢，应仅用于调试。使用示例如下。

```cpp
#include <tbb/concurrent_queue.h>
#include <iostream>

int main() {
  tbb::concurrent_queue<int> queue;
  for( int i=0; i<10; ++i )
    queue.push(i);
  for( tbb::concurrent_queue<int>::const_iterator
       i(queue.unsafe_begin()); 
       i!=queue.unsafe_end();
       ++i )
    std::cout << *i << " ";
  std::cout << std::endl;
  return 0;
}
```

​							code 4.迭代并发队列的调试代码示例。
注意 begin 和 end 上的 unsafe_ 前缀，以强调这些方法的调试唯一性和非线程安全特性。
这些方法的非线程安全性质。

输出为：

```cmd
0 1 2 3 4 5 6 7 8 9
```

**为什么使用这个并发队列：A-B-A 问题**

我们在本文一开始就提到，由并行专家编写的容器对我们来说有着重要的价值，我们只需“直接使用”它们。没有人应该希望为每个应用程序重新发明良好可扩展的实现。

作为动机，我们偏离一下，提到 A-B-A 问题 - 这是一个典型的计算机科学中并行问题的例子！乍一看，并发队列似乎很容易只需编写自己的。但事实并非如此。使用TBB的`concurrent_queue`，或者任何其他经过充分研究和实施的并发队列都是一个好主意。如果 A-B-A 问题妨碍了我们的意图，那么上一篇文中的更新习惯（compare_and_swap）是不合适的。当尝试为链接数据结构设计非阻塞算法时，包括并发队列时，这是一个常见的问题。TBB的设计者已经在并发队列的解决方案中打包了 A-B-A 问题的解决方案。我们可以完全依赖它。当然，这是开源代码，所以如果你感兴趣，你可以在代码中查看解决方案。如果你查看源代码，你会发现***<u>竞技场管理</u>***也必须处理 A-B-A 问题。

当然，你可以只使用TBB而不需要知道任何这些。我们只是想强调，解决并发数据结构问题并不像看起来那么容易 - 这就是我们喜欢使用TBB支持的并发数据结构的原因。

**ABA问题**

理解 A-B-A 问题是训练我们在设计自己的算法时思考并发性影响的关键方法之一。尽管 TBB 在实现并发队列和其他 TBB 结构时避免了 A-B-A 问题，但这提醒我们需要“思考并行”。

A-B-A 问题发生在一个线程检查某个位置以确保其值为 A，并且仅在值为 A 时才进行更新的情况下。问题是，如果其他任务以第一个任务未能检测到的方式更改了相同的位置，那么是否会成为问题：

1. 任务从 globalx 读取值 A。
2. 其他任务将 globalx 从 A 更改为 B，然后再次更改为 A。
3. 步骤 1 中的任务执行其 compare_and_swap 操作，读取 A，因此未检测到中间的更改为 B。

如果任务在错误的假设下继续操作，即自从首次读取后位置未发生更改，那么任务可能会继续损坏对象或以其他方式得到错误的结果。

考虑一个带有链表的示例。假设有一个链表 W(1)→X(9)→Y(7)→Z(4)，其中字母表示节点位置，数字表示节点中的值。假设某个任务遍历列表以查找要出列的节点 X。该任务获取下一个指针 X.next（即 Y），打算将其放入 W.next。但是，在执行交换之前，该任务被暂停了一段时间。 在暂停期间，其他任务正在忙碌。它们出列 X，然后偶然重用相同的内存，并在某个时间点排队一个新版本的节点 X，同时出列 Y 并添加 Q。现在，列表变成了 W(1)→X(2)→Q(3)→Z(4)。 一旦原始任务最终醒来，它发现 W.next 仍然指向 X，因此它将 W.next 交换为 Y，从而彻底破坏了链表。

原子操作是解决这个问题的方法，如果它们为我们的算法提供足够的保护。如果 A-B-A 问题可能会引发问题，我们需要找到更复杂的解决方案。tbb::concurrent_queue 具有必要的额外复杂性来解决这个问题！

何时不使用队列：考虑算法！ 在并行程序中，队列广泛用于缓冲生产者和消费者之间的数据。在使用显式队列之前，我们需要考虑使用 parallel_do 或 pipeline。出于以下原因，这些选项通常比队列更高效： 

• 队列本质上是瓶颈，因为它们必须维护顺序。 

• 弹出值的线程将在队列为空时阻塞，直到有值被推入。 

• 队列是一种被动数据结构。如果一个线程推送一个值，直到弹出该值可能需要一些时间，与此同时，该值（及其引用的任何内容）会在缓存中变得cold。更糟糕的是，另一个线程可能会弹出该值，并且该值（及其引用的任何内容）必须移动到另一个处理器核心。 相比之下，parallel_do 和 pipeline 避免了这些瓶颈。由于它们的线程是隐式的，它们会优化工作线程的使用方式，使其在出现值之前进行其他工作。它们还尽量保持项目在缓存中保持hot。例如，当将另一个工作项添加到 parallel_do 时，除非另一个空闲线程可以在hot线程处理之前偷走它，否则它将保留在添加它的线程中。这样，项目更有可能由hot线程处理，从而减少获取数据的延迟。

**并发向量（Concurrent Vector）**
TBB 提供了一个名为 concurrent_vector 的类。concurrent_vector<T> 是 T 的动态可增长数组。即使在其他线程也在操作它的元素，甚至增加它们的情况下，也可以安全地增长 concurrent_vector。为了安全地并发增长，concurrent_vector 具有支持动态数组常见用途的三个方法：push_back、grow_by 和 grow_to_at_least。
code5显示了 concurrent_vector 的简单用法，下面的结果图显示了向量内容的转储，显示了并行线程同时添加的效果。如果按数字顺序排序，同一程序的输出将完全相同。
**何时使用 tbb::concurrent_vector 而不是 std::vector**
concurrent_vector<T> 的关键价值在于它能够同时增长向量，并且能够保证元素在内存中不会移动。

concurrent_vector 相对于 std::vector 具有更多的开销。因此，在我们需要在其他访问正在（或可能）进行时动态调整大小或要求元素永不移动时，应使用 concurrent_vector。

```cpp
#include <iostream>

#include <tbb/concurrent_vector.h>
#include <tbb/parallel_for.h>

void oneway() {
//  Create a vector containing integers
    tbb::concurrent_vector<int> v = {3, 14, 15, 92};

    // Add more integers to vector IN PARALLEL 
    for( int i = 100; i < 1000; ++i ) {
	v.push_back(i*100+11);
	v.push_back(i*100+22);
	v.push_back(i*100+33);
	v.push_back(i*100+44);
    }

    // Iterate and print values of vector (debug use only)
    for(int n : v) {
      std::cout << n << std::endl;
    }
}

void allways() {
//  Create a vector containing integers
    tbb::concurrent_vector<int> v = {3, 14, 15, 92};

    // Add more integers to vector IN PARALLEL 
    tbb::parallel_for( 100, 999, [&](int i){
	v.push_back(i*100+11);
	v.push_back(i*100+22);
	v.push_back(i*100+33);
	v.push_back(i*100+44);
      });

    // Iterate and print values of vector (debug use only)
    for(int n : v) {
      std::cout << n << std::endl;
    }
}

int main() {
  oneway();
  std::cout << std::endl;
  allways();
  return 0;
}
```

​						Code 5. Concurrent vector small example

![image-2](/assets/image/posts/2023-09-18-Data Structures for Concurrency/2.png)

图6.输出结果，左侧是使用 for时产生的输出，右侧是使用 parallel_for 时产生的输出。
**元素永不移动**
concurrent_vector 直到数组被清除之前永不移动元素，即使对于单线程代码，这也可能是优势，与 STL std::vector 不同，concurrent_vector 在增长时不会移动现有元素。该容器分配一系列连续的数组。第一次预留、增长或分配操作决定了第一个数组的大小。使用少量元素作为初始大小会导致跨缓存行的碎片化，可能会增加元素访问时间。shrink_to_fit() 方法将几个较小的数组合并成一个连续的数组，可能会提高访问时间。

**concurrent_vector 的并发增长**
虽然并发增长与理想的异常安全性基本不兼容，但 concurrent_vector 确实提供了一定程度的异常安全性。元素类型必须具有永不抛出异常的析构函数，如果构造函数可能抛出异常，则析构函数必须是nonvirtual的，并且在零填充内存上正常工作。

push_back(x) 方法安全地将 x 追加到向量。grow_by(n) 方法安全地追加 n 个连续元素，并初始化为 T()。这两种方法都返回一个指向第一个追加元素的迭代器。每个元素都使用 T() 初始化。以下例程安全地将 C 字符串附加到共享向量。

```cpp
void Apeend( concurrent_vector<char> & vector, const char*string){
    size_t n = strlen(string) + 1;
    std::copy( string, string+n, vector.grow_by(n));
}
```

grow_to_at_least(n) 如果向量较短，则将向量扩展到大小 n。对增长方法的并发调用不一定按照追加到向量的元素的顺序返回。

size() 返回向量中的元素数量，这可能包括仍在通过 push_back、grow_by 或 grow_to_at_least 方法进行并发构造的元素。前面的示例使用了 std::copy 和迭代器，而不是 strcpy 和指针，因为 concurrent_vector 中的元素可能不在连续地址上。在 concurrent_vector 增长时使用迭代器是安全的，只要迭代器永远不会超过 end() 的当前值。但是，迭代器可能引用正在进行并发构造的元素。因此，我们需要同步构建和访问。

concurrent_vector 上的操作在增长方面是与并发安全相关的，但不适用于清除或销毁向量。如果 concurrent_vector 上有其他操作正在进行，请勿调用 clear()。

总结
在本文中，我们讨论了 TBB 中支持的三种关键数据结构（Hash/Map/Set、Queue和Vector）。这些 TBB 提供的支持具有线程安全性（允许并发运行）以及具有良好扩展性的实现。我们提供了一些需要避免的建议，因为它们往往会在并行程序中引起问题，包括仅使用Map/Set返回的迭代器引用查找的项以外的任何内容。我们回顾了 A-B-A 问题，既是使用 TBB 而不是编写自己的动机，也是需要在并行程序共享数据时进行思考的一个极好示例。

尽管所有这些容器都支持并行使用，但我们不能强调足够的概念，即通过思考算法以最小化任何类型的同步来实现高性能并行编程至关重要。如果您可以通过使用 parallel_do、pipeline、parallel_reduce 等来避免共享数据结构，您可能会发现您的程序具有更好的可扩展性，因为深思熟虑这一点对于最有效的并行编程非常重要。

