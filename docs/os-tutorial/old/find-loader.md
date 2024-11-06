总是困在小小的引导扇区之中，也不是长久之计，毕竟只有 446 个字节能给我们自由支配，而保护模式的栈动不动就 512 字节，一个引导扇区完全盛不下。所以我们有必要进入一个跳板模块，并在其中进行初始化工作，再进入内核。

这时候又该有人问了：

> 啊所以为什么不直接进内核呢？

emmm，事实上也有这种系统（比如 `haribote`），但这样的一个缺点就是你的内核文件结构必须很简单甚至根本没有结构才行。

所以我们还是老老实实地跳入 `Loader` 再进内核吧，不过话说回来，我们现在连一个正经八百的 `Loader` 都还没有，不着急，我们马上创建一个：

**代码 3-1 极简 `Loader`（loader.asm）**
```asm
    org 0100h

    mov ax, 0B800h
    mov gs, ax ; 将gs设置为0xB800，即文本模式下的显存地址
    mov ah, 0Fh ; 显示属性，此处指白色
    mov al, 'L' ; 待显示的字符
    mov [gs:((80 * 0 + 39) * 2)], ax ; 直接写入显存
    
    jmp $ ; 卡死在此处
```

这个 `Loader` 的作用很简单，只是在屏幕第一行的正中央显示一个白色的 “L” 。

现在最主要的问题就是我们应该怎样寻找 `Loader` 呢？

这个很简单，在根目录区中是一个一个一个32字节的文件结构，其中就包含文件名，我们在根目录区中查找即可。

依照 FAT12 文件系统，根目录区排在 FAT 表和引导扇区后面，因此它的起始扇区是 BPB_RsvdSecCnt + BPB_NumFATs * BPB_FATSz16 = 19 号扇区；它的结束位置则是 19 + BPB_RootEntCnt * 32 / BPB_BytsPerSec = 33。

于是我们的思路便有了：从第 19 号扇区开始，依次读取根目录区，并在其中查找 `LOADER  BIN`（loader.bin写入之后的文件名）。如果已经读到第 34 扇区而仍然没有找到 `LOADER  BIN`，那么就默认该磁盘内不存在 `loader` 。

那么我们该怎么读取磁盘呢？事实上，BIOS的 `int 13h` 也给我们提供了这个功能：

> AH=02h：读取磁盘
>
> AL：待读取扇区数
>
> CH：起始扇区所在的柱面
>
> DH：起始扇区所在的磁头
>
> CL：起始扇区在柱面内的编号
>
> DL：驱动器号
>
> ES:BX：读入缓冲区的地址
>
> 返回值：
> >
> > FLAGS.CF=0：操作成功，AH=0，AL=成功读入的扇区总数
> >
> > FLAGS.CF=1：操作失败，AH 存放错误编码

错误编码我们并不需要，只需要保证 `FLAGS.CF` 的值为 `0` 就可以了。对此，我们可以执行一个 `jc` 跳转命令，它的作用是当 `FLAGS.CF` 为 `1` 时跳转。

思路有了，读盘功能也有了，我们就开始写程序吧。首先在 `DispStr` 函数的后面加入一个读取扇区的函数 `ReadSector`，它的作用是从第 `ax` 号扇区开始，读取 `cl` 个扇区至 `es:bx`：

**代码 3-2 读取软盘的函数（boot.asm）**
```asm
ReadSector:
    push bp
    mov bp, sp
    sub esp, 2 ; 空出两个字节存放待读扇区数（因为cl在调用BIOS时要用）

    mov byte [bp-2], cl
    push bx ; 这里临时用一下bx
    mov bl, [BPB_SecPerTrk]
    div bl ; 执行完后，ax将被除以bl（每磁道扇区数），运算结束后商位于al，余数位于ah，那么al代表的就是总磁道个数（下取整），ah代表的是剩余没除开的扇区数
    inc ah ; +1表示起始扇区（这个才和BIOS中的起始扇区一个意思，是读入开始的第一个扇区）
    mov cl, ah ; 按照BIOS标准置入cl
    mov dh, al ; 用dh暂存位于哪个磁道
    shr al, 1 ; 每个磁道两个磁头，除以2可得真正的柱面编号
    mov ch, al ; 按照BIOS标准置入ch
    and dh, 1 ; 对磁道模2取余，可得位于哪个磁头，结果已经置入dh
    pop bx ; 将bx弹出
    mov dl, [BS_DrvNum] ; 将驱动器号存入dl
.GoOnReading: ; 万事俱备，只欠读取！
    mov ah, 2 ; 读盘
    mov al, byte [bp-2] ; 将之前存入的待读扇区数取出来
    int 13h ; 执行读盘操作
    jc .GoOnReading ; 如发生错误就继续读，否则进入下面的流程

    add esp, 2
    pop bp ; 恢复堆栈

    ret
```

之所以这么包装一下，是因为如果每次都去手动填写上面的几个参数的话，实在是太麻烦了（总共有6个之多），现在我们只需要准备3个。

下一步，我们定义几个常量，它们的作用是增加可读性，毕竟满篇写死的根目录大小14之类的，很难让人看懂。

**代码 3-3 放在开头的常量定义（boot.asm）**
```asm
BaseOfStack             equ 07c00h ; 栈的基址
BaseOfLoader            equ 09000h ; Loader的基址
OffsetOfLoader          equ 0100h  ; Loader的偏移
RootDirSectors          equ 14     ; 根目录大小
SectorNoOfRootDirectory equ 19     ; 根目录起始扇区
```

常量过后还有变量，我们在这个程序中将要用到的变量也不少，它们将被放置在 `DispStr` 函数的前面。

**代码 3-4 放在中间的变量定义（boot.asm）**
```asm
wRootDirSizeForLoop dw RootDirSectors ; 查找loader的循环中将会用到
wSectorNo           dw 0              ; 用于保存当前扇区数
bOdd                db 0              ; 这个其实是下一节的东西，不过先放在这也不是不行

LoaderFileName      db "LOADER  BIN", 0 ; loader的文件名

MessageLength       equ 9 ; 下面是三条小消息，此变量用于保存其长度，事实上在内存中它们的排序类似于二维数组
BootMessage:        db "Booting  " ; 此处定义之后就可以删除原先定义的BootMessage字符串了
Message1            db "Ready.   " ; 显示已准备好
Message2            db "No LOADER" ; 显示没有Loader
```

`BootMessage` 改过之后，`DispStr` 也做了微调，现在可以用 `dh` 传递消息编号来打印了：

**代码 3-5 改进后的 `DispStr`（boot.asm）**
```asm
DispStr:
    mov ax, MessageLength
    mul dh ; 将ax乘以dh后，结果仍置入ax（事实上远比此复杂，此处先解释到这里）
    add ax, BootMessage ; 找到给定的消息
    mov bp, ax ; 先给定偏移
    mov ax, ds
    mov es, ax ; 以防万一，重新设置es
    mov cx, MessageLength ; 字符串长度
    mov ax, 01301h ; ah=13h, 显示字符的同时光标移位
    mov bx, 0007h ; 黑底灰字
    mov dl, 0 ; 第0行，前面指定的dh不变，所以给定第几条消息就打印到第几行
    int 10h ; 显示字符
    ret
```

一切准备工作均已办妥，下面我们开始主循环吧……且慢，我们还有一点点预备知识要补充，下面是 `int 13h` 的另一种用途。

> AH=00h：复位磁盘驱动器

> DL=驱动器号

> 返回值：

> > FLAGS.CF=0：操作成功

> > FLAGS.CF=1：操作失败，AH=错误代码

这里我们直接假定 `FLAGS.CF` 为0，不做任何判断了。下面便是主体代码：

**代码 3-6 查找 `Loader` 的代码主体（boot.asm）**
```asm
LABEL_START:
    mov ax, cs
    mov ds, ax
    mov es, ax ; 将ds es设置为cs的值（因为此时字符串和变量等存在代码段内）
    mov ss, ax ; 将堆栈段也初始化至cs
    mov sp, BaseOfStack ; 设置栈顶

    xor ah, ah ; 复位
    xor dl, dl
    int 13h ; 执行软驱复位

    mov word [wSectorNo], SectorNoOfRootDirectory ; 开始查找，将当前读到的扇区数记为根目录区的开始扇区（19）
LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
    cmp word [wRootDirSizeForLoop], 0 ; 将剩余的根目录区扇区数与0比较
    jz LABEL_NO_LOADERBIN ; 相等，不存在Loader，进行善后
    dec word [wRootDirSizeForLoop] ; 减去一个扇区
    mov ax, BaseOfLoader
    mov es, ax
    mov bx, OffsetOfLoader ; 将es:bx设置为BaseOfLoader:OffsetOfLoader，暂且使用Loader所占的内存空间存放根目录区
    mov ax, [wSectorNo] ; 起始扇区：当前读到的扇区数（废话）
    mov cl, 1 ; 读取一个扇区
    call ReadSector ; 读入

    mov si, LoaderFileName ; 为比对做准备，此处是将ds:si设为Loader文件名
    mov di, OffsetOfLoader ; 为比对做准备，此处是将es:di设为Loader偏移量（即根目录区中的首个文件块）
    cld ; FLAGS.DF=0，即执行lodsb/lodsw/lodsd后，si自动增加
    mov dx, 10h ; 共16个文件块（代表一个扇区，因为一个文件块32字节，16个文件块正好一个扇区）
LABEL_SEARCH_FOR_LOADERBIN:
    cmp dx, 0 ; 将dx与0比较
    jz LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR ; 继续前进一个扇区
    dec dx ; 否则将dx减1
    mov cx, 11 ; 文件名共11字节
LABEL_CMP_FILENAME: ; 比对文件名
    cmp cx, 0 ; 将cx与0比较
    jz LABEL_FILENAME_FOUND ; 若相等，说明文件名完全一致，表示找到，进行找到后的处理
    dec cx ; cx减1，表示读取1个字符
    lodsb ; 将ds:si的内容置入al，si加1
    cmp al, byte [es:di] ; 此字符与LOADER  BIN中的当前字符相等吗？
    jz LABEL_GO_ON ; 下一个文件名字符
    jmp LABEL_DIFFERENT ; 下一个文件块
LABEL_GO_ON:
    inc di ; di加1，即下一个字符
    jmp LABEL_CMP_FILENAME ; 继续比较

LABEL_DIFFERENT:
    and di, 0FFE0h ; 指向该文件块开头
    add di, 20h ; 跳过32字节，即指向下一个文件块开头
    mov si, LoaderFileName ; 重置ds:si
    jmp LABEL_SEARCH_FOR_LOADERBIN ; 由于要重新设置一些东西，所以回到查找Loader循环的开头

LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
    add word [wSectorNo], 1 ; 下一个扇区
    jmp LABEL_SEARCH_IN_ROOT_DIR_BEGIN ; 重新执行主循环

LABEL_NO_LOADERBIN: ; 若找不到loader.bin则到这里
    mov dh, 2
    call DispStr; 显示No LOADER
    jmp $

LABEL_FILENAME_FOUND: ; 找到了则到这里
    jmp $ ; 什么都不做，直接死循环
```

代码不长，60多行，虽然不多，作用很强。如果直接按照上文的方法，先 `nasm` 后 `dd`，一顿操作猛如虎的话，那么运行结果应该是这样的：

![图 3-1 直接运行的结果](https://cdn.luogu.com.cn/upload/image_hosting/2oos0b2t.png)

（图 3-1 直接运行的效果）

第三行将会出现一个 `No LOADER` 的标识，虽然不符合预期（应该没有任何输出才对），但这也正好说明了我们的主循环在工作。

那么下面我们的工作就是把 `Loader` 写入磁盘了，不过您可能会发现，我们甚至都没有编译 `Loader`，没事，马上编译一下：
```plain
nasm loader.asm -o loader.bin
```

虽然得到了 `loader.bin`，但我们的写入工作在此处就有两个分支了。如果您使用的是 `Linux` 或 `macOS`，请使用下列命令将 `loader.bin` 写入磁盘：
```plain
mkdir floppy
sudo mount -o loop a.img ./floppy/
cp loader.bin ./floppy/ -v
sudo umount ./floppy/
rmdir floppy
```

在 `Windows` 下我们则需要这样：
```plain
edimg imgin:a.img copy from:loader.bin to:@: imgout:a.img
```

无论用什么方式，只要您成功把 `Loader` 写入了磁盘，便无大碍。总之，写入之后的运行结果是这样的：

![图 3-2 写入后再运行，第 3 行的 No LOADER 消失不见](https://cdn.luogu.com.cn/upload/image_hosting/6ycty7vj.png)

（图 3-2 写入后再运行，第 3 行已经没有了 `No LOADER`）

如果您的运行结果与之相符，那么您就可以进入下一节的学习，我们将要加载我们的 `Loader`，并跳入其中，这样，我们的可支配空间就从 `0.5KB` 扩张到了 `64KB`，足有 `128` 倍的提升。