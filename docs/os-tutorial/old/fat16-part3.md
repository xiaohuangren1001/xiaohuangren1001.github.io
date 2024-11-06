本节是实现 FAT16 的最后一个小节，我们将实现一套可供用户使用的系统调用，包括 `open`、`read`、`write`、`unlink` 以及 `lseek`。有了这五个函数，我们就可以任意地进行文件读写以及对文件进行删除和创建。

本节代码极大程度上参照了《操作系统真象还原》以及 [PlantOS](http://github.com/min0911Y/Plant-OS) 的代码，因此在细节上可能有所欠缺，请见谅。

首先，如同多任务一样，直接把 `fileinfo_t` 当作一个文件的底层抽象是相当不合适的：由于 `fileinfo_t` 与硬件强相关，所以后期将难以扩展。

所以，最终，经过包装，我们创建了一个 `file_t` 结构体，用于表示一个抽象的文件的概念。

**代码 20-1 文件的底层抽象（include/file.h）**
```c
typedef enum FILE_TYPE {
    FT_USABLE,
    FT_REGULAR,
    FT_UNKNOWN
} file_type_t;

typedef enum oflags {
    O_RDONLY,
    O_WRONLY,
    O_RDWR,
    O_CREAT = 4
} oflags_t;

typedef struct FILE_STRUCT {
    void *handle;
    void *buffer;
    int pos;
    int size;
    int open_cnt;
    file_type_t type;
    oflags_t flags;
} file_t;
```

由于完全不打算实现目录，所以在 `file_type_t` 里，只有 `FT_REGULAR` 一种正常值，剩下两个一个用于判断文件是否已被占用，另一个则没什么用。

至于 `O_RDONLY` 这些，则是一个简单的判断读写的小机制，这样可以创建只读以及只写的文件，虽然这没什么用吧。

再往下的 `file_t` 可以说是十分灵活，我们甚至没有限制 `handle` 必须是 `fileinfo_t`，你往里面塞什么牛鬼蛇神都行，只要你能在后面的 `read` 这些地方圆回来。

这个 `buffer` 具体的用处到后面 `read` 和 `write` 时再讲。

下面的 `pos` 则是代表文件的读写位置，`lseek` 移动的就是它。

最后的 `open_cnt` 暂时没用，大概到了下一节或者最后一节才会有用。

下面是一些小小的改动：实际上，打开文件的操作是由任务执行的，所以至少在任务的层面，应该对文件给予一些支持。

**代码 20-2 新版 `task_t` 结构体（include/mtask.h）**
```c
#define MAX_FILE_OPEN_PER_TASK 32

typedef struct TASK {
    uint32_t sel;
    int32_t flags;
    exit_retval_t my_retval;
    int fd_table[MAX_FILE_OPEN_PER_TASK]; // here
    tss32_t tss;
} task_t;
```

与 Linux 0.01 不同的是，这里并未直接存储 `file_t`，而是存储了一个 `int`，它代表对应的 `file_t` 在一个文件表中的索引。这么做的目的是节省空间，以及可能带来的更高氨醛性——毕竟理论上可以顺着 `malloc`（虽然还没实现）找到 `task_t` 来给文件一锅端了（确信）。

这个数组需要在 `task_alloc` 时进行初始化：

**代码 20-3 初始化任务中的文件描述符表（kernel/mtask.c）**
```c
            task->fd_table[0] = 0; // 标准输入，占位
            task->fd_table[1] = 1; // 标准输出，占位
            task->fd_table[2] = 2; // 标准错误，占位
            for (int i = 3; i < MAX_FILE_OPEN_PER_TASK; i++) {
                task->fd_table[i] = -1; // 其余文件均可用
            }
```

然后，在 `fs/file.c` 中，我们来添加这个文件表：

**代码 20-4 创建文件表（fs/file.c）**
```c
#include "file.h"
#include "mtask.h"
#include "memory.h"

static file_t file_table[MAX_FILE_NUM];
```

然后我们需要提供一个/一组把 `fileinfo_t` 转换为 `file_t` 并安装到当前任务当中的函数，这由下面的 `install_to_global` 和 `install_to_local` 实现：

**代码 20-5 将 `fileinfo_t` 安装到任务（fs/file.c）**
```c
static int install_to_global(fileinfo_t finfo)
{
    int i = MAX_FILE_NUM;
    for (i = 0; i < MAX_FILE_NUM; i++) {
        if (file_table[i].type == FT_USABLE) break; // 当前文件空闲，则占用
    }
    if (i == MAX_FILE_NUM) return -1; // 没有文件空闲，则退出
    fileinfo_t *safer_finfo = (fileinfo_t *) kmalloc(sizeof(fileinfo_t)); // 分配一个finfo指针，准备挂到handle上
    if (!safer_finfo) return -1;
    *safer_finfo = finfo; // 装入
    file_table[i].handle = safer_finfo; // 这就是其内部的handle
    file_table[i].type = FT_REGULAR; // 类型为正常文件
    file_table[i].pos = 0; // 由于刚刚注册，pos设为0
    return i; // 返回其在文件表内的索引
}

static int install_to_local(int global_fd)
{
    task_t *task = task_now(); // 获取当前任务
    int i;
    for (i = 3; i < MAX_FILE_OPEN_PER_TASK; i++) { // fd 0 1 2分别代表标准输入 标准输出 标准错误，所以从3开始找起
        if (task->fd_table[i] == -1) break; // 这里还空着，直接用
    }
    if (i == MAX_FILE_OPEN_PER_TASK) return -1; // 到达任务可打开的文件上限，返回-1
    task->fd_table[i] = global_fd; // 将文件表索引安装到任务的文件描述符表
    return i; // 返回索引，这就是对应的文件描述符了
}
```

有了这两个函数，实现打开文件和创建文件就畅通无阻了。在 Linux 系统中，这两个功能被整合进 `open` 这一个函数中，这是一个我也说不上好不好的设计，总之我决定模仿。

**代码 20-6 `open`：打开文件、创建文件（fs/file.c）**
```c
int sys_open(char *filename, uint32_t flags)
{
    fileinfo_t finfo; // 准备接收打开的文件
    if (flags & O_CREAT) { // flags中含有O_CREAT，则需要创建文件
        int status = fat16_create_file(&finfo, filename); // 调用创建文件的函数
        if (status == -1) return status; // 创建失败则直接不管
    } else {
        int status = fat16_open_file(&finfo, filename); // 调用打开文件的函数
        if (status == -1) return status; // 打开失败则直接不管
    }
    int global_fd = install_to_global(finfo); // 先安装到全局文件表
    file_table[global_fd].open_cnt++; // open个数+1，没什么用
    file_table[global_fd].size = finfo.size; // 设置文件大小
    file_table[global_fd].flags = flags | (~O_CREAT); // flags中剔除O_CREAT
    file_table[global_fd].buffer = kmalloc(finfo.size + 5); // 分配一个缓冲区
    if (finfo.size) { // 如果有内容
        int status = fat16_read_file(&finfo, file_table[global_fd].buffer); // 则直接读到缓冲区里来
        if (status == -1) { // 如果读不进缓冲区，那就只好这样了
            kfree(file_table[global_fd].handle); // 释放占有的资源
            kfree(file_table[global_fd].buffer);
            return status;
        }
    }
    return install_to_local(global_fd); // 最后安装到任务里
}
```

这个缓冲区很快就会在下面的 `read` 和 `write` 中用到，由于这两者操作逻辑较为类似，合并到同一个代码块里来：

**代码 20-7 `read`、`write`：读写文件（fs/fat16.c）**
```c
int sys_write(int fd, const void *msg, int len)
{
    if (fd <= 0) return -1; // 是无效fd，返回
    if (fd == 1 || fd == 2) { // 往标准输出或标准错误中输出
        char *s = (char *) msg; // 转换为char *
        for (int i = 0; i < len; i++) monitor_put(s[i]); // 直接用monitor_put逐字符输出
        return len; // 一切正常
    }
    task_t *task = task_now(); // 获取当前任务
    int global_fd = task->fd_table[fd]; // 获取文件表中索引
    file_t *cfile = &file_table[global_fd]; // 获取文件表中的文件指针
    if (cfile->flags == O_RDONLY) return -1; // 只读，不可写，返回
    for (int i = 0; i < len; i++) { // 对于每一个字节
        if (cfile->pos >= cfile->size) { // 如果超出了原本的范围
            cfile->size++; // 大小+1
            void *new_buffer = krealloc(cfile->buffer, cfile->size); // 使用krealloc扩容，增长缓冲区大小
            if (new_buffer) cfile->buffer = new_buffer; // 如果缓冲区分配成功，那么这里就是新的缓冲区
        }
        char *buf = (char *) cfile->buffer; // 文件的缓冲区，相当于文件的当前内容了
        char *content = (char *) msg; // 要写入的内容
        buf[cfile->pos] = content[i]; // 向读写指针处写入当前内容
        cfile->pos++; // 文件指针后移
    }
    int status = fat16_write_file(cfile->handle, cfile->buffer, cfile->size); // 写入完毕，立刻更新到硬盘
    if (status == -1) return status; // 写入失败，返回
    return len; // 否则，返回实际写入的长度len
}

int sys_read(int fd, void *buf, int count)
{
    int ret = -1;
    if (fd < 0 || fd == 1 || fd == 2) return ret; // 从标准输入/标准错误中读或是fd非法都是不允许的
    if (fd == 0) { // 如果是标准输入
        char *buffer = (char *) buf; // 先转成char *
        uint32_t bytes_read = 0; // 读了多少个
        while (bytes_read < count) { // 没达到count个
            while (fifo_status(&decoded_key) == 0); // 只要没有新的键我就不读进来
            *buffer = fifo_get(&decoded_key); // 获取新的键
            bytes_read++;
            buffer++; // buffer指向下一个
        }
        ret = (bytes_read == 0 ? -1 : (int) bytes_read); // 如果啥也没读着就-1，否则就正常返回就行了
        return ret;
    }
    task_t *task = task_now(); // 获取当前任务
    int global_fd = task->fd_table[fd]; // 获取fd对应的文件表索引
    file_t *cfile = &file_table[global_fd]; // 获取文件表中对应文件
    if (cfile->flags == O_WRONLY) return -1; // 只写，不可读，返回-1
    ret = 0; // 记录到底读了多少个字节
    for (int i = 0; i < count; i++) {
        if (cfile->pos >= cfile->size) break; // 如果已经到达末尾，返回
        char *filebuf = (char *) cfile->buffer; // 文件缓冲区
        char *retbuf = (char *) buf; // 接收缓冲区
        retbuf[i] = filebuf[cfile->pos]; // 逐字节拷贝内容
        cfile->pos++; // 读写指针后移
        ret++; // 读取字节数+1
    }
    return ret; // 返回读取字节数
}
```

这里从 `sys_write` 的实现中就可以知道 `buffer` 的作用了：在前面读写文件时，我们只实现了覆盖整个文件，想要对文件的特定位置进行修改，只能全部读进来，在软件层面修改后再全部写回去。而且，这样的实现还可以加快 `sys_read` 的速度，也算是一种奇妙的优化吧。

同时，改完这里以后，原先 `kernel/syscall.c` 中的 `sys_read` 和 `sys_write` 就可以删除了。

扩容用的 `krealloc` 写在了 `kernel/memory.c` 中：

**代码 20-8 `krealloc`（kernel/memory.c）**
```c
void *krealloc(void *buffer, int size)
{
    void *res = NULL;
    if (!buffer) return kmalloc(size); // buffer为NULL，则realloc相当于malloc
    if (!size) { // size为NULL，则realloc相当于free
        kfree(buffer);
        return NULL;
    }
    // 否则实现扩容
    res = kmalloc(size); // 分配新的缓冲区
    memcpy(res, buffer, size); // 将原缓冲区内容复制过去
    kfree(buffer); // 释放原缓冲区
    return res; // 返回新缓冲区
}
```

接下来是关闭文件用的 `sys_close`，基本上就是对文件使用资源的释放。

**代码 20-9 `close`：关闭文件（fs/file.c）**
```c
int sys_close(int fd)
{
    int ret = -1; // 返回值
    if (fd > 2) { // 的确是被打开的文件
        task_t *task = task_now(); // 获取当前任务
        uint32_t global_fd = task->fd_table[fd]; // 获取对应文件表索引
        task->fd_table[fd] = -1; // 释放文件描述符
        file_t *cfile = &file_table[global_fd]; // 获取对应文件
        kfree(cfile->buffer); // 释放缓冲区
        kfree(cfile->handle); // install_to_global中使用kmalloc分配fileinfo指针
        cfile->type = FT_USABLE; // 设置type为可用
        return 0; // 关闭完成
    }
    return ret; // 否则返回-1
}
```

移动读写指针用的 `sys_lseek` 纯属软件操作，`sys_unlink` 则只是 `fat16_delete_file` 套皮，这里一并放上来。

**代码 20-10 `lseek`、`unlink`：最后一个部分（fs/file.c）**
```c
int sys_lseek(int fd, int offset, uint8_t whence)
{
    if (fd < 3) return -1; // 不是被打开的文件，返回
    if (whence < 1 || whence > 3) return -1; // whence只能为123，分别对应SET、CUR、END，返回
    task_t *task = task_now(); // 获取当前任务
    file_t *cfile = &file_table[task->fd_table[fd]]; // 获取fd对应的文件
    fileinfo_t *fhandle = (fileinfo_t *) cfile->handle; // 文件实际上对应的fileinfo
    int size = fhandle->size; // 获取大小，总归是有用的
    int new_pos = 0; // 新的文件位置
    switch (whence) {
        case SEEK_SET: // SEEK_SET就是纯设置
            new_pos = offset; // 直接设置
            break;
        case SEEK_CUR: // 从当前位置算起移动offset位置
            new_pos = cfile->pos + offset; // 用当前pos加上offset
            break;
        case SEEK_END: // 从结束位置算起移动offset位置
            new_pos = size + offset; // 用大小加上offset
            break;
    }
    if (new_pos < 0 || new_pos > size - 1) return -1; // 如果新的位置超出文件，返回-1
    cfile->pos = new_pos; // 设置新位置
    return new_pos; // 返回新位置
}

int sys_unlink(const char *filename)
{
    return fat16_delete_file((char *) filename); // 直接套皮，不多说
}
```

好，那么到此为止，历时四节，我们的 FAT16 文件系统实现的征程到此结束！鼓掌！

或许有人会觉得：这看上去也不难嘛……那不妨自己查询资料写一个试试哦（

再往下三节，我们来实现应用程序的执行，彻底结束这个破烂不堪的操作系统教程。
