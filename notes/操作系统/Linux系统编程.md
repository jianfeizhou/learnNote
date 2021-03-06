# 一、I/O 操作

## 1. 文件 I/O 

### 1. 文件操作

- **文件表**： 内核会替每个进程维护一份已打开文件列表

  > - 文件描述符： 非负整数，作为文件表的索引，打开一个文件就会返回一个文件描述符，而随后的读写操作会以“文件描述符”为主要参数

- `open()`： 用来将路径名称 name 所指定的文件映射至一个文件描述符，并于映射成功后返回该文件描述符

  > ```c
  > #include<sys/types.h>
  > #include<sys/stat.h>
  > #include<fcntl.h>
  > 
  > int open(const char *name,int flags);
  > int open(const char *name,int flags,mode_t mode); //mode 在创建新文件时使用
  > ```
  >
  > `flags` 参数： 
  >
  > - 操作文件方式： 
  >
  >   |    标志    |      用途      |
  >   | :--------: | :------------: |
  >   | `O_RDONLY` | 以只读方式打开 |
  >   | `O_WRONLY` | 以只写方式打开 |
  >   |  `O_RDWR`  | 以读写方式打开 |
  >
  > - 附加方式： 以 OR 方式连接
  >
  >   | 标志          | 用途                                              |
  >   | ------------- | ------------------------------------------------- |
  >   | `O_APPEND`    | 在文件尾部追加数据                                |
  >   | `O_ASYNC`     | 当I/O操作可行时，产生信号(signal)通知进程         |
  >   | `O_CREAT`     | 若文件不存在则创建                                |
  >   | `O_DIRECT`    | 无缓存的输入/输出                                 |
  >   | `O_DIRECTORY` | 如果pathname不是目录，则失败                      |
  >   | `O_EXCL`      | 结合O_CREAT参数使用，创建文件时，若文件存在则失败 |
  >   | `O_LARGEFILE` | 在32位系统中使用此标志打开大文件                  |
  >   | `O_NOCTTY`    | 不要让 pathname 成为控制终端                      |
  >   | `O_NOFOLLOW`  | 不自动对符号链接解引用                            |
  >   | `O_NONBLOCK`  | 以非阻塞方式打开                                  |
  >   | `O_SYNC`      | 以同步方式写入文件                                |
  >   | `O_TRUNC`     | 截断已有文件,使其长度为零                         |
  >   | `O_CLOEXEC`   | 设置 close-on-exec 标志                           |
  >   | `O_NOATIME`   | 调用 read() 时，不修改文件的最近访问时间          |
  >   | `O_DSYNC`     | 提供同步的 I/O 数据完整性                         |
  >
  > - 例子： 
  >
  >   ```c
  >   int fd = open("/xx/xx",O_WRONLY | O_TRUNC)
  >   ```
  >
  > **新文件**： 
  >
  > - 新文件的拥有者： 
  >
  >   - 判断拥有用户： 文件拥有者的 uid 就是创建文件的进程的有效 uid
  >
  >   - 判断拥有组： 将文件的拥有组设定为创建文件的进程的有效 gid
  >
  >     > BSD 方式： 文件的拥有组会被设定为上层目录的 gid
  >
  > - 新文件的使用权限： 参数 `mode` 用来为新创建的文件设定使用权限
  >
  >   | 示例      | 作用                               |
  >   | --------- | ---------------------------------- |
  >   | `S_IRWXU` | 拥有者具有读取、写入、执行权限     |
  >   | `S_IRUSR` | 拥有者具有读取权限                 |
  >   | `S_IWUSR` | 拥有者具有写入权限                 |
  >   | `S_IXUSR` | 拥有者具有执行权限                 |
  >   | `S_IRWXG` | 组具有读取、写入、执行权限         |
  >   | `S_IRGRP` | 组具有读取权限                     |
  >   | `S_IWGRP` | 组具有写入权限                     |
  >   | `S_IXGRP` | 组具有执行权限                     |
  >   | `S_IRWXO` | 其他所有人具有读取、写入、执行权限 |
  >   | `S_IROTH` | 其他所有人具有读取权限             |
  >   | `S_IWOTH` | 其他所有人具有写入权限             |
  >   | `S_IXOTH` | 其他所有人具有执行权限             |

- `read()`： 调用会从 `fd` 参数所引用文件的当前文件位置读取 `len` 个字节到 `buf`

  > 执行成功后会返回写入 `buf` 的字节数目，执行失败会返回 `-1` 并设定 `errno` 
  >
  > ```c
  > #include<unistd.h>
  > ssize_t read(int fd,void *buf,size_t len);
  > ```
  >
  > - 读取成功后，文件位置会前进到从 `fd` 所读取的字节数目
  >
  >   > 如果 `fd` 所代表的对象不具有查找位置的能力，则每次都只从“当前”位置读取数据
  >
  > - 若没有可供读取的 `fd` 个字节，则调用会阻塞(休眠状态)，直到出现可供读取的字节
  >
  > 返回错误类型：
  >
  > - `EINTR`： 表示在任何字节被读取之前收到了一个信号，可以再次进行此调用
  > - `EAGAIN`： 表示读取操作被阻挡
  > - 其他： 严重错误
  >   - `EBADF`： 所指定的文件描述符无效或未打开以备读取
  >   - `EFAULT`： buf 所提供的指针并非指向进行调用进程的地址空间范围内
  >   - `EINVAL`： 文件描述符被映射到一个不允许读取操作的对象
  >   - `EIO`： 发生了一个低级的 I/O 错误

- `write()`： 会从 `buf` 开始将 `count` 个字节写入 `fd` 所指定文件的当前文件位置

  > ```c
  > #include<unistd.h>
  > ssize_t write(int fd,const void *buf,size_t count);
  > ```
  >
  > 错误代码： 
  >
  > | 错误码   | 作用                                                         |
  > | -------- | ------------------------------------------------------------ |
  > | `EBADF`  | 所指定的文件描述符无效或未打开以备写入                       |
  > | `EFAULT` | `buf` 所提供的指针并非指向进行调用进程的地址空间范围内       |
  > | `EFBIG`  | 写入操作所产生的文件大小超过了每个进程文件大小上限或内部所实施的限制 |
  > | `EINVAL` | 所指定的文件描述符被映射到一个不允许写入操作的对象           |
  > | `EIO`    | 发生了一个低级的 `I/O` 错误                                  |
  > | `ENOSPC` | 文件系统于背后所指定的文件描述符没有足够的空间可用           |
  > | `EPIPE`  | 所指定的文件描述符关联到读取端已经关闭的 pipe 或 socket      |

- `close()`： 可以取消已打开文件描述符 `fd` 的映射关系，以及让进程与相关联文件分离

  > ```c
  > #include<unistd.h>
  > int close(int fd);
  > ```

- `lseek()`： 可将一个文件描述符的文件位置设定为指定的值

  > ```c
  > #include<sys/types.h>
  > #include<unistd.h>
  > off_t lseek(int fd,off_t pos,int origin);
  > ```
  >
  > `lseek()` 行为取决于 `origin` 参数： 
  >
  > | 参数       | 作用                                                         |
  > | ---------- | ------------------------------------------------------------ |
  > | `SEEK_CUR` | `fd` 的当前文件位置会被设定为当前值+pos<br/> pos 为零时，返回的是当前的文件位置值 |
  > | `SEEK_END` | `fd` 的当前文件位置会被设定为当前长度+pos<br/> pos 为零时，偏移值会被设定为文件的末端 |
  > | `SEEK_SET` | `fd` 的当前文件位置会被设定为 pos<br/> pos 为零时，偏移值会被设定为文件的开头 |

- 截短文件： 将一个文件的大小删减成比当前长度短

  - `ftruncate()`：可用于处理 `fd` 所指定的文件描述符，`fd` 必须已打开 

    > ```c
    > #include<unistd.h>
    > #include<sys/types.h>
    > int ftruncate(int fd,off_t len)
    > ```

  - `truncate()`：可用于操作 `path` 所指定的文件，该文件必须可写入

    > ```c
    > #include<unistd.h>
    > #include<sys/types.h>
    > int truncate(const char *path,off_t len)
    > ```

### 2. 同步化

- `fsync()`： 可确保文件描述符 `fd` 所映射的文件中所有脏数据会被写回磁盘

  > ```c
  > #include<unistd.h>
  > int fsync(int fd);
  > ```

- `fdatasync()`： 同 `fsync` 但只刷新数据，因此更快，但无法保证元数据与磁盘同步

  > ```c
  > #include<unistd.h>
  > int fdatasync(int fd);
  > ```

- `sync()`： 可确保缓冲区与磁盘同步化

  > ```c
  > #include<unistd.h>
  > void sync(void);
  > ```
  >
  > Linux 会一直等到所有缓冲区都被交付为止，因此花费的时间较长

### 3. 多任务式

- **作用**： 多任务式 I/O 可以让一个应用程序可以同时服务多个文件描述符，以及在其中有任何一个就绪可进行读取或写入时收到通知而不会受到阻挡

- **运行流程**： 

  1. 多任务式 I/O： 当文件描述符中有能进行 I/O，就通知提醒

  2. 休眠： 直到一个或多个文件描述符准备妥当

  3. 唤醒： 所有准备就绪

  4. 处理所有就绪可进行 I/O 的文件描述符而不会受到阻挡
  5. 回到步骤 1，从头开始

- **解决方案**： I/O 复用模型，`select,poll,epoll`

  ![](../../pics/netty/socket_10.png)

### 4. 内核

- **虚拟文件系统(VFS)**： 通过**公共文件模型**来实现，提供了==挂钩==来支持读取，创建链接，同步化等功能

  > - 每种文件系统可以注册其特有的函数，以便处理其所能进行的操作
  >
  > 优势： 
  >
  > - 单一系统调用可以读取任何存储媒介上的任何操作系统
  > - 单一实用程序可以从一个文件系统复制到任何其他文件系统
  > - 所有文件系统均支持相同的概念，相同的接口以及相同的调用

- **页面缓存**： 一块内存存储区，用来临时存放最近从磁盘文件系统上访问的数据

  > 页面缓存使用了时间局部性，顺序局部性
  >
  > - 时间局部性： 一种引用局部性，指一个资源被访问后，在不久的将来再次被访问的概率很高
  >
  > - 顺序局部性： 另一种引用局部性，指数据顺序被引用，内核还实现了页面缓存区的预读功能
  >
  >   > 预读： 指随着每个读取请求将额外的数据从磁盘读进页面缓存区的行为

- **页面写回**： 当进行写入时，数据会先复制到缓冲区，同时该缓冲区被标记成“已被改变”

  > “写回”发生的情况：
  >
  > - **内存缓冲区已被写满**，则触发“写出磁盘”操作
  > - 内存缓冲区的数据保存时间**超过指定时间**，则触发“写回磁盘”操作

## 2. 缓冲式 I/O

### 1. 流操作

- **打开文件**： `FILE *fopen(const char *path,const char *mode)`

  > 模式 `mode` 的取值： 
  >
  > - `r`： 打开文件以备读取，流位于文件开端
  >
  > - `r+`： 打开文件以备读取和写入，流位于文件开端
  >
  > - `w`： 打开文件以备写入，流位于文件开端
  >
  >   > - 如果文件已存在，它会被截短成零长度
  >   > - 如果文件不存在，则会被创建
  >
  > - `w+`： 打开文件以备读取和写入，流位于文件开端
  >
  > - `a`： 打开文件以备写入，并且进入附加模式，流位于文件末端
  >
  > - `a+`： 打开文件以备读取和写入，并且进入附加模式

- **经文件描述符打开流**： `FILE *fdopen(int fd,const char *mode)`

  > `mode` 取值同 `fopen`，但必须兼容于 `fopen` 的模式

- **关闭流**： `int fclose(FILE *stream)`

  > 当前缓冲数据会先被刷新至磁盘，而后关闭流

- **关闭所有流**： `int fcloseall(void)`

  > 关闭前，所有流会被刷新至磁盘

- **读取流**： 

  - **读取一个字符**： 

    > `int fgetc(FILE *stream)`： 
    >
    > - 从 `stream` 读取下一个字符，把它从 `unsigned char` 转型成 `int` 并返回
    >
    > - 转型目的： 提供足够的空间给 `end-of-file` 或错误的通知(`EOF`)
    >
    > `int ungetc(int c,FILE *stream)`： 可以把 `c` (会被转型为 `char`)推回 `stream`

  - **读取一整行**： `char *fgets(char *str,int size,FILE *stream)`

    > - 从 `stream` 读取 `size-1` 个字节并将结果存入 `str`

  - **读取二进制数据**： `size_t fread(void *buf,size_t size,size_t nr,FILE *stream)`

    > - 从 `stream` 读取 `nr * zise` 个字节到 `buf` 所指向的缓冲区

- **写入流**： 

  - **写入单一字符**： `int fputc(int c,FILE *stream)`

    > - 将参数 `c` (会被转型成 `char`)写入 `stream` 所指向的流

  - **写入一个字符串**： `int fputs(const char *str,FILE *stream)`

    > - 将参数 `str` 所指向的以 `null` 分隔的字符串全都写入参数 `stream` 所指向的流

  - **写入二进制数据**： `size_t fwrite(void *buf,size_t size,size_t nr,FILE *stream)`

    > - 将参数 `buf` 所指向的数据写入参数 `stream` 所指向的流，写入字节数 `nr * size`

- **查找流**： 

  > - `int fseek(FILE *stream,long offset,int whence)`： 通过 `whence,offset` 来操纵 `stream` 位置
  >   - `whence` 为 `SEEK_SET`，则文件位置为 `offset`
  >   - `whence` 为 `SEEK_CUR`，则文件位置为 当前位置 + `offset`
  >   - `whence` 为 `SEEK_END`，则文件位置为 文件末端 + `offset`
  >
  > - `int fsetpos(FILE *stream,fpos_t *pos)`： 将 `stream` 的流位置设为 `pos`
  >
  >   > 尽量避免使用该函数，可能存在平台的不兼容问题
  >   >
  >   > `int fgetpos(FILE *stream,fpos_t *pos)`： 返回 0 并将 `stream` 的当前流位置放于 `pos`
  >
  > - `void rewind(FILE *stream)`： 将位置重新设定为流的开头，等同 `fseek(stream,0,SEEK_SET)`
  >
  > - `long ftell(FILE *stream)`： 返回 `stream` 当前流的位置

- **刷新流**： `int fflush(FILE *stream)`

  > - `stream` 所指向的流中的任何未被写至内核的数据会被刷新至内核

- **获取指定流的文件描述符**： `int fileno(FILE *stream)`

### 2. 错误与 EOF

- `int ferror(FILE *stream)`： 用于测试 `stream` 之上是否设定了错误指示器

  > 错误指示器由其他标准 I/O 接口设定，用于返回错误状态。若未设定，返回 0

- `int feof(FILE *stream)`： 用于测试 `stream` 之上是否设定了 `EOF` 指示器

  > `EOF` 指示器由其他的标准 `I/O` 接口设定，用于反映抵达了文件的末端
  >
  > 如果设定了指示器，则返回一个非零值，否则返回 0

- `void clearerr(FILE *stream)`： 用于清除 `stream` 之上的错误与 `EOF` 指示器 

### 3. 缓冲机制

不同的用户缓冲机制： 

- **未经缓冲**： 数据直接提交给内核，标准错误默认未经缓冲
- **经行缓冲**： 以行为基础，每遇到一个 `newline`(或 `\n`)字符，缓冲区就被提交给内核
- **经块缓冲**： 以块为基础，默认与文件相关的流都是经块缓冲

控制使用的缓冲类型： `int setvbuf(FILE *stream,char *buf,int mode,size_t size)`

> 将 `stream` 的缓冲类型设定为 `mode`： 
>
> - `_IONBF`： 未经缓冲
> - `_IOLBF`： 经行缓冲
> - `_IOFBF`： 经块缓冲

### 4. 锁机制

- `void flockfile(FILE *stream)`： 会等到 `stream` 不再被锁定后取得流的锁，递增锁定计数，成为流的拥有线程，然后返回

- `void funlockfile(FILE *stream)`： 用于递减与 `stream` 相关联的锁定计数，直到锁定计数为零

- `int ftrylockfile(FILE *stream)`： `flockfile` 的不受阻挡的版本

  > - 如果 `stream` 已被锁定，则会立即返回一个非零值
  >
  > - 如果 `stream` 未被锁定，则会取得流的锁，递增锁定计数，成为 `stream` 的拥有者，然后返回 0

## 3. 高级文件 I/O

### 1. 分散-聚集 I/O

- **概念**： 让单一调用可以同时对多个缓冲区进行数据的读取或写入操作，可用于将各个数据结构的字段聚集成单次 I/O 操作

- **优点**： 

  - 更自然的 I/O 操作： 可以对分段数据，进行直观的 I/O 操作
  - 效率： 单次向量 I/O 操作可以取代多次线性 I/O 操作
  - 性能： 可以减少系统调用次数，同时还可经内部的优化
  - 原子性： 单次向量操作，不会与另一个进程操作交叉的风险

- **实现**： 

  > ```c
  > struct iovec{
  >     void *iov_base;
  >     size_t iov_len; //字段长度
  > }
  > ```

  - `readv()`： 把 `count` 个段从文件描述符 `fd` 读取进 `iov` 所描述的缓冲区

    > `ssize_t readv(int fd,const struct iovec *iov,int count)`

  - `writev()`： 把 `count` 个段从 `iov` 所描述的缓冲区写入至文件描述符 `fd`

    > `ssize_t writev(int fd,const struct iovec *iov,int count)`

### 2. 事件轮询接口

- `select 与 poll` 问题： 每次调用需要一份所要查看的文件描述符的完整列表，同时进行处理

  > 问题： 当该列表变大时，可能包含成百上千个文件描述符

- `epoll` 处理方式： 让事件监视器的注册于实际的事件监视器工作脱钩
  - 第一个系统调用用于初始化 `epoll` 的上下文
  - 第二个系统调用用于将所要查看的文件描述符加入上下文或从上下文中移除文件描述符
  - 第三个系统调用则实际执行事件等待

- `epoll` 函数： 

  - `int epoll_create(int size)`： 用于创建一个 `epoll` 上下文

    > - 执行成功，会创建一个新的 `epoll` 实例，并且返回一个与实例相关联的文件描述符
    >
    >   > 该文件描述符与真实的文件没关系，只是一个可供随后的调用使用 `epoll` 设备的句柄
    >
    > - 执行失败，返回 `-1`，并设定 `errno`：
    >
    >   - `EINVAL`： size 参数不是一个正数
    >   - `ENFILE`： 系统已经到达所能打开文件数目的上限
    >   - `ENOMEM`： 可用内存不足
    >
    > `size`： 用于提示内核要监视的文件描述符数目

  - `int epoll_ctl(int epfd,int op,int fd,struct epoll_event *event)`： 将文件描述符加入指定的  `epoll` 上下文，或从指定的 `epoll` 上下文移除文件描述符

    > - `op`： 指定如何操作与 `fd` 相关联的文件
    >
    >   - `EPOLL_CTL_ADD`： 根据 `event` 定义的事件，将特定文件上的一个监视器加入 `epoll` 实例
    >   - `EPOLL_CTL_DEL`： 从 `epoll` 实例移除特定文件上的一个事件监视器
    >   - `EPOLL_CTL_MOD`： 以 `event` 所指定的新事件来修改 `fd` 上的一个现有事件监视器
    >
    > - `event`： 进一步描述操作的行为
    >
    > - `epoll_event` 结构：
    >
    >   ```c
    >   #include<sys/epoll.h>
    >   struct epoll_event{
    >       __u32 events; //事件
    >       union{
    >           void *ptr;
    >           int fd;
    >           __u32 u32;
    >           __u64 u64;
    >       }data;
    >   }
    >   ```

  - `int epoll_wait(int epfd,struct epoll_event *events,int maxevents,int timeout)`： 最多可以为特定 `epoll` 实例相关联的文件上的特定事件等待 `timeout` 毫秒的时间

### 3. 内存映射 I/O

- 将一个文件映射至内存，让文件的 I/O 可以通过简单的内存操纵来进行，可用于某些类型的 I/O

- 相关函数： 
  - `void *mmap(void *addr,size_t len,int prot,int flags,int fd,off_t offset)`： 可要求内核将文件描述符 `fd` 所代表的对象映射至内存

    > - `len`： 代表对象中有多少个字节将被映射至内存
    > - `offset`： 代表从对象中的何处开始映射
    > - `addr`： 用于指向内存中所要使用的起始地址
    > - `prot`： 用于指示访问权限
    > - `flags`： 用于指定额外的行为

  - `int munmap(void *addr,size_t len)`： 用于移除 `mmap()` 创建的内存映射

  - `void *mremap(void *addr,size_t old_size,size_t new_size,unsigned long flags)`： 用于扩大或缩小所指定的映射空间

  - `int mprotect(const *addr,size_t len,int prot)`： 变更现有内存区域的权限

  - `int msync(void *addr,size_t len,int flags)`： 将经 `mmap()` 所映射的文件变动刷新至磁盘

  - `int madvise(void *addr,size_t len,int advice)`： 为内核提供建议，指出使用何种映射，然后内核优化其行为以便利用映射的预定用法

  - `long sysconf(int name)`： 返回 `name` 的配置项，如： 页面大小

  - `int getpagesize(void)`： 返回一个页面的大小

- **`mmap` 的优缺点**：

  - **优点**： 

    - 对内存映射文件进行读写时，可避免使用 `read()` 或 `write()` 系统调用时产生的无关副本

      > **读写时的副本**： 数据必须被复制到一个用户空间缓冲区，并从用户空间缓冲区复制回来

    - 对内存映射文件的读写操作不会产生任何系统调用或环境切换的开销，除页面失误外

    - 当多个进程将同一个对象映射至内存时，数据由这些进程共享

    - 映射的查找仅涉及很少的指针操作，不需 `lseek()` 系统调用

  - **缺点**： 

    - 内存映射往往是页面大小的整数倍，会闲置无法填满页面的空间
    - 内存映射必须适合放入进程的地址空间，即需要连续的地址空间
    - 创建和维护内存映射以及内核内部相关的数据结构需要一定的开销

### 4. 文件建议

- 允许进程提示内核自己的使用情况，可以改善 I/O 的性能

- 函数调用： 

  - `int posix_fadvise(int fd,off_t offset,off_t len,int advice)`： 将对文件描述符 `fd` 位于 `[offset,offset+len]` 范围内的建议提供给内核

    > 注： `len` 为 `0` 表示建议整个文件

  - `ssize_t readahead(int fd,off64_t offset,size_t count)`： 将文件描述符 `fd` 于 `[offset,offset+count]` 范围内的数据填入页面缓存区

### 5. 异步 I/O

- **同步与异步化**： 

  - 写入操作的同步状态：

    |      | 同步化                                                       | 异步化                                                       |
    | ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
    | 同步 | 写入操作会等到数据被刷新至磁盘后才返回                       | 写入操作会等到数据被存入内核缓冲区后才返回                   |
    | 异步 | 写入操作会在写入请求被排入队列后返回。等到该写入操作执行时，数据保证会被写入磁盘 | 写入操作会在写入请求被排入队列后返回。等到该写入操作实际执行时，数据保证至少会被存入内核缓冲区 |

  - 读取操作的同步状态： 

    |      | 同步化                                                       |
    | ---- | ------------------------------------------------------------ |
    | 同步 | 读取操作会在最新的数据被存入所提供的缓冲区后返回             |
    | 异步 | 读取操作会在读取请求被排入队列后返回，但当读取操作实际执行时会返回最新的数据 |

- **读取延迟**： 因为每个读取请求必须返回最新的数据，因此如果被请求的数据并未出现在页面缓存区中，读取进程必须遭到阻挡，直到可从磁盘读取数据为止

- I/O 调度程序采用的避免饿死机制： 

  - **`Linus Elevator` 莱纳斯电梯**： 在队列中出现待得很久的请求时，便停止插入排序的动作

    > - 公平对待每个请求，可以改善读取延迟，但该探试性机制并不是很友好
    > - **实现**： 维护一份经排序的未决 I/O 请求列表，位于队列头部的 I/O 请求是下一个要被服务的对象

  - **Deadline(期限) I/O 调度程序**： 

    > - 实现： 保留莱纳斯电梯队列，同时引进**读取请求 FIFO 队列和写入请求 FIFO 队列**
    >
    >   > FIFO 队列中的每条请求都会被赋予一个到期时间，读取 FIFO 为 500毫秒，写入 FIFO 为 5 秒
    >
    > - **执行**： 当一个新 I/O 请求被提交时，会被插入至标准队列，并且会被放在相应的 FIFO 队列尾端
    >
    >   > - 送给磁盘的 I/O 请求会来自己排序标准队列的头部，该 FIFO 队列可以减少对标准队列的查找操作，提高性能，因为标准队列会按块编号排序
    >   > - Deadline I/O 调度程序可以对 I/O 请求实施软性期限，即当 FIFO 队列头部出现已到期项目时，I/O 调度程序会停止从标准队列派送 I/O 请求，而会开始服务 FIFO 队列请求
    >
    > - **缺陷**： 当一系列 FIFO 的第一个请求快到期时，假设应用程序执行后会送出另一个读取请求，进而查找磁盘，如此来回查找会持续一段时间，影响整体性能

  - **Anticipatory(预测) I/O 调度程序**： 

    > **读取依赖问题**： 
    >
    > - 只有在前一个读取请求返回时才可以送出新的读取请求
    > - 等到应用程序取得读取数据并准备运行时，I/O 调度程序却切换来执行其他请求
    > - 进而导致每个读取请求浪费了两次查找时间： 请求开始时，磁盘查找一次；往回运行时又查找
    >
    > Anticipatory 运作方式： 
    >
    > - 一开始是 Deadline(期限) I/O 调度程序
    >
    > - 同时具备预测机制： 
    >
    >   - 若有一个读取请求被提交，则 Anticipatory I/O 调度程序会在期限内对它提供服务
    >
    >   - 接着会等待至多 6 毫秒来送出另一个针对文件系统中同一部分的读取请求，该请求会立刻获得服务： 
    >
    >     - 若6毫秒内没有出现读取请求，则会去服务已排序的标准队列
    >
    >     - 若有适当数量的请求被预测成功，则可以节省大量时间
    >
    >       > 原因： 每成功一次等于两次代价高的查找操作，由于读取请求的依赖高，因此预测算法很值得

  - **CFQ(完全公平队列) I/O 调度程序**： 

    > - 使用 CFQ 时，每个进程都会被指定一个独立的队列，而每个队列都会被指定一个时间片
    >
    > - I/O 调度程序会以轮转的方式扫描每个队列，服务队列里的请求，直到该队列的时间片用完或该队列已无未决请求
    >
    > - 当队列无请求时，CFQ I/O 调度程序将闲置10毫秒(默认)，等待队列里出现新的请求
    >
    >   > 该方式类似预测，由于同步化请求优先于异步化请求，该方式可避免写入饿死问题
    >
    > 因为每个进程设置了队列，所以 CFQ I/O 调度程序会公平对待所有进程，同时提高整体性能

  - **Noop(无运算) I/O 调度程序**： 

    > Noop I/O 调度程序不会排序，只会进行基本的合并，可用于无需排序 I/O 请求的特殊设备

### 6. 优化 I/O 性能

优化 I/O 措施： **减少 I/O 操作**，即将许多规模较小的操作凝聚成少数规模较大的操作

- 执行对齐块大小的 I/O
- 使用用户缓冲机制
- 利用高级 I/O 技术，如： 向量式 I/O、特定位置的 I/O、异步 I/O

I/O 调度程序会执行的步骤： 

- **逐块地排序请求**
- **最小化查找**
- **让磁盘的读写头平稳地以线性的方式移动**

用户空间应用程序的三种排序方式： 

- **按路径排序**： 最简单且最接近按块排序

  > - 优点： 多数文件系统使用的是**布局算法**，因此每个目录中的文件更可能在物理磁盘上相邻
  >
  > - 缺点： **未考虑碎片问题**

- **按 inode 编号排序**： Unix 风格的文件系统

  > - 概念： **每个文件都有一个 inode，且每个 inode 都有一个独立编码**
  > - 执行： 每个 I/O 请求涉及的文件都关联一个 inode，所以这些请求可按照 inode 编号从小到大排序

- **按物理块排序**： 每个文件会被划分成数个逻辑块，而逻辑块是文件系统的最小配置单元

  > 执行： 按照所指文件的第一个逻辑块的位置来排序
  >
  > 缺陷： 该方法需要 root 权限

# 二、进程

## 1. 进程管理

### 1. 进程 ID

- **进程 ID `pid`**：每个进程具有的唯一标识符 

- Linux 启动时运行的初始化进程由内核决定

  > 初始化进程 `init` 存放位置，顺序读取：`/sbin/init,/etc/init,/bin/init,/bin/sh`

- **进程 ID 的分配**： 内核默认进程 ID 最大值为 $2^{16}$ = 32768，可通过 `/proc/sys/kernel/pid_max` 修改

  > - 内核会使用严格的**线性方式**替进程分配进程 ID
  > - 内核不会重复使用 pid 值，除非超过最大值，才会使用较小值

- **进程的层次结构**： 

  > - 每个子进程都有一个父进程 `ppid`
  >
  > - 每个进程由一个用户`user` 和组 `group` 所拥有，所有权可用于控制资源的访问权限

- **获取进程 ID**： 
  - 获取当前进程 ID：`pid_t getpid(void)`
  - 获取父进程 ID： `pid_t getppid(void)`

### 2. 运行新进程

- **`exec` 调用**： 

  > `exec` 系列函数： 
  >
  > - `int execl(const char *path,const char *args,..)`： 将 path 参数所指向的程序加载至内存
  >
  >   > 成功的 `execl()` 调用不仅会改变地址空间和进程影像，也会改变进程的某些属性： 
  >   >
  >   > - 任何未决信号会遗失
  >   > - 进程所捕捉到的任何信号会回到默认行为
  >   > - 内存的任何锁定会被丢弃
  >   > - 多数线程属性会回到默认值
  >   > - 多数进程统计数据会被重置
  >   > - 与进程的内存有关的任何东西，包括所映射的任何文件，会被丢弃
  >   > - 单独存在于用户空间的任何东西，包括 C 链接库的功能，会被丢弃
  >
  > - `int execlp(const char *file,const char *arg,...)`
  >
  > - `int execle(const char *path,const char *arg,...,char * const envp[])`
  >
  > - `int execv(const char *path,const char *const argv[])`
  >
  > - `int execvp(const char *file,const char *const argv[])`
  >
  > - `int execve(const char *filename,const char *const argv[],char *const envp[])` 
  >
  > 对比： 
  >
  > - `l` 与 `v` 用于界定参数是提供列表，还是数组
  > - `p` 表示用户的完整路径，可用于搜索指定的文件
  > - `e` 表示新的进程提供一个新的环境

- **`fork` 调用**： 创建一个新的进程来运行与当前相同的映像

  > 子进程与父进程几乎一致，除了： 
  >
  > - 子进程的 pid 与父进程的不同
  > - 子进程的 ppid 被设定为父进程的 pid
  > - 子进程中的资源统计数据会被重置为零
  > - 任何未决信号会被清除，而且不会被子进程继承
  > - 所取得的任何文件锁定不会被子进程继承
  >
  > **进程的派生复制**： 
  >
  > - 早期 Linux 进程派生时，内核会替内部的所有数据结构创建副本，复制进程的页表项，以及将父进程的地址空间逐页复制到子进程的新地址空间
  >
  >   > 这种逐页复制费时
  >
  > - 现代 Linux 进程派生时，采用**写入时才复制(COW)**页面
  >
  >   > - 目的： 一种延迟化策略，减轻复制资源的开销
  >   > - 前提： 如果要读取一个资源，则会得到指向资源的指针；如果要写入资源，则会复制资源的副本，即写入复制

- **`vfork` 调用**： 同 `fork` 调用，但同时会执行 `exec` 或 `_exit()` 调用

  > 目的： 如果后面立即进行 `exec` 动作，`fork` 动作期间会浪费时间进行地址空间复制

### 3. 终止进程

- `void exit(int status)`： 终止当前进程

  > 步骤： 
  >
  > - 终止进程前，C 链接库会依次执行 `shutdown` 步骤
  > - 接着 `exit()` 会调用 `_exit()` 让内核处理终止进程的其余工作

- `shutdown` 步骤： 
  - 按照与注册相反的顺序调用以 `atexit()` 或 `on_exit()` 注册的任何函数
  - 刷新 `flush` 所有已打开的标准 I/O 流
  - 移除任何由 `tmpfile()` 函数所创建的临时文件

- `int atexit(void (*function)(void))`： 用于注册于进程终止时所调用的函数

  > 注册的函数原型： `void my_function(void)`

- `int on_exit(void (*function)(int,void *),void *arg)`： 同 `atexit`

  > 注册的函数原型： `void my_function(int status,void *arg)`

- `pid_t wait(int *status)`： 返回已终止子进程的 pid

  > 僵尸进程： 若子进程的死亡时间先于它的父进程，则内核让子进程进入特殊的进程状态
  >
  > > 该状态的进程只保留最小的骨架： 
  > >
  > > - 一些可能会包含有用数据的基本内核数据结构

- `pid waitpid(pid_t pid,int *status,int options)`： 

  > - 参数 `pid` 用于指定要等待的是进程 ID
  >   - `<-1`： 等待进程==组== ID 等于此值绝对值的任何子进程
  >   - `-1`： 等待任何子进程，此刻行为如同 `wait()`
  >   - `0`： 等待与 `calling process`(进行调用的进程) 均属于同一个进程组的任何子进程
  >   - `>0`： 等待其 pid 等于所指定值的任何子进程
  >
  > - 参数 `status`： 同 `wait()` 参数
  >
  > - 参数 `options`： 可以是零个或多个以下选项的二元 OR 逻辑运算
  >   - `WNOHANG`： 不要阻挡，如果没有相符的子进程已经终止(或停止，或继续)，则立即返回
  >   - `WUNTRACED`： 让我们实现更一般化的作业控制
  >   - `WCONTINUED`： 可用于实现一个 shell

- `int waitid(idtype_t idtype,id_t id,siginfo_t *infop,int options)`： 可用于等待并取得关于一个子进程的状态变更(终止，停止，继续)信息

  > 参数 `idtype` 和 `id` 用于指定所要等待的子进程

- `int system(const char *command)`： 可用于产生一个新进程并等待它的终止

  > 作用示例： 若一个进程产生了一个子进程且立即等待它的终止，则可以使用此接口







## 2. 高级进程管理











# 三、文件与内存管理

## 1. 文件和目录管理







## 2. 内存管理











# 四、信号与时间

## 1. 信号







## 2. 时间











