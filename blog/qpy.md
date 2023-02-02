### QuickPy：动态类型语言的终极性能

##### QuickPy是什么？

QuickPy是一门新编程语言，它的语法和Python完全相同，但性能却和C++相同，QuickPy编译管线如下：

![](/pic/qpy.PNG)

##### 特性如下：

- 动态类型。即同一变量可以赋值不同类型。QuickPy的设计思路与Julia类似，但运行速度比Julia更快。
- 0性能损失。内置全调用链类型推导，尽可能推导出变量的类型以便加速。大多数情况下，类型推导会生效，这时可以做到编译出来的C++代码和手写的C++代码完全相同的性能。QuickPy是目前运行性能最强的动态类型AOT编译器。
- 无GC。当类型推导失效时变量为动态类型，这时会交给一个带引用计数的runtime执行。
- 无GIL。多线程性能和C++一样。
- 与CPython生态完美融合。QuickPy最终生成的pyd文件是标准的CPython模块，因此与CPython、pip 100%兼容。支持直接import其它Python库并调用，也支持被其它Python库调用。该特性得益于ken与CPython的无缝融合。
- 0学习成本。和Python语法完全相同，不新增语法，现有95%的Python代码可以不经修改直接用QuickPy编译运行，充分保护用户已有投资。
- 最小化用户击键次数。因为语法和Python完全相同。换句话说就是，QuickPy拥有C++的性能却可以比C++代码量更小。
- 牺牲编译速度换取运行速度。因为编译管线多了两个步骤，编译速度比C++慢0.5倍。

##### 性能测试：（win10 x64，i5-8500 3.00 GHZ）

||C++（VS2019 /O2 x64）|QuickPy -O0 x64|QuickPy -O2 x64|pypy-3.8-v7.3.7-win64|
|:---:|:---:|:---:|:---:|:---:|
|递归计算fib|1.97 s|3.75 s|1.83 s|7.08 s|
|循环叠加字符串|-|4.07 s|3.42 s|3.47 s|
|双重循环整数计算|0.19 s|2.13 s|0.19 s|1.39 s|

```python
import time

def fib(n):
	if n < 2:
		return 1
	return fib(n - 1) + fib(n - 2)

start = time.time()	
print(fib(43))
print(time.time() - start)
```

```cpp
#include <stdio.h>
#include <windows.h>
typedef __int64 int64;

int64 fib(int64 n)
{
	if (n < 2ll)
	{
		return 1ll;
	}
	return fib(n - 1ll) + fib(n - 2ll);
}

int main()
{
	auto start = GetTickCount();
	printf("%I64d\n", fib(43ll));
	printf("%d\n", GetTickCount() - start);
	return 0;
}
```

```python
import time

start = time.time()	
s = ""
for i in range(100):
	for j in range(1000):
		s += str(i * j)
print(len(s))
print(time.time() - start)
```

```python
import time

start = time.time()	
sum = 0
for i in range(100000):
	for j in range(10000):
		sum += i * j
print(sum)
print(time.time() - start)
```

```cpp
#include <stdio.h>
#include <windows.h>
typedef __int64 int64;

int main()
{
	auto start = GetTickCount();
	int64 sum = 0ll;
	for (int64 i = 0ll;i < 100000ll; i++)
	{
		for (int64 j = 0ll;j < 10000ll; j++)
		{
			sum += i * j;
		}
	}
	printf("%I64d\n", sum);
	printf("%d\n", GetTickCount() - start);
	return 0;
}
```

因为QuickPy的整数为64位，C++这边也使用int64以示公平，递归计算的0.14 s差距是因为误差，本文作者已多次校验，QuickPy生成的C++代码是0性能损失，代码片段如下：

```cpp
( start = ( time . time ) ( ) ) ;
( sum = 0ll ) ;
( i = 0ll ) ;
_for_judge_2 : ;
( _COND = ( i < 100000ll ) ) ;
if ( _COND == 0ll ) { goto _for_end_2 ; } ;
( j = 0ll ) ;
_for_judge_3 : ;
( _COND = ( j < 10000ll ) ) ;
if ( _COND == 0ll ) { goto _for_end_3 ; } ;
( sum = ( sum + ( i * j ) ) ) ;
_for_add_3 : ;
( j = ( j + 1ll ) ) ;
goto _for_judge_3 ;
_for_end_3 : ;
_for_add_2 : ;
( i = ( i + 1ll ) ) ;
goto _for_judge_2 ;
_for_end_2 : ;
print ( sum ) ;
print ( ( ( time . time ) ( ) - start ) ) ;
```

##### QuickPy的设计哲学：

1. 极简主义。如无必要，不增实体。一个极简的东西必然是精致的，QuickPy的目标是成为精致的艺术品，艺术优先于技术。
2. 如无必要，不发明新东西。
3. 运行性能优先，在保证性能的基础上尽可能保持Python语法兼容性。
4. 死扣每一处细节。生成的C++代码与手写的C++必须保证相同的性能，慢1纳秒都不行。死扣性能细节也是C++的哲学。
5. 0开销抽象。这一条是上一条的变种。
6. 追求极致语言表达力，为此可以牺牲编译速度。
7. 迎合大多数人的喜好，而不是语言设计者的喜好。只有这样才能提升社会整体生产力。
8. 充分利用现有的轮子和生态，蹭CPython的库、蹭最新C++的特性、蹭成熟C++编译器的优化。但它的设计目标为工业级产品，只蹭大规模使用并成熟稳定的东西（比如CPython、cl.exe等），不蹭pypy、cython、mingw等未经考验的产品。
9. 只做有效改进，绝不做无效改进或倒退式变化。不管对语言的设计还是编译器的实现都是一样。像把var换成local、把花括号换成begin end，这些属于典型的无效改进，甚至是倒退式变化，因为换一种写法会增加人们的记忆负担。像C++的匿名函数、atomic、thread、thread_local，这些属于典型的有效改进，新增的功能可以帮助人们更快更好地解决问题。另外像编译器的优化也属于有效改进。

##### 为什么编译成C++而不是LLVM IR？

- 可以蹭最先进的C++特性。
- 轻松使用CPython API与CPython交互。
- 编译成C++就有众多C++编译器后端可以选择，cl.exe在windows上比clang更成熟、更稳定、更快。若选择LLVM IR就只能在clang/LLVM这一棵树上吊死。
- 编译成C++也有缺点，就是会降低整体编译速度。

##### QuickPy是一门新语言还是Python的一种新实现？

这是一个哲学问题，本质上和忒修斯之船类似，当船上的每一块木板都换掉了，船的外观不变，但这艘船真的还是原来的船吗？

虽然QuickPy和Python外观看起来一样，但是底层实现完全不同。在类型推导生效时，它就像一个披着Python外衣的C++。所以不管把它称为新语言还是Python新实现，都是可以的，这其实是同一个事物的两种不同视角。

本文作者更倾向于把QuickPy当作代码量更少的C++，而不是更快速的Python。

##### 为什么取名QuickPy？
向QuickJs的作者致敬。

##### 支持调试吗？
目前不支持，未来计划支持。

##### 会在Python语法基础上新增语法吗？
暂时不新增语法。未来可能新增thread、thread_local、atomic、co_yield等关键字，充分利用最先进的C++特性。但是依然会保证对CPython语法的兼容性，最大程度保证已有Python代码可以不经修改直接在QuickPy上运行。

##### 为什么只能兼容95%的已有Python代码？
C++也不能100%兼容C，win10也不能100%兼容win7上的软件。因为QuickPy底层与CPython完全不同，100%兼容是不可能的。

##### 不兼容的代码怎么办？
目前QuickPy可以支持文件级部分编译，也就是和CPython混合执行，这样就可以把不兼容的文件扔给CPython执行。未来还打算支持函数级部分编译、自动标注不兼容代码、自动转换不兼容代码为兼容代码，这些技术可以略提高兼容性，但仍不可能100%兼容。

##### 静态类型推导成功率多高？

推导失败的情况：

- 调用的是一个第三方的没有源码的pyd文件，该文件的所有函数返回值均无法推导出类型。
- 同一变量赋值为不同类型。
- 异型list、dict。即list或dict内的元素存在不同类型。

推导成功的情况：

- 在QuickPy管辖范围内的代码，即经过QuickPy处理的py文件或pyc文件。
- 调用的是一个第三方的没有源码的pyd文件，但该文件内的函数返回值用构造函数包围了，如int(f(...))、str(f(...))。
- 变量始终为同一类型。
- 同型list、dict。
- 推导失败的部分性能下降到pypy水平，这部分代码达不到C++的性能。但推导失败不会影响程序正确性。

##### 闭包如何处理？
暂时用C++的静态闭包。后续可能加上静态闭包和动态闭包混合执行，即在静态闭包处理不了的情况下切到动态闭包，动态闭包依然由前面提到的引用计数runtime接管。当然，这些对用户不可见。

##### 有可能比C/C++更快吗？
在计算机体系结构和计算机理论没有重大突破前，单线程运行性能几乎不可能比C/C++更快，任何语言或编译器都做不到。QuickPy的设计目标是达到和C/C++完全一样的性能，而不是超越。

##### 什么是动态类型语言的终极性能？
类型推导、AOT、顶级静态类型优化编译器，三者合一就是动态类型语言的终极性能，这就是QuickPy。
