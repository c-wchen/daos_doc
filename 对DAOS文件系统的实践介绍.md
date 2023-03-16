## 介绍

DAOS允许通过DFS库（libdfs)来访问存储在DAOS容器中POSIX命名空间中的数据，如果应用程序不想使用libdfs API访问DAOS容器的话，也可以使用FUSE（dfuse)守护进程公开透明的访问libdfs

请参阅[daos.io上的POSIX命名空间文档](https://docs.daos.io/user/posix/)。



 

![three-paths-to-use-libdfs](https://www.intel.cn/content/dam/develop/external/us/en/images/three-paths-to-use-libdfs.png)应用程序访问libdfs的不同方式( (image source: https://docs.daos.io/graph/posix.png).

 

本文展示应用程序如何访问DAOS容器的三种路径（dfuse、dfuse + libioil、libdfs)来访问DAOS POSIX命名空间数据，对于libdfs(对应于直接路径)，将展示使其工作所需的源代码修改，以及与之等价的传统POSIX。

## 创建容器

使用前需要先创建*dfs_pool*和它的容器*dfs_cont*，运行以下命令:

```bash
$ dmg pool create --label=dfs_pool --size=8G
Creating DAOS pool with automatic storage allocation: 8.0 GB total, 6,94 tier ratio
Pool created with 100.00%,0.00% storage tier ratio
--------------------------------------------------
  UUID                 : b405ad74-5136-4325-a549-eea40d76fc66
  Service Ranks        : 0
  Storage Ranks        : 0
  Total Size           : 8.0 GB
  Storage tier 0 (SCM) : 8.0 GB (8.0 GB / rank)
  Storage tier 1 (NVMe): 0 B (0 B / rank)
$ daos container create --type=POSIX --pool=dfs_pool --label=dfs_cont
  Container UUID : 51fe3022-6b32-47a9-8485-1fe669deafe1
  Container Label: dfs_cont
  Container Type : POSIX

Successfully created container 51fe3022-6b32-47a9-8485-1fe669deafe1
```

pool的大小为8GiB，container的类型设置为POSIX。该类型用于底层DAOS对象模型指定预定义的数据布局。例如，类型有可以是HDF5和PYTHON。

如果运行容器检查，应该在新创建的POSIX容器中看到两个对象:

```bash
$ daos container check --pool=dfs_pool --cont=dfs_cont
check container dfs_cont started at: Wed Nov 10 05:49:47 2021

check container dfs_cont completed at: Wed Nov 10 05:49:47 2021

checked: 2
skipped: 0
inconsistent: 0
run_time: 1 seconds
scan_speed: 2 objs/sec
```

一个对象对应于保存元数据的**POSIX容器超级块**，另一个是命名空间中的根目录(即"/")。有关更多信息，请参阅[DAOS github中的DFS自述文件](https://github.com/daos-stack/daos/blob/master/src/client/dfs/README.md)。

## Using the DFS Container with Dfuse

[FUSE(用户空间文件系统)](https://www.kernel.org/doc/html/latest/filesystems/fuse.html)是一个由普通用户空间进程提供的文件系统，可以通过内核接口访问。FUSE有很多应用，例如：它可用于挂载加密文件系统(参见[EncFS](https://wiki.archlinux.org/title/EncFS))、远程文件系统(参见[sshfs](https://github.com/libfuse/sshfs)或[s3fs](https://github.com/s3fs-fuse/s3fs-fuse))或特定设备(如iphone和ipod)(参见[iFuse](https://libimobiledevice.org/))。你可以在[arch Linux wiki page for FUSE](https://wiki.archlinux.org/title/FUSE)以及[Wikipedia page for FUSE](https://en.wikipedia.org/wiki/Filesystem_in_Userspace#Example_uses)中查看项目的全面列表。

对于dfuse而言，提供给内核的是一个由DAOS容器支持的远程文件系统。

使用dfuse的第一步是为POSIX命名空间创建挂载点。创建目录/opt/dfs:

```bash
$ mkdir /opt/dfs
```

现在我们可以运行dfuse。如果想在前台运行查看任何错误日志，可以传递选项```--foreground```(如果在前台运行，则打开一个新的终端):

```bash
$ dfuse --pool=dfs_pool --container=dfs_cont --foreground --mountpoint=/opt/dfs
```

要检查dfuse是否已挂载，请运行:

```bash
$ df -h | grep dfuse
dfuse                         7.5G  2.2M  7.5G   1% /opt/dfs
```

 

**注意:**在dfuse中缓存是默认启用的。这可能导致并行应用程序失败，因为并行缓存之间的数据没有同步。可以使用选项--disable-caching在dfuse中禁用缓存。有关更多信息，请参阅[DAOS文件系统文档的缓存部分](https://docs.daos.io/v2.0/user/filesystem/#caching)。

创建一个文件，向其中写入一些内容(然后再读取):

```bash
$ echo "Hello World" > /opt/dfs/hello.txt
$ cat /opt/dfs/hello.txt
Hello World
$
```

运行容器检查来验证我们现在有3个DAOS对象，其中额外的对象是我们刚刚创建的文件:

```bash
$ daos container check --pool=dfs_pool --cont=dfs_cont
check container dfs_cont started at: Wed Nov 10 05:56:43 2021

check container dfs_cont completed at: Wed Nov 10 05:56:43 2021

checked: 3
skipped: 0
inconsistent: 0
run_time: 1 seconds
scan_speed: 3 objs/sec
```

完成后，dfuse可以通过fusermount卸载:

```bash
$ fusermount3 -u /opt/dfs
```

## Dfuse+Libioil

为了避免一些FUSE性能瓶颈，我们可以使用libioil拦截应用程序中的所有读/写(和lseek)系统调用，以绕过操作系统。

为了演示如何操作，创建了以下C程序:

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define CHECK_ERROR(ret, funcname)                                      \
        do {                                                            \
                if (ret < 0)                                            \
                {                                                       \
                        printf("Error on %s(): %s\n",   funcname,       \
                                                strerror(errno));       \
                        return -1;                                      \
                }                                                       \
        } while (0)

int main (int argc, char *argv[])
{
        int             fd;
        int             ret;
        off_t           offset;
        ssize_t         bytes;
        char            buf[128];

        if (argc < 2)
        {
                printf("USE: %s <file-path>\n", argv[0]);
                return -1;
        }
        fd = open(argv[1], O_RDWR | O_CREAT | O_TRUNC, S_IRWXU);
        CHECK_ERROR(fd, "open");

        bytes = write(fd, "Hello World", strlen("Hello World"));
        CHECK_ERROR(bytes, "write");

        offset = lseek(fd, 0, SEEK_SET);
        CHECK_ERROR(offset, "lseek");

        bytes = read(fd, buf, 128);
        CHECK_ERROR(bytes, "read");
        printf("data read = %.*s\n", bytes, buf);

        ret = close(fd);
        CHECK_ERROR(ret, "close");

        return 0;
}
```

如你所见，代码很简单。该程序首先打开作为参数传入的文件(如果它不存在，则创建它;如果有，则将其截断)。然后向它写入Hello World。之后，程序寻找描述符定位0(文件的开头)，读取文件的内容并打印出来，然后关闭文件。

在使用拦截库之前，我们运行perf以可视化系统调用。Perf允许我们捕获大量预定义事件:

```bash
# perf record -e 'syscalls:*open' -e 'syscalls:*write' -e 'syscalls:*lseek' -e 'syscalls:*read' -e 'syscalls:*close' ./write_and_read /opt/dfs/hello.txt
data read = Hello World
[ perf record: Woken up 28 times to write data ]
[ perf record: Captured and wrote 0.028 MB perf.data (22 samples) ]
# perf script
...
  write_and_read 45849 [056] 13442525.391293:            syscalls:sys_enter_open: filename: 0x7ffe7f0bd535, flags: 0x00000242, mode: 0x000001c0
  write_and_read 45849 [056] 13442525.393287:             syscalls:sys_exit_open: 0x3
  write_and_read 45849 [056] 13442525.393289:           syscalls:sys_enter_write: fd: 0x00000003, buf: 0x004009ca, count: 0x0000000b
  write_and_read 45849 [056] 13442525.393832:            syscalls:sys_exit_write: 0xb
  write_and_read 45849 [056] 13442525.393834:           syscalls:sys_enter_lseek: fd: 0x00000003, offset: 0x00000000, whence: 0x00000000
  write_and_read 45849 [056] 13442525.393834:            syscalls:sys_exit_lseek: 0x0
  write_and_read 45849 [056] 13442525.393836:            syscalls:sys_enter_read: fd: 0x00000003, buf: 0x7ffe7f0bbf10, count: 0x00000080
  write_and_read 45849 [056] 13442525.393837:             syscalls:sys_exit_read: 0xb
  write_and_read 45849 [056] 13442525.393885:           syscalls:sys_enter_write: fd: 0x00000001, buf: 0x7f720d41f000, count: 0x00000018
  write_and_read 45849 [056] 13442525.393916:            syscalls:sys_exit_write: 0x18
  write_and_read 45849 [056] 13442525.393919:           syscalls:sys_enter_close: fd: 0x00000003
  write_and_read 45849 [056] 13442525.393920:            syscalls:sys_exit_close: 0x0
```

我们可以看到，代码中发出的所有系统调用(open()、write()、lseek()、read()和close())都确实调用了系统。注意，close()前面有一个额外的write()，这与printf()对应，在printf()中，写入描述符1(标准输出)而不是3。

现在让我们添加拦截库。我们要做的就是把通往bioil的路径加进去。因此，将文件写入LD_PRELOAD环境变量:

```bash
$ export LD_PRELOAD=<DAOS_INSTALLATION_DIR>/lib64/libioil.so:$LD_PRELOAD
```

如果再次运行perf，会得到以下结果:

```bash
# perf record -e 'syscalls:*open' -e 'syscalls:*write' -e 'syscalls:*lseek' -e 'syscalls:*read' -e 'syscalls:*close' ./write_and_read /opt/dfs/hello.txt
data read = Hello World
[ perf record: Woken up 25 times to write data ]
[ perf record: Captured and wrote 1.154 MB perf.data (12292 samples) ]
# perf script
...
  write_and_read 45944 [054] 13442652.584153:            syscalls:sys_enter_open: filename: 0x7ffc4d31650f, flags: 0x00000242, mode: 0x000001c0
  write_and_read 45944 [054] 13442652.586117:             syscalls:sys_exit_open: 0x3
...
  write_and_read 45944 [019] 13442652.720267:           syscalls:sys_enter_write: fd: 0x00000001, buf: 0x7f986881c000, count: 0x00000018
  write_and_read 45944 [019] 13442652.720298:            syscalls:sys_exit_write: 0x18
  write_and_read 45944 [019] 13442652.720317:           syscalls:sys_enter_close: fd: 0x00000003
  write_and_read 45944 [019] 13442652.720318:            syscalls:sys_exit_close: 0x0
...
```

这一次，描述符3的open()和close()之间没有事件了。因此，拦截库有效地将它们重定向到libdfs。

还可以通过导出D_IL_REPORT环境变量来检查拦截库是否正常工作。例如，在不拦截的情况下，我们有:

```bash
$ D_IL_REPORT=-1 ./write_and_read /opt/dfs/hello.txt 
data read = Hello World
```

拦截:

```bash
$ D_IL_REPORT=-1 LD_PRELOAD=<DAOS_INSTALLATION_DIR>/lib64/libioil.so ./write_and_read /opt/dfs/hello.txt  
[libioil] Intercepting write of size 11
[libioil] Intercepting read of size 128
data read = Hello World
[libioil] Performed 1 reads and 1 writes from 1 files
```

有关更多信息，请参阅[DAOS文件系统文档的监视活动部分](https://docs.daos.io/v2.0/user/filesystem/#monitoring-activity)。

## Libdfs

在本节中，我们将转换上面显示的简单C代码，使其直接使用libdfs提供的API。注意，这个示例是有限的，没有涵盖整个API。如果您想了解更多信息，请参阅[daos_fs.h](https://github.com/daos-stack/daos/blob/master/src/include/daos_fs.h)。

```c
#include <fcntl.h>
#include <daos.h>
#include <daos_fs.h>
#include <sys/stat.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define CHECK_ERROR(ret, funcname)                                      \
        do {                                                            \
                if (ret < 0)                                            \
                {                                                       \
                        printf("Error on %s(): %s\n",   funcname,       \
                                                strerror(errno));       \
                        return -1;                                      \
                }                                                       \
        } while (0)
#define CHECK_ERROR_DAOS(ret, funcname)                                 \
        do {                                                            \
                if (ret < 0)                                            \
                {                                                       \
                        printf("Error on %s(): %d\n",   funcname, ret); \
                        return -1;                                      \
                }                                                       \
        } while (0)

int main (int argc, char *argv[])
{
        daos_handle_t   poh;
        daos_handle_t   coh;
        dfs_t           *dfs;
        dfs_obj_t       *file_obj;
        d_sg_list_t     sgl_data;
        d_iov_t         iov_buf;
        int             ret;
        char            buf_in[128];
        char            buf_out[128];
        daos_size_t     read_size;

        if (argc < 4)
        {
                printf("USE: %s <pool_label> <container_label> <file_name>\n",
                       argv[0]);
                return -1;
        }
        /** init phase */
        ret = daos_init();
        CHECK_ERROR_DAOS(ret, "daos_init");
        ret = daos_pool_connect(argv[1], NULL, DAOS_PC_RW, &poh, NULL, NULL);
        CHECK_ERROR_DAOS(ret, "daos_pool_connect");
        ret = daos_cont_open(poh, argv[2], DAOS_COO_RW, &coh, NULL, NULL);
        CHECK_ERROR_DAOS(ret, "daos_cont_open");
        ret = dfs_mount(poh, coh, O_RDWR, &dfs);
        CHECK_ERROR(ret, "dfs_mount");

        /** equivalent to open() */
        ret = dfs_open(dfs, NULL, argv[3], S_IRWXU | S_IFREG,
                        O_RDWR | O_CREAT | O_TRUNC, 0, 0, NULL, &file_obj);
        CHECK_ERROR(ret, "dfs_open");

        /** equivalent to write() */
        sgl_data.sg_nr          = 1;
        sgl_data.sg_nr_out	= 0;
        sgl_data.sg_iovs        = &iov_buf;
        strcpy(buf_in, "Hello World");
        d_iov_set(&iov_buf, buf_in, strlen(buf_in));
        ret = dfs_write(dfs, file_obj, &sgl_data, 0, NULL);
        CHECK_ERROR(ret, "dfs_write");

        /** equivalent to read() */
        d_iov_set(&iov_buf, buf_out, 128);
        ret = dfs_read(dfs, file_obj, &sgl_data, 0, &read_size, NULL);
        CHECK_ERROR(ret, "dfs_read");
        printf("data read = %.*s\n", read_size, buf_out);

        /** equivalent to close() */
        ret = dfs_release(file_obj);
        CHECK_ERROR(ret, "dfs_release");

        /** finish phase */
        ret = dfs_umount(dfs);
        CHECK_ERROR(ret, "dfs_close");
        ret = daos_cont_close(coh, NULL);
        CHECK_ERROR_DAOS(ret, "daos_cont_close");
        ret = daos_pool_disconnect(poh, NULL);
        CHECK_ERROR_DAOS(ret, "daos_pool_disconnect");
        ret = daos_fini();
        CHECK_ERROR_DAOS(ret, "daos_fini");

        return 0;
}
```

在代码本身中，添加了注释，以显示新代码的哪些部分对应于旧代码。还有两个新部分init和finish。让我们一步一步来:

**Init**

```c
ret = daos_init();
CHECK_ERROR_DAOS(ret, "daos_init");
ret = daos_pool_connect(argv[1], NULL, DAOS_PC_RW, &poh, NULL, NULL);
CHECK_ERROR_DAOS(ret, "daos_pool_connect");
ret = daos_cont_open(poh, argv[2], DAOS_COO_RW, &coh, NULL, NULL);
CHECK_ERROR_DAOS(ret, "daos_cont_open");
ret = dfs_mount(poh, coh, O_RDWR, &dfs);
CHECK_ERROR(ret, "dfs_mount");
```

每次程序运行只需要执行一次init阶段。它由4个函数调用组成:

- daos_init(): 初始化daos客户端库。
- daos_pool_connect(): 连接pool
- daos_cont_open(): 打开池内的容器。我们在这里假设容器存在。如果没有，也可以调用daos_cont_create()在这里创建它。
- dfs_mount(): 创建一个处理程序，允许对容器进行文件和目录操作。

**Open**

```c
ret = dfs_open(dfs, NULL, argv[3], S_IRWXU | S_IFREG,
                O_RDWR | O_CREAT | O_TRUNC, 0, 0, NULL, &file_obj);
CHECK_ERROR(ret, "dfs_open");
```

函数open()被dfs_open()取代，用于打开文件(如果文件不存在，则创建文件)。

**Write**

```c
sgl_data.sg_nr          = 1;
sgl_data.sg_nr_out      = 0;
sgl_data.sg_iovs        = &iov_buf;
strcpy(buf_in, "Hello World");
d_iov_set(&iov_buf, buf_in, strlen(buf_in));
ret = dfs_write(dfs, file_obj, &sgl_data, 0, NULL);
CHECK_ERROR(ret, "dfs_write");
```

对write()的调用被对dfs_write()的调用取代。对于dfs_write()，我们不能传递一个常量字符串作为输入。输入必须作为缓冲区传递到d_sg_list_t数据结构(sgl_data)中。为此，需要调用d_iov_set()将原始输入缓冲区设置为d_iov_t数据结构(iov_buf)，然后在d_sg_list_t结构中使用该数据结构。

**Read**

```c
d_iov_set(&iov_buf, buf_out, 128);
ret = dfs_read(dfs, file_obj, &sgl_data, 0, &read_size, NULL);
CHECK_ERROR(ret, "dfs_read");
printf("data read = %.*s\n", read_size, buf_out);
```

对read()的调用被对dfs_read()的调用取代。与dfs_write()一样，我们必须将缓冲区作为d_sg_list_t变量传递。在这种情况下，我们改变了赋值给iov_buf变量的指针(buf_out)。由于iov_buf已经分配给sgl_data，我们可以将其传递给dfs_read()。**Close**

```c
ret = dfs_release(file_obj);
CHECK_ERROR(ret, "dfs_release");
```

函数close()被dfs_release()取代。

**Finish**

```c
ret = dfs_umount(dfs);
CHECK_ERROR(ret, "dfs_close");
ret = daos_cont_close(coh, NULL);
CHECK_ERROR_DAOS(ret, "daos_cont_close");
ret = daos_pool_disconnect(poh, NULL);
CHECK_ERROR_DAOS(ret, "daos_pool_disconnect");
ret = daos_fini();
CHECK_ERROR_DAOS(ret, "daos_fini");
```

最后，我们执行清理代码以关闭打开的处理程序并调用daos_fini()。这个代码段只需要在程序末尾执行一次。

## 总结

在本文中，展示了如何使用应用程序可用的三个代码路径（dfuse、dfuse+libioil和libdfs）来访问存储在DAOS容器中的POSIX命名空间中的数据。以libdfs(对应于直接路径)为例，给出了使其工作所需的源代码修改，以及与之等价的传统POSIX。

## 原文参考

[A Hands-on Introduction to the DAOS File System Library (intel.cn)](https://www.intel.cn/content/www/cn/zh/developer/articles/training/introduction-to-dfs.html)