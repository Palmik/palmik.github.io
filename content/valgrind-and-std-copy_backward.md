+++
title = "Valgrind and std::copy_backward"
draft = true
date = 2011-11-09

[taxonomies]
categories = []
tags = ["C++"]
+++

Today I came across a weird valgrind report. It's consistently reproducible on my machine using this sample code:

```cpp
#include <algorithm>
#include <iostream>

using std::cout;
using std::endl;

int main() {
  const std::size_t size = 10;
  int data[size + 1] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

  // The data should now contain 1, 2, 2, 3, 4, 5, 6, 7, 8, 9
  std::copy_backward(data + 2, data + size, data + size + 1);

  for (std::size_t i = 0; i < size + 1; ++i) {
      cout << data[i] << ' ';
  }

  cout << endl;
}
```

It does output what we would expect (see [cppreference](http://en.cppreference.com/w/cpp/algorithm/copy_backward)). Of great importance is the footnote of Section 25.2.2. of the C++ Standard, which says:

> copy_backward should be used instead of copy when last is in the range [result - (last - first), result).

Which is indeed our case.

But yet valgrind finds this code problematic. Here is what it has to say about that program’s execution:

```
Source and destination overlap in memcpy(0x7ff00039c, 0x7ff000398, 32)
   at 0x4C28B46: memcpy (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
   by 0x400B18: int* std::__copy_move_backward<false, true, std::random_access_iterator_tag>::__copy_move_b<int>(int const*, int const*, int*) (stl_algobase.h:561)
   by 0x400AB8: int* std::__copy_move_backward_a<false, int*, int*>(int*, int*, int*) (stl_algobase.h:581)
   by 0x400A58: int* std::__copy_move_backward_a2<false, int*, int*>(int*, int*, int*) (stl_algobase.h:590)

   by 0x4009E8: int* std::copy_backward<int*, int*>(int*, int*, int*) (stl_algobase.h:625)
   by 0x400902: main (Test.cpp:13)
```

Interesting, so what's the problem here?

If `memcpy` is indeed called, then valgrind is right to point this out, since using `memcpy` in this case leads to undefined behaviour according to the C Standard, Section 7.21.2.1:

> If copying takes place between objects that overlap, the behavior is undefined.

Instead of `memcpy` you should use `memmove` in such cases since it avoids the undefined behaviour by making temporal copy first.

But is `memcpy` really called? Well, not according to the libstdc++ documentation. I did not yet confirm it firsthand, but on a first glance it seems that the documentation is correct. This is line 561 of `stl_algobase.h` reffered by valgrind:

```cpp
__builtin_memmove(__result - _Num, __first, sizeof(_Tp) * _Num);
```

In the end it turned out to be related to these [[1](https://bugs.kde.org/show_bug.cgi?id=278502), [2](https://bugs.kde.org/show_bug.cgi?id=275284)] bug reports. 
