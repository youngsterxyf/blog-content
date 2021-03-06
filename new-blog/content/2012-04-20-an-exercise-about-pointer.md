Title: 关于指针的一道笔试题
Date: 2012-04-20
Author: youngsterxyf
Slug: an-exercise-about-pointer
Tags: 笔试, C/C++

同学找实习，遇到这样一道笔试题:

	:::c
    int *a[2][3];
    sizeof(a) = ?
    sizeof(*a) = ?
    sizeof(**a) = ?
    sizeof(***a) = ? 

这题还是有点小意思的。遇到这种题，脑子一定要清楚，注意分析。

对于int \*a[2][3]应该这么理解：

**a是个数组，有两个元素；元素也是数组，其有3个元素，每个元素是指向int类型的指针。**

指针的长度固定为4个字节，C语言的int类型也是4个字节。

这样一分析，这题就简单了。

	:::text
    sizeof(a)意思是求a数组的长度，数组的长度=数组元素的个数*元素的长度，所以sizeof(a) = 2 * 3 * 4 = 24个字节
    sizeof(*a)中的*a是指a的第一个元素，所以sizeof(*a)就是求a数组的第一个元素的长度，即sizeof(*a) = 3 * 4 = 12个字节
    sizeof(**a)则是求a的第一个元素(也是数组)的第一个元素(是指向int类型的指针)的长度，所以sizeof(**a) = 4个字节(即为一个指针的长度)
    sizeof(***a)，因为**a是一个指针类型，那么***a即为*(**a)，那为指针指向的值，这里为int类型，所以sizeof(***a) = 4个字节(int类型的长度)
