## 内核参数vm.max_map_count

### 起源

Elastic Search启动报错“max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]”，有必要看一下这个内核参数是什么意思。   
搜索到此参数的描述

```
This file contains the maximum number of memory map areas a process may have. Memory map areas are used as a side-effect of calling malloc, directly by mmap and mprotect, and also when loading shared libraries.  While most applications need less than a thousand maps, certain programs, particularly malloc debuggers, may consume lots of them, e.g., up to one or two maps per allocation.
```

简单来说，此参数设定了一个进程可以拥有最大内存映射区域，如果对Virtual Memory没有基本概念的，可以去看下操作系统相关的书籍，或者看懂维基百科上的[介绍](https://en.wikipedia.org/wiki/Virtual_memory)。

### 利用代码进行验证

深刻理解这个参数的最好方式是编写程序来验证，这里在go语言中采用两种方式来检验这个内核参数。

1. 调用make([]byte, n)来多次分配内存，直觉上，这样的方式并不一定会触发内存映射区域增加到上限。因为首先，go语言的make原语并不一定会立即触发malloc，试想如果我是go语言作者，我肯定希望在内存分配上尽可能避免系统调用，malloc会触发系统调用brk或mmap。在内存分配时，肯定更倾向先尽可能多地像OS要内存，在后续的内存使用和分配中，则尽可能地复用先前申请到的内存。另外，就算是malloc本身，也不见得每次都会调用mmap去开辟新的虚拟内存区域。

2. 直接调用mmap来做内存映射，这种方式肯定会触发内存映射区域增加到上限。

相关代码与执行环境：

我们先将这个参数设定为一个比较小的数量，比如设定为512。

```bash
echo 512>/proc/sys/vm/max_map_count
```

内存分配代码
```go

const MBytes = 1024*1024

func memAllocate(n int) {
    for i := 0; i < n; i++ {
        var buf = make([]byte, MBytes)
        if len(buf) != MBytes {
            panic("invalid allocate")
        }
    }
}

```

Mmap相关代码

```go

func mmapMem(n int) error {
    for i := 0; i < n; i++ {
        var fPath = fmt.Sprintf("./tmp/tmp%d.dat", i)
	    var f, err = os.OpenFile(fPath, os.O_RDWR|os.O_CREATE, 0666)
	    if err != nil {
		    return err
	    }
        mmap, err := syscall.Mmap(int(f.Fd()), 0, MBytes, syscall.PROT_READ|syscall.PROT_WRITE, syscall.MAP_SHARED)
	    if err != nil {
		    return err
	    }
    }
    return nil
}

```

测试结果

将max_map_count设为512时：
1. 分配大量的内存块都没有任何问题，这也证明了make->malloc并不是每次都会调用mmap来映射内存。
2. 映射文件时，在映射到490个文件时就已经报错，这证明了512个虚拟内存区域的上限限制了此进程的虚拟内存映射个数。为什么在490个文件时就报错了呢，这是因为此进程其余部分已经消耗了一些虚拟内存映射区域(比如go运行时已经消耗了一些内存映射区域)。

### 为什么是Elastic Search?

如果你对数据中间件有一定的了解，你会发现大部分数据中间件都会使用mmap系统调用，这种方式不会直接通过read/write的系统调用去操作磁盘上的文件，而是将这些文件映射到虚拟内存中，让上面以类似于操作内存的方式来操作文件，这种方法更高效。其余的中间件，例如MySQL，HDFS，Redis，MongoDB都没有Elastic Search这样的限制，那是因为这些中间件大多都采用大文件的方式来存储数据，文件本身的数量并不多。而Elastic Search则需要打开大量的文件句柄，进入Elastic Search任何一个节点的数据目录，运行tree命令都可以看到，"2856 directories, 21716 files"，有非常多的小文件。这也反映出Elastic Search与别的数据中间件的差异，它有着更多的小文件，调高内核参数vm.max_map_count是必要的。

### 永久生效

```bash
# 临时调高
echo 262144>/proc/sys/vm/max_map_count

#永久生效
vim /etc/sysctl.conf

#编辑内容
vm.max_map_count = 262144
```


