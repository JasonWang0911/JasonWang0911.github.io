---
title: Go 语言递归函数
---

递归，就是在运行的过程中调用自己。

语法格式如下：
```go
func recursion() {
   recursion() /* 函数调用自身 */
}

func main() {
   recursion()
}
```
Go 语言支持递归。但我们在使用递归时，开发者需要设置退出条件，否则递归将陷入无限循环中。

递归函数对于解决数学上的问题是非常有用的，就像计算阶乘，生成斐波那契数列等。

## **阶乘**
  以下实例通过 Go 语言的递归函数实例阶乘：
#### **代码实例**
```go
package main

import "fmt"

func Factorial(n uint64)(result uint64) {
	if (n > 0) {
		result = n * Factorial(n-1)
		return result
	}
	return 1
}

func main() {
	var i int = 20
	fmt.Printf("%d 的阶乘是 %d\n", i, Factorial(uint64(i)))
}

```
以上代码实例执行输出结果为：
```go
20 的阶乘是 2432902008176640000
```
## **斐波那契数列**

以下实例通过 Go 语言的递归函数实现斐波那契数列：

#### **代码实例**

```go
package main

import "fmt"

func Fibonacci(n int) int {
	if n < 2 {
		return n
	}
	return Fibonacci(n-2) + Fibonacci(n-1)
}

func main() {
	var i int
	for i = 0; i < 20; i++ {
		fmt.Printf("%d\t", Fibonacci(i))
	}
}

```
以上代码实例执行输出结果为：
```go
0	1	1	2	3	5	8	13	21	34	55	89	144	233	377	610	987	1597	2584	4181	
```
斐波纳契数列以如下被以递归的方法定义：F0=0，F1=1，Fn=F(n-1)+F(n-2)（n>=2，n∈N*）。
在这里的:
```go
return Fibonacci(n-2) + Fibonacci(n-1)
```
是指的 n-2 是 n 前面第二项的值，而不是 n-2=x 的值，那么 n-1 也与此同理。
更好的一种 fibonacci 实现，用到多返回值特性，降低复杂度：
```go
func fibonacci2(n int) (int,int) {
  if n < 2 {
    return 0,n
  }
  a,b := fibonacci2(n-1)
  return b,a+b
}


func fibonacci(n int) int {
  a,b := fibonacci2(n)
  return b
}
```

#### **求平方根**

**原理**: 计算机通常使用循环来计算 x 的平方根。从某个猜测的值 z 开始，我们可以根据 z² 与 x 的近似度来调整 z，产生一个更好的猜测：
```go
z -= (z*z - x) / (2*z)
```
重复调整的过程，猜测的结果会越来越精确，得到的答案也会尽可能接近实际的平方根。
### **代码实例**
```go
package main
import "fmt"

func sqrt(x float64,i float64) (float64,float64){
    remain:=(i*i-x)/(2*i);
    i=i-remain
    if(remain>0){
        return sqrt(x,i);
    }else{
        return i,remain
    }
}
func get_sqrt(x float64) float64{
    i,_ :=sqrt(x,x);
    return i;
}
func main(){
    var x, y int = 8959405, 7543758
    fmt.Println(x, "的平方根是：", get_sqrt(float64(x)))
    fmt.Println(y, "的平方根是：", get_sqrt(float64(y)))
}
```
以上代码实例执行输出结果为：
```go
8959405 的平方根是： 2993.2265199947697
7543758 的平方根是： 2746.5902497460374
```
