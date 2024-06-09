---
title: IL2Cpp 逆向工程：字符串提取（绕路版）
date: 2024-06-09 21:28:03
tags:
  - Reverse
categories:
  - Reverse
toc: true
---
书接某位[可爱女大（本文中的🐱）的上回](https://www.neko.ink/2023/10/15/dump-il2cpp-executable-from-memory/)，在凌晨四点的一个Twitter Space里面，我们开始研究IL2Cpp的字符串这个大坑。

<!-- more -->
# `global-metadata.dat`
IL2Cpp中引用的字符串字面量都存储在Global Metadata文件中，并在应用运行时按需加载，要取得字符串池，就需要先获得这个文件。在元气骑士的最新版中，这个文件在APK中存储的版本是加密的，并且相关算法进行了保护，难以直接取得。

这里参考了 https://www.bilibili.com/read/cv25143217/ 这篇文章，大致的思路是对`vm::MetadataLoader::LoadMetadataFile`这个函数进行Hook，从而取得实际加载进引擎的文件。

```javascript
var finded = false
var addr;
var notFirstTime = true
var s_GlobalMetadataHeader;

setInterval(() => {
    if (finded) {
        if (addr != null) {
            let offset = 0x3A5CEB4; //此值需要更改
            let funcOff = addr.base.add(offset);
            if (notFirstTime) {
                notFirstTime = false;
                Interceptor.attach(funcOff, {
                    onEnter: function (args) {
                        console.log("MetadataLoader::LoadMetadataFile");
                    },
                    onLeave: function (retval) {
                        console.log("s_GlobalMetadataHeader：" + retval.toString(16));
                        s_GlobalMetadataHeader = retval;
                        save(get_size())
                    }
                })
            }
        } else {
            addr = Process.findModuleByName("libil2cpp.so");
            if (addr == null) {
                return;
            }
            let offset = 0x3A5CEB4; //此值需要更改
            let funcOff = addr.base.add(offset);
            if (notFirstTime) {
                notFirstTime = false;
                Interceptor.attach(funcOff, {
                    onEnter: function (args) {
                        console.log("MetadataLoader::LoadMetadataFile");
                    },
                    onLeave: function (retval) {
                        console.log("s_GlobalMetadataHeader：" + retval.toString(16));
                        s_GlobalMetadataHeader = retval;
                        save(get_size())
                    }
                })
            }
        }
    } else {
        try {
            addr = Process.findModuleByName("libil2cpp.so");
            finded = true;

        } catch (ex) {
        }
    }
}, 5);

function get_size() {
    const metadataHeader = s_GlobalMetadataHeader;
    let fileOffset = 0x10C;
    let lastCount = 0;
    let lastOffset = 0;
    while (true) {
        lastCount = Memory.readInt(ptr(metadataHeader).add(fileOffset));
        if (lastCount !== 0) {
            lastOffset = Memory.readInt(ptr(metadataHeader).add(fileOffset - 4));
            break;
        }
        fileOffset -= 8;
        if (fileOffset <= 0) {
            console.log("get size failed!");
            break;
        }
    }
    return lastOffset + lastCount;
}

function save(size) {
    var file = new File("/data/data/com.ChillyRoom.DungeonShooter/files/global-metadata.dump.dat", "wb");
    var contentBuffer = Memory.readByteArray(s_GlobalMetadataHeader, size);
    file.write(contentBuffer);
    file.flush();
    file.close;
    console.log("global-metadata已导出到/data/data/com.ChillyRoom.DungeonShooter/files/global-metadata.dump.dat")
}
```
对原文的脚本稍作修改，并对路径进行处理（`/data/local/tmp`有的时候对用户应用不可写），成功得到了解密后的Metadata文件。

# Begin of the Rabbit Hole
<details>
	<summary>剧透</summary>
	这事其实没有我们想象的复杂，掉坑里去了，但还是做一下记录，以备不时之需。
</details>

{% asset_img "Pasted image 20240609214524.png" %}
把文件加载进解析工具[Il2CppDumper](https://github.com/Perfare/Il2CppDumper)之后，我们发现这玩意无论如何都跑不动。在启动视觉工作室™对Il2CppDumper进行了一些小修改后，可以导出以字母表顺序排序的字符串池（这一部分没有问题）。但是🐱表示希望能获得以引用地址排序的字符串池（她表示之前的版本就是这样的），以方便后续的逆向，所以还需要继续研究后面的结构。

由于游戏里面确实有一些防护，我们的第一想法是有部分加载逻辑被修改了，于是开始尝试寻找正确的文件结构。

# String Literals是如何存储的（v24.1）
{% asset_img "Notes_240609_215132.jpg" %}
在文件头`Il2CppGlobalMetadataHeader`中，有一组偏移量`metadataUsagePairs{Offset, Count}`记录了Metadata Usage表在文件中的位置，这张表记录了Metadata中的各类资源与可执行文件中的代码和其他数据结构之间的关联关系，字符串本身的存储较为简单，这里不再详述。

Metadata Usage表由两个字段组成，描述了一对从字符串到偏移地址的对应关系。
```c++
typedef struct Il2CppMetadataUsagePair
{
	// 该 Metadata 在可执行文件中某数据结构的索引
    uint32_t destinationIndex;
    // 该 Metadata 的类型和在本文件中的索引
    // 对于字符串，就是在字符串池中的索引
    // 其中 index = encodedSourceIndex & 0x1FFFFFFF /* 这个在 v27+ 不一样 */
    // usage = (encodedSourceIndex & 0xE0000000) >> 29
    // usage取值为1-6，0和7均无效
    uint32_t encodedSourceIndex;
} Il2CppMetadataUsagePair;
```
相对应的，在可执行文件中有如下结构：
```c++
typedef struct Il2CppMetadataRegistration
{
    int32_t genericClassesCount;
    Il2CppGenericClass* const * genericClasses;
    int32_t genericInstsCount;
    const Il2CppGenericInst* const * genericInsts;
    int32_t genericMethodTableCount;
    const Il2CppGenericMethodFunctionsDefinitions* genericMethodTable;
    int32_t typesCount;
    const Il2CppType* const * types;
    int32_t methodSpecsCount;
    const Il2CppMethodSpec* methodSpecs;

    FieldIndex fieldOffsetsCount;
    const int32_t** fieldOffsets;

    TypeDefinitionIndex typeDefinitionsSizesCount;
    const Il2CppTypeDefinitionSizes** typeDefinitionsSizes;
    const size_t metadataUsagesCount; /****/
    void** const* metadataUsages; /****/
} Il2CppMetadataRegistration;
```
这个结构体在进行代码生成时即被填入数据，存储在`.data.rel.ro`段中。`metadataUsages`中的指针在运行时[会被替换为指向实际字符串的指针](https://github.com/4ch12dy/il2cpp/blob/93f63348743a017cfb7267f64b2b7a2cdae8af51/unity_2019_x/libil2cpp/vm/MetadataCache.cpp#L1642)。

在IL2Cpp生成的代码中，字符串被引用的方式可以参考 [Investigating an issue about creating string literals in multiple threads in Unity](https://medium.com/@alanliu90/investigating-an-issue-about-creating-string-literals-in-multiple-threads-in-unity-cdb2989f1f25) 一文。每一个字符串都是一个全局变量（也就是会在`.got`中生成一项）,在`.got`中的顺序大致上与字符串在函数中引用的顺序一致，可以作为逆向的参考。
```c++
// Il2CppMetadataUsage.cpp  
String_t* _stringLiteral3495210533;  
​  
extern void** const g_MetadataUsages[73910] =   
{  
    // ...  
    (void**)(&_stringLiteral3495210533),  
    // ...  
};  
​  
// Il2CppMetadataRegistration.cpp  
extern void** const g_MetadataUsages[];  
extern const Il2CppMetadataRegistration g_MetadataRegistration =   
{  
    33686,  
    s_Il2CppGenericTypes,  
    9827,  
    g_Il2CppGenericInstTable,  
    77179,  
    s_Il2CppGenericMethodFunctions,  
    74908,  
    g_Il2CppTypeTable,  
    80321,  
    g_Il2CppMethodSpecTable,  
    14575,  
    g_FieldOffsetTable,  
    14575,  
    g_Il2CppTypeDefinitionSizesTable,  
    73137,  
    g_MetadataUsages,  
};  
​  
// Il2CppCodeRegistration.cpp  
void s_Il2CppCodegenRegistration()  
{  
    il2cpp_codegen_register (&g_CodeRegistration, &g_MetadataRegistration, &s_Il2CppCodeGenOptions);  
}  
#if RUNTIME_IL2CPP  
static il2cpp::utils::RegisterRuntimeInitializeAndCleanup s_Il2CppCodegenRegistrationVariable (&s_Il2CppCodegenRegistration, NULL);  
#endif
```
这里我发现，文件头和字符串索引表有16个字节的重叠，说明文件头的大小并不正确：
{% asset_img "Pasted image 20240609223541.png" %}
在尝试将每一组`Offset`和`Count`都当成Metadata Usage Pairs解析后，发现都不满足该结构的条件，于是先放弃处理元数据，看看可执行文件那边是什么情况。
# `g_MetadataRegistration`，你在哪里？
该结构体的偏移量在有符号的时候可以直接拿到，没有符号的时候Dumper可以进行搜索，但这里由于Metadata文件无法解析，采用了手动的方式。在`MetadataCache`的初始化过程中，会用到一个指向该结构体的指针，稍作对比，即可找出引用了`s_Il2CppMetadataRegistration`这个指针的代码：
{% asset_img "Pasted image 20240609221841.png" %}
```c++
// libil2cpp/vm/MetadataCache.cpp

static const Il2CppCodeRegistration * s_Il2CppCodeRegistration;
static const Il2CppMetadataRegistration * s_Il2CppMetadataRegistration;
static const Il2CppCodeGenOptions* s_Il2CppCodeGenOptions;
static CustomAttributesCache** s_CustomAttributesCaches;

bool il2cpp::vm::MetadataCache::Initialize()
{
    s_GlobalMetadata = vm::MetadataLoader::LoadMetadataFile("global-metadata.dat");

    if (!s_GlobalMetadata)
        return false;

    s_GlobalMetadataHeader = (const Il2CppGlobalMetadataHeader*)s_GlobalMetadata;
    IL2CPP_ASSERT(s_GlobalMetadataHeader->sanity == 0xFAB11BAF);
    IL2CPP_ASSERT(s_GlobalMetadataHeader->version == 24);

    // Pre-allocate these arrays so we don't need to lock when reading later.
    // These arrays hold the runtime metadata representation for metadata explicitly
    // referenced during conversion. There is a corresponding table of same size
    // in the converted metadata, giving a description of runtime metadata to construct.
    s_TypeInfoTable = (Il2CppClass**)IL2CPP_CALLOC(s_Il2CppMetadataRegistration->typesCount, sizeof(Il2CppClass*));
    s_TypeInfoDefinitionTable = (Il2CppClass**)IL2CPP_CALLOC(s_GlobalMetadataHeader->typeDefinitionsCount / sizeof(Il2CppTypeDefinition), sizeof(Il2CppClass*));
    s_MethodInfoDefinitionTable = (const MethodInfo**)IL2CPP_CALLOC(s_GlobalMetadataHeader->methodsCount / sizeof(Il2CppMethodDefinition), sizeof(MethodInfo*));
    s_GenericMethodTable = (const Il2CppGenericMethod**)IL2CPP_CALLOC(s_Il2CppMetadataRegistration->methodSpecsCount, sizeof(Il2CppGenericMethod*));
    s_ImagesCount = s_GlobalMetadataHeader->imagesCount / sizeof(Il2CppImageDefinition);
    s_ImagesTable = (Il2CppImage*)IL2CPP_CALLOC(s_ImagesCount, sizeof(Il2CppImage));
    s_AssembliesCount = s_GlobalMetadataHeader->assembliesCount / sizeof(Il2CppAssemblyDefinition);
    s_AssembliesTable = (Il2CppAssembly*)IL2CPP_CALLOC(s_AssembliesCount, sizeof(Il2CppAssembly));

	// ...
}
```
于是使用Frida动态调试，找出实际的偏移量：
```javascript
new NativePointer(Process.findModuleByName("libil2cpp.so").base.add(0x9CFF4C8)).readPointer().sub(Process.findModuleByName("libil2cpp.so").base)
// 0x93e29d8
```
没毛病，但是`metadataUsagesCount`怎么是0呢？
{% asset_img "Pasted image 20240609222341.png" %}
于是开始在源码里寻找用到了上面提到的这些数据的代码，找到了`il2cpp::vm::MetadataCache::IntializeMethodMetadataRange`这个函数，里面有一个标志性的`switch`，在翻看xref的时候很容易注意到：
{% asset_img "Pasted image 20240609222630.png" %}
加载字符串的实际逻辑：
{% asset_img "Pasted image 20240609222823.png" %}
由于IDA对`switch`的处理比较苦手，此处没有F5源代码可供参考，但是看起来和公开的源码没有很大的出入。

# 小丑竟是我自己
为啥不能用呢？这个时候我决定动调看看：
```shell
frida-trace -U  'Soul Knight' -a 'libil2cpp.so!0x3A5D078'
```
```javascript
{
  onEnter(log, args, state) {
log(`sub_3a5d078(${args[0].sub(Process.findModuleByName("libil2cpp.so").base)}, ${args[1]}, ${args[2]})`);
    state.ptr = args[0];
    log(`==> ${state.ptr.readPointer()}`);
  },
  onLeave(log, retval, state) {
    log(`<== ${state.ptr.readPointer()}`);
  }
}
```
```
  1167 ms  sub_3a5d078(0x99124d0, 0x1, 0xb400007d85d38900)
  1167 ms  0x2000f3eb
  1168 ms  0x7d143167c0
  1168 ms  sub_3a5d078(0x9967768, 0x1, 0x0)
  1168 ms  0xc0070c03
  1168 ms  0x7d1431b8a0
  1168 ms  sub_3a5d078(0x9a04f30, 0x1, 0xb400007d85abb798)
  1168 ms  0xa00154fb
  1168 ms  0x7d3e8e27d0
  1168 ms  sub_3a5d078(0x9a04f38, 0x1, 0x32)
  1168 ms  0xa00154fd
  1168 ms  0x7d3e8e2780
```
注意到第一个参数是个指针，而且指向ELF中的区域，并且指针的目标数据在调用前后发生了变化。

查看ELF中的对应区域（也就是输出中调用前的值），发现全部都满足Usage Pairs的结构条件，也就是说我们苦苦寻找的数据区段，并不在Metadata里面。这个时候我想起来IL2CppDumper代码中，对v27+的Metadata文件，并没有读取相关区段，[而是在ELF中进行查找](https://github.com/Perfare/Il2CppDumper/blob/217f1d4737cd9d9d16ab5bef355156bcbc44f9e0/Il2CppDumper/Outputs/StructGenerator.cs#L266)，就开始觉得不对劲了，难不成版本不对？

先继续进行处理，注意到每一项在`.got`中都有引用，而`.got`中的顺序是有意义的：
```
.data:00000000099124D8 qword_99124D8   DCQ 0x2003077F          ; DATA XREF: il2cpp:000000000514623C↑r
.data:00000000099124D8                                         ; .got:off_96975A0↑o
.data:00000000099124E0 qword_99124E0   DCQ 0x20030781          ; DATA XREF: il2cpp:0000000005145D28↑r
.data:00000000099124E0                                         ; .got:off_9694B70↑o
.data:00000000099124E8 qword_99124E8   DCQ 0x2000F3F1          ; DATA XREF: il2cpp:0000000003B9C92C↑r
.data:00000000099124E8                                         ; il2cpp:0000000003B9D4C0↑r ...
.data:00000000099124F0 qword_99124F0   DCQ 0x20030783          ; DATA XREF: il2cpp:loc_3C7DF80↑r
.data:00000000099124F0                                         ; .got:off_9625870↑o
.data:00000000099124F8 qword_99124F8   DCQ 0x20030785          ; DATA XREF: il2cpp:loc_8D2E62C↑r
.data:00000000099124F8                                         ; .got:off_9710F08↑o
.data:0000000009912500 qword_9912500   DCQ 0x20030787          ; DATA XREF: il2cpp:0000000008F12634↑r
.data:0000000009912500                                         ; .got:off_9718F28↑o
.data:0000000009912508 qword_9912508   DCQ 0x2000F3FD          ; DATA XREF: il2cpp:loc_85BDDF8↑r
.data:0000000009912508                                         ; il2cpp:00000000085BDE1C↑r ...
.data:0000000009912510 qword_9912510   DCQ 0x20030789          ; DATA XREF: il2cpp:0000000004F4B3A4↑r
.data:0000000009912510                                         ; il2cpp:loc_4F4CBD0↑r ...
```
于是编写脚本，以`.got`中的顺序输出字符串引用：
```python
import lief
import struct, json

lib = lief.ELF.parse("libil2cpp.so")
raw_lib = open("libil2cpp.so", "rb").read()
out_strings = []

with open('stringliteral.json', 'rb') as f:
    strings = json.load(f,)

missing_strings = set(range(len(strings)))

for rel in lib.relocations:
    target = raw_lib[rel.addend:rel.addend+4]
    if len(target) != 4:
        continue

    encoded,  = struct.unpack('<I', target)
    usage = (encoded & 0xE0000000) >> 29
    index = (encoded & 0x1FFFFFFE) >> 1

    if usage == 5 and index < len(strings):
        missing_strings.discard(index)
        out_strings.append(strings[index])

print("Refs:",  len(out_strings))
print("Not referenced in GOT", missing_strings)

with open('strings.json', 'w', encoding='utf-8') as f:
    json.dump(out_strings, f, indent=4, ensure_ascii=False)
```
细心的朋友可能注意到了，在解析`index`时使用的公式，和上文并不一致，这是因为...
# String Literals是如何存储的（v27+）
我们找错版本啦！
{% asset_img "Notes_240609_225337.jpg" %}
上面我们逆向分析出来的加载机制，实际上和v27和以上版本的加载机制完全一致：字符串是全局变量放在`.got`里面，GOT指向Usage Pairs的位置，`IntializeMethodMetadataRange`函数接收指向Usage Pair的指针，在Metadata中查找相关的字符串或其他结构，并分配内存，用该结构的指针覆盖Usage Pair。

尝试将`global-metadata.data`中的版本号直接由0x18改为0x1D，不做任何其他处理，并使用`Il2CppDumper`的最新版（注意之前的版本对v29的处理有问题，这里也卡了很久），配合🐱从内存中获取的解除保护的Dump文件，终于成功解析了所有内容，绕了个巨大的圈子：
```
> Il2CppDumper.exe Z:/7452280000_com.ChillyRoom.DungeonShooter_libil2cpp.so 'Z:/global-metadata.dat'
Initializing metadata...
Metadata Version: 29
Initializing il2cpp file...
Applying relocations...
Il2Cpp Version: 29
Detected this may be a dump file.
Input il2cpp dump address or input 0 to force continue:
7452280000
Searching...
Change il2cpp version to: 29.1
CodeRegistration : 745b332f20
MetadataRegistration : 745b6629d8
Dumping...
Done!
Generate struct...
Done!
Generate dummy dll...
Done!
Press any key to exit...
```
比较可惜的是，使用工具导出的字符串表，依旧是以A-Z的顺序排序的，无法满足最早的需求，所以或许之前的那个巨大的Rabbit Hole，还是必须跳的呢？

# 总结
* 逆向工程用的工具，尽量还是用最新版（至少不要“不小心”用到旧版）
* 某些场景下，一些表象是障眼法的可能性比真的花大力气做了处理的可能性大
* Frida的可用性真的是玄学，🐱换了两台机器都没跑起来脚本，我自己在处理的时候也经常需要从好好的`-f`启动应用改成传入进程名称然后拼手速