# 通过异步生成 dSYM 实现极速打包

## 背景

对于头条这种百万行级别的大型应用来说，即使使用 Mac Pro 进行编译打包，耗时也接近一小时。公司搭建了组件化平台后，组件得以提前编译为二进制，大大降低的应用的 CI 编译时间，目前耗时大约为八分钟左右。

通过对编译时间的进一步分析发现，大约有两分钟的时间用于生成 dSYM 文件，这个文件是 Release 模式下应用的符号表，由于 CI 打出的包不管是用于灰度测试还是内部研发人员测试，均有发生 crash 的风险和追查 crash 的需求，因此生成 dSYM 文件的步骤是不可省略的。

既然这一步不可省略，直觉告诉我们可以通过异步的方式去生成，从而避免阻塞编译构建。具体的做法为：

1. 将一次编译构建拆分为两次
2. 第一次编译不生成 dSYM 文件
3. 第二次编译再生成 dSYM 文件，由于使用了相同的代码和缓存，因此速度非常快。

## 符号化流程

### UUID 关联

经过实际测试后发现，这种做法并不可行。这里首先介绍一下 crash 日志的解析流程。

当安装在手机上的 App 发生崩溃后，系统会生成一份崩溃日志，其中记录了每个线程的调用堆栈，但是只有进程地址，没有函数名称。将进程地址转换为函数名称依赖于 dSYM 文件，这个过程也称为符号化。

一份崩溃日志必须要有对应的 dSYM 才能解析，它们通过一个叫做 UUID 的标志关联。具体流程如下：

1. 通过 xcodebuild 命令编译产物时，会生成一个 .app 文件和对应的 dSYM 文件，它们都是 Mach-O 可执行文件，都有自己的唯一标示，即 UUID。且两者的 UUID 相同。
2. App 崩溃后，系统生成一份崩溃日志，并且在其中记录下 UUID
3. 通过 UUID 找到关联的 dSYM 文件完成符号化。

崩溃日志中的 UUID 一般在 `Binary Images` 中的下一行：

<p class="img-tip" data-str="image.png"><img src='https://sf3-ttcdn-tos.pstatp.com/img/tos-cn-v-0000/1e92ae5486624b35b2e22320b3f77bba~noop.png' height=538 width=1000/></p>

dSYM 文件的 UUID 可以通过 `dwarfdump --uuid` 命令获取：

<p class="img-tip" data-str="image.png"><img src='https://sf3-ttcdn-tos.pstatp.com/img/tos-cn-v-0000/4305629a409746dfb81b9af84df35f46~noop.png' height=40 width=1140/></p>

### 符号化方式

崩溃日志的符号化一般有两种方式：

1. 使用 `symbolicatecrash` 命令，传入 dSYM 文件和崩溃日志，可以生成符号化以后的崩溃日志。
2. 使用 `atos` 命令，传入 dSYM 文件和崩溃日志中的具体地址，可以得到这个地址对应的函数符号。

经过测试验证，我们发现：

1. 第一种符号化方式，要求 dSYM 文件和崩溃日志中的 UUID 相同才能解析，一般个人用户会使用这种方案。
2. 第二种符号化方式，由于不涉及崩溃日志，表面上看不需要关联 UUID。著名的 Fabric 平台，和公司内的 Slardar 平台采用这种方式，并且单独存储 dSYM 文件。但在解析崩溃日志时，依然依赖 UUID 字段去找到对应 dSYM 文件。

需要说明的是，UUID **仅用于两个文件之间的关联**，苹果并不对它们的值有任何限制。以 `symbolicatecrash` 命令为例，崩溃日志和 dSYM 文件的 UUID 只要一致，**不管值是什么，均可以成功符号化**。

### 异步导出 dSYM 的方案

至此，我们已经摸清楚了最初异步导出 dSYM 方案失败的原因。在两次打包中，即使代码和缓存都一样，系统依然会产生两个 UUID。

假设第一次编译产生的 `.app` 文件的 UUID 为 A，第二次编译产生的 dSYM 文件的 UUID 为 B。在 Slardar 解析时，崩溃日志中的 UUID 为 A，但是平台只存储了 UUID 为 B 的 dSYM，虽然两者实际上可以通用，但是无法在平台上正确的关联上，导致解析失败。

因此，解决方案有以下几种：

1. 保存一份 A -> B 的 UUID 映射表，Slardar 平台根据这个关联
2. 保存一份 A -> B 的 UUID 映射表，hook 系统生成 crash 日志的流程，将其中的 UUID 从 A 改成 B。
3. 修改第二次编译生成的 dSYM 文件，将它的 UUID 改成和第一次生成的 `.app` 文件的 UUID 一致。

显然第三种方案操作更简单，并且对已有系统完全透明，无侵入和耦合。

## Mach-O 文件

### 结构

Mach-O 文件由三个部分组成，分别是 Header、Load Commands 和 Data。借用比较知名的图片来展示下：

<p class="img-tip" data-str="image.png"><img src='https://sf3-ttcdn-tos.pstatp.com/img/tos-cn-v-0000/8150ec327f714b9893ac13ab65937e3b~noop.png' height=401 width=365/></p>

而 UUID 就是其中一个 Load Command，名字叫 `LC_UUID`。可以通过 MachOViewer 看下其中的结构：

<p class="img-tip" data-str="image.png"><img src='https://sf3-ttcdn-tos.pstatp.com/img/tos-cn-v-0000/e091dabdb28248199530c895adedc27c~noop.png' height=191 width=1140/></p>

可以看到这个 Load Command 的结构：

- 前四个字节 `0000001B` 中的 `1B` 表示这是 `LC_UUID` 段。（通过 `#import <mach-o/ldsyms.h>` 并输入 `LC_UUID` 可以验证，并且可以看到所有 Load Command 的枚举）
- 接下来四个字节 `00000018` 中的 `18` 是 16 进制，对应到 10 进制表示这个 Load Command 的大小为 24 字节
- 因此可以推算出来，剩下 `24 - 4 - 4 = 16` 个字节就是实际存储的 UUID 的数据

### 修改

借助开源工具 [LIEF](https://github.com/lief-project/LIEF/) 可以获对 Mach-O 文件做解析，分析 Header、Load Commands 等各个部分的数据。

然而这个库当前发布的所有 Release 版本均有严重的 Bug，它的解析结果是正确的，但是写入结果有问题。而最新的 master 分支虽然写入没问题，但是处理大文件时会卡死（也可能是笔者姿势不对）。因此无奈之下，仅用这个库进行解析，获取必要的数据。

写入部分其实也很简单，通过 `mmap` 把文件映射到内存中，借助 `LIEF` 的分析结果，找到 `LC_UUID` 的偏移量，手动修改指针并写回文件即可。

部分核心逻辑如下（有删减）：

```c++
// mmap 读取文件
int fileDescriptor = open(argv[1], O_RDONLY, 0);
size_t size = _GetFileSize(fileDescriptor);
char *contents = mmap(0, size, PROT_READ | PROT_WRITE, MAP_PRIVATE, fileDescriptor, 0);
```

主要的修改部分：

```c++
void modifyDsymUUID(char *contents, FatBinary *macho, string instructionSet, string UUID) {
    // 先找到对应的指令集
    for (Binary &binary :*macho) {
        Header header = binary.header();
        // 根据 header 判断是不是当前需要处理的指令集，如果不是的话就略过
        if (!isCurrentInstructionSet(header, instructionSet)) {
            continue;
        }

        // 开始处理，找到 LC_UUID 段，以及这一段的偏移量和大小
        UUIDCommand uuidCommand = binary.uuid();
        uint64_t binaryFatOffset = binary.fat_offset();
        uint64_t commandOffset = uuidCommand.command_offset();
        uint32_t commandSize = uuidCommand.size();

        // 生成新的 uuid 数据并逐个替换
        std::vector<uint8_t> newUUID = rawUUID(UUID);
        for (int i = 8; i < commandSize; ++i) {
            contents[binaryFatOffset + commandOffset + i] = newUUID[i - 8];
        }
    }
}
```

## 后记

感谢公司内外各位大神的指教，然而受限于时间和笔者的能力，目前这个工具仅用于修改 UUID，不区分是否是 dSYM 文件，且**仅支持 armv7 和 arm64 架构**。因此项目开源在：https://github.com/bestswifter/bsUUIDModifier

欢迎修改订制与交流指正。
