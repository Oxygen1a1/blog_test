---
title: c++学习模板与泛型
date: 2023-04-12 16:10:22
categories: 编程语言学习
password: c++stl
encrypt: false
---
# STL

## Stl的呈现方式

STL全程`Standard Template Library`,也就是叫做标准模板库;

标准库是以header files的形式呈现的;

而且C++引用的时候,使用#include <xxx>而不是#include<xxx.h>的形式;

## Stl的六大部件

标准库的60-70%都是STL;

分为

![image-20230412180258332](E:\notes\Notes\C++进阶学习\assets\image-20230412180258332.png)

- **(Container)容器**
- **(Allocator)分配器**
- **(Algorithms)算法**
- **(Iterators)迭代器**
- **(Adapters)适配器**
- **(Functions)仿函数**

事实上,STL的设计方式是GP,也就是模板编程;数据是放在容器中的,但是算法却是在另一个文件;

比如下面的代码

```c++
#include <algorithm>
#include <functional>
#include <iostream>
#include <vector>

using namespace std;

int main() {
  int ia[6] = {27, 210, 12, 47, 109, 83};
  vector<int, allocator<int>> vi(ia, ia + 6);
  cout << count_if(vi.begin(), vi.end(), not1(bind2nd(less<int>(), 40)));
  return 0;
}

```

`vector<int,allocator<int>>`,第二个参数就是分配器,因为容器是基于分配器的;

allocator是标准库的分配器,同样的,allocator也是一个**模板**;==分配器和容器的类型==应该是一样的;

`count_if`是算法中的,给定条件,返回条件;

用到了这些STL的六大部件;

![image-20230412181213401](E:\notes\Notes\C++进阶学习\assets\image-20230412181213401.png)

### stl的前闭后开区间

即标准库规定,所有的`容器`的起始规定

**开始就是第一开始的位置,但是结束的迭代器指向位于最后一个元素的下一个;**

![image-20230412181932108](E:\notes\Notes\C++进阶学习\assets\image-20230412181932108.png)

因此在循环的时候,经常会用

`for(;ite!=c.end();ite++){}`,当然这种for比较老,现在`C++11`直接使用`for(auto item:container)`来进行容器的遍历;

# STL的容器使用

STL的容器一般分为

- 连续容器

### Array(cpp 11+)

Array(C++11才有的,包装的[],也就是数组)

![image-20230414104420811](E:\notes\Notes\C++进阶学习\assets\image-20230414104420811.png)

下面是测试C++11的Array容器

```c++
array<long,ASIZE> c;

int compareLongs(const void* a, const void* b)
{
	return (*(long*)a - *(long*)b);
}

int main() {
	clock_t timeStart = clock();
	for (long i = 0; i < ASIZE; ++i) {
		c[i] = rand();
	}
	//放50W元素耗时
	cout << "milli-seconds : " << (clock() - timeStart) << endl;	//
	
	//array的使用
	cout << "array.size():" << c.size() << endl;
	cout << "array.front:" << c.front()<<endl;
	cout << "array.back():" << c.back() << endl;
	cout << "array.data():" << c.data() << endl;//获取指针

	long target = 0;
	cout << "target (0~" << RAND_MAX << "):" << endl;
	cin >> target;

	timeStart = clock();
	//快速查找 c语言标准库
	::qsort(c.data(), c.size(),sizeof(long), (_CoreCrtNonSecureSearchSortCompareFunction)compareLongs);
	
	long* pItem = (long*)::bsearch(&target, c.data(), c.size(), sizeof(long), (_CoreCrtNonSecureSearchSortCompareFunction)compareLongs);
	cout << "qsort()+bsearch(), milli-seconds : " << (clock() - timeStart) << endl;
	if (pItem != NULL)
		cout << "found, " << *pItem << endl;
	else
		cout << "not found! " << endl;


	return 0;
}

```

涉及的Array的操作有

```c++
	//array的使用
	cout << "array.size():" << c.size() << endl;
	cout << "array.front:" << c.front()<<endl;
	cout << "array.back():" << c.back() << endl;
	cout << "array.data():" << c.data() << endl;//获取指针
```

**后面则是使用cstdlib的函数进行一些查找**

**比如`qsort`和`bsearch`;**

### Vector

Vector:也是数组,但是可以自动增长;

![image-20230414104426529](E:\notes\Notes\C++进阶学习\assets\image-20230414104426529.png)

他和Array相比,好处之一就是Vector可以动态增长,也就说,==可能在某个区间进行Vector.push()可能耗时和Array差不多,但是一旦超过,那么会涉及到内存换地,这样就会非常慢;==

事实上,`Vector`有两个大小,一个是`.size()`,另一个是`.capacity()`,其中size就是大小,表示元素的个数。而`capacity`则是表示当前Vector的最大容量,一旦超过这个容量,就会增持;

而vector的增长通常是`按照2倍数`增长,知道堆内存用完;

下面是Vector的一些测试

```c++
namespace test_vector {

	auto test_vector(const long& vaule)->void {
		vector<string> c;
		char buf[10];

		clock_t timeStart = clock();


		for (long i = 0; i < vaule; i++) {

			sprintf_s(buf, "%d", rand());
			c.push_back(buf);
		}

		cout << "vector.max_size():" << c.max_size() << endl;
		cout << "vector.size()" << c.size() << endl;
		cout << "vector.front()" << c.front() << endl;
		cout << "vector.back()" << c.back() << endl;
		cout << "vector.data()" << c.data() << endl;
		cout << "vector.capacity" << c.capacity() << endl;

		string s;
		cout << "input find:";
		cin >> s;

		auto pItem = ::find(c.begin(), c.end(), s);
		
		if (pItem != c.end()) {
			cout << "find!" << endl;

		}
		else {
			cout << "not find" << endl;
		}

		sort(c.begin(), c.end());


	}


}
```

基本上就是`.size`,.`data`	`.push_back`

还有`front`和`back`;

而sort是stl的函数,本质上是一个模板函数;find也是;

==值得一提的是==,函数模板其实不用加`<typename>`,他似乎会自动推导,`find和sort`就是这样;

### Deque

Deque:双向队列

![image-20230414104437278](E:\notes\Notes\C++进阶学习\assets\image-20230414104437278.png)

他也是两边可以扩充,而且是连续的;事实上Deque是一块块指针,每一个指针都是指向一个`连续的`

所以Deque是一个`分段连续的`双向队列;

![image-20230415091706927](E:\notes\Notes\C++进阶学习\assets\image-20230415091706927.png)

那他是怎么使用`[]`和`++,--`这些运算符的呢?

毫无疑问,运算符重载;

```c++
#include <deque>
namespace test_deque {

	auto test_deque(const long& vaule) -> void {

		deque<string> d;
		char buf[10];
		for (int i = 0; i < vaule; ++i) {
			_itoa_s(rand(), buf, 10);
			d.push_back(string(buf));

		}

		cout <<"d.back():" << d.back() << endl;
		cout << "d.front():" << d.front() << endl;
		cout << "d.size():" << d.size() << endl;
		//cout << "d.push_front" << d.push_front(string("555"));
		string target;
		cout << "input target:";
		cin >> target;

		//stl
		auto pItem = find(d.begin(), d.end(), target);
		if (pItem != d.end()) {
			cout << "find" << endl;
		}
		else {

			cout << "not find" << endl;
		}

		sort(d.begin(), d.end());
	}


}
```

和vector有区别的就是,它可以`push_front`来添加前面,因为他是双向的队列;

### List

List:双向链表

![image-20230414104510293](E:\notes\Notes\C++进阶学习\assets\image-20230414104510293.png)

这个标准库有一个`sort`,而链表自己也有一个`sort`,只要容器自己有一个自己的算法函数,一定用容器自己的是最快的;

而且因为是双向的,可以查最前面和最后面;

```c++
#include <list>
namespace test_list {

	auto test_list(const long& value) -> void {

		list<string> l;
		for (int i = 0; i < value; ++i) {

			char buf[10];
			_itoa_s(rand(), buf, 10);
			l.push_front(string(buf));

		}

		l.push_front(string("1111"));

		cout << "front():" << l.front() << endl;
		cout << "back():" << l.back() << endl;
		cout << "size()" << l.size() << endl;
		
		l.sort();

		string target;
		cout << "input targetl";
		cin >> target;
		auto pItem=find(l.begin(), l.end(), target);
		if (pItem != l.end()) {

			cout << "find" << endl;

		}
		else {
			cout << "not find" << endl;
		}
	}

}
```

值得注意的是,List有自己的`sort`函数;

### Forward-List

Forward-list:单向链表

![image-20230414104956227](E:\notes\Notes\C++进阶学习\assets\image-20230414104956227.png)

## 连续式容器的总结

STL的容器应该按需使用,

所以从连续式容器可以看出,所有的数值是连续的,都是容易找,但是空间利用率低;

而LIst一般是查找慢,但是空间利用率高;

除了array,List,插入元素一旦超过当前的`capacity`,就会换地方,这个过程会超级慢;

而array和List不会出现这种情况;

**但是无论怎么样,连续式容器的查找都很慢,而关联式容器的查找非常快速;**

- 关联式容器

### Map/MultiMap

==一般是需要用大量查找的时候需要==

![image-20230414105157150](E:\notes\Notes\C++进阶学习\assets\image-20230414105157150.png)

标准库中,Map/MultiMap都是红黑树写的;

Set和Map的区别就是,Map有`Key`和`Vaule`

同样的,还有MultiSet/Map,这是因为Key和Map的`Key`是无法重复的;

但是MultiSet/Map可以多个;

下面是测试,==值得一提的是==Map可以直接使用`[]`下标志来进行`insert`,而Map则不行,只能用pair<>()的形式进行插入;

另外,Map使用自带的`find`查找到的是一个pair<>()的形式;因此想要查值需要`x.second`来进行获取键值对的值;

```c++
#include <map>

namespace test_map {

	auto test_map(const long& value)->void {

		map<int, string> m;
		for (int i = 0; i < value; i++) {

			char buf[10];
			_itoa_s(rand(), buf, 10);
			//m.insert(pair<int,string>(i, string(buf)));
			m[i] = string(buf);
		}

		cout << "size:" << m.size() << endl;
		
		//find
		int target;
		cout << "input target:";
		cin >> target;

		auto item=m.find(target);
		cout << "find:" << item->second << endl;
	}

}
```

- 无序容器

### 无序容器Unordered_set

事实上,他就是一种关联式容器;

但是是一个Hash-Table;也就是不是用红黑树做的;

和用红黑树做的区别就是,

![image-20230415110122593](E:\notes\Notes\C++进阶学习\assets\image-20230415110122593.png)

通过一个hash函数,把键通过计算放到`哈希桶`中,然后下次查找的时候,只需要Find找这个哈希桶;

当然可能会出现`哈希碰撞`,因此哈希表实现的关联容器使用了指针的结构;每个桶事实上都是一个指针,指向一个链表;

下面是用`unordered_set`进行的测试

```c++
#include <unordered_set>
namespace test_unordered_set {
	
	auto test_unordered_set(const int& value) {

		unordered_set<string> uset;

		for (int i = 0; i < value; ++i) {

			char buf[10];
			_itoa_s(rand(), buf, 10);

			uset.insert(string(buf));
		}

		clock_t runtime;
		string keys;
		cout << "input find keys:";
		cin >> keys;
		runtime = clock();
		auto item=uset.find(keys);
		auto cost_time = runtime - clock();

		cout << "find:" << *item <<"cost time:"<<cost_time<<"ms"<<endl;
	}

}
```

上面的示例是用`unordered_set`来进行的；因此find查找直接可以查找的只要`keys`;

猜测如果set<typename>是别的结构体,可能要提供给find一个函数或者是仿函数来进行比较;

# 分配器

分配器就是STL支持容器的内存的;

一般来说,使用容器会使用默认的分配器;

```c++
_EXPORT_STD template <class _Ty, class _Alloc = allocator<_Ty>>
class deque {}

```

比如可以看到,这个是默认的分配器`allocator<>`,分配器就是一个class;

事实上,有很多分配器

```c++
	//欲使用 std::allocator 以外的 allocator, 得自行 #include <ext\...> 
#ifdef __GNUC__		
#include <ext\array_allocator.h>
#include <ext\mt_allocator.h>
#include <ext\debug_allocator.h>
#include <ext\pool_allocator.h>
#include <ext\bitmap_allocator.h>
#include <ext\malloc_allocator.h>
#include <ext\new_allocator.h>  
#endif
```

只不过都在ext扩展目录里面;并不是定义在STL里面;而是扩展的;

# OOP VS GP

在早期版本的STL中,主要是用了`GP`编程,好处之一是能够更好的看源代码;

所谓OOP就是`数据`和`操作`放在class中;

而`GP`是把data和methods是分开来的;表现在标准库中就是

在`vector`和`deque`都没有`sort`这个函数,而是在另一个文件中

使用`Alorithms`使用`泛型`来进行计算;

其实采用`GP`的思想,其实可以使团队分开开发;只要把`接口`定义好即可;

其实有点像`C#`的泛型接口了;只不过C#更简单

比如这样写

```c++
template <class T,class Compare>
inline const T& min(const T& a,const T& b,Compare com){
    
    return com(a,b)?b:a;
    
}
```

用一个函数传进来,这个函数就很想`接口`的概念;

 # STL基础

## 运算符重载

STL基本都是泛型和`GP`编写的;因此操作符重载和模板泛型是非常重要的;

| 表达式  |     作为成员函数     |  作为非成员函数  |                             示例                             |
| :-----: | :------------------: | :--------------: | :----------------------------------------------------------: |
|   @a    |  (a).operator@ ( )   |  operator@ (a)   | ![std::cin](https://zh.cppreference.com/w/cpp/io/cin) 调用 [std::cin](https://zh.cppreference.com/w/cpp/io/cin).operator!() |
|   a@b   |  (a).operator@ (b)   | operator@ (a, b) | [std::cout](https://zh.cppreference.com/w/cpp/io/cout) << 42 调用 [std::cout](https://zh.cppreference.com/w/cpp/io/cout).operator<<(42) |
|   a=b   |  (a).operator= (b)   |   不能是非成员   | 给定 [std::string](https://zh.cppreference.com/w/cpp/string/basic_string) s; ， s = "abc"; 调用 s.operator=("abc") |
| a(b...) | (a).operator()(b...) |   不能是非成员   | 给定 [std::random_device](https://zh.cppreference.com/w/cpp/numeric/random/random_device) r; ， auto n = r(); 调用 r.operator()() |
|  a[b]   |  (a).operator[](b)   |   不能是非成员   | 给定 [std::map](https://zh.cppreference.com/w/cpp/container/map)<int, int> m; ， m[1] = 2; 调用 m.operator[](1) |
|   a->   |  (a).operator-> ( )  |   不能是非成员   | 给定 auto p = [std::make_unique](https://zh.cppreference.com/w/cpp/memory/unique_ptr/make_unique)<S>(); p->bar() 调用 p.operator->() |
|   a@    |  (a).operator@ (0)   | operator@ (a, 0) | 给定 [std::vector](https://zh.cppreference.com/w/cpp/container/vector)<int>::iterator i = v.begin(); ， i++ 调用 i.operator++(0) |

事实上,有四个重载符号不能重载,`::`,`.`

# 标准库的六大部件

## 分配器(Allocators)

分配器的效率非常重要,一般不会见到他;

事实上,所有的分配(应用层)基本上都是最后调用malloc;

在C++中,我们学到了`auto operator new(size_t size)->void{}`

事实上,在STL默认使用的分配器是allocator<_Ty>;

分配器最重要的函数一个是`allocate`,一个是`deallocate`

![image-20230415145335606](E:\notes\Notes\C++进阶学习\assets\image-20230415145335606.png)

上述代码是VC6自带的,因此,在VC6中,allocator只是最终会调用operatpr new和operator delete

所以如果想要用分配器调用的话,需要使用

`int* p=allocator<int>().allocate(512,0);`来进行分配内存;`allocator<int>()`的作用是搞一个临时对象;

# List

在标准库中,List是一个环形的双向链表

![image-20230415152628513](E:\notes\Notes\C++进阶学习\assets\image-20230415152628513.png)

还有iterator,begin()就指向第一个,而end()就是最后一个,那个数据是没有东西的;

## List的Iterator

本质上,iterator就是一个`smart_prt`,用来模仿指针;

 首先,所有的容器都有一个`typedef`或者是`using`,因为stl库中的iterator都是这种;

比如`list<xx>::iterator`,本质上是通过class内的`typedef`来实现的;

`typedef __list_iterator<T,T&,T*> iterator`,用来代表这个类的iterator;

iterator需要模仿指针;也就是对iterator进行对象重载;`->`,`*`,`++`,`--`

而一个iterator需要传三个模板参数,即`T,T&,T*`，另外,在iterator

![image-20230415154405098](E:\notes\Notes\C++进阶学习\assets\image-20230415154405098.png)



必须要用这几个typedef,这些后面会解释,但是在iterator的涉及必须满足;

另外;此外,在操作符重载的函数中,使用重载了的运算符,并不会触发这些运算符的重载,而是原生的;

### Iterator的traits

traites是特征的意思,通过`traits`用于萃取出某个东西的特征;

每个东西基本上都有traits,毫无疑问,`iterator`是`Algorithms`是`containers`的桥梁,因此算法在`GP`的时候需要使用萃取来进行判断属性

事实上,iterator_traits也是个类,用来萃取来判断,萃取出这个iterator的属性,比如他的移动属性,两个iterator的距离等等

![image-20230415161127356](E:\notes\Notes\C++进阶学习\assets\image-20230415161127356.png)

所以Iterator必须提供5种供算法询问`Traits`,也就是

![image-20230415161654390](E:\notes\Notes\C++进阶学习\assets\image-20230415161654390.png)

因此算法只需这样问

![image-20230415161751977](E:\notes\Notes\C++进阶学习\assets\image-20230415161751977.png)

但是事实上stl用了`traits`来询问，而不是通过typedef来进行;

但是由于有可能算法可能受到的是`原始指针`,因此加了一个中介层,然后来一个`偏特化`

 ![image-20230415162608825](E:\notes\Notes\C++进阶学习\assets\image-20230415162608825.png)

偏特化来分离出指针的形式;直接`typedef T vaule_type`来回答询问;







