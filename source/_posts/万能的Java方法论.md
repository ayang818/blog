---

title: IDEA远程调试

date: 2019-10-03 20:17:24

tags: [Java]

---

> java世界里的一切东西都是在拼Java命令行参数



## Debug之前

调试一切Java程序，想清楚两点

- 想清楚要调试的代码跑在哪个JVM中
- 找到要调试的源代码



随时随地Debug

```bash
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=1044 FileName
```
<!-- more -->
1. IDEA连接Remote 调试

2. 一个鬼畜的方法，调试Java代码

   ```bash
   export JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=1044
   # 取消
   unset JAVA_TOOL_OPTIONS
   
   ls -alts
   ```


## IDEA Debug技巧

- step over ：逐行调试，跳过函数(接收结果)
- step into ：单步调试，进入函数体(不进入JDK原生方法)
- force step into ：同上，进入JDK原生方法
- step out ：跳出方法
- resume problem ：跳至下一个断点