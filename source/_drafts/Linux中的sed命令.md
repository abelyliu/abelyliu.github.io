---
title: Linux中的sed命令
tags:
---
在Unix体系中，一切皆文件。许多复杂的操作只需要修改几个配置文件即可，如果通过脚本实现一些功能，学会修改文件则是必不可少的。这里谈谈Linux中sed的常见用法。

```sh 
echo "This is a test" | sed 's/test/big test/'

output:
This is a big test
```

sed命令的一般格式为`sed options script file`，上面的例子中省略了选项，s代表执行替换操作，/为分隔符，test为被替换的字符串，big test为要替换的字符串。分隔符可以为任意字符，但必须前后一样，比如用-，则为`echo "This is a test" | sed 's-test-big test-'`

sed命令会将输出输出到STDOUT，如果输入是文件，执行过sed命令后并不会修改源文件，这点值得注意。
## 参考链接
1. Linux命令行与Shell脚本编程大全

