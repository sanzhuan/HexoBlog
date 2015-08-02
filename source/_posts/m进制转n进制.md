title: "m进制转n进制"
date: 2015-08-01 22:12:37
tags: c++
---

## 问题描述：
写一个函数把m进制转成n进制，m和n都是大于等于2，小于等于36，进制数字集合为（0 1 2 3 4 5 6 7 8 9 A B C D E F G H I J K L M  N O P Q R S T U V W X Y Z ）。
<!-- more -->
## 解题思路：
先把m进制转成10进制，再把10进制转成n进制，代码如下：
``` cpp
/************************************
* dest 转换完毕的n进制字符串
* n 目的进制数（十六进制填16、二进制填2等）
* src 初始m进制字符串
* m 初始进制数，与目的进制数表示方法相同
* 注意，初始m进制字符以10进制表示必须不超过int表示范围，由调用方保证输入的合法性。
*************************************/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

long toDecimal(const char *src,unsigned int m)  
{     
	long i,b=1,sum=0;  
	int length=strlen(src);  
	for (i=length-1;i>=0;i--)  
	{  
		if (src[i]>='A')  
		{  
			sum+=(src[i]-'A'+10)*b;  
			b*=m;  
		}  
		else  
		{  
			sum+=(src[i]-'0')*b;  
			b*=m;  
		}  
	}  
	return sum;  
}  

long toN(char *dest,unsigned int n,long decimal)  
{
	char *p = dest;
	char c;
	long remainder = 0;

	while(decimal)  
	{
		remainder = decimal % n;
		if(n > 10 && remainder >= 10)  
			*p++ = remainder + 'A';
		else
			*p++ = remainder + '0';


		decimal = decimal / n;
	}  
	*p-- = '\0';

	while (p > dest) {
		c = *p;
		*p-- = *dest;
		*dest++ = c;
	}

}

void m2n(char *dest, unsigned int n, const char *src, unsigned int m)
{
	long decimal = 0;

	/* m进制字符串src转为10进制数 */
	decimal = toDecimal(src,m);

	/* 10进制数转成n进制数，保存在dest中 */
	toN(dest,n,decimal);
}


int main(int argc,char *argv[])
{
	char *result = (char *)malloc(sizeof(char) * strlen(argv[2]) * 4);
	if(argc < 4)
	{
		printf("usage:./m2n 目的进制 源进制数据 源进制");
		exit(1);
	}

	m2n(result, atoi(argv[1]), argv[2], atoi(argv[3]) );

	printf("result=%s\n",result);

	free(result);
	result=NULL;

	return 0;
}

```
注意点：
+ 对超过十进制和不超过的要分开处理，如上述代码*p++ = remainder + 'A';和*p++ = remainder + '0';
+ 这个程序没有考虑整形溢出的问题
+ 这个程序不知道要分配多大的内存才不会溢出，我想的是十进制的9表示成二进制需要4位，这里直接把内存设成4倍的源数据大小，对不对还有待考虑