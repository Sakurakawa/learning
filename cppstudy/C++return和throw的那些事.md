@[toc]

## C++return 和 throw 赋值过程中临时对象的问题

我们知道，`C++`中常用的`return`语句并不是直接将返回值作为右值赋值给左值的，在返回值的时候，首先有一个临时对象被创建，它与返回的函数值做交互，然后（如果你需要返回值）才与赋值表达式的左值交互。`throw`表现上和`return`有点相似，但也有点不一样。首先从实现上来说，`return`一个引用非常普遍，因为程序员可以手动指定函数返回类型嘛，而`throw`一个引用是不可能的，其次还有值和引用的一些不同，这是本文主要讨论的。

### 值相关

#### return

考虑一下代码：

```
class Sample {
	public:
		int name;
		Sample(int _name){
			this->name = _name;
			cout << name << " called the default constructor, address: " << this << endl;
		}
		Sample(const Sample& s) {
			this->name = s.name;
			cout  <<name<< " called the copy constructor, address: "  <<this
                  << ", copying "<<s.name  << " address: "  <<&s<< endl;
		}
		~Sample() {
			cout << name << " destruct, address: " << this << endl;
		}
	};
	Sample gen() {//返回值
		Sample s1(1);
		cout << "in gen()" << endl;
		return s1;
	}
	void sampleRun() {
		cout << "gen() start..." << endl;
		Sample s2 = gen();
		cout << "gen() end." << endl;
		s2.name =2;
		cout << "2, address: " <<&s2<< endl;
	}
```

在 VS2017 中编译并执行，执行结果：

```
gen() start...
1 called the default constructor, address: 010FFC64
in gen()
1 called the copy constructor, address: 010FFD5C, copying 1 address: 010FFC64
1 destruct, address: 010FFC64
gen() end.
2, address: 010FFD5C
2 destruct, address: 010FFD5C
```

首先在函数`gen()`中，地址为`010FFC64`的对象`s1`通过默认构造函数（其实在这里是一般构造函数，因为传参了）构造，`s1`是局部对象，地址在栈内存中，最终会在`gen()`执行完毕后会编译器销毁，在这之前，它需要保留数据，从执行结果来看，是一个地址为`010FFD5C`的对象担任这项工作，它是一个临时对象，没有名字，在这里姑且称它为`temp`。  
随后`s1`被销毁，函数`gen()`执行完毕。注意到从`temp`“变成”`s2`的过程，没有任何构造函数被调用，说明编译器秘密地让`s2`引用了`temp`，最后只有一个析构函数被调用证明了这一点。  
注意，由于信息不足，我们不知道`s2`引用`temp`是发生在`s1`被销毁前还是后。这里先说结论，是发生在销毁后。

#### throw

`throw`的情况有点不同，考虑一下代码：

```
void sampleTry() {
		try {
			Sample s1(1);
			cout << "in try" << endl;
			throw s1;
		}
		catch (Sample s2) {//注意这里catch的是值
			s2.name = 2;
			cout << "2, address: " << &s2 << endl;
		}
	}
```

VS2017 编译并执行：

```
1 called the default constructor, address: 005FF8C8
in try
1 called the copy constructor, address: 005FF7F0, copying 1 address: 005FF8C8
1 called the copy constructor, address: 005FF8BC, copying 1 address: 005FF7F0
1 destruct, address: 005FF8C8
2, address: 005FF8BC
2 destruct, address: 005FF8BC
1 destruct, address: 005FF7F0
```

一眼看上去，马上发现华点了：有三个地址！这说明有三个对象被实例化了，并且由代码执行顺序，很容易知道这些地址的归属：对象`s1`地址`005FF8C8`，临时对象`temp`地址`005FF7F0`，对象`s2`地址`005FF8BC`。
是的，`s1`返回被`temp`通过拷贝构造函数复制，后者接着以同样的方式被`s2`复制。`s2`和`temp`都是有独立地址的对象，因此最后都要被析构。
另外，这里我们能发现，从返回值到临时对象，再到赋值完成后，局部对象`s1`才被销毁。

### 引用相关

#### return

引用就大有不同了，考虑一下代码：

```
Sample& gen() {//返回的是引用
		Sample s1(1);
		cout << "in gen()" << endl;
		return s1;
	}
	void sampleRun() {
		cout << "gen() start..." << endl;
		Sample s2 = gen();
		cout << "gen() end." << endl;
		s2.name =2;
		cout << "2, address: " <<&s2<< endl;
	}
```

执行结果：

```
gen() start...
1 called the default constructor, address: 00D9FC04
in gen()
1 destruct, address: 00D9FC04
-858993460 called the copy constructor, address: 00D9FCF8, copying -858993460 address: 00D9FC04
gen() end.
2, address: 00D9FCF8
2 destruct, address: 00D9FCF8
```

我们知道，从实现的角度，临时对象`temp`一定要在`s1`销毁前保留数据，因此在`s1`调用析构函数前，`temp`引用了`s1`（因为没有构造函数被调用），随后`s1`销毁，`temp`和`s1`是一个地址，`temp`也被销毁了，成员`name`变成了垃圾值，`s2`随后复制`temp`，理所当然也是垃圾值了。由于`temp`提前跟随`s1`被销毁，因此最后只有`s2`调用析构函数。
这也解释了为什么不能返回函数内部对象的引用。

#### throw

而`throw`在这方面优化了：

```
void sampleTry() {
		try {
			Sample s1(1);
			cout << "in try" << endl;
			throw s1;
		}
		catch (Sample &s2) {//catch的是引用
      cout <<"before name is changed, "<<s2.name<< ", address: " << &s2 << endl;
			s2.name = 2;
			cout << "2, address: " << &s2 << endl;
		}
	}
```

执行结果：

```
1 called the default constructor, address: 00AFF6E0
in try
1 called the copy constructor, address: 00AFF608, copying 1 address: 00AFF6E0
1 destruct, address: 00AFF6E0
before name is changed, 1, address: 00AFF608
2, address: 00AFF608
2 destruct, address: 00AFF608
```

由`throw`的特性，完成赋值语句后，`try`中的局部对象`s1`才被销毁，这之前只有一个拷贝构造函数被调用，说明要么`temp`复制了`s1`然后`s2`引用了`temp`，要么`temp`引用了`s1`然后`s2`复制了`temp`。这里根据输出结果并不能判断到底属于哪种情况。
`throw`的先完成赋值再销毁局部变量的特性天然支持`catch`一个引用，而且有利于`catch`一个派生类（值传递`catch`派生类相当于基类转派生类），`C++`标准也推荐`catch`引用，百利无一害。
