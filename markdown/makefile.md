# makefile知识点总结

## 1. makefile的作用
makefile用于工程文件的自动化生成，其主要原理是利用makefile中提供的依赖关系，确定指定目
标是否比其依赖项更新，从而决定是否重新生成该目标。因而它一方面描述了目标之间的依赖关系
和生成规则，另一方面也能减少很多不必要的编译构建时间。

## 2. makefile的隐式规则
> `make -p`

上面的命令能打印make的内置变量及隐式规则

## 3. makefile的内置函数

## 4. makefile变量

## 5. makefile依赖关系

## 6. makefile扩展功能
### 1. include
### 2. define

## 7. makefile与shell关系