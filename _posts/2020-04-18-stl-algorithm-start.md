---
layout: post
title:  "STL算法(1)"
date:   2020-04-18 19:00:00
excerpt: "C++ STL算法入门"
categories: [code]
comments: true
---

## STL 算法 概要

* 算法
* 函数对象(function objects)
* 函数适配器(function adapters)

{% highlight C++%}
#include<algorithm>
#include<numeric>
#include<functional>
{% endhighlight %}

## 函数对象

### 如何使用函数对象

{% highlight C++%}
//下面为使用函数对象的例子
//set是自动排序的红黑树
//less<type>()是预定义的函数对象
set<int> a;		//默认为less<int>
set<int, less<int> > b;	//同a，也是自动从小到大排序
set<int, greater<int> > c;	//采用从大到小排序
{% endhighlight %}


### 函数对象与算法进行结合

for_each()实际上是一个函数，向它传两个迭代器，第三个参数可以是函数或函数对象。它会对两个迭代器之间的数据用这个函数或函数对象来处理。

{% highlight C++ %}
void print(int elem){
      cout<<elem;
}

vector<int> vec;
for(int i=0;i<9;i++)
      vec.push_back(i);
for_each(vec.begin,vec.end(),print)；
{% endhighlight %}

### 如何定义函数对象

函数对象实际上是一个类，为了把它做成函数对象，就必须有一个名为operator的成员函数，这样我们使用起来就像一个函数。它利用构造函数创建一个临时的对象，调用的时候是自动调用operator这个方法。

## 最大最小值算法

最大最小值算法一共有4个，可以用在所有的容器当中,函数返回值为迭代器：

{% highlight C++ %}
min_element(b,e);
min_element(b,e,op);
max_element(b,e);
max_element(b,e,op);
{% endhighlight %}

以下为使用deque做例子：

{% highlight C++ %}
#include<iostream>
#include<algorithm>
#include<deque>

using namespace std;

int main(){
	deque<int> deq;
	for(int i=2;i<=8;i++){
		deq.insert(deq.end(),i);
	}
	for(int i=-3;i<=5;i++){
		deq.insert(deq.end(),i);
	}
	cout<<"最小值为 ："<<*min_element(deq.begin(),deq.end())<<endl;
	cout<<"最大值为 ："<<*max_element(deq.begin(),deq.end())<<endl;
}
{% endhighlight %}

