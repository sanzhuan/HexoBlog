title: "写时复制（Copy on Write）技术"
date: 2015-07-06 21:10:13
tags: [c++,Copy-on-Write]
---

##前言
参加阿里实习生招聘的时候问道了写时复制技术怎么实现的，当时猜是用引用计数实现的，但是具体怎么实现的，还真不清楚。
<!-- more --> 
##写时复制
Scott Meyers在《More Effective C++》中举了个例子，不知你是否还记得？在你还在上学的时候，你的父母要你不要看电视，而去复习功课，于是你把自己关在房间里，做出一副正在复习功课的样子，其实你在干着别的诸如给班上的某位女生写情书之类的事，而一旦你的父母出来在你房间要检查你是否在复习时，你才真正捡起课本看书。这就是“拖延战术”，直到你非要做的时候才去做。

当然，这种事情在现实生活中时往往会出事，但其在编程世界中摇身一变，就成为了最有用的技术，正如C++中的可以随处声明变量的特点一样，Scott Meyers推荐我们，在真正需要一个存储空间时才去声明变量（分配内存），这样会得到程序在运行时最小的内存花销。执行到那才会去做分配内存这种比较耗时的工作，这会给我们的程序在运行时有比较好的性能。必竟，20%的程序运行了80%的时间。

当然，拖延战术还并不只是这样一种类型，这种技术被我们广泛地应用着，特别是在操作系统当中，当一个程序运行结束时，操作系统并不会急着把其清除出内存，原因是有可能程序还会马上再运行一次（从磁盘把程序装入到内存是个很慢的过程），而只有当内存不够用了，才会把这些还驻留内存的程序清出。

写时才拷贝（Copy-On-Write）技术，就是编程界“懒惰行为”——拖延战术的产物。举个例子，比如我们有个程序要写文件，不断地根据网络传来的数据写，如果每一次fwrite或是fprintf都要进行一个磁盘的I/O操作的话，都简直就是性能上巨大的损失，因此通常的做法是，每次写文件操作都写在特定大小的一块内存中（磁盘缓存），只有当我们关闭文件时，才写到磁盘上（这就是为什么如果文件不关闭，所写的东西会丢失的原因）。更有甚者是文件关闭时都不写磁盘，而一直等到关机或是内存不够时才写磁盘，Unix就是这样一个系统，如果非正常退出，那么数据就会丢失，文件就会损坏。

呵呵，为了性能我们需要冒这样大的风险，还好我们的程序是不会忙得忘了还有一块数据需要写到磁盘上的，所以这种做法，还是很有必要的。

STL类std::string的Copy-On-Write

在我们经常使用的STL标准模板库中的string类，也是一个具有写时才拷贝技术的类。C++曾在性能问题上被广泛地质疑和指责过，为了提高性能，STL中的许多类都采用了Copy-On-Write技术。这种偷懒的行为的确使使用STL的程序有着比较高要性能。

这里，我想从C++类或是设计模式的角度为各位揭开Copy-On-Write技术在string中实现的面纱，以供各位在用C++进行类库设计时做一点参考。

在讲述这项技术之前，我想简单地说明一下string类内存分配的概念。通过常，string类中必有一个私有成员，其是一个char*，用户记录从堆上分配内存的地址，其在构造时分配内存，在析构时释放内存。因为是从堆上分配内存，所以string类在维护这块内存上是格外小心的，string类在返回这块内存地址时，只返回const char*，也就是只读的，如果你要写，你只能通过string提供的方法进行数据的改写。
特性

由表及里，由感性到理性，我们先来看一看string类的Copy-On-Write的表面特征。让我们写下下面的一段程序：
``` C++
    
#include <stdio.h>
#include <string>
using namespace std;
  
main()
{
    string str1 = "hello world";
    string str2 = str1;
 
    printf ("Sharing the memory:/n");
    printf ("/tstr1's address: %x/n", str1.c_str() );
    printf ("/tstr2's address: %x/n", str2.c_str() );
 
    str1[1]='q';
    str2[1]='w';
 
    printf ("After Copy-On-Write:/n");
    printf ("/tstr1's address: %x/n", str1.c_str() );
    printf ("/tstr2's address: %x/n", str2.c_str() );
 
    return 0;
}
```
这个程序的意图就是让第二个string通过第一个string构造，然后打印出其存放数据的内存地址，然后分别修改str1和str2的内容，再查一下其存放内存的地址。程序的输出是这样的（我在VC6.0和g++ 2.95都得到了同样的结果）：
``` bash
    
> g++ -o stringTest stringTest.cpp
> ./stringTest
Sharing the memory:
        str1's address: 343be9
        str2's address: 343be9
After Copy-On-Write:
        str1's address: 3407a9
        str2's address: 343be9
```
从结果中我们可以看到，在开始的两个语句后，str1和str2存放数据的地址是一样的，而在修改内容后，str1的地址发生了变化，而str2的地址还是原来的。从这个例子，我们可以看到string类的Copy-On-Write技术。
 深入

在深入这前，通过上述的演示，我们应该知道在string类中，要实现写时才拷贝，需要解决两个问题，一个是内存共享，一个是Copy-On-Wirte，这两个主题会让我们产生许多疑问，还是让我们带着这样几个问题来学习吧：

1、  Copy-On-Write的原理是什么？

2、  string类在什么情况下才共享内存的？

3、  string类在什么情况下触发写时才拷贝（Copy-On-Write）?

4、  Copy-On-Write时，发生了什么？

5、  Copy-On-Write的具体实现是怎么样的？

喔，你说只要看一看STL中stirng的源码你就可以找到答案了。当然，当然，我也是参考了string的父模板类basic_string的源码。但是，如果你感到看STL的源码就好像看机器码，并严重打击你对C++自信心，乃至产生了自己是否懂C++的疑问，如果你有这样的感觉，那么还是继续往下看我的这篇文章吧。

OK，让我们一个问题一个问题地探讨吧，慢慢地所有的技术细节都会浮出水面的。

 
Copy-On-Write的原理是什么？

有一定经验的程序员一定知道，Copy-On-Write一定使用了“引用计数”，是的，必然有一个变量类似于RefCnt。当第一个类构造时，string的构造函数会根据传入的参数从堆上分配内存，当有其它类需要这块内存时，这个计数为自动累加，当有类析构时，这个计数会减一，直到最后一个类析构时，此时的RefCnt为1或是0，此时，程序才会真正的Free这块从堆上分配的内存。

是的，引用计数就是string类中写时才拷贝的原理！

不过，问题又来了，这个RefCnt该存在在哪里呢？如果存放在string类中，那么每个string的实例都有各自的一套，根本不能共有一个RefCnt，如果是声明成全局变量，或是静态成员，那就是所有的string类共享一个了，这也不行，我们需要的是一个“民主和集中”的一个解决方法。这是如何做到的呢？呵呵，人生就是一个糊涂后去探知，知道后和又糊涂的循环过程。别急别急，在后面我会给你一一道来的。

 
string类在什么情况下才共享内存的？

 

这个问题的答案应该是明显的，根据常理和逻辑，如果一个类要用另一个类的数据，那就可以共享被使用类的内存了。这是很合理的，如果你不用我的，那就不用共享，只有你使用我的，才发生共享。

 

使用别的类的数据时，无非有两种情况，1）以别的类构造自己，2）以别的类赋值。第一种情况时会触发拷贝构造函数，第二种情况会触发赋值操作符。这两种情况我们都可以在类中实现其对应的方法。对于第一种情况，只需要在string类的拷贝构造函数中做点处理，让其引用计数累加；同样，对于第二种情况，只需要重载string类的赋值操作符，同样在其中加上一点处理。

唠叨几句：

1）构造和赋值的差别

对于前面那个例程中的这两句：
``` c++
    
string str1 = "hello world";
 
string str2 = str1;
```
不要以为有“=”就是赋值操作，其实，这两条语句等价于：
``` c++
    
string str1 ("hello world");   //调用的是构造函数
 
string str2 (str1);            //调用的是拷贝构造函数
```
如果str2是下面的这样情况：
``` c++
    
string str2;      //调用参数默认为空串的构造函数：string str2(“”);
 
str2 = str1;     //调用str2的赋值操作：str2.operator=(str1);
```
2) 另一种情况
``` c++
    
char tmp[]=”hello world”;
 
string str1 = tmp;
 
string str2 = tmp;
```
这种情况下会触发内存的共享吗？想当然的，应该要共享。可是根据我们前面所说的共享内存的情况，两个string类的声明和初始语句并不符合我前述的两种情况，所以其并不发生内存共享。而且，C++现有特性也无法让我们做到对这种情况进行类的内存共享。
string类在什么情况下触发写时才拷贝（Copy-On-Write）?

哦，什么时候会发现写时才拷贝？很显然，当然是在共享同一块内存的类发生内容改变时，才会发生Copy-On-Write。比如string类的[]、=、+=、+、操作符赋值，还有一些string类中诸如insert、replace、append等成员函数,包括类的析构时。

修改数据才会触发Copy-On-Write，不修改当然就不会改啦。这就是托延战术的真谛，非到要做的时候才去做。
Copy-On-Write时，发生了什么？

我们可能根据那个访问计数来决定是否需要拷贝，参看下面的代码：
``` C++   
If  ( RefCnt>0 ) {
    char* tmp =  (char*) malloc(strlen(_Ptr)+1);
    strcpy(tmp, _Ptr);
    _Ptr = tmp;
}
```
 

上面的代码是一个假想的拷贝方法，如果有别的类在引用（检查引用计数来获知）这块内存，那么就需要把更改类进行“拷贝”这个动作。

我们可以把这个拷的运行封装成一个函数，供那些改变内容的成员函数使用。
Copy-On-Write的具体实现是怎么样的？

最后的这个问题，我们主要解决的是那个“民主集中”的难题。请先看下面的代码：
``` C++
    
string h1 = “hello”;
string h2= h1;
string h3;
h3 = h2;
  
string w1 = “world”;
string w2(“”);
w2=w1;
```
很明显，我们要让h1、h2、h3共享同一块内存，让w1、w2共享同一块内存。因为，在h1、h2、h3中，我们要维护一个引用计数，在w1、w2中我们又要维护一个引用计数。

如何使用一个巧妙的方法产生这两个引用计数呢？我们想到了string类的内存是在堆上动态分配的，既然共享内存的各个类指向的是同一个内存区，我们为什么不在这块区上多分配一点空间来存放这个引用计数呢？这样一来，所有共享一块内存区的类都有同样的一个引用计数，而这个变量的地址既然是在共享区上的，那么所有共享这块内存的类都可以访问到，也就知道这块内存的引用者有多少了。

请看下图：

![o_string](/images/o_string.jpg)

于是，有了这样一个机制，每当我们为string分配内存时，我们总是要多分配一个空间用来存放这个引用计数的值，只要发生拷贝构造可是赋值时，这个内存的值就会加一。而在内容修改时，string类为查看这个引用计数是否为0，如果不为零，表示有人在共享这块内存，那么自己需要先做一份拷贝，然后把引用计数减去一，再把数据拷贝过来。下面的几个程序片段说明了这两个动作：
``` c++
    
//构造函数（分存内存）
    string::string(const char* tmp)
{
    _Len = strlen(tmp);
    _Ptr = new char[_Len+1+1];
    strcpy( _Ptr, tmp );
    _Ptr[_Len+1]=0;  // 设置引用计数  
}
  
//拷贝构造（共享内存）
    string::string(const string& str)
    {
         if (*this != str){
              this->_Ptr = str.c_str();   //共享内存
              this->_Len = str.szie();
              this->_Ptr[_Len+1] ++;  //引用计数加一
         }
}
  
//写时才拷贝Copy-On-Write
char& string::operator[](unsigned int idx)
{
    if (idx > _Len || _Ptr == 0 ) {
         static char nullchar = 0;
return nullchar;
          }
    
_Ptr[_Len+1]--;   //引用计数减一
    char* tmp = new char[_Len+1+1];
    strncpy( tmp, _Ptr, _Len+1);
    _Ptr = tmp;
    _Ptr[_Len+1]=0; // 设置新的共享内存的引用计数
    
    return _Ptr[idx];
}
 
//析构函数的一些处理
~string()
{ 
_Ptr[_Len+1]--;   //引用计数减一
    
         // 引用计数为0时，释放内存 
    if (_Ptr[_Len+1]==0) {
        delete[] _Ptr;
         }
  
}
```
哈哈，整个技术细节完全浮出水面。

 

不过，这和STL中basic_string的实现细节还有一点点差别，在你打开STL的源码时，你会发现其取引用计数是通过这样的访问：_Ptr[-1]，标准库中，把这个引用计数的内存分配在了前面（我给出来的代码是把引用计数分配以了后面，这很不好），分配在前的好处是当string的长度扩展时，只需要在后面扩展其内存，而不需要移动引用计数的内存存放位置，这又节省了一点时间。

STL中的string的内存结构就像我前面画的那个图一样，_Ptr指着是数据区，而RefCnt则在_Ptr-1 或是_Ptr[-1]处。
副作用

是谁说的“有太阳的地方就会有黑暗”？或许我们中的许多人都很迷信标准的东西，认为其是久经考验，不可能出错的。呵呵，千万不要有这种迷信，因为任何设计再好，编码再好的代码在某一特定的情况下都会有Bug，STL同样如此，string类的这个共享内存/写时才拷贝技术也不例外，而且这个Bug或许还会让你的整个程序crash掉！

不信？！那么让我们来看一个测试案例。假设有一个动态链接库（叫myNet.dll或myNet.so）中有这样一个函数返回的是string类：
``` c++    
string GetIPAddress(string hostname)
{
    static string ip;
    ……
    ……
    return ip;
}
```
而你的主程序中动态地载入这个动态链接库，并调用其中的这个函数：
``` c++
    
main()
{
    //载入动态链接库中的函数
    hDll = LoadLibraray(…..);
    pFun =  GetModule(hDll, “GetIPAddress”);
      
    //调用动态链接库中的函数
    string ip = (*pFun)(“host1”);
    ……
    ……
    //释放动态链接库
    FreeLibrary(hDll);
    ……
    cout << ip << endl;
}
```
让我们来看看这段代码，程序以动态方式载入动态链接库中的函数，然后以函数指针的方式调用动态链接库中的函数，并把返回值放在一个string类中，然后释放了这个动态链接库。释放后，输入ip的内容。

根据函数的定义，我们知道函数是“值返回”的，所以，函数返回时，一定会调用拷贝构造函数，又根据string类的内存共享机制，在主程序中变量ip是和函数内部的那个静态string变量共享内存（这块内存区是在动态链接库的地址空间的）。而我们假设在整个主程序中都没有对ip的值进行修改过。那么在当主程序释放了动态链接库后，那个共享的内存区也随之释放。所以，以后对ip的访问，必然做造成内存地址访问非法，造成程序crash。即使你在以后没有使用到ip这个变量，那么在主程序退出时也会发生内存访问异常，因为程序退出时，ip会析构，在析构时就会发生内存访问异常。

内存访问异常，意味着两件事：1）无论你的程序再漂亮，都会因为这个错误变得暗淡无光，你的声誉也会因为这个错误受到损失。2）未来的一段时间，你会被这个系统级错误所煎熬（在C++世界中，找到并排除这种内存错误并不是一件容易的事情）。这是C/C++程序员永远的心头之痛，千里之堤，溃于蚁穴。而如果你不清楚string类的这种特征，在成千上万行代码中找这样一个内存异常，简直就是一场噩梦。

备注：要改正上述的Bug，有很多种方法，这里提供一种仅供参考：   
``string ip = (*pFun)(“host1”).cstr();``
后记

 

文章到这里也应该结束了，这篇文章的主要有以下几个目的：

1）向大家介绍一下写时才拷贝/内存共享这种技术。
2）以STL中的string类为例，向大家介绍了一种设计模式。
3）在C++世界中，无论你的设计怎么精巧，代码怎么稳固，都难以照顾到所有的情况。智能指针更是一个典型的例子，无论你怎么设计，都会有非常严重的BUG。
4）C++是一把双刃剑，只有了解了原理，你才能更好的使用C++。否则，必将引火烧身。如果你在设计和使用类库时有一种“玩C++就像玩火，必须千万小心”的感觉，那么你就入门了，等你能把这股“火”控制的得心应手时，那才是学成了。

更新：在最新的STL中，这个特性已经被去掉了。有一个原因是线程不安全！COW其实还是比较危险的。

##读时复制
C++的std::string的读时也拷贝技术！

嘿嘿，你没有看错，我也没有写错，是读时也拷贝技术。什么?我的错，你之前听说写过时才拷贝，嗯，不错的确有这门技术，英文是Copy On Write，简写就是COW,非常’牛’！那么我们就来看看这个’牛’技术的效果吧。

我们先编写一段程序
``` c++
    
#include <string>
#include <iostream>
#include <sys/time.h>
 
static long getcurrenttick()
{
    long tick ;
    struct timeval time_val;
    gettimeofday(&time_val , NULL);
    tick = time_val.tv_sec * 1000 + time_val.tv_usec / 1000 ;
    return tick;
}
 
 
int main( )
{
    string the_base(1024 * 1024 * 10, 'x');
    long begin =  getcurrenttick();
    for( int i = 0 ;i< 100 ;++i ) {
       string the_copy = the_base ;
    }
    fprintf(stdout,"耗时[%d] \n",getcurrenttick() - begin );
}
```
嗯，一个非常大的字符串，有10M字节的x，并且执行了100此拷贝。编译执行它，非常快，在我的虚拟机甚至不要1个毫秒。

现在我们来对这个string加点料！
``` c++ 
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    long begin =  getcurrenttick();
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
        the_copy[0] = 'y';
    }
    fprintf(stdout,"耗时[%d] \n",getcurrenttick() - begin );
}
```
现在我们再编译并执行这断程序，居然需要4~5秒！哇！非常美妙的写时才拷贝技术，性能和功能的完美统一。

我们再来看看另外一种情况！
``` C++    
string original = "hello";
char & ref = original[0];
string clone = original;
ref = 'y';
```
我们生成了一个string，并保留了它首字符的引用，然后复制这个string，修改string中的首字符。因为写操作只是直接的修改了内存中的指定位置，这个string就根本不能感知到有写发生，如果写时才拷贝是不成熟的，那么我们将同时会修改original和clone两个string。那岂不是灾难性的结果？幸好上述问题不会发生。clone的值肯定是没有被修改的。看来COW就是非常的牛！

以上都证明了我们的COW技术非常牛！

有太阳就有黑暗，这句说是不是有点耳熟？
``` C++    
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    fprintf(stdout,"the_base's first char is [%c]\n",the_base[0] );
    long begin =  getcurrenttick();
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
    }
    fprintf(stdout,"耗时[%d] \n",getcurrenttick() - begin );
}
```
啊，居然也是4~5秒！你可能在想，我只是做了一个读，没有写嘛，这到底是怎么回事？难道还有读时也拷贝的技术！。

不错，为了避免了你通过[]操作符获取string内部指针而直接修改字符串的内容，在你使用了the_base[0]后，这个字符串的写时才拷贝技术就失效了。

C++标准的确就是这样的，C++标准认为，当你通过迭代器或[]获取到string的内部地址的时候，string并不知道你将是要读还是要写。这是它无法确定，为此，当你获取到内部引用后，为了避免不能捕获你的写操作，它在此时废止了写时才拷贝技术！

这样看来我们在使用COW的时候，一定要注意，如果你不需要对string的内部进行修改，那你就千万不要使用通过[]操作符和迭代器去获取字符串的内部地址引用，如果你一定要这么做，那么你就必须要付出代价。当然，string还提供了一些使迭代器和引用失效的方法。比如说push_back，等， 你在使用[]之后再使用迭代器之后，引用就有可能失效了。那么你又回到了COW的世界！比如下面的一个例子
``` c++   
int main( )
{
    struct timeval time_val;
    string the_base(1024 * 1024 * 10, 'x');
    long begin = 0 ;
    fprintf(stdout,"the_base's first char is [%c]\n",the_base[0] );
    the_base.push_back('y');
    begin = getcurrenttick();
    for( int i = 0 ;i< 100 ;++i ) {
        string the_copy = the_base ;
    }
    fprintf(stdout,"耗时[%d] \n",getcurrenttick() - begin );
}
```
一切又恢复了正常！如果对[]返回引用进行了操作又会发生情况呢，有兴趣的朋友可以试试！结果非常令人惊讶。

另外：上述例子是在linux环境下编译的，使用STL是GNU的STL。windows上我用的是vs2003，但是非常明显vs2003一点都不支持COW。

这篇文章出自http://ridiculousfish.com/blog/archives/2009/09/17/i-didnt-order-that-so-why-is-it-on-my-bill-episode-2/ 这里，我使用了它的例子。但是我重新自己组织了内容。

编写这篇文章的同时，我还参考了耗子的《标准C＋＋类string的Copy-On-Write技术》一文 
## I Didn't Order That, So Why Is It On My Bill, Episode 2
This is the second episode of I Didn't Order That, So Why Is It On My Bill: C++ features you never use, but pay for anyways. See episode one here.

For those who like their punchlines first: std::string is designed to allow copy-on-write. But once operator[] is called, that string is ruined for COW forevermore. The reason is that it has to guard against a future write at any time, because you may have stashed away the internal pointer that operator[] returns - a feature that you surely know better than to use, but that costs you nevertheless.

The C++ standard string class is std::string. Here's a string containing 10 megabytes of the letter 'x':
``` C++
#include <string>
using std::string;
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
}
```
Big string. Now let's make a hundred copies, one by one:
``` C++
#include <string>
using std::string;
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
    }
}
```
This runs in .02 seconds, which is very fast. Suspiciously fast! This old iMac can't schlep a gigabyte of data in two hundredths of a second. It's using copy-on-write.

So let's write and measure the copy:
``` C++
#include <string>
using std::string;
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
        the_copy[0] = 'y';
    }
}
```
The program now runs in 2.5 seconds. These scant 100 writes resulted in a 100x slowdown. Not surprising: it's exactly what we would expect from a COW string.

the_copy[0] returns a reference to a char. But maybe we aren't going to write into it - perhaps we only want to read. How can these copy-on-write strings know when the write occurs, so they can copy? The answer is that they cannot - they must be pessimistic and assume you will write, even if you never do. We can trick it: don't assign anything, and still see the performance hit:
``` C++
#include <string>
using std::string;
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
        the_copy[0];
    }
}
```
No writes! Just a read! But same performance as before. This isn't copy on write - it's copy on read!

Things get even worse. operator[] returns a reference: basically a sugary pointer. You can read from it or write to it. Or you can be a jerk and just hang onto it:
``` C++
string original = "hello";
char & ref = original[0];
string clone = original;
ref = 'y';
```
In case that's not clear, we made a string, stashed away a reference to the first character, copied the string, and then wrote to the string's guts. The write is just a normal store to a memory location, so the poor string is caught completely unaware. If the copy-on-write were naive, then we would have modified both the original and the copy.

Oh man, is that evil. But the string's got to be paranoid and guard against it anyways - in fact, the C++ standard explicitly calls out this case as something that has to work! In other words, when you get a reference to a char inside a string, the string is tainted. It can never participate in copy-on-write again, because you just might have stashed away the internal pointer it handed back, and write into the string as a completely unexpected time.

Of course you know better than to hang onto internal pointers. You probably don't need this feature. But you pay for it anyways:
``` C++
#include <string>
using std::string;
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    the_base[0];
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
    }
}
```
This takes the 2.5 seconds. Just one silly read wrecked all of the COW machinery for all the future copies. You're paying for writes that never come!

Could the C++ standard have fixed this? Sure - the C++ standard goes into great detail about when iterators and references into strings are invalidated, and all it would have taken would be to add the copy constructor and operator=() to the list of reference-invalidating functions (section 23.1.5, for those who are following along in the standard). A second option would have been to declare that operator=() invalidates existing references for writing, but not for reading. But the standards committee preferred simplicity, convenience and safety to performance - uncharacteristically, if you'll forgive a smidgen of editorializing.

If you call a reference-invalidating function, you can, in principle, get back in the COW fast lane. Although GNU STL does not take advantage of this in all cases, it does when resizing strings:
``` C++
#include <string>
using std::string;
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    the_base[0];
    the_base.push_back('x');
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
    }
}
```
Allowing the string to modify itself allowed it to know that any previous references were invalidated. We're fast again! It's called a COW, and it's not about to blow it now.

The moral is that you should avoid using operator[], the at() function, iterators, etc. to read from a non-const string - especially a long string - if it may be copied in the future. If you do use these functions, you risk paying for a write that you don't make, and for hanging onto an internal pointer that you released. Unfortunately, there's no good function for just reading from a non-const string. You can do it by adding a crazy cast:
``` C++
#include <string>
using std::string;
int main(void) {
    string the_base(1024 * 1024 * 10, 'x');
    const_cast<const string &>(the_base)[0];
    for (int i = 0; i < 100; i++) {
        string the_copy = the_base;
    }
}
```
This hits the fast path too.

Scene. I hope you enjoyed this post. As usual the point is not to rag on C++, but to explore aspects of its design that, for whatever reason, tilt a tradeoff away from performance.
