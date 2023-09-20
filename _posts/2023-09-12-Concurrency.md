---
title: Concurrency
date: 2023-09-12 18:18:18 +0800
categories: concurrency
tags: tbb
---



先看一个例子：

```cpp
#include <vector>
#include <iostream>
#include <algorithm>

#include <tbb/parallel_for.h>
#include <tbb/blocked_range.h>
#include <tbb/cache_aligned_allocator.h>
#include <tbb/enumerable_thread_specific.h>
#include <tbb/parallel_reduce.h>
#include <tbb/combinable.h>
#include <tbb/queuing_mutex.h>
#include <tbb/tick_count.h>
#include <atomic>

int main(int argc, char** argv) {
  size_t N = 100000000;
  int nth = 2;

  std::cout << "Par_count with N: " << N << " and nth: " << nth << std::endl;
  // tbb::task_scheduler_init init{nth};

  // for (int test = 0; test < 10; ++test)
  // {

  // 创建一个包含N个元素的向量，元素初始化为1
  std::vector<uint8_t> vec(N);
  std::generate(vec.begin(), vec.end(), []() { return 1; });

  // 1.串行执行
  long long sum = 0;
  tbb::tick_count t0_s = tbb::tick_count::now();
  for_each(vec.begin(), vec.end(), [&](uint8_t i) { sum += i; });
  tbb::tick_count t1_s = tbb::tick_count::now();
  double t_s = (t1_s - t0_s).seconds();
  std::cout << "Serial \t result: " << sum << " Time: " << t_s << std::endl;

  // 2.并行执行（错误的方式）
  tbb::tick_count t0_p = tbb::tick_count::now();
  long long sum_g = 0;
  parallel_for(tbb::blocked_range<size_t>{0, N},
    [&](const tbb::blocked_range<size_t>& r)
    {
      for (int i = r.begin(); i < r.end(); ++i) sum_g += vec[i];
    });
  tbb::tick_count t1_p = tbb::tick_count::now();
  double t_p = (t1_p - t0_p).seconds();
  std::cout << "Parallel (Mistaken) \t result: " << sum_g;
  if (sum_g != sum) std::cout << " (INCORRECT!) ";
  std::cout << " Time: " << t_p;
  std::cout << "\tSpeedup: " << t_s / t_p << std::endl;

  // 3.并行执行（粗粒度锁）
  using my_mutex_t = tbb::queuing_mutex;
  my_mutex_t my_mutex;
  t0_p = tbb::tick_count::now();
  sum_g = 0;
  parallel_for(tbb::blocked_range<size_t>{0, N},
    [&](const tbb::blocked_range<size_t>& r) {
      my_mutex_t::scoped_lock mylock{ my_mutex };
      for (int i = r.begin(); i < r.end(); ++i) sum_g += vec[i];
    });
  t1_p = tbb::tick_count::now();
  t_p = (t1_p - t0_p).seconds();
  std::cout << "Parallel (Hardy) \t result: " << sum_g;
  if (sum_g != sum) std::cout << " (INCORRECT!) ";
  std::cout << " Time: " << t_p;
  std::cout << "\tSpeedup: " << t_s / t_p << std::endl;

  // 4.并行执行（细粒度锁）
  t0_p = tbb::tick_count::now();
  sum_g = 0;
  parallel_for(tbb::blocked_range<size_t>{0, N},
    [&](const tbb::blocked_range<size_t>& r)
    {
      for (int i = r.begin(); i < r.end(); ++i) {
        my_mutex_t::scoped_lock mylock{ my_mutex };
        sum_g += vec[i];
      }
    });
  t1_p = tbb::tick_count::now();
  t_p = (t1_p - t0_p).seconds();
  if (sum_g != sum) std::cout << "INCORRECT" << std::endl;
  std::cout << "Parallel (Laurel) \t result: " << sum_g;
  if (sum_g != sum) std::cout << " (INCORRECT!) ";
  std::cout << " Time: " << t_p;
  std::cout << "\tSpeedup: " << t_s / t_p << std::endl;

  // 5.并行执行（原子操作）
  t0_p = tbb::tick_count::now();
  std::atomic<long long> sum_a{ 0 };
  parallel_for(tbb::blocked_range<size_t>{0, N},
    [&](const tbb::blocked_range<size_t>& r)
    {
      std::for_each(vec.data() + r.begin(), vec.data() + r.end(),
        [&](const int i) { sum_a += i; });
    });
  t1_p = tbb::tick_count::now();
  t_p = (t1_p - t0_p).seconds();
  std::cout << "Parallel (Nuclear) \t result: " << sum_a;
  if (sum_a != sum) std::cout << "INCORRECT";
  std::cout << " Time: " << t_p;
  std::cout << "\tSpeedup: " << t_s / t_p << std::endl;

  // 6.并行执行（线程本地存储 - 可枚举线程特定）
  t0_p = tbb::tick_count::now();
  using priv_s_t = tbb::enumerable_thread_specific<long long>;
  priv_s_t priv_s{ 0 };
  parallel_for(tbb::blocked_range<size_t>{0, N},
    [&](const tbb::blocked_range<size_t>& r)
    {
      priv_s_t::reference my_s = priv_s.local();
      for (int i = r.begin(); i < r.end(); ++i) my_s += vec[i];
    });
  long long sum_p = 0;
  for (auto& i : priv_s) { sum_p += i; }
  t1_p = tbb::tick_count::now();
  t_p = (t1_p - t0_p).seconds();
  std::cout << "Parallel (Local ETS) \t result: " << sum_p;
  if (sum_p != sum) std::cout << "INCORRECT";
  std::cout << " Time: " << t_p;
  std::cout << "\tSpeedup: " << t_s / t_p << std::endl;

  // 7.并行执行（线程本地存储 - 可组合）
  t0_p = tbb::tick_count::now();
  tbb::combinable<long long> priv_sum{ []() { return 0; } };
 

 parallel_for(tbb::blocked_range<size_t>{0, N},
    [&](const tbb::blocked_range<size_t>& r)
    {
      long long& my_s = priv_sum.local();
      for (int i = r.begin(); i < r.end(); ++i) my_s += vec[i];
    });
  sum_p = priv_sum.combine([](long long a, long long b) -> long long
    { return a + b; });
  t1_p = tbb::tick_count::now();
  t_p = (t1_p - t0_p).seconds();
  std::cout << "Parallel (Local) \t result: " << sum_p;
  if (sum_p != sum) std::cout << "INCORRECT";
  std::cout << " Time: " << t_p;
  std::cout << "\tSpeedup: " << t_s / t_p << std::endl;

  // 8.并行执行（parallel_reduce）
  t0_p = tbb::tick_count::now();
  sum_p = parallel_reduce(tbb::blocked_range<size_t>{0, N},
    /*identity*/ 0,
    [&](const tbb::blocked_range<size_t>& r, const long long& mysum)
    {
      long long res = mysum;
      for (int i = r.begin(); i < r.end(); ++i) res += vec[i];
      return res;
    },
    [&](const long long& a, const long long& b)
    {
      return a + b;
    });
  t1_p = tbb::tick_count::now();
  t_p = (t1_p - t0_p).seconds();
  std::cout << "Parallel (Wise) \t result: " << sum_p;
  if (sum_p != sum) std::cout << " (INCORRECT!) ";
  std::cout << " Time: " << t_p;
  std::cout << "\tSpeedup: " << t_s / t_p << std::endl;

  // }
  return 0;
}
```

这个代码使用Intel TBB库实现了多种并行计算方法，包括串行执行和多种并行执行方式，以及使用不同的锁策略和线程本地存储。

看下在树莓派3B+上的执行结果：

![1.result](/assets/image/posts/2023-09-13-Concurrency/1.result.png)

### 1.串行执行

没啥好说的

### 2.并行执行（错误的方式）

出现问题的原因是，在并行实现中，不同的线程可能同时递增相同的共享容器。换句话说，我们的代码不是线程安全的。更正式地说，我们的并行不安全代码表现出“未定义的行为”。

发生这种情况是因为+=操作不是原子的，它通常由三个汇编级操作组成：将变量从内存加载到寄存器中，执行求和寄存器操作，然后将寄存器存储回内存中。
使用更正式的术语，这种操作称为读取-修改-写入或 RMW 操作。对共享变量进行并发写入正式称为共享可变状态。

### 3.第一个安全并行实现：粗粒度锁定

我们首先解决共享数据结构的并行访问问题。我们需要一种机制，当另一个线程已经在写入同一变量时，防止其他线程读取和写入共享变量。
用更通俗的话说，我们想要一个试衣间，一个人可以进入试衣间，看看衣服合不合身，然后离开试衣间给队列中的下一个人。下图说明了试衣间的一扇关闭的门将其他门排除在外。
在并行编程中，试衣间的门称为互斥锁，当一个人进入试衣间时，他通过关闭和锁定门来获取并保持互斥锁上的锁，当人离开时，他们通过保持门打开来释放锁并解锁。
用更正式的术语来说，互斥体是一个用于在受保护的代码区域的执行中提供互斥的对象。
这个需要通过互斥保护的代码区域通常称为“critical section”。试衣间示例还说明了争用的概念，即一种资源（试衣间）同时被多个人需要的状态，如图所示。
由于试衣间一次只能由一个人占用，因此试衣间的使用是“serialized”的。同样，任何受互斥体保护的内容都会降低程序的性能，首先是因为管理互斥体对象带来了额外的开销，其次也是更重要的是因为它可能引发争用和序列化。

我们希望尽可能减少同步的一个关键原因是避免争用和序列化，这反过来又限制了并行程序的扩展。

![image-fitting room](/assets/image/posts/2023-09-13-Concurrency/2.fitting room.png)

### 4.细粒度锁定

现在我们不仅需要向量数组，还需要一个相同长度的互斥对象数组。这意味着需要更大的内存，而且需要缓存更多的数据，并且会受到错误共享和真实共享的影响。

除了锁固有的开销之外，锁还是另外两个问题的根源：convoying和死锁。让我们首先讨论“convoying”。这个名字来自于所有线程以第一个线程的较低速度一个接一个地传送。
我们需要一个例子来更好地说明这种情况，如图所示。假设我们有线程 1、2、3 和 4 在同一核心上执行相同的代码，其中有一个由自旋互斥锁 A 保护的临界区。
如果这些线程在不同时间持有锁，它们会愉快地运行而不会发生争用（情况 1）。但可能会发生线程 1 在释放锁之前用完其时间片的情况，这会将 A 发送到就绪状态队列的末尾（情况 2）。

![image-oversubscription](/assets/image/posts/2023-09-13-Concurrency/3.oversubscription.png)

线程2,3,4可以获取它们对应的时间片，但是它们没法获取锁，因为 1 仍然是所有者（情况 3）。这意味着 2、3 和 4 现在可以让行或旋转，但无论如何，它们都被困在一档大卡车后面。当再次调度1时，会释放锁A（情况4）。
现在，2、3、4 都准备争夺锁，只有一个成功，其他都在等待。这种情况会反复出现，特别是如果现在线程 2、3 和 4 需要多个时间片来运行其受保护的临界区。
此外，线程 2、3 和 4 现在无意中进行了协调，所有线程都在代码的同一区域中运行，这导致互斥体争用的可能性更高！
请注意，当核心超额订阅时（如本示例中四个线程竞争在单个核心上运行），护航尤其严重，这也强化了我们避免超额订阅的建议。

由锁引起的另一个众所周知的问题是“死锁”。下图A显示了一个噩梦般的情况，即使有可用资源（没有汽车可以使用的空线），也没有人能够取得进展。
这在现实生活中是僵局，但是请把这个形象从你的脑海中移走（如果可以的话！），然后回到我们的并行编程的虚拟世界。
如果我们有一组 N 个线程，它们持有一个锁，并且还在等待获取该组中任何其他线程已经持有的锁，那么我们的 N 个线程就会死锁。
下图B给出了只有两个线程的示例：线程 1 持有互斥锁 A 的锁，并正在等待获取互斥锁 B 的锁，但线程 2 已经持有互斥锁 B 的锁，并等待获取互斥锁 B 的锁。获取互斥体 A 上的锁。
显然，任何线索都不会前进，永远注定要陷入致命的拥抱！如果线程已经持有一个互斥体，我们可以通过不需要获取不同的互斥体来避免这种不幸的情况。或者至少，让所有线程始终以相同的顺序获取锁。

![image-deadlock](/assets/image/posts/2023-09-13-Concurrency/4.deadlock.png)

如果已经持有锁的线程调用也获取不同锁的函数，我们可能会无意中引发死锁。
如果我们不知道函数的作用，建议避免在持有锁时调用该函数（通常建议不要在持有锁时调用其他人的代码）。
或者，我们应该仔细检查后续函数调用链是否会导致死锁。当然，我们还可以尽可能避免锁！

实施中，它们应该帮助我们相信锁带来的问题往往比它们解决的问题还要多，而且它们并不是获得高并行性能的最佳替代方案。
仅当争用概率较低并且执行关键部分的时间最短时，锁才是可以接受的选择。在这些情况下，基本的 spin_lock 或speculative_spin_lock 可以产生一些加速。
但在任何其他情况下，基于锁的算法的可扩展性都会受到严重损害，最好的建议是跳出框框思考并设计一种完全不需要互斥体的新实现。
但是，我们能否在不依赖多个互斥对象的情况下获得细粒度的同步，从而避免相应的开销和潜在的问题呢？

### 5.第三种安全并行实现：原子

幸运的是，在许多情况下，我们可以采用一种更便宜的机制来摆脱互斥体和锁。我们可以使用原子变量来执行原子操作。增量操作不是原子操作，而是可分为三个较小的操作（加载、增量和存储）。但是，如果我们声明一个原子变量并执行以下操作：

```cpp
#include <atomic>
std::atomic<long long> sum_a{ 0 };
//...
sum_a += i;
```

原子变量的增量是一个原子操作。这意味着访问 sum_a值的任何其他线程都将“看到”该操作，就好像增量是在单个步骤中完成的一样（不是三个较小的操作，而是单个操作）。
也就是说，任何其他“目光敏锐”的线程都会观察到操作是否完成，但它永远不会观察到半完成的增量。

原子操作不会受到传送或死锁的影响，并且比互斥替代方案更快。但是，并非所有操作都可以原子执行，并且那些可以原子执行的操作并不适用于所有数据类型。更准确地说，当 T 是整型、枚举或指针数据类型时，atomic 支持原子操作。下图列出了此类原子类型的变量 x 支持的原子操作。

![image-atomic_support_x](/assets/image/posts/2023-09-13-Concurrency/5.atomic_support_x.png)

通过这五个操作，可以实现大量的派生操作。例如，x++、x--、x+=... 和 x-=... 都派生自 x.fetch_and_add()。

关于std::atomic，相对于tbb::atomic，如果在“弱有序”架构上（如ARM或PowerPC；相反，Intel CPU具有强有序内存模型）开发无锁算法和数据结构，可以获得一些额外的性能。对于我们在这里的目的，只需说fetch_and_store、fetch_and_add和compare_and_swap默认遵循顺序一致性（C++术语中的memory_order_seq_cst），这可以防止一些乱序执行，因此会额外消耗一点时间。为了考虑到这一点，TBB还提供了释放和获取语义：默认情况下，在原子读取中采用获取语义（...=x）；在原子写入中默认采用释放语义（x=...）。所需的语义也可以使用模板参数来指定，例如，x.fetch_and_add仅强制释放内存顺序。在C++11中，还允许使用其他更轻松的内存顺序（memory_order_relaxed和memory_order_consume），在特定情况和架构下，这些顺序可以允许读取和写入的顺序更加灵活，从而挤出一些额外的性能。

如果我们希望为了最终的性能而更接近底层，即使知道会增加额外的编码和调试负担，那么C++11的低级特性就在那里等着我们，而且我们可以将它们与TBB提供的高级抽象结合使用。

在这五个基本操作中，比较与交换（CAS）可以被视为所有原子读-修改-写（RMW）操作的起源，这是因为所有原子RMW操作都可以在CAS操作之上实现。
请注意，如果您需要保护一个小的临界区并且已经确定要尽量避免锁定，那么让我们稍微深入了解一下CaS操作的细节。假设我们的代码需要将共享整数变量v原子地乘以3（不要问为什么！我们有我们的原因！）。尽管我们知道乘法不包含在原子操作中，但我们的目标是实现无锁解决方案，这就是CaS发挥作用的地方。首先，将v声明为原子变量：

```cpp
tbb::atomic<uint_32_t> v;
```
现在我们可以调用v.compare_and_swap(new_v, old_v)，它原子地执行以下操作，即仅当v等于old_v时，我们才能使用新值更新v。在任何情况下，我们都会返回ov（在“==”比较中使用的共享v）。现在，实现我们的“乘以3”原子乘法的关键是编写所谓的CaS循环：
```cpp
uint_32_t fetch_and_triple(tbb::atomic<uint_32_t>& v)
{
    uint_32_t old_v = v.load();
    do
    {
        if (v.compare_and_swap(old_v * 3, old_v) == old_v)
            return old_v;
    } while (true);
}
```
我们的新fetch_and_triple是线程安全的（可以由多个线程同时安全调用），即使在调用时传递相同的共享原子变量。这个函数基本上是一个do-while循环，在这个循环中，我们首先对共享变量进行快照（这对于后来比较其他线程是否已经修改了它至关重要）。然后，在原子方式下，如果没有其他线程更改v（v==old_v），我们就会更新它（v=old_v*3），并返回v。由于在这种情况下v==old_v（再次强调：没有其他线程更改了v），我们离开do-while循环，并成功更新了我们的共享v。

然而，在获取快照后，其他线程可能会更新v。在这种情况下，v!=old_v，这意味着（i）我们不会更新v，（ii）我们会继续留在do-while循环中，希望下次幸运女士会对我们微笑（当没有其他贪婪的线程敢于在我们获取快照和成功更新v之间的间隙触碰我们的v时）。下图说明了v是如何由线程1或线程2更新的。有可能其中一个线程必须重试一次或多次，但在设计良好的场景中，这不应该是一个大问题。
这种策略的两个缺点是（i）它的性能扩展很差，（ii）它可能会受到“aBa问题”的影响。关于第一个问题，考虑p个线程竞争同一个原子的情况，只有一个线程成功，其他线程都要重试，然后另一个线程成功，再有p-2个线程重试，然后p-3个线程重试，依此类推，导致二次复杂度的工作量。这个问题可以通过采用“指数回退”策略来改善，该策略将连续重试的速率减小以减少争用。另一方面，aBa问题发生在获取快照和成功更新v之间的间隙时间内，不同的线程将v从值A更改为值B，然后再更改回值A。我们的CaS循环可以在不注意介入线程的情况下成功，这可能会有问题。如果需要在开发中使用CaS循环，请仔细检查并理解这个问题及其后果。

![image-cas](/assets/image/posts/2023-09-13-Concurrency/6.cas.png)

好了，回到我们的例子中。

在这个实现中，我们摆脱了互斥对象和锁，结果是，我们获得了直观锁策略下的向量的并行增量，但在互斥管理和互斥存储方面成本较低。
然而，从性能角度来看，前面的实现仍然过于慢：

```yaml
Serial: 0.533142, Parallel(Nuclear): 4.89581, Speed-up: 0.108897
```
除了原子递增开销之外，虚假共享和真实共享是我们尚未解决的问题。虚假共享可以通过利用对齐分配器和填充技术来解决。虚假共享是一个经常阻碍并行性能的问题，我们后面再讨论这个。
假设我们已经解决了虚假共享问题，那么真实共享呢？两个不同的线程最终会增加相同的sum_a，这将在不同缓存之间来回传递。我们需要更好的方法来解决这个问题。

### 6.并行执行（线程本地存储 - enumerable_thread_specific）

避免共享的常见解决方案是将其私有化。在并行编程方面也不例外。

TBB的一个重要方面是我们不知道在任何给定时间有多少个线程正在使用。即使我们在32核系统上运行，并使用parallel_for进行32次迭代，我们也不能假设会有32个活动线程。这是使我们的代码可组合的关键因素，这意味着它将在并行程序内部调用时工作，或者如果它调用一个并行运行的库。因此，即使在我们的parallel_for示例中有32次迭代，我们也不知道需要多少个线程本地数据的副本。TBB的线程本地存储模板类是为了提供一种抽象方式，以请求TBB为我们分配、操作和组合正确数量的副本，而无需担心有多少个副本。这使我们可以创建可扩展、可组合和可移植的应用程序。

TBB提供了两个线程本地存储的模板类。它们都提供了对每个线程的本地元素的访问，并在需要时（延迟）创建元素。它们在其预期使用模型上有所不同：
- 可枚举线程特定（enumerable_thread_specific）类提供了一个行为类似STL容器的线程本地存储，每个线程有一个元素。容器允许使用通常的STL迭代习惯对元素进行迭代。任何线程都可以迭代所有的本地副本，查看其他线程的本地数据。
- 可组合（combinable）类提供了线程本地存储，用于保存每个线程的子计算结果，稍后将这些结果减少为单个结果。每个线程只能看到它自己的本地数据，或者在调用combine后，可以看到组合后的数据。

让我们首先看看如何使用enumerable_thread_specific类实现。

首先声明了一个类型为long long的enumerable_thread_specific对象priv_s。然后，在parallel_for中，不确定数量的线程将处理迭代空间的各个块，对于每个块，parallel_for的主体（在我们的示例中是lambda）将被执行。负责处理特定块的线程调用priv_s_t::reference my_s = priv_s.local();，它的工作原理如下。如果这个线程第一次调用local()成员函数，将为该线程创建一个新的私有向量。相反，如果不是第一次，那么该向量已经被创建，我们只需要重用它。在两种情况下，都会返回对私有向量的引用，并将其分配给my_s，然后在parallel_for中使用它来更新给定块的求和。这样，处理不同块的线程将为第一个块创建私有vector，然后在后续块中重用它。相当巧妙，不是吗？

在parallel_for结束时，我们会得到不确定数量的私有vector，需要将它们组合起来计算最终的和，累积所有部分结果。但是，如果我们甚至不知道vector的数量，如何进行这种减少呢？幸运的是，enumerable_thread_specific不仅为T类型的元素提供线程本地存储，还可以像STL容器一样从头到尾进行迭代。在其中变量i（类型为priv_s_t::const_iterator）顺序遍历不同的私有vector上累积所有条目计数。

由于减少操作是频繁的操作，enumerable_thread_specific还提供了两个额外的成员函数来执行减少：combine_each()和combine()。

成员函数combine_each()的原型如下：
```cpp
template<typename Func>
void combine_each(Func f)
```
通常，combine_each()成员函数对enumerate_thread_specific对象中的每个元素调用一元函数。这个combine函数通常以void(T)或void(const T&)的形式减少私有副本到全局变量。

另一个成员函数combine()返回类型为T的值，原型如下：
```cpp
template<typename Func>
T combine(Func f)
```
如果要缩减的私有副本的数量很大或者缩减操作是计算密集的。 
我们还将解决这个问题，但首先我们想介绍实现线程本地存储的第二种选择。

### 7.并行执行（线程本地存储 - 可组合）

一个combinable<T>对象为每个线程提供了一个本地实例，类型为T，用于在并行计算期间保存线程本地值。与先前描述的ETS类不同，combinable对象不能使用priv_h那样进行迭代。然而，由于combinable类的唯一目的是实现本地数据存储的减少，因此提供了combine_each()和combine()成员函数。

在这种情况下，priv_sum是一个combinable对象，构造函数提供了一个lambda函数，该函数将在每次调用priv_sum.local()时调用。成员函数combine_each()和combine()与我们看到的ETS类的等效成员函数基本相同。请注意，这种减少仍然是按顺序进行的，因此仅适用于要减少的对象数量较小和/或减少两个对象的时间较短的情况。

###   8.并行执行（parallel_reduce）

TBB已经提供了一个高级并行算法，可以轻松实现并行减少。那么，如果我们想要实现并行减少，为什么不依赖于这个parallel_reduce模板呢？

parallel_reduce的第一个参数只是迭代的范围，它将自动分割成块并分配给线程。第一个lambda负责部分和的私有和本地计算。最后，第二个lambda实现了减少操作。

我们明确地识别了三种不同的行为集。不安全、细粒度锁定和原子解决方案在四个核心上比顺序执行要慢得多（这里的“慢得多”意味着慢一个数量级以上！）。正如我们所说，由于锁定和伪共享/真共享导致频繁的同步是一个真正的问题。细粒度解决方案是最糟糕的，因为我们的vector和互斥向量都存在伪共享和真共享。作为其自己类别的一个代表，粗粒度解决方案仅比顺序解决方案略差一点。请记住，这只是一个“并行化然后序列化”的版本，在这个版本中，粗粒度锁定强制线程依次进入关键部分。粗粒度版本性能略有下降，实际上是测量并行化和互斥管理的开销，但现在我们不再受伪共享或真共享的困扰。最后，私有化+减少解决方案（TLS和parallel_reduction）位居前列。它们的扩展性相当不错，甚至超过线性，因为parallel_reduction由于树状减少略微较慢，在这个问题中不值得。核心数量较少，并且减少所需的时间微不足道。对于这个小问题，使用TLS类实现的顺序减少已经足够好了。

### 总结

TBB库提供了不同类型的互斥锁和原子变量，以帮助我们在需要安全访问共享数据时同步线程。该库还提供了线程局部存储（TLS）类（如ETS和combinable）和算法（如parallel_reduction），帮助我们避免需要同步的情况。对于这个示例，我们从一个不正确的实现开始，然后迭代了不同的同步替代方案，如粗粒度锁定、细粒度锁定和原子操作，最终得出了一些根本不使用锁的替代实现。在这个过程中，我们停在一些显著的地方，介绍了允许我们刻画互斥锁的属性，TBB库中可用的不同类型的互斥锁以及通常在依赖互斥锁来实现算法时出现的常见问题。现在，在这段旅程的尽头，本章的主要信息显而易见：除非性能不是你的目标，否则不要使用锁！