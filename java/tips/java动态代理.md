

我们知道所有基于jvm的语言，最终都会被自己的编译器编译成符合jvm规范的字节码文件提供给jvm。  
如图：

<img src="https://raw.githubusercontent.com/dchack/java_read_learn/master/view/java-class.jpg" width="50%" height="50%">  




##### CGLib






.class是十六进制
1.首先以二进制方式编辑这个文件
vi -b datafile

2.使用xxd转换为16进制
:%!xxd




JDK动态代理使用简单，它内置在JDK中，因此不需要引入第三方Jar包，但相对功能比较弱。CGLIB和Javassist都是高级的字节码生成库，总体性能比JDK自带的动态代理好，而且功能十分强大。ASM是低级的字节码生成工具，使用ASM已经近乎在于使用Javabytecode编程，对开发人员要求较高，也是性能最好的一种动态代理生辰工具。但ASM的使用是在过于繁琐，而且性能也没有数量级的提升，与CGLIB等高级字节码生成工具相比，ASM程序的可维护性也较差。
