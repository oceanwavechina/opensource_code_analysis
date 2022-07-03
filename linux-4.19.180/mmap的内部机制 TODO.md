# mmap的内部机制

mmap可以实现把一个文件映射到内存区域中，这样一来，直接修改内存就可以达到更新文件的目的。

一个mmap的 [例子](https://gitee.com/oceanwave/linux_playground/blob/master/linux_api/mmap_copy_files.cpp),演示了如何把文件映射到内存中

<br>

## 1. 内核中的调用栈
----
<br>

通过 `ag mmap | grep SYSCALL` 可以找到该系统调用的入口在 `arch/x86/kernel/sys_x86_64.c:91` 

``` cpp
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, off)
        |
        |-- ksys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
            |
            |   file = fget(fd);
            |-- vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
                |
                |   security_mmap_file(file, prot, flag);
                |-- do_mmap_pgoff(file, addr, len, prot, flag, pgoff, &populate, &uf)
                    |
                    |-- do_mmap(file, addr, len, prot, flags, 0, pgoff, populate, uf)
                        |
                        |-- mmap_region(file, addr, len, vm_flags, pgoff, uf)
                            |
                            |   vm_area_alloc(mm)
                            |-- call_mmap(file, vma)
                                |
                                |-- file->f_op->mmap(file, vma)
```

可以看到最终调用的是文件上的函数指针mmap，来把file中的内种和vma关联起来。

我们以普通磁盘文件系统为例 `fs/reiserfs/file.c:250` (搜索 `ag '.mmap = '`):

``` cpp
{.mmap = generic_file_mmap}
    |
    |-- int generic_file_mmap(struct file * file, struct vm_area_struct * vma)
        |
        |-- vma->vm_ops = &generic_file_vm_ops;
```

``` cpp
const struct vm_operations_struct generic_file_vm_ops = {
	.fault		= filemap_fault,
	.map_pages	= filemap_map_pages,
	.page_mkwrite	= filemap_page_mkwrite,
};
```

可以看到是最终是 ```vm_area_struct``` op指向了 ```generic_file_vm_ops```

<br>

## 2. 调用层分析
----
<br>

接下来，我们按上诉调用栈依次看看mmap的内部流程

<br>

### 2.1 系统调用入口: ```SYSCALL_DEFINE6(mmap)```
<br>

我们先看看入口， 源代码在 `arch/x86/kernel/sys_x86_64.c:91`

``` cpp
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, off)
{
	long error;
	error = -EINVAL;
	if (off & ~PAGE_MASK)
		goto out;

	error = ksys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
out:
	return error;
}
```

``` cpp
/* PAGE_SHIFT determines the page size */
#define PAGE_SHIFT	13
#define PAGE_SIZE	(_AC(1,UL) << PAGE_SHIFT)
#define PAGE_MASK	(~(PAGE_SIZE-1))
```

这里有一个参数校验，其目的主要是判断 off 这个参数（也就是offset），确保offset 是 PAGE_SIZE 的整数倍。

玩法是这样的： ~PAGE_MASK = (PAGE_SIZE-1) = 8191 = 0001 1111 1111 1111b
    
任何不是page_size 整数倍的off 在计算 `off & ~PAGE_MASK` 后都是非0的 这也就是掩码的作用

还有一个问题，为什么一定要是 page_size 的整数倍，参考资料中有个解释很通俗易懂。

因为 os 管理内存和文件的最小单元是page，如果一个offset不是 page_size 的整数倍，os需要判断offset所在的page。


<br>

### 2.2 ksys_mmap_pgoff: ```SYSCALL_DEFINE6(mmap)```
<br>

这个函数的作用是根据传入的 fd 获取对应的 ``` struct file * ```


<br><br><br>


## 参考资料
----
<br>

* [Why file starting offset in mmap() must be multiple of the page size](https://stackoverflow.com/questions/20093473/why-file-starting-offset-in-mmap-must-be-multiple-of-the-page-size)