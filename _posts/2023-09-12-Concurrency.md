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

  // 串行执行
  long long sum = 0;
  tbb::tick_count t0_s = tbb::tick_count::now();
  for_each(vec.begin(), vec.end(), [&](uint8_t i) { sum += i; });
  tbb::tick_count t1_s = tbb::tick_count::now();
  double t_s = (t1_s - t0_s).seconds();
  std::cout << "Serial \t result: " << sum << " Time: " << t_s << std::endl;

  // 并行执行（错误的方式）
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

  // 并行执行（粗粒度锁）
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

  // 并行执行（细粒度锁）
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

  // 并行执行（原子操作）
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

  // 并行执行（线程本地存储 - 可枚举线程特定）
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

  // 并行执行（线程本地存储 - 可组合）
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

  // 并行执行（归约）
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

![1.result]({{ site.url }}/assets/image/posts/2023-09-13-Concurrency/1.result.png)