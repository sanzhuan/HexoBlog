layout: c++
title: "cin输入类型不匹配时发生了什么"
date: 2015-05-20 20:06:35
tags: c++ 
---

输入类型不匹配是指输入的数据类型与所期望的类型不匹配，如 `int n; cin >> n;` 
但输入的数据为字符串时，这种情况就是输入类型不匹配。那么当出现这种情况时，变量`n`的值有没有改变呢，又该如何检测这种情况呢？

<!-- more --> 

首先
变量n的值并没有改变。
不匹配的字符串仍留在输入缓冲区中。
`cin`类中的一个错误标志被设置了
`cin`方法转换成的bool类型返回值为`false`
 
检测方法：
如下面的示例程序，首先用 `good()` 方法检测输入是否出错，然后分别检测出错类型，先检测是否遇到`EOF`，使用 `eof()` 方法，
然后检测是否出现输入类型不匹配的情况，使用 `fail()` 方法，注意 `fail()` 方法在当遇到 `EOF` 或者出现输入类型不匹配时都返回`true`。 
还可以使用`bad()`方法检测文件损坏或硬件错误而出现的输入错误。

``` c++
#include <iostream>
#include <fstream> // file I/O support
#include <cstdlib> // support for exit()
const int SIZE = 60;
int main()
{
    using namespace std;
    char filename[SIZE];
    ifstream inFile; // object for handling file input
    cout << “Enter name of data file: “;
    cin.getline(filename, SIZE);
    inFile.open(filename); // associate inFile with a file
    if (!inFile.is_open()) // failed to open file
    {
        cout << “Could not open the file “ << filename << endl;
        cout << “Program terminating.\n”;
        exit(EXIT_FAILURE);
    }
    double value;
    double sum = 0.0;
    int count = 0; // number of items read
    inFile >> value; // get first value
    while (inFile.good()) // while input good and not at EOF
    {
        ++count; // one more item read
        sum += value; // calculate running total
        inFile >> value; // get next value
    }
    if (inFile.eof())
        cout << “End of file reached.\n”;
    else if(inFile.fail())
        cout << “Input terminated by data mismatch.\n”;
    else
        cout << “Input terminated for unknown reason.\n”;

    if (count == 0)
        cout << “No data processed.\n”;
    else
    {
        cout << “Items read: “ << count << endl;
        cout << “Sum: “ << sum << endl;
        cout << “Average: “ << sum / count << endl;
    }

    inFile.close(); // finished with the file
    return 0;
}
```
