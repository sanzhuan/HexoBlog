title: "new/delete和malloc/free"
date: 2015-08-28 19:32:48
tags: c++
---
前记：帮老师检查项目内存泄露的时候，看到了之前的学生用new出来的内存，结果用free()来释放。老师教导我们new/delete和malloc/free可以混用，但是不能把new和free混用，也不能把malloc和delete混用。But why?在讲为啥之前我们先来瞅瞅new/delete 和 malloc/free的区别
<!-- more --> 
## new/delete 和 malloc/free的区别
new/delete 和 malloc/free 的一个显著区别是new/delete比malloc/free多一步操作，new会调用operator new在已经分配的内存中构造出来对象，即初始化内存。delete会调用operator delete析构分配的对象。以Fred *p = new Fred() 来说，其实包括了下面两步:
1.调用void* operator new(size_t nbytes)函数分配sizeof(Fred) 字节大小的内存。void* operator new(size_t nbytes)和malloc(size_t nbytes)差不多，但是两个是不等价的，不保证他们使用的是相同的内存区域。
2.在第一步分配的内存上调用构造函数构造出Fred类型的对象。第一步返回的指针作为参数传给构造函数，这一步包裹在 try … catch块中来处理异常，保证出现异常时析构掉已经分配的sizeof(Fred) 字节大小的内存。
从功能上来说，new Fred()相当于以下代码:
``` c++
// Original code: Fred* p = new Fred();
Fred* p;
void* tmp = operator new(sizeof(Fred));
try {
  new(tmp) Fred();  // Placement new
  p = (Fred*)tmp;   // 只有上面的new(tmp) Fred()成功执行，p才被赋值。
}
catch (...) {
  operator delete(tmp);  // 析构分配的内存
  throw;                 // 重新抛出异常
}
```
“Placement new” operator new 调用 Fred的构造函数。在构造函数中Fred::Fred()，p相当于this指针。

同样地delete p包含两步，调用析构函数，释放内存。delete p相当于:
``` c++
// Original code: delete p;
if (p) {    // or "if (p != nullptr)"
  p->~Fred();
  operator delete(p);
}
```
'p->~Fred()'调用Fred的析构函数。

operator delete(p) 调用void operator delete(void* p)函数。void operator delete(void* p)和 free(void* p)差不多，但是两者不能交换使用，因为不保证它们使用的是同一个heap。

new/delete 和 malloc/free区别还有很多，详细如下：
new/delete
+ 内存从'Free Store'分配
+ 返回某种类型的指针，是类型安全的
+ new从不会返回NULL ，而是有可能抛出异常。
+ 不用指定大小 (编译器会根据我们指定的类型来计算大小)。
+ new []可以处理数组。
+ 由于复制构造函数，重新分配来获取更多空间不是很方便。
+ 根据具体实现来决定是否调用malloc/free。
+ 可以用新的内存分配器来处理少内存的情况。(set_new_handler)
+ new/delete是操作符，可以重载。 
+ 使用构造函数/析构函数来初始化/销毁分配的对象。

malloc/free
+ 内存从'Heap'中分配
+ 返回void*，不是类型安全的
+ 失败时返回NULL
+ 必须以字节大小指定分配的内存大小。
+ 分配数组需要手动计算需要的空间。
+ 可以重新分配大块内存relloc。
+ 不会调用new/delete
+ 在内存少的情况下无法指定内存分配器来处理。
+ malloc/free是库函数，不能重载。

通过以上可以看出，new分配的内存来自于'Free Store'而malloc分配的内存来自于'Heap'。'Free Store'和'Heap'是否是同一内存区域是跟实现相关的，所以不能把new和free混用。
 
总结：new/delete相比于malloc/free有很多好处，比如: 调用Constructors/destructors, 类型安全, 可重载。是不是说我们应该使用new/delete而不是malloc/free呢？其实不是这样，我们不应该手动管理内存，应该使用智能指针来管理，如：
``` c++
auto p1 = make_shared<widget>(); // C++11, type is shared_ptr
auto p2 = make_unique<widget>(); // new in C++14, type is unique_ptr
```

参考：
http://stackoverflow.com/questions/4061514/c-object-created-with-new-destroyed-with-free-how-bad-is-this
http://stackoverflow.com/questions/240212/what-is-the-difference-between-new-delete-and-malloc-free
https://isocpp.org/wiki/faq/freestore-mgmt#mixing-malloc-and-delete
https://isocpp.org/wiki/faq/cpp14-library#make-unique

