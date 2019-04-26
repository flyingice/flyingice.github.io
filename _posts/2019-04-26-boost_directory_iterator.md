---
title: 用boost中directory_iterator遍历目录碰到的问题
tag: c++ boost posix
---

最近在公司写项目碰到一个“奇怪”的问题，局部需求很简单：c++实现一个函数，找到目标路径下最近被更新过的文件（假设路径下只有普通文件，无子目录无符号链接）。以下所有代码片段都只保留问题相关的核心逻辑，省略了错误和异常处理。

因为公司的gcc版本还不支持c++17，所以用boost的filesystem库来做。开始没想太多，写了如下实现：

```c++
#include <algorithm>
#include <boost/filesystem.hpp>
using namespace boost::filesystem;
using namespace std;

// version 1
path findLastWriteFile(const path& iAbsPath) {
  auto it = max_element(directory_iterator(iAbsPath),
                        directory_iterator(),
                        [](const directory_entry& l, const directory_entry& r) {
                          return last_write_time(l.path()) < last_write_time(r.path());
                        });

  return it == directory_iterator() ? path() : it->path();
}
```

gtest单元测试一下，发现version 1的结果和预期不符。对着上面代码愣了半天，好像没有什么问题啊。于是试着把遍历目录的过程展开成for循环方便debug：

```c++
#include <boost/filesystem.hpp>
using namespace boost::filesystem;

// version 2
path findLastWriteFile(const path& iAbsPath) {
  path aPath;
  for (auto it = directory_iterator(iAbsPath); it != directory_iterator(); it++) {
    if (aPath.empty() || last_write_time(aPath) < last_write_time(it->path())) {
      aPath = it->path();
    }
  }

  return aPath;
}
```

 非常意外，version 2的结果居然是对的。再继续对比下两个版本的代码实现，实在发现不了两者语义上有什么区别，看上去两种方法实现的功能是完全相同的。那么，version 1的问题到底在哪里？



-----华丽的分割线-----



看下boost源码里面[directory_iterator](https://github.com/boostorg/filesystem/blob/feature/readdir_r/src/operations.cpp)在unix平台的实现，自增操作的调用栈为directory_iterator_increment -> dir_itr_increment -> readdir_r_simulator。以下摘取函数readdir_r_simulator的关键片段：

```c++
struct dirent * p;
*result = 0;
if ((p = ::readdir(dirp))== 0)
  return errno;
std::strcpy(entry->d_name, p->d_name);
*result = entry;
```

其中dirp是DIR*类型，在调用opendir的时候产生。每次迭代器自增时，调用readdir读取当前条目，直到readdir返回空指针时循环结束，最后closedir关闭用opendir打开的DIR结构释放资源。对于readdir系统调用, APUE里没有细讲，但是[POSIX](http://pubs.opengroup.org/onlinepubs/9699919799/)规范里有这么一段：

*The application shall not modify the structure to which the return value of readdir() points, nor any storage areas pointed to by pointers within the structure. The returned pointer, and pointers within the structure, might be invalidated or the structure or the storage areas might be overwritten by a subsequent call to readdir() on the same directory stream.*

这里的关键在于DIR是一个目录流，它和FILE文件流有些类似。每次调用readdir后之所以前一次返回的指针内容会失效，是因为实现中使用了静态内存，这片内存在opendir的时候就已经分配好了。换句话说，readdir是会在多次调用中共享状态的，这点上和[strtok](https://en.cppreference.com/w/cpp/string/byte/strtok)一样。

再来看下macOS 10.14上对C++标准库里[max_element](https://en.cppreference.com/w/cpp/algorithm/max_element)的实现：

```c++
template <class _ForwardIterator, class _Compare>
inline _LIBCPP_INLINE_VISIBILITY _LIBCPP_CONSTEXPR_AFTER_CXX11
_ForwardIterator
max_element(_ForwardIterator __first, _ForwardIterator __last, _Compare __comp)
{
    if (__first != __last)
    {
        _ForwardIterator __i = __first;
        while (++__i != __last)
            if (__comp(*__first, *__i))
                __first = __i;
    }
    return __first;
}
```

max_element移动指针i来遍历所有元素，过程中通过指针first来记住当前已访问的所有元素中的最大值的**位置**。回到[operations.hpp](https://github.com/boostorg/filesystem/blob/feature/readdir_r/include/boost/filesystem/operations.hpp)看下directory_iterator的定义，该类的定义中仅包含一个指向dir_itr_imp类型的shared_ptr，dir_itr_imp真正实现了所有的迭代器操作。由于directory_iterator类没有显式地定义复制构造函数，所以编译器自动生成了一个默认版本来复制shared_ptr成员。这样使得在max_element里面用first给i赋值的时候，实际只进行了指针的浅拷贝，并没有真正复制指针拥有的资源。在上面的while循环中，无论if条件是真是假，指针first始终和指针i指向相同对象，只不过该对象每次迭代之后会因为readdir覆盖之前的静态内存内容而变化。下面的例子可以让问题更加明显：

```c++
#include <boost/filesystem.hpp>
#include <iomanip>
#include <iostream>
using namespace boost::filesystem;
using namespace std;

int main()
{
   auto it = directory_iterator(".");
   auto it_cpy = it;
   it++;
   cout << boolalpha << (it != it_cpy) << endl;		// output: false
   return 0;
}
```

运行结果屏幕打印false，原因刚刚解释过了。

为了从直观上理解这个问题，可以认为流对象是不能回退的。也就是说，已经读取过的内容都会失效。上面max_element的实现中需要通过保存流对象过去的状态来返回结果，导致程序和期望的语义不一致。

除了version 2的方法，还有两个思路能避开上述问题。一种方法是对version 1稍加修改，利用额外内存在遍历过程中保存结果（version 3）；另一种是保留version 1的代码不动，自己实现一个直接比较元素值的max_element，不过这么做还不如直接写成version 2形式，意义不大。

```c++
#include <algorithm>
#include <boost/filesystem.hpp>
#include <vector>
using namespace boost::filesystem;
using namespace std;

// version 3
path findLastWriteFile(const path& iAbsPath) {
  vector<directory_entry> vec((directory_iterator(iAbsPath)), directory_iterator());

  auto it = max_element(vec.begin(), vec.end(), [](const directory_entry& l, const directory_entry& r) {
    return last_write_time(l.path()) < last_write_time(r.path());
  });

  return it == vec.end() ? path() : it->path();
};
```

version 3的结构相比version 2程序意图更明显，缺点是会占用更多的临时内存。

附代码测试环境：macOS 10.14.4, LLVM version 10.0.1 (clang-1001.0.46.4), boost 1.69.0