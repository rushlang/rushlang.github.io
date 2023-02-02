### ken：C++ 、Python混合前端编译器

##### ken是什么？
ken是一个C++ 、Python混合前端编译器，也可以把它当作C++和Python的交互媒介，利用它可以让C++、Python无缝融合，它的编译管线如下：

![](/pic/ken.PNG)

##### 主要解决什么问题？
目前最好的Python调用C++的框架是pybind11。但在pybind11中，C++调用Python非常繁琐，且传递嵌套的复杂结构（如vector<vector<double>>）也很繁琐。

ken克服了pybind11的缺点，它支持：

- Python和C++双向无缝互相调用。
- 双向传递任意复杂的嵌套模板和任意复杂的嵌套结构体对象。比pybind11更简单、好用，更不必说ctypes、SWIG了。
- 支持C++和Python混合在一个文件中，两种语言融会贯通，互相调用十分方便。（当然也支持分开在不同的文件中）

##### 它还有什么其它的特性？
- 0性能损失。C++部分保持性能不变，Python部分也保持性能不变。因此ken的性能与pybind11是一样的，理所当然就比cython、Nuitka更高效。
- 与CPython生态完美融合。ken利用CPython API与CPython交互，最终生成的pyd文件是标准的CPython模块，因此与CPython、pip 100%兼容。支持直接import其它Python库并调用，也支持被其它Python库调用。
- 完全接管并控制了C++的宏、模板和类型系统。也就是说最终的C++文件不包含宏、模板。你可能会联想到，UE4或QT也有对C++代码进行简单预处理，但ken和它们完全不同，ken对C++代码进行了深度解析，所以可以获取全部的C++元信息。

##### 为什么取名ken？
向C语言创造者之一Ken Thompson致敬。

##### 应用场景：

- 海量数据处理。Python有很多库可以用来做数据分析和制表，但若有大量计算则需要把计算部分挪到C++。这时使用ken可轻松对接两种语言，几乎不用付出任何努力。
- AI。Python是AI训练的首选语言，数据预处理部分可以用C++加速。
- 游戏引擎。Python可以用来写界面和脚本逻辑，C++写核心引擎。
- 扩充C++库。Python是目前库最多的语言，比C++的库更多，Python库使用也比C++库方便得多。使用ken可以直接在C++中import任意Python库，大大扩展了C++的能力。

##### 混合编程有什么好处？

- 对于简短的双语言函数不用多开一个文件，写起来更方便。
- 若有两个Python模块a.py和b.py，a想调用b中的函数，同时b也想调用a中的函数，在CPython中，这两个模块无法同时互相import，这时只能将两个函数挪到一个文件里。C++函数和Python函数互相调用也会有这个需求。
- 每种语言都有自己的优势，C++运行快，Python写起来方便，混合编程可以同时发挥两种语言的优势。
- ken不仅支持文件粒度的混合编程，还支持函数粒度、语句粒度、表达式粒度的混合，也就是说，可以在一个C++函数里混写Python变量和C++变量，并且两种变量之间可以自动隐式互相转换，用起来十分方便。但是，ken目前暂时不支持在Python函数里混写C++变量（理论上可以支持）。

##### 运行环境：
新版本win7 x64可以直接运行，老版本win7 x64需要打KB4474419和KB4490628两个补丁才能执行，因为VS2019中的c1xx.dll需要它们。

win10 x64可以直接运行。

##### 示例代码：
示例1：

```cpp
#include "var.kpy"

def add(a, b):
	return a + b
	
int main()
{
	KPrintL(add(1, 2));
	KPrintL(add(0.99, 0.98));
	KPrintL(add(string("ab"), string("cd")));
	return 0;
}
```

示例2：

```cpp
#include "var.kpy"

_kpy int add(int a, int b)
{
	return a + b;
}

def test():
	print(add(1, 2))
	
int main()
{
	test();
	return 0;
}
```

示例3：

```cpp
#include "var.kpy"

struct C
{
	_kpy static void test1(int a)
	{
		KPrintL(a);
	}
	
	_kpy static int test2(int a)
	{
		return a + 1;
	}
};

def testCallMemberFunc():
	C::test1(111)
	print(C::test2(111))

int main()
{
	testCallMemberFunc();
	return 0;
}
```

示例4，将Python的函数对象扔给C++，并在C++里执行：

```cpp
#include "var.kpy"

def add(a, b):
	return a + b
	
def getFunc():
	return add

int main()
{
	auto f = getFunc();
	int temp = f(1, 2);
	KPrintL(temp);
	return 0;
}
```

示例5：

```cpp
#include "var.kpy"

import os

def out(a):
	print(a, type(a))
	
def testPyCallCpp():
	pyCallCpp(123456, "abcde", 
		[[1, 2, 123456], [1, 2, 123456], [1, 2, 123456]],
		[1, "abc", [11, 22, 33], 0], 
		1.98788, (-1, 2, -1234567), 
		{"a" : 22, "bc" : 33})
	
struct A
{
	int a;
	string b;
	vector<int> c;
	void* d;
};

struct B
{
	A a;
	A b;
};

//加上_kpy关键字表示函数既可被Python调用，又可被C++调用
_kpy void pyCallCpp(int a, string b, vector<vector<int>> c, A d, double e, 
	vector<int> f, map<string, int> g)
{
	KPrintL(a);
	KPrintL(b);
	KPrintL(c.Count());
	KPrintL(c[0][0]);
	KPrintL(c[1][1]);
	KPrintL(c[2][2]);
	KPrintL(d.a);
	KPrintL(d.b);
	KPrintL(d.c[0]);
	KPrintL(d.c[1]);
	KPrintL(e);
	KPrintL(f[0]);
	KPrintL(f[1]);
	KPrintL(f[2]);
	KPrintL(g.Count());
	KPrintL(g["a"]);
	KPrintL(g["bc"]);
}

int main()
{
	for(int i = 0; i < 80000; i++)
	{
		out(99999999);
		out(string("abc"));
		out(int64(99999999));
		out("abcd");
		out(0.98778);
		
		//ken有一套自己的基础库，也可以跟STL混合使用，即vector和std::vector是不同的
		vector<string> a;
		a.Push(string("aaaaaa"));
		a.Push(string("bc"));
		
		vector<vector<string>> b;
		b.Push(a);
		b.Push(a);
		
		out(a);
		out(b);
		
		vector<int> c;
		c.Push(222222222);
		c.Push(222222223);
		
		vector<vector<int>> d;
		d.Push(c);
		d.Push(vector<int>());
		d.Push(c);
		
		out(c);
		out(d);
		
		set<int> e;
		e.Insert(22222);
		e.Insert(333333333);
		
		out(e);
		
		vector<set<int>> f;
		f.Push(e);
		f.Push(set<int>());
		
		out(f);
		
		A g;
		g.a = 9999999;
		g.b = "abcd";
		g.c = c;
		g.d = null;
		
		out(g);
		
		B h;
		h.a = g;
		h.b = g;
		
		out(h);
		
		map<string, double> d1;
		d1["abc"] = 0.5;
		d1["de"] = 1.9784;
		
		out(d1);
		
		map<int, vector<double>> d2;
		d2[99] = vector<double>(2, 0.99);
		d2[98] = vector<double>(3, 0.98);
		
		out(d2);
		
		testPyCallCpp();
	
		os.system("dir"); 
		
		KPrintL(i);
	}
	KPrintL("end");
	return 0;
}
```

##### 一种新的编译期计算技术 -- cpdef：
如果你以为ken到上面就结束了，但其实那只是个开始，好戏还在后头。

接下来本文将介绍ken的核心技术之一，cpdef。这种技术大家可能闻所未闻，但是它却是实现混合编程的核心基础设施。它无论在性能还是可读性、易用性、扩展性上都可以吊打并替代模板元编程。这项技术最早在RPP上引入，但当时只是雏形，很不成熟。现在它在ken中已经成熟了，引入cpdef的ken就像老虎插上了翅膀，变成了一只会飞的老虎，这是相当可怕的。

C++的模板元本质上是一种编译期计算，看起来是一个很炫酷的东西，但模板元为什么成为了少数智力精英的玩物？甚至沦为奇技淫巧？原因就是它晦涩难懂，简单地说就是难读又难写。cpdef克服了模板元的缺点，它既简单又高效。本文作者相信，自从有了cpdef，模板元的日子就难过了。

请看示例6，这个例子实现了泛型加法计算，当然它也可以用模板实现，这里只是为了说明问题故意用了cpdef：

```cpp
#include "var.kpy"

cpdef add():
	type = _vtype[0]
	s = type + " add(" + type + " a, " + type + " b){return a + b;}"
	genFunc(s)
	
int main()
{
	KPrintL(add(1, 2));
	KPrintL(add(0.5, 0.6));
	KPrintL(add(string("ab"), string("cd")));
	return 0;
}
```

示例7，这个例子实现了可变参数函数：

```cpp
#include "var.kpy"

cpdef add():
	s = "int add("
	for i in range(_count):
		if i != 0:
			s += ","
		s += "int a" + str(i)
	s += "){"
	s += "return "
	for i in range(_count):
		if i != 0:
			s += "+"
		s += "a" + str(i)
	s += ";"
	s += "}"
	genFunc(s)
	
int main()
{
	KPrintL(add(1, 2));
	KPrintL(add(1, 2, 3, 4, 5));
	return 0;
}
```

从上面可以看出，cpdef的语法和Python完全相同，只需要把def替换成cpdef，用户完全不需要学习成本。但cpdef并不是调用CPython解释器执行的，因为CPython太重度了，cpdef通过一个内置的动态类型轻量解释器执行，启动速度非常快（可达到lua的水平），执行效率也很高，比只支持递归的模板元快得多，因为模板元是通过编译器内部递归实现的。

cpdef就像一个编译期钩子，比如上面设定了一个钩子add，当语法分析器扫描到add时，就会触发这个钩子。而cpdef函数就像一个普通的Python函数一样，可以支持选择、循环、递归，显然它是图灵完备的。

另外cpdef还可以通过内部函数或内部变量与编译器通信，比如genFunc就是一个内部函数，当执行genFunc时，编译器会解析一段字符串代码并插入。第一次遇到add(1, 2)时，语法分析器进入cpdef解释器，第二次遇到add(1, 2)时，由于编译器已经找到了int add(int a0, int a1)，可以匹配，所以不会启动cpdef解释器，所以不用担心重复执行。

对于普通的C++模板，只有类型和数字是可以泛化的，而cpdef可以泛化整个函数，不只是类型和数字，这就是它的扩展性强于C++模板的地方。换句话说，cpdef可以无中生有地创造任意一个你想要的函数，模板元有的功能它都有，模板元没有的功能它也有，因此它的功能比模板元强得多。（当然，cpdef本质上也是一种元编程技术，即写代码去生成代码。通常来说写代码去生成代码是编译器干的事，而元编程可以赋予程序员修改编译器的能力，且不需要重新编译编译器源码。）

请看示例8，这个例子比较复杂，它实现了任意一个Python变量自动隐式转换为vector<T>，其中var是对PyObject*的包装类。该功能用普通的C++是很难实现的。因为代码比较长只截取片段：

```cpp
template <typename T>
struct vector
{
	~vector<T>()
	{
		...
	}
	
	vector<T>()
	{
		...
	}
	
	cpdef vector<T>():
		type = _vtype[0]
		if type != "var":
			return
		classType = "vector<" + T + ">"
		s = "final " + classType + "(const var& a) {"
		if isValidType(T):
			s += "if (PyTuple_Check(a.pyObj)){"

			s += "int size = KCast(int, PyTuple_Size(a.pyObj));"
			s += "Init();"
			s += "Alloc(size);"
			s += "for (int i = 0; i < size; i++)"
			s += "{"
			s += "point[i] = var(PyTuple_GetItem(a.pyObj, i), 1);"
			s += "}"

			s += "}else{"

			s += "int size = KCast(int, PyList_Size(a.pyObj));"
			s += "Init();"
			s += "Alloc(size);"
			s += "for (int i = 0; i < size; i++)"
			s += "{"
			s += "point[i] = var(PyList_GetItem(a.pyObj, i), 1);"
			s += "}"

			s += "}"

			s += "}"
			genFunc(s)
	
	...
};
```

示例9，cpdef自身也是可以递归的，并可以调用其它的cpdef函数，这是一个编译期计算阶乘的例子：

```cpp
#include "var.kpy"

cpdef f(n):
	if n <= 1:
		return 1
	return n * f(n - 1)

cpdef test():
	genFunc("int test(){return " + str(f(5)) + ";}")
	
int main()
{
	KPrintL(test());
	return 0;
}
```

##### cpdef对比C++模板：
- C++只能add<1, 2>()、add<1, 2, 3>()，但cpdef可以支持add(1, 2)、add(1, 2, 3)，这种可变参数是最自然和直接的。
- C++的可变参数模板因为是递归的，遇到复杂的返回对象会造成编译器无法优化，导致性能问题。cpdef没有这个问题，cpdef可以做到和手写代码完全相同的性能。
- cpdef可以泛化整个函数，除了上面的例子，它还可以做其它更神奇的事，制约你的只有想象力。

##### 一种新的宏编译期计算技术 -- #cpdef：
cpdef是在语义分析阶段进行的，它的执行时机和模板函数一样，因此它可以取到变量的类型。而#cpdef不同，#cpdef是在预处理和词法分析阶段进行的，因此它只能处理文本。简单地说，cpdef可以替代模板函数，#cpdef可以替代#define。下面是一个#cpdef求和并进行文本替换的例子：

```cpp
#include "var.kpy"

#cpdef sum():
	sum = 0
	for i in range(len(_vparam)):
		sum += int(_vparam[i])
	replaceText(str(sum))

int main()
{
	KPrintL(sum(1, 2));
	KPrintL(sum(1, 2, 3, 4, 5));
	return 0;
}
```
