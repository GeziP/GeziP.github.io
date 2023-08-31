# string_view

## 什么是string_view

转自Google的Abseil项目： [Tip of the Week #1](https://link.zhihu.com/?target=https%3A//abseil.io/tips/1)，和其他一些博客内容。

当你创建一个将（常量）字符串作为参数的函数时，你有四个选择，你可能知道两个，但不知道另外两个：

```cpp
void TakesCharStar(const char* s);             // C convention
void TakesString(const string& s);             // Old Standard C++ convention
void TakesStringView(absl::string_view s);     // Abseil C++ convention
void TakesStringView(std::string_view s);      // C++17 C++ convention
```

当调用者已经有已提供的格式的字符串时，前两者方法最有效，但是当需要进行转换（如从const char *到string 或 string到char *）时发生什么呢？

调用者需要将字符串转换为const char *时，需要用（高效但不方便）c_str()函数：

```cpp
void AlreadyHasString(const string& s) {
  TakesCharStar(s.c_str());               // explicit conversion
}
```

调用者需要将const char *转换为字符串时，不需要做任何其他操作（这是好消息）；但是将创建临时字符串（方便但效率低），并复制该字符串的内容（这是坏消息）。

```cpp
void AlreadyHasCharStar(const char* s) {
  TakesString(s); // compiler will make a copy
}
```

## String有什么缺点

> 本节内容主要摘自博客[【现代C++】性能控的工具箱之string_view](https://link.zhihu.com/?target=https%3A//segmentfault.com/a/1190000018387368)。

在数据传递中减少拷贝是提高性能的最常用办法。在C中指针是完成这一目的的标准数据结构，而在C++中引入了安全性更高的引用类型。所以在C++中若传递的数据仅仅可读，const string&成了C++天然的方式。但这并非完美，从实践上来看，它至少有以下几方面问题：

- 字符串字面值、字符数组、字符串指针的传递依然要数据拷贝

这三类低级数据类型与string类型不同，传入时编译器要做隐式转换，即需要拷贝这些数据生成string临时对象。const  string&指向的实际上是这个临时对象。通常字符串字面值较小，性能损失可以忽略不计；但字符串指针和字符数组某些情况下可能会比较大（比如读取文件的内容），此时会引起频繁的内存分配和数据拷贝，影响程序性能。

- substr O(n)复杂度

substr是个常用的函数，好在std::string提供了这个函数，美中不足的时每次都要返回一个新生成的子串，很容易引起性能热点。实际上我们本意不是要改变原字符串，为什么不在原字符串基础上返回呢？

## 怎么办

在**C++17**中引入了`string_view`，能很好的解决以上两个问题。

> std::string_view是C++  17标准中新加入的类，正如其名，它提供一个字符串的视图，即可以通过这个类以各种方法“观测”字符串，但不允许修改字符串。由于它只读的特性，它并不真正持有这个字符串的拷贝，而是与相对应的字符串共享这一空间。即——构造时不发生字符串的复制。同时，你也可以自由的移动这个视图，移动视图并不会移动原定的字符串。

- 通过调用 string_view 构造器可将字符串转换为 string_view 对象。string 可隐式转换为 string_view。
- string_view 是只读的轻量对象，它对所指向的字符串没有所有权。
- string_view通常用于函数参数类型，可用来取代 const char* 和 const string&。string_view 代替 const string&，可以避免不必要的内存分配。
- string_view的成员函数即对外接口与 string 相类似，但只包含读取字符串内容的部分。
  string_view::substr()的返回值类型是string_view，不产生新的字符串，不会进行内存分配。string::substr()的返回值类型是string，产生新的字符串，会进行内存分配。
- string_view字面量的后缀是 sv。（string字面量的后缀是 s）

```cpp
#include <string_view>
#include <iostream>
 
int main()
{
    using namespace std::literals;
 
    std::string_view s1 = "abc\0\0def";
    std::string_view s2 = "abc\0\0def"sv;
    std::cout << "s1: " << s1.size() << " \"" << s1 << "\"\n";
    std::cout << "s2: " << s2.size() << " \"" << s2 << "\"\n";
}
```

输出：

```cpp
s1: 3 "abc"
s2: 8 "abc^@^@def"
```

以上例子能很好看清二者的语义区别，`\0`对于字符串而言，有其特殊的意义，即表示字符串的结束，字符串视图根本不care，它关心实际的字符个数。

Google首选通过string*view接受这样的字符串参数。这是C++17的“pre-adopted”类型，*在C++17的构建中，您应该使用std::string_view，在任何不依赖C++17的代码中，您应该使用absl::string_view（Abseil是Google开源的C++库）。

string_view类的实例可以看作是现有字符串缓冲区的“视图”。具体来说，string_view仅由一个指针和一个长度组成，用于标记不是string _view拥有且不能被该视图修改的字符串数据部分。所以，复制string_view是一项浅层的操作：不复制任何字符串数据。

string_view有来自const char * 和 const  string&的隐式转换构造函数，并且由于string_view不拷贝，因此进行浅拷贝不产生O(n)内存损失。在传递cosnt  string&的情况下，构造函数在O(1)时间进行。在传递const  char*的情况下，构造函数会自动调用strlen()（或者你可以使用具有两个参数的string_view构造函数）。

```cpp
void AlreadyHasString(const string& s) {
  TakesStringView(s); // no explicit conversion; convenient!
}

void AlreadyHasCharStar(const char* s) {
  TakesStringView(s); // no copy; efficient!
}
```

因为string_view不拥有数据，所以string_view所指的任何字符串必须具有已知的生命周期，并且必须比string_view本身生命周期更长。这意味着使用string_view进行存储通常是有问题的：你需要一些证据证明基础数据的生命周期将超过string_view。

如果你的API仅需在一次调用中引用字符串数据，而无需修改数据，则接受string_view就足够了。如果以后需要引用数据或需要修改数据，则可以使用string（my_string_view）显式转换为C ++字符串对象。

将string_view添加到现有代码库中并非总是正确的答案：更改参数以通过string_view传递可能效率不高，如果这些参数随后传递给需要字符串或以NUL终止的const char *的函数。最好从实用程序代码开始向上使用string_view，或者在启动新项目时保持完全一致。

### 成员函数

下面列举其成员函数：忽略了函数的返回值，若函数有重载，括号内用`...`填充。这样可以对其有个整体轮廓。

```cpp
// 迭代器
begin()
end()
cbegin()
cend()
rbegin()
rend()
crbegin()
crend()
 
// 容量
size()
length()
max_size()
empty()
 
// 元素访问
operator[](size_type pos)
at(size_type pos)
front()
back()
data()
 
// 修改器
remove_prefix(size_type n)
remove_suffix(size_type n)
swap(basic_string_view& s)
 
copy(charT* s, size_type n, size_type pos = 0)
string_view substr(size_type pos = 0, size_type n = npos)
compare(...)
starts_with(...)
ends_with(...)
find(...)
rfind(...)
find_first_of(...)
find_last_of(...) 
find_first_not_of(...)
find_last_not_of(...)
```

从函数列表来看，几乎跟`string`的只读函数一致，使用`string_view`的方式跟`string`基本一致。有几个地方需要特别说明：

1. `string_view`的`substr`函数的时间复杂度是O(1)，解决了**背景**部分的第二个问题。
2. **修改器**中的三个函数仅会修改`string_view`的数据指向，不会修改指向的数据。

除此之外，函数名基本是自解释的。

### 示例

**Haskell**中有一个常用函数`lines`，会将字符串切割成行存储在容器里。下面我们用**C++**来实现

**string-版本**

```cpp
#include <string>
#include <iostream>
#include <vector>
#include <algorithm>
#include <sstream>

void lines(std::vector<std::string> &lines, const std::string &str) {
    auto sep{"\n"};
    size_t start{str.find_first_not_of(sep)};
    size_t end{};

    while (start != std::string::npos) {
        end = str.find_first_of(sep, start + 1);
        if (end == std::string::npos)
            end = str.length();

        lines.push_back(str.substr(start, end - start));
        start = str.find_first_not_of(sep, end + 1);
    }
}
```

上面我们用`const std::string &`类型接收待分割的字符串，若我们传入指向较大内存的字符指针时，会影响程序效率。

使用`std::string_view`可以避免这种情况：
**string_view-版本**

```cpp
#include <string>
#include <iostream>
#include <vector>
#include <algorithm>
#include <sstream>
#include <string_view>

void lines(std::vector<std::string> &lines, std::string_view str) {
    auto sep{"\n"};
    size_t start{str.find_first_not_of(sep)};
    size_t end{};

    while (start != std::string_view::npos) {
        end = str.find_first_of(sep, start + 1);
        if (end == std::string_view::npos)
            end = str.length();

        lines.push_back(std::string{str.substr(start, end - start)});
        start = str.find_first_not_of(sep, end + 1);
    }
}
```

上面的例子仅仅是把`string`类型修改成了`string_view`就获得了性能上的提升。一般情况下，将程序中的`string`换成`string_view`的过程是比较直观的，这得益于两者的成员函数的相似性。但并不是所有的“翻译”过程都是这样的，比如：

```cpp
void lines(std::vector<std::string> &lines, const std::string& str) {
    std::stringstream ss(str);
    std::string line;

    while (std::getline(ss, line, '\n')) {
        lines.push_back(line);
    }
}
```

这个版本使用`stringstream`实现`lines`函数。由于`stringstream`没有相应的构造函数接收`string_view`类型参数，所以没法采用直接替换的方式，所以翻译过程要复杂点。

## 使用陷阱

世上没有免费的午餐。不恰当的使用`string_view`也会带来一系列的问题。

1. `string_view`范围内的字符可能不包含`\0`

如

```cpp
#include <iostream>
#include <string_view>

int main() {
    std::string_view str{"abc", 1};

    std::cout << str.data() << std::endl;

    return 0;
}
```

本来是要打印`a`，但输出了`abc`。这是因为字符串相关的函数都有一条兼容C的约定：`\0`代表字符串的结尾。上面的程序打印从开始到字符串结束的所有字符，虽然`str`包含的有效字符是`a`，但`cout`认`\0`。好在这块内存空间有合法的字符串结尾符，如果`str`指向的是一个没有`\0`的字符数组，程序很有可能会出现内存问题，所以我们在将`string_view`类型的数据传入接收字符串的函数时要非常小心。

2.从`[const] char*`构造`string_view`对象时间复杂度`O(n)`
 这是因为获取字符串的长度需要从头开始遍历。如果对`[const] char*`类型仅仅是一些`O(1)`的操作，相比直接使用`[const] char*`，转为`string_view`是没有性能优势的。只不过是相比`const string&`，`string_view`少了拷贝的损耗。实际上我们完全可以用`[const] char*`接收所有的字符串，但这个类型太底层了，不便使用。在某些情况下，我们转为`string_view`可能仅仅是想用其中的一些函数，比如`substr`。

3.`string_view`指向的内容的生命周期可能比其本身短
`string_view`并不拥有其指向内容的所有权，用Rust的术语来说，它仅仅是暂时`borrow`（借用）了它。如果拥有者提前释放了，你还在使用这些内容，那会出现内存问题，这跟`悬挂指针`(dangling pointer)或悬挂引用（dangling references）很像。Rust专门有套机制在编译时分析变量的生命期，保证`borrow`的资源在使用期间不会被释放，但C++没有这样的检查，需要人工保证。下面列出一些典型的问题情况：

```cpp
std::string_view sv = std::string{"hello world"}; 
string_view foo() {
    std::string s{"hello world"};
    return string_view{s};
}
auto id(std::string_view sv) { return sv; }

int main() {
    std::string s = "hello";
    auto sv = id(s + " world"); 
}
```



## 其他事项

- 与其他字符串类型不同，你应该按值传递string_view，就像int或double一样，因为string_view是一个很小的值。
- string_view不一定是NUL终止的。因此，编写以下内容并不安全：

```cpp
printf("%s\n", sv.data()); // DON’T DO THIS
```

但是，下面是好的代码：

```cpp
printf("%.*s\n", static_cast<int>(sv.size()), sv.data());
```

- 你可以输出string_view，就像输出字符串或const char*一样：

```cpp
std::cout << "Took '" << s << "'";
```

- 大多数情况下，你可以将接受const string&或NUL终止的const char*的现有例程安全的转换为string_view。在执行此操作时遇到的唯一危险是，如果已获取函数的地址，则将导致编译中断，因为生成的函数指针类型将有所不同。

## 总结

`string_view`解决了一些痛点，但同时也引入了指针和引用的一些老问题。C++标准并没有对这个类型做太多的约束，这引来的问题是我们可以像平常的变量一样以多种方式使用它，如，可以传参，可以作为函数返回值，可以做普遍变量，甚至我们可以放到容器里。随着使用场景的复杂，人工是很难保证指向的内容的生命周期足够长。所以，推荐的使用方式：**仅仅作为函数参数**，因为如果该参数仅仅在函数体内使用而不传递出去，这样使用是安全的。

