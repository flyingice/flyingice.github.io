---
title: std::remove_if的陷阱
tag: c++ STL
---

想必C++中std::remove_if不会真正擦除元素的坑大家都知道，这里要说的是remove操作完成后容器中新的逻辑结束位置之后元素值的问题。

其实这个问题[文档](https://en.cppreference.com/w/cpp/algorithm/remove)有所说明，但是一直没有仔细读过：

*Iterators pointing to an element between the new logical end and the physical end of the range are still dereferenceable, but the elements themselves have unspecified values (as per MoveAssignable post-condition).*

即新的逻辑结束位置之后的元素是**未定义**的，remove_if函数的后置条件很明确，但是我还是掉进了这个坑。之前一直错误地认为remove_if是通过swap()来交换元素来实现的，所以调用结束后符合断言的元素值不变，只是被移动到了容器末尾。

比如有一个字符串数组，现在需要将所有不以数字开头的字符串（假设每个字符串非空）全部移动到数组尾部，那么可以这么做：

```c++
int tail = 0, sz = v.size();
for(int i = 0; i < sz; i++) {
  if(isdigit(v[i][0])) swap(v[i], v[tail++]);
}
```

以上所有操作都是in-place的，循环结束之后tail指向新的逻辑结束位置，tail及其之后的所有元素都不以数字开头。后来我“灵光一闪”，上面的操作不就是标准库中remove_if做的事情吗，那么干脆不要自己实现，于是有了以下代码：

```c++
remove_if(v.begin(), v.end(), [](const string& word) {
  return !isdigit(word[0]);
});
```

看上去没什么问题，但是之后进一步处理被移动到数组末尾的元素的时候产生了意料之外的结果，debug半天也没看出个所以然来，于是去翻STL里remove_if的实现（macOS 10.14.6）：

```c++
template <class _ForwardIterator, class _Predicate>
_LIBCPP_CONSTEXPR_AFTER_CXX17 _ForwardIterator
remove_if(_ForwardIterator __first, _ForwardIterator __last, _Predicate __pred)
{
    __first = _VSTD::find_if<_ForwardIterator, typename add_lvalue_reference<_Predicate>::type>
                           (__first, __last, __pred);
    if (__first != __last)
    {
        _ForwardIterator __i = __first;
        while (++__i != __last)
        {
            if (!__pred(*__i))
            {
                *__first = _VSTD::move(*__i);
                ++__first;
            }
        }
    }
    return __first;
}
```

这下终于明白了，注意std::move()操作这行， remove_if并没有调用swap来交换元素位置，而是直接赋值覆盖原值。

总而言之，遇事还是不要太想当然。



