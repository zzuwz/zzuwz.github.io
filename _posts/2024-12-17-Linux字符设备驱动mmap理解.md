---
title: Linux字符设备驱动mmap理解
date: 2024-12-17 18:14:00 +0800
categories: [Linux]
tags: [Linux内核,Linux驱动]
---
<script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>

<link rel="stylesheet" href="https://use.fontawesome.com/releases/v5.3.1/css/all.css" integrity="sha384-mzrmE5qonljUremFsqc01SB46JvROS7bZs3IO2EmfFsd15uHvIt+Y8vEf7N7fWAU" crossorigin="anonymous">

<p align="right"><i class="fa fa-eye"></i> 阅读次数: <span id="busuanzi_value_page_pv"><i class="fa fa-spinner fa-spin"></i></span></p>

## mmap的本质
mmap的本质就是如下图所示，内核态申请一块空间，用户态想直接控制这段内存空间，利用`mmap`机制，实现内核态和用户态不同的虚拟地址共用一块物理内存，实现内存映射，读写内存时，可以提高读写速度。

![](/assets/image/WEBRESOURCEca4636cada85f39f921e92744f6c6bf8image.png)

在用户态可以通过查看 `/proc/pid/maps`  会展示该进程对应的所有虚拟内存段。

![](/assets/image/WEBRESOURCE76bb3bdec32e65d1d915f28ed53743e0image.png)

![](/assets/image/WEBRESOURCE7fa04104017c7546040d61b76fe6e296image.png)

`/proc/pid/maps`中的每一行其实都对应一个上面的结构体。内核进程内存分配如下：

![](/assets/image/WEBRESOURCE6aec4936aff441dad9261d7b91b5d32cimage.png)


## 代码详细分析

以两个例子进行分析，一个共享8KB的共享内存，一个共享20MB的共享内存   大内存和小内存是有一定的差异的。

内核通过`kmalloc` 申请8KB的内容，进行映射。

### 内核态：

```c
#include <linux/cdev.h>
#include <linux/delay.h>
#include <linux/errno.h>
#include <linux/fs.h>
#include <linux/gpio.h>
#include <linux/init.h>
#include <linux/ioctl.h>
#include <linux/kernel.h>
#include <linux/list.h>
#include <linux/miscdevice.h>
#include <linux/mm.h>
#include <linux/module.h>
#include <linux/moduleparam.h>
#include <linux/pci.h>
#include <linux/slab.h>
#include <linux/string.h>
#include <linux/types.h>

#define DEVICE_NAME "mymap"

static unsigned char array[10] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
static unsigned char *buffer;

static int my_open(struct inode *inode, struct file *file) { return 0; }


static int my_map(struct file *filp, struct vm_area_struct *vma) {
  unsigned long page;
  unsigned char i;
  unsigned long start = (unsigned long)vma->vm_start;
  // unsigned long end =  (unsigned long)vma->vm_end;
  unsigned long size = (unsigned long)(vma->vm_end - vma->vm_start);
  printk(KERN_INFO"the size is %d\n",size);
  // 得到物理地址
  page = virt_to_phys(buffer);
  // 将用户空间的一个vma虚拟内存区映射到以page开始的一段连续物理页面上
  if (remap_pfn_range(
          vma, start, page >> PAGE_SHIFT, size,
          PAGE_SHARED)) // 第三个参数是页帧号，由物理地址右移PAGE_SHIFT得到
    return -1;

  // 往该内存写10字节数据
  for (i = 0; i < 10; i++)
    buffer[i] = array[i];

  return 0;
}


static long my_ioctl(struct file *file,        /* ditto */
                 unsigned int cmd,      /* number and param for ioctl */
                 unsigned long param)
{
        /* ioctl回调函数中一般都使用switch结构来处理不同的输入参数（cmd） */
        switch(cmd){
        case 0:
        {
                printk(KERN_INFO "[TestModule:] Inner function (ioctl 0) finished.\n");
                printk(KERN_INFO "the mem value is %d\n",buffer[0]);
                break;
        }
        default:
                printk(KERN_INFO "[TestModule:] Unknown ioctl cmd!\n");
                return -EINVAL;
        }
        return 0;
}


static struct file_operations dev_fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .mmap = my_map,
    .unlocked_ioctl = my_ioctl,
    
};

static struct miscdevice misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = DEVICE_NAME,
    .fops = &dev_fops,
};

static int __init dev_init(void) {
  int ret;

  // 注册混杂设备
  ret = misc_register(&misc);
  // 内存分配
  buffer = (unsigned char *)kmalloc(2*PAGE_SIZE, GFP_KERNEL);
  // 将该段内存设置为保留
  SetPageReserved(virt_to_page(buffer));

  return ret;
}

static void __exit dev_exit(void) {
  // 注销设备
  misc_deregister(&misc);
  // 清除保留
  ClearPageReserved(virt_to_page(buffer));
  // 释放内存
  kfree(buffer);
}

module_init(dev_init);
module_exit(dev_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("LKN@SCUT");

```

### 用户态：

```c
#include <stdio.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<unistd.h>
#include<sys/mman.h>
#include <stdlib.h>
#include <string.h>

int main()
{
    int fd;
    char *start;
    int i;
    
    /*打开文件*/
    fd = open("/dev/mymap",O_RDWR);
        
    start=mmap(NULL,10,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
    
    /* 读出数据 */
    for( i=0;i<10;i++){
        printf("%d ",start[i]);
    }
    printf("\n");

    // /* 写入数据 */
    // strcpy(start,"Buf Is Not Null!");
    
    // memset(buf, 0, 100);
    // strcpy(buf,start);
    // sleep (1);
    // printf("buf 2 = %s\n",buf);

       
    close(fd);  
    return 0;    
}
```

核心代码还是在内核态：

1：内核态申请内存，并使内存常驻。

```c
  // 内存分配
  buffer = (unsigned char *)kmalloc(32*PAGE_SIZE, GFP_KERNEL);
  // 将该段内存设置为保留
 ClearPageReserved(virt_to_page(buffer));
```


kmalloc 申请的是物理内存中的连续内存，最大128KB。`vmalloc`可以申请更大的，但是物理内存就不连续。

2：`mmap`的实现。

```c
static int my_map(struct file *filp, struct vm_area_struct *vma) {
  unsigned long page;
  unsigned char i;
  unsigned long start = (unsigned long)vma->vm_start;
  // unsigned long end =  (unsigned long)vma->vm_end;
  unsigned long size = (unsigned long)(vma->vm_end - vma->vm_start);
  printk(KERN_INFO"the size is %d\n",size);
  // 得到物理地址
  page = virt_to_phys(buffer);
  // 将用户空间的一个vma虚拟内存区映射到以page开始的一段连续物理页面上
  if (remap_pfn_range(
          vma, start, page >> PAGE_SHIFT, size,
          PAGE_SHARED)) // 第三个参数是页帧号，由物理地址右移PAGE_SHIFT得到
    return -1;

  // 往该内存写10字节数据
  for (i = 0; i < 10; i++)
    buffer[i] = array[i];

  return 0;
}
```

这里面的`vm_area_struct`其实对应的是用户态调用mmap的时候内核为该进程分配的一个虚拟内存区域。这个`size` 对应`vma->vm_end - vma->vm_start` 大小跟用户态内存映射函数设置的大小有关，同时跟4096 对齐，保证是整页。  这里面用户态设置的是内存映射是10,因此虚拟内存要对齐，所以是4096.可以通过查看对应的maps文件进行验证。

![](/assets/image/WEBRESOURCE626d506315e5fc4c413448d2e0c3d158image.png)

这个内存空间就是1个page。

如果申请5000个，那么就会变成两个page了。

![](/assets/image/WEBRESOURCEcddf6108cb73aa818af679aa145fc33cimage.png)

如果我申请10000个，也不会失败，但是实际操作就会有问题，因为我底层内核只申请了8192个字节。应该是内核做了一些措施，没有导致段错误，但是越界对实际内存操作应该是无效的。

`remap_pfn_range`函数最重要的是第三个参数，首先获得那块buffer的物理地址，然后右移12位就得到了对应物理页的页帧号。

如果要针对大内存的申请，就不连续了，就需要用`vmalloc_user`函数了。

`mmap`的函数也更加简单一些，输入地址就可以了：


```c
static int hnc_shm_driver_mmap(struct file *filp, struct vm_area_struct *vma)
{
	int ret;
	hnc_shmem_t *dev = filp->private_data;

	ret = remap_vmalloc_range(vma, dev->addr, vma->vm_pgoff);
	if (ret != 0) {
		printk(KERN_INFO "remap failed:%d", ret);
		return -1;
	}

	return 0;
}
```


## 参考链接

1. [ Linux内核空间内存申请函数kmalloc、kzalloc、vmalloc的区别](https://blog.csdn.net/lu_embedded/article/details/51588902)
2. [vmalloc与mmap](https://blog.csdn.net/juS3Ve/article/details/83629302)

