本来是想写应用程序的，但是吧，这个问题吧，它这个这个，略微有一些难度，知道吧，所以说先挑比较简单的写。

本节我们要重温第 1-6 节的恐惧，用一节的时间速通一个在硬盘上的引导加载器，然后就可以抛掉现在这个不伦不类的软盘启动，硬盘放数据的框架了。

引导扇区比较好改，先从引导扇区开始吧。和软盘版的引导扇区相比，主要要修改的部分有以下几点：

* `ReadSector` 要从读取软盘改成读取硬盘。

* 硬盘使用 FAT16 文件系统，所以 `GetFATEntry` 需要同步修改。

* 还是因为 FAT16 文件系统，主循环中判断文件是否结束的条件也要略作修改。

其余的部分均可保持不变。

我们先进入项目根目录，然后新建 `boot.asm` 和 `loader.asm`，`loader.asm` 我们仍旧选择使用第 3-4 节使用的白板 Loader：

**代码 21-1 白板 Loader（loader.asm）**
```asm
    org 0100h

    mov ax, 0B800h
    mov gs, ax ; 将gs设置为0xB800，即文本模式下的显存地址
    mov ah, 0Fh ; 显示属性，此处指白色
    mov al, 'L' ; 待显示的字符
    mov [gs:((80 * 0 + 39) * 2)], ax ; 直接写入显存
    
    jmp $ ; 卡死在此处
```

将白板 Loader 用 `ftcopy` 命令写入硬盘：

![](https://img2024.cnblogs.com/blog/3014678/202408/3014678-20240825104524809-1837687064.png)
（图 21-1 具体命令）

在 `boot.asm` 中粘贴原先软盘版的 `boot.asm` 的所有内容，并将 `load.inc` 和 `pm.inc` 一并复制到根目录，然后就可以开始修改了。

首先来修改 `boot.asm` 中获取 FAT 项的部分：

**代码 21-2 硬盘版 `GetFATEntry`（boot.asm）**
```asm
GetFATEntry: ; 返回第ax个簇的值
    push es
    push bx
    push ax ; 都会用到，push一下
    mov ax, BaseOfLoader
    sub ax, 0100h
    mov es, ax
    pop ax
    mov bx, 2
    mul bx ; 每一个FAT项是两字节，给ax乘2就是偏移
LABEL_GET_FAT_ENTRY:
    ; 将ax变为扇区号
    xor dx, dx
    mov bx, [BPB_BytsPerSec]
    div bx ; dx = ax % 512, ax /= 512
    push dx ; 保存dx的值
    mov bx, 0 ; es:bx已指定
    add ax, SectorNoOfFAT1 ; 对应扇区号
    mov cl, 1 ; 一次读一个扇区即可
    call ReadSector ; 直接读入
    ; bx 到 bx + 512 处为读进扇区
    pop dx
    add bx, dx ; 加上偏移
    mov ax, [es:bx] ; 读取，那么这里就是了
LABEL_GET_FAT_ENTRY_OK: ; 胜利执行
    pop bx
    pop es ; 恢复堆栈
    ret
```

修改的部分主要有：乘1.5的部分变成了乘2；读取的扇区数由两个降到一个；删掉了 FAT12 时期对 FAT 解压缩的处理。

读取扇区的部分则直接仿着四节前的那个硬盘驱动写就行了：

**代码 21-3 硬盘版 `ReadSector`（boot.asm）**
```asm
ReadSector: ; 读硬盘扇区
; 从第eax号扇区开始，读取cl个扇区至es:bx
    push esi
    push di
    push es
    push bx
    mov esi, eax
    mov di, cx ; 备份ax,cx

; 读硬盘 第一步：设置要读取扇区数
    mov dx, 0x1f2
    mov al, cl
    out dx, al

    mov eax, esi ; 恢复ax

; 第二步：写入扇区号
    mov dx, 0x1f3
    out dx, al ; LBA 7~0位，写入0x1f3

    mov cl, 8
    shr eax, cl ; LBA 15~8位，写入0x1f4
    mov dx, 0x1f4
    out dx, al

    shr eax, cl
    mov dx, 0x1f5
    out dx, al ; LBA 23~16位，写入0x1f5

    shr eax, cl
    and al, 0x0f ; LBA 27~24位
    or al, 0xe0 ; 表示当前硬盘
    mov dx, 0x1f6 ; 写入0x1f6
    out dx, al

; 第三步：0x1f7写入0x20，表示读
    mov dx, 0x1f7 
    mov al, 0x20
    out dx, al

; 第四步：检测硬盘状态
.not_ready:
    nop
    in al, dx ; 读入硬盘状态
    and al, 0x88 ; 分离第4位，第7位
    cmp al, 0x08 ; 硬盘不忙且已准备好
    jnz .not_ready ; 不满足，继续等待

; 第五步：将数据从0x1f0端口读出
    mov ax, di ; di为要读扇区数，共需读di * 512 / 2次
    mov dx, 256
    mul dx
    mov cx, ax
    
    mov dx, 0x1f0
.go_on_read:
    in ax, dx
    mov [es:bx], ax
    add bx, 2
    loop .go_on_read
; 结束
    pop bx
    pop es
    pop di
    pop esi
    ret
```

这里需要注意，`ReadSector` 调用前后会修改 `bx`、`di` 和 `esi`，如果自己写的话要注意备份。

由于换了 FAT16，boot.asm 开头的 `%include "fat12hdr.inc"` 也要同步更换为 `%include "fat16hdr.inc"`，这里面的内容对照着格式化函数和 `file.h` 很容易写出：

**代码 21-4 FAT16 相关常量（fat16hdr.inc）**
```asm
    BS_OEMName     db 'tutorial'    ; 固定的8个字节
    BPB_BytsPerSec dw 512           ; 每扇区固定512个字节
    BPB_SecPerClus db 1             ; 每簇固定1个扇区
    BPB_RsvdSecCnt dw 1             ; MBR固定占用1个扇区
    BPB_NumFATs    db 2             ; 我们实现的FAT16文件系统有2个FAT表
    BPB_RootEntCnt dw 512           ; 根目录区32个扇区，一个目录项32字节，共计32*512/32=512个目录项
    BPB_TotSec16   dw 0             ; 80MB硬盘的大小过大，不足以放到TotSec16
    BPB_Media      db 0xF8          ; 介质描述符，硬盘为0xF8
    BPB_FATSz16    dw 32            ; 一个FAT表所占的扇区数，FAT16 文件系统固定为32个扇区
    BPB_SecPerTrk  dw 63            ; 每磁道扇区数，80MB硬盘为63
    BPB_NumHeads   dw 16            ; 磁头数，bximage 的输出告诉我们是16个
    BPB_HiddSec    dd 0             ; 隐藏扇区数，没有
    BPB_TotSec32   dd 41943040      ; 若之前的 BPB_TotSec16 处没有记录扇区数，则由此记录，如果记录了，这里直接置0即可
    BS_DrvNum      db 0x80          ; int 13h 调用时所读取的驱动器号，由于挂载的是硬盘所以0x80 
    BS_Reserved1   db 0             ; 未使用，预留
    BS_BootSig     db 29h           ; 扩展引导标记
    BS_VolID       dd 0             ; 卷序列号，由于只挂载一个盘所以为0
    BS_VolLab      db 'OS-tutorial' ; 卷标，11个字节
    BS_FileSysType db 'FAT16   '    ; 由于是 FAT16 文件系统，所以写入 FAT16 后补齐8个字节

FATSz                   equ 32      ; BPB_FATSz16
RootDirSectors          equ 32      ; 根目录大小
SectorNoOfRootDirectory equ 65      ; 根目录起始扇区
SectorNoOfFAT1          equ 1       ; 第一个FAT表的开始扇区
DeltaSectorNo           equ 63      ; 由于第一个簇不用，所以RootDirSectors要-2再加上根目录区首扇区和偏移才能得到真正的地址，故把RootDirSectors-2封装成一个常量
```

最后一处修改是在主循环的 `LABEL_GOON_LOADING_FILE` 附近：

**代码 21-5 硬盘版主循环（boot.asm）**
```asm
    cmp ax, 0FFFFh ; 这里！原本是0FFF，但FAT16的文件结束时FFFF，所以这里要修改
    jz LABEL_FILE_LOADED ; 若此项=0FFFF，代表文件结束，直接跳入Loader
    push ax ; 重新存储FAT号，但此时的FAT号已经是下一个FAT了
```

至此，硬盘版引导扇区修改完成，完整代码如下：

**代码 21-6 硬盘引导扇区-完整版（boot.asm）**
```asm
    org 07c00h ; 告诉编译器程序将装载至0x7c00处
 
BaseOfStack             equ 07c00h ; 栈的基址

    jmp short LABEL_START
    nop ; BS_JMPBoot 由于要三个字节而jmp到LABEL_START只有两个字节 所以加一个nop
 
%include "fat16hdr.inc" ; 没错它会db一遍
%include "load.inc" ; 代替之前的常量

LABEL_START:
    mov ax, cs
    mov ds, ax
    mov es, ax ; 将ds es设置为cs的值（因为此时字符串和变量等存在代码段内）
    mov ss, ax ; 将堆栈段也初始化至cs
    mov sp, BaseOfStack ; 设置栈顶
 
    mov ax, 0600h ; AH=06h：向上滚屏，AL=00h：清空窗口
    mov bx, 0700h ; 空白区域缺省属性
    mov cx, 0 ; 左上：(0, 0)
    mov dx, 0184fh ; 右下：(80, 25)
    int 10h ; 执行
 
    mov dh, 0
    call DispStr ; Booting
 
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

LABEL_FILENAME_FOUND:
    mov ax, RootDirSectors ; 将ax置为根目录首扇区（19）
    and di, 0FFE0h ; 将di设置到此文件块开头
    add di, 01Ah ; 此时的di指向Loader的FAT号
    mov cx, word [es:di] ; 获得该扇区的FAT号
    push cx ; 将FAT号暂存
    add cx, ax ; +根目录首扇区
    add cx, DeltaSectorNo ; 获得真正的地址
    mov ax, BaseOfLoader
    mov es, ax
    mov bx, OffsetOfLoader ; es:bx：读取扇区的缓冲区地址
    mov ax, cx ; ax：起始扇区号

LABEL_GOON_LOADING_FILE: ; 加载文件
    push ax
    push bx
    mov ah, 0Eh ; AH=0Eh：显示单个字符
    mov al, '.' ; AL：字符内容
    mov bl, 0Fh ; BL：显示属性
; 还有BH：页码，此处不管
    int 10h ; 显示此字符
    pop bx
    pop ax ; 上面几行的整体作用：在屏幕上打印一个点
 
    mov cl, 1
    call ReadSector ; 读取Loader第一个扇区
    pop ax ; 加载FAT号
    call GetFATEntry ; 加载FAT项
    cmp ax, 0FFFFh
    jz LABEL_FILE_LOADED ; 若此项=0FFF，代表文件结束，直接跳入Loader
    push ax ; 重新存储FAT号，但此时的FAT号已经是下一个FAT了
    mov dx, RootDirSectors
    add ax, dx ; +根目录首扇区
    add ax, DeltaSectorNo ; 获取真实地址
    add bx, [BPB_BytsPerSec] ; 将bx指向下一个扇区开头
    jmp LABEL_GOON_LOADING_FILE ; 加载下一个扇区

LABEL_FILE_LOADED:
    mov dh, 1 ; 打印第 1 条消息（Ready.）
    call DispStr
    jmp BaseOfLoader:OffsetOfLoader ; 跳入Loader！
 
wRootDirSizeForLoop dw RootDirSectors ; 查找loader的循环中将会用到
wSectorNo           dw 0              ; 用于保存当前扇区数
bOdd                db 0              ; 这个其实是下一节的东西，不过先放在这也不是不行
 
LoaderFileName      db "LOADER  BIN", 0 ; loader的文件名
 
MessageLength       equ 9 ; 下面是三条小消息，此变量用于保存其长度，事实上在内存中它们的排序类似于二维数组
BootMessage:        db "Booting  " ; 此处定义之后就可以删除原先定义的BootMessage字符串了
Message1            db "Ready.   " ; 显示已准备好
Message2            db "No LOADER" ; 显示没有Loader

DispStr:
    mov ax, MessageLength
    mul dh ; 将ax乘以dh后，结果仍置入ax（事实上远比此复杂，此处先解释到这里）
    add ax, BootMessage ; 找到给定的消息
    mov bp, ax ; 先给定偏移
    mov ax, ds
    mov es, ax ; 以防万一，重新设置es
    mov cx, MessageLength ; 字符串长度
    mov ax, 01301h ; ah=13h, 显示字符的同时光标移位
    mov bx, 0007h ; 黑底白字
    mov dl, 0 ; 第0行，前面指定的dh不变，所以给定第几条消息就打印到第几行
    int 10h ; 显示字符
    ret

ReadSector: ; 读硬盘扇区
; 从第eax号扇区开始，读取cl个扇区至es:bx
    push esi
    push di
    push es
    push bx
    mov esi, eax
    mov di, cx ; 备份ax,cx

; 读硬盘 第一步：设置要读取扇区数
    mov dx, 0x1f2
    mov al, cl
    out dx, al

    mov eax, esi ; 恢复ax

; 第二步：写入扇区号
    mov dx, 0x1f3
    out dx, al ; LBA 7~0位，写入0x1f3

    mov cl, 8
    shr eax, cl ; LBA 15~8位，写入0x1f4
    mov dx, 0x1f4
    out dx, al

    shr eax, cl
    mov dx, 0x1f5
    out dx, al ; LBA 23~16位，写入0x1f5

    shr eax, cl
    and al, 0x0f ; LBA 27~24位
    or al, 0xe0 ; 表示当前硬盘
    mov dx, 0x1f6 ; 写入0x1f6
    out dx, al

; 第三步：0x1f7写入0x20，表示读
    mov dx, 0x1f7 
    mov al, 0x20
    out dx, al

; 第四步：检测硬盘状态
.not_ready:
    nop
    in al, dx ; 读入硬盘状态
    and al, 0x88 ; 分离第4位，第7位
    cmp al, 0x08 ; 硬盘不忙且已准备好
    jnz .not_ready ; 不满足，继续等待

; 第五步：将数据从0x1f0端口读出
    mov ax, di ; di为要读扇区数，共需读di * 512 / 2次
    mov dx, 256
    mul dx
    mov cx, ax
    
    mov dx, 0x1f0
.go_on_read:
    in ax, dx
    mov [es:bx], ax
    add bx, 2
    loop .go_on_read
; 结束
    pop bx
    pop es
    pop di
    pop esi
    ret

GetFATEntry: ; 返回第ax个簇的值
    push es
    push bx
    push ax ; 都会用到，push一下
    mov ax, BaseOfLoader
    sub ax, 0100h
    mov es, ax
    pop ax
    mov bx, 2
    mul bx ; 每一个FAT项是两字节，给ax乘2就是偏移
LABEL_GET_FAT_ENTRY:
    ; 将ax变为扇区号
    xor dx, dx
    mov bx, [BPB_BytsPerSec]
    div bx ; dx = ax % 512, ax /= 512
    push dx ; 保存dx的值
    mov bx, 0 ; es:bx已指定
    add ax, SectorNoOfFAT1 ; 对应扇区号
    mov cl, 1 ; 一次读一个扇区即可
    call ReadSector ; 直接读入
    ; bx 到 bx + 512 处为读进扇区
    pop dx
    add bx, dx ; 加上偏移
    mov ax, [es:bx] ; 读取，那么这里就是了
LABEL_GET_FAT_ENTRY_OK: ; 胜利执行
    pop bx
    pop es ; 恢复堆栈
    ret
 
times 510 - ($ - $$) db 0
db 0x55, 0xaa ; 确保最后两个字节是0x55AA
```

编译，运行，命令如下：

![](https://img2024.cnblogs.com/blog/3014678/202408/3014678-20240825114440154-1299637965.png)
（图 21-2 编译运行命令）

效果如下：

![](https://img2024.cnblogs.com/blog/3014678/202408/3014678-20240825114346553-1563979487.png)
（图 21-3 白色的 `L`，很熟悉对吧）

对于 Loader，在进行完上述修改以后，把 `LABEL_FILE_LOADED` 中的 `call KillMotor` 以及 `KillMotor` 函数一并删除即可，这里不多赘述，贴一遍完整代码：

**代码 21-7 硬盘版 Loader-完整版（loader.asm）**
```asm
    org 0100h ; 告诉编译器程序将装载至0x100处
 
BaseOfStack                 equ 0100h ; 栈的基址

    jmp LABEL_START
 
%include "fat16hdr.inc" ; 没错它会再db一遍
%include "load.inc" ; 代替之前的常量
%include "pm.inc" ; 保护模式相关

; GDT
LABEL_GDT:          Descriptor 0,            0, 0                            ; 占位用描述符
LABEL_DESC_FLAT_C:  Descriptor 0,      0fffffh, DA_C | DA_32 | DA_LIMIT_4K   ; 32位代码段，平坦内存
LABEL_DESC_FLAT_RW: Descriptor 0,      0fffffh, DA_DRW | DA_32 | DA_LIMIT_4K ; 32位数据段，平坦内存
LABEL_DESC_VIDEO:   Descriptor 0B8000h, 0ffffh, DA_DRW | DA_DPL3             ; 文本模式显存，后面用不到了
 
GdtLen equ $ - LABEL_GDT                                                    ; GDT的长度
GdtPtr dw GdtLen - 1                                                        ; gdtr寄存器，先放置长度
       dd BaseOfLoaderPhyAddr + LABEL_GDT                                   ; 保护模式使用线性地址，因此需要加上程序装载位置的物理地址（BaseOfLoaderPhyAddr）
 
SelectorFlatC       equ LABEL_DESC_FLAT_C  - LABEL_GDT                      ; 代码段选择子
SelectorFlatRW      equ LABEL_DESC_FLAT_RW - LABEL_GDT                      ; 数据段选择子
SelectorVideo       equ LABEL_DESC_VIDEO   - LABEL_GDT + SA_RPL3            ; 文本模式显存选择子

LABEL_START:
    mov ax, cs
    mov ds, ax
    mov es, ax ; 将ds es设置为cs的值（因为此时字符串和变量等存在代码段内）
    mov ss, ax ; 将堆栈段也初始化至cs
    mov sp, BaseOfStack ; 设置栈顶
    
    mov dh, 0
    call DispStr ; Loading
 
    mov word [wSectorNo], SectorNoOfRootDirectory ; 开始查找，将当前读到的扇区数记为根目录区的开始扇区（19）
    xor ah, ah ; 复位
    xor dl, dl
    int 13h ; 执行软驱复位
LABEL_SEARCH_IN_ROOT_DIR_BEGIN:
    cmp word [wRootDirSizeForLoop], 0 ; 将剩余的根目录区扇区数与0比较
    jz LABEL_NO_KERNELBIN ; 相等，不存在Kernel，进行善后
    dec word [wRootDirSizeForLoop] ; 减去一个扇区
    mov ax, BaseOfKernelFile
    mov es, ax
    mov bx, OffsetOfKernelFile ; 将es:bx设置为BaseOfKernel:OffsetOfKernel，暂且使用Kernel所占的内存空间存放根目录区
    mov ax, [wSectorNo] ; 起始扇区：当前读到的扇区数（废话）
    mov cl, 1 ; 读取一个扇区
    call ReadSector ; 读入
 
    mov si, KernelFileName ; 为比对做准备，此处是将ds:si设为Kernel文件名
    mov di, OffsetOfKernelFile ; 为比对做准备，此处是将es:di设为Kernel偏移量（即根目录区中的首个文件块）
    cld ; FLAGS.DF=0，即执行lodsb/lodsw/lodsd后，si自动增加
    mov dx, 10h ; 共16个文件块（代表一个扇区，因为一个文件块32字节，16个文件块正好一个扇区）
LABEL_SEARCH_FOR_KERNELBIN:
    cmp dx, 0 ; 将dx与0比较
    jz LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR ; 继续前进一个扇区
    dec dx ; 否则将dx减1
    mov cx, 11 ; 文件名共11字节
LABEL_CMP_FILENAME: ; 比对文件名
    cmp cx, 0 ; 将cx与0比较
    jz LABEL_FILENAME_FOUND ; 若相等，说明文件名完全一致，表示找到，进行找到后的处理
    dec cx ; cx减1，表示读取1个字符
    lodsb ; 将ds:si的内容置入al，si加1
    cmp al, byte [es:di] ; 此字符与KERNEL  BIN中的当前字符相等吗？
    jz LABEL_GO_ON ; 下一个文件名字符
    jmp LABEL_DIFFERENT ; 下一个文件块
LABEL_GO_ON:
    inc di ; di加1，即下一个字符
    jmp LABEL_CMP_FILENAME ; 继续比较

LABEL_DIFFERENT:
    and di, 0FFE0h ; 指向该文件块开头
    add di, 20h ; 跳过32字节，即指向下一个文件块开头
    mov si, KernelFileName ; 重置ds:si
    jmp LABEL_SEARCH_FOR_KERNELBIN ; 由于要重新设置一些东西，所以回到查找Kernel循环的开头

LABEL_GOTO_NEXT_SECTOR_IN_ROOT_DIR:
    add word [wSectorNo], 1 ; 下一个扇区
    jmp LABEL_SEARCH_IN_ROOT_DIR_BEGIN ; 重新执行主循环

LABEL_NO_KERNELBIN: ; 若找不到kernel.bin则到这里
    mov dh, 2
    call DispStr ; 显示No KERNEL
    jmp $

LABEL_FILENAME_FOUND:
    mov ax, RootDirSectors ; 将ax置为根目录首扇区（19）
    and di, 0FFF0h ; 将di设置到此文件块开头
 
    push eax
    mov eax, [es:di + 01Ch]
    mov dword [dwKernelSize], eax
    pop eax
 
    add di, 01Ah ; 此时的di指向Kernel的FAT号
    mov cx, word [es:di] ; 获得该扇区的FAT号
    push cx ; 将FAT号暂存
    add cx, ax ; +根目录首扇区
    add cx, DeltaSectorNo ; 获得真正的地址
    mov ax, BaseOfKernelFile
    mov es, ax
    mov bx, OffsetOfKernelFile ; es:bx：读取扇区的缓冲区地址
    mov ax, cx ; ax：起始扇区号

LABEL_GOON_LOADING_FILE: ; 加载文件
    push ax
    push bx
    mov ah, 0Eh ; AH=0Eh：显示单个字符
    mov al, '.' ; AL：字符内容
    mov bl, 0Fh ; BL：显示属性
; 还有BH：页码，此处不管
    int 10h ; 显示此字符
    pop bx
    pop ax ; 上面几行的整体作用：在屏幕上打印一个点
 
    mov cl, 1
    call ReadSector ; 读取Kernel第一个扇区
    pop ax ; 加载FAT号
    call GetFATEntry ; 加载FAT项
    cmp ax, 0FFFFh
    jz LABEL_FILE_LOADED ; 若此项=0FFF，代表文件结束，直接跳入Kernel
    push ax ; 重新存储FAT号，但此时的FAT号已经是下一个FAT了
    mov dx, RootDirSectors
    add ax, dx ; +根目录首扇区
    add ax, DeltaSectorNo ; 获取真实地址
    add bx, [BPB_BytsPerSec] ; 将bx指向下一个扇区开头
    jmp LABEL_GOON_LOADING_FILE ; 加载下一个扇区

LABEL_FILE_LOADED:
    mov dh, 1 ; "Ready."
    call DispStr
; 准备进入保护模式
    lgdt [GdtPtr] ; 加载gdt
    cli ; 关闭中断

    in al, 92h ; 开启A20地址线
    or al, 00000010b
    out 92h, al

    mov eax, cr0
    or eax, 1 ; CR0.PE=1，进入保护模式
    mov cr0, eax
 
    jmp dword SelectorFlatC:(BaseOfLoaderPhyAddr + LABEL_PM_START) ; 进入32位段，彻底进入保护模式
 
dwKernelSize        dd 0              ; Kernel大小
wRootDirSizeForLoop dw RootDirSectors ; 查找Kernel的循环中将会用到
wSectorNo           dw 0              ; 用于保存当前扇区数
bOdd                db 0              ; 这个其实是下一节的东西，不过先放在这也不是不行
 
KernelFileName      db "KERNEL  BIN", 0 ; Kernel的文件名
 
MessageLength       equ 9 ; 下面是三条小消息，此变量用于保存其长度，事实上在内存中它们的排序类似于二维数组
BootMessage:        db "Loading  " ; 此处定义之后就可以删除原先定义的BootMessage字符串了
Message1            db "Ready.   " ; 显示已准备好
Message2            db "No KERNEL" ; 显示没有Kernel

DispStr: ; void DispStr(char idx);
; idx -> dh
; 基于bios功能：
; int 10h : ah=13h, 打印字符串
    mov ax, MessageLength
    mul dh ; 将ax乘以dh后，结果仍置入ax（事实上远比此复杂，此处先解释到这里）
    add ax, BootMessage ; 找到给定的消息
    mov bp, ax ; 先给定偏移
    mov ax, ds
    mov es, ax ; 以防万一，重新设置es
    mov cx, MessageLength ; 字符串长度
    mov ax, 01301h ; ah=13h, 显示字符的同时光标移位
    mov bx, 0007h ; 黑底白字
    mov dl, 0 ; 第0行，前面指定的dh不变，所以给定第几条消息就打印到第几行
    add dh, 3 ; 给dh加3，避免与boot打印的消息重叠
    int 10h ; 显示字符
    ret

ReadSector: ; 读硬盘扇区
; 从第eax号扇区开始，读取cl个扇区至es:bx
    push esi
    push di
    push es
    push bx
    mov esi, eax
    mov di, cx ; 备份ax,cx

; 读硬盘 第一步：设置要读取扇区数
    mov dx, 0x1f2
    mov al, cl
    out dx, al

    mov eax, esi ; 恢复ax

; 第二步：写入扇区号
    mov dx, 0x1f3
    out dx, al ; LBA 7~0位，写入0x1f3

    mov cl, 8
    shr eax, cl ; LBA 15~8位，写入0x1f4
    mov dx, 0x1f4
    out dx, al

    shr eax, cl
    mov dx, 0x1f5
    out dx, al ; LBA 23~16位，写入0x1f5

    shr eax, cl
    and al, 0x0f ; LBA 27~24位
    or al, 0xe0 ; 表示当前硬盘
    mov dx, 0x1f6 ; 写入0x1f6
    out dx, al

; 第三步：0x1f7写入0x20，表示读
    mov dx, 0x1f7 
    mov al, 0x20
    out dx, al

; 第四步：检测硬盘状态
.not_ready:
    nop
    in al, dx ; 读入硬盘状态
    and al, 0x88 ; 分离第4位，第7位
    cmp al, 0x08 ; 硬盘不忙且已准备好
    jnz .not_ready ; 不满足，继续等待

; 第五步：将数据从0x1f0端口读出
    mov ax, di ; di为要读扇区数，共需读di * 512 / 2次
    mov dx, 256
    mul dx
    mov cx, ax
    
    mov dx, 0x1f0
.go_on_read:
    in ax, dx
    mov [es:bx], ax
    add bx, 2
    loop .go_on_read
; 结束
    pop bx
    pop es
    pop di
    pop esi
    ret

GetFATEntry: ; 返回第ax个簇的值
    push es
    push bx
    push ax ; 都会用到，push一下
    mov ax, BaseOfLoader
    sub ax, 0100h
    mov es, ax
    pop ax
    mov bx, 2
    mul bx ; 每一个FAT项是两字节，给ax乘2就是偏移
LABEL_GET_FAT_ENTRY:
    ; 将ax变为扇区号
    xor dx, dx
    mov bx, [BPB_BytsPerSec]
    div bx ; dx = ax % 512, ax /= 512
    push dx ; 保存dx的值
    mov bx, 0 ; es:bx已指定
    add ax, SectorNoOfFAT1 ; 对应扇区号
    mov cl, 1 ; 一次读一个扇区即可
    call ReadSector ; 直接读入
    ; bx 到 bx + 512 处为读进扇区
    pop dx
    add bx, dx ; 加上偏移
    mov ax, [es:bx] ; 读取，那么这里就是了
LABEL_GET_FAT_ENTRY_OK: ; 胜利执行
    pop bx
    pop es ; 恢复堆栈
    ret

[section .s32]
align 32
[bits 32]
LABEL_PM_START:
    mov ax, SelectorVideo ; 按照保护模式的规矩来
    mov gs, ax ; 把选择子装入gs

    mov ah, 0Fh
    mov al, 'P'
    mov [gs:((80 * 0 + 39) * 2)], ax ; 这一部分写入显存是通用的
     
    mov ax, SelectorFlatRW ; 数据段
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov ss, ax
    mov esp, TopOfStack

; cs的设定已在之前的远跳转中完成
    call InitKernel ; 重新放置内核
    jmp SelectorFlatC:KernelEntryPointPhyAddr ; 进入内核，OS征程从这里开始

MemCpy: ; void memcpy(void *dest, const void *src, size_t size);
; ds:参数2 ==> es:参数1，大小：参数3
    push ebp
    mov ebp, esp ; 保存ebp和esp的值
 
    push esi
    push edi
    push ecx ; 暂存这三个，要用
 
    mov edi, [ebp + 8] ; [esp + 4] ==> 第一个参数，目标内存区
    mov esi, [ebp + 12] ; [esp + 8] ==> 第二个参数，源内存区
    mov ecx, [ebp + 16] ; [esp + 12] ==> 第三个参数，拷贝的字节大小
.1:
    cmp ecx, 0 ; if (ecx == 0)
    jz .2 ; goto .2;
 
    mov al, [ds:esi] ; 从源内存区中获取一个值
    inc esi ; 源内存区地址+1
    mov byte [es:edi], al ; 将该值写入目标内存
    inc edi ; 目标内存区地址+1
 
    dec ecx ; 拷贝字节数大小-1
    jmp .1 ; 重复执行
.2:
    mov eax, [ebp + 8] ; 目标内存区作为返回值
 
    pop ecx ; 以下代码恢复堆栈
    pop edi
    pop esi
    mov esp, ebp
    pop ebp
 
    ret

InitKernel: ; void InitKernel();
    xor esi, esi ; esi = 0;
    mov cx, word [BaseOfKernelFilePhyAddr + 2Ch] ; 这个内存地址存放的是ELF头中的e_phnum，即Program Header的个数
    movzx ecx, cx ; ecx高16位置0，低16位置入cx
    mov esi, [BaseOfKernelFilePhyAddr + 1Ch] ; 这个内存地址中存放的是ELF头中的e_phoff，即Program Header表的偏移
    add esi, BaseOfKernelFilePhyAddr ; Program Header表的具体位置
.Begin:
    mov eax, [esi] ; 首先看一下段类型
    cmp eax, 0 ; 段类型：PT_NULL或此处不存在Program Header
    jz .NoAction ; 本轮循环不执行任何操作
    ; 否则的话：
    push dword [esi + 010h] ; p_filesz
    mov eax, [esi + 04h] ; p_offset
    add eax, BaseOfKernelFilePhyAddr ; BaseOfKernelFilePhyAddr + p_offset
    push eax
    push dword [esi + 08h] ; p_vaddr
    call MemCpy ; 执行一次拷贝
    add esp, 12 ; 清理堆栈
.NoAction: ; 本轮循环的清理工作
    add esi, 020h ; 下一个Program Header
    dec ecx
    jnz .Begin ; jz过来的话就直接ret了
 
    ret

[section .data1]
StackSpace: times 1024 db 0 ; 栈暂且先给1KB
TopOfStack  equ $ - StackSpace ; 栈顶
```

如今硬盘 bootloader 已成，直接把原本的 `boot.asm` 和 `loader.asm` 替换为现在的 `boot.asm` 和 `loader.asm`，并用 `fat16hdr.inc` 替换 `fat12hdr.inc` 即可。

Makefile 也要进行修改，从此以后不再生成 `a.img` 了，而是生成 `hd.img`：

**代码 21-8 新版 `Makefile`（Makefile）**
```makefile
hd.img : out/boot.bin out/loader.bin out/kernel.bin
	ftimgcreate hd.img -t hd -size 80
	ftformat hd.img -t hd -f fat16
	ftcopy out/loader.bin -to -img hd.img
	ftcopy out/kernel.bin -to -img hd.img
	dd if=out/boot.bin of=hd.img bs=512 count=1

run : hd.img
	qemu-system-i386 -hda hd.img
```

由于现在每次编译都会重新创建硬盘镜像，所以之前的写入测试文件 `iloveado.fai` 将不复存在，那就返璞归真，用一行简单的打印证明我们进入了内核吧：

**代码 21-9 kernel/main.c**
```c
void kernel_main() // kernel.asm会跳转到这里
{
    monitor_clear();
    init_gdtidt();
    init_memory();
    init_timer(100);
    init_keyboard();
    asm("sti");

    task_t *task_a = task_init();
    task_t *task_shell = create_kernel_task(shell);
    //task_run(task_shell);

    printk("Hello, HD Boot!");

    while (1);
}
```

（shell：请你认真的看一看我……我至今为止有被用过哪怕一次么？）

使用 `make default` 全部重新编译，运行，效果如下图：

![](https://img2024.cnblogs.com/blog/3014678/202408/3014678-20240825120014809-1395748390.png)
（图 21-4 硬盘启动成功）

终于，在整整21节之后，我们并不是很紧地跟上了时代潮流，将软盘扔进了历史的垃圾堆，事实证明这是颇不具有里程碑意义的一件不是很大的事。

这一节本来是想放在最后写的，耐不住老有人催，所以提前写了，应用程序什么的就放在下一节吧！
