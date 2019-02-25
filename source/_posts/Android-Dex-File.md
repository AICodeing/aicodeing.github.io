---
title: Android-Dex File
date: 2019-02-25 14:35:48
tags:
	- Android
	- Dalvik
	- Dex
category: Android
comments: ture
---

文章[Android-Dalvik And ART](/Android-Dalvik-And-ART/)中介绍了，`ART`和`Dalvik`两种虚拟机，这次介绍以下 他们的可执行文件 `.dex`。

### Dex 文件的生成

`Android`开发，通常是使用 `Java`语言进行开发 (即使是 `Kotlin`最终也是编译成 Java 字节码)。而 	`Java`文件的编译产物就是 `.class`字节码文件，而 Android的`Dalvik`虚拟机或 `ART`是不能直接运行 `.class` 文件的，需要将 `.class`文件转成 `.dex`文件。目前 Android 官方提供两种方式，一种是 `dx`工具，另一种是`d8`工具。
这两种编译工具在 Android SDK Platform 的 build-tools目录；  
如: `your_sdk_path/build-tools/28.0.3`, `28.0.3`是 version，如果你有其他 version的编译工具，替换 version 即可。  在 该目录下，存在 `dx`和 `d8`两个可执行文件。    

#### 生成 class 文件
例如 我编写了一个Hi.java 文件  

```java
public class Hi {

	private static String text = "Hi Class";

	public static void main(String[] args){
		System.out.println("main -->");
		Hi hi = new Hi();
		hi.print(text);
	}

	public void print(String text){
		System.out.println(text);
	}
}
```

通过执行 `javac`命令把 `Hi.java` 文件编译换成 `Java` 字节码 `.class` 文件  

```shell
➜  ~ javac Hi.java
```
在当前目录生成文件 `Hi.class`，由于 class文件是 Java虚拟机的可执行文件，所以可以直接使用 `java` 命令执行  

```
➜  ~ java Hi
main -->
Hi Class
```

#### 生成 dex 文件

有了 `.class`文件之后，利用 `dx`或 `d8`工具生成 `.dex`文件  

```shell
➜  ~ dx --dex  --output=Hi.dex Hi.class
```

生成 `Hi.dex`文件

或 使用 `d8`编译器:  

```shell
➜  ~ d8 Hi.class
```
在当前目录生成了 `classes.dex`文件

这个 `.dex`文件就可以在Android 运行时环境直接执行。  


### Dex 文件

现在分析一下 `Dex`的格式, `Dex`文件的定义在 `dalvik/libdex/DexFile.h`中

```c++
/*
 * Structure representing a DEX file.
 *
 * Code should regard DexFile as opaque, using the API calls provided here
 * to access specific structures.
 */
struct DexFile {
    /* directly-mapped "opt" header */
    const DexOptHeader* pOptHeader;

    /* pointers to directly-mapped structs and arrays in base DEX */
    const DexHeader*    pHeader;
    const DexStringId*  pStringIds;
    const DexTypeId*    pTypeIds;
    const DexFieldId*   pFieldIds;
    const DexMethodId*  pMethodIds;
    const DexProtoId*   pProtoIds;
    const DexClassDef*  pClassDefs;
    const DexLink*      pLinkData;

    /*
     * These are mapped out of the "auxillary" section, and may not be
     * included in the file.
     */
    const DexClassLookup* pClassLookup;
    const void*         pRegisterMapPool;       // RegisterMapClassPool

    /* points to start of DEX file data */
    const u1*           baseAddr;

    /* track memory overhead for auxillary structures */
    int                 overhead;

    /* additional app-specific data structures associated with the DEX */
    //void*               auxData;
};
```

图形化表示:  

![](/img/dex/dex_struct.jpg)

在介绍之前，先铺垫一些符号的定义  

|Name|Desc|
|---|---|
|byte|8位的有符号数|
|ubyte|8位的无符号数|
|short|16位的有符号数|
|ushort|16位的无符号数|
| int|32位的有符号数|
| uint| 32位的无符号数|
| long|64位的有符号数|
| ulong|64位的无符号数|
| sleb128|有符号的 LEB128格式,可变长度|
| uleb128|无符号的 LEB128格式，可变长度|
|uleb128p1|无符号的 LEB128格式 +1，可变长的|

什么是 `LEB128` 呢？ LEB128是小端数据格式，它用于任意有符号或无符号整数量的可变长度编码。在 dex 文件中，`LEB128`仅用于编码32位数量。  
每个LEB128编码值由一到五个字节组成，它们一起代表一个32位值。除了最后一个字节之外，每个字节都有一个符号位，位于每个字节的最高位，其余的7位是 有效数据。这样的一个字节或多个字节就形成了 `LEB128数值`。  

对于一个LEB128的有符号数据(`sleb128`)，最后一个字节的最高位被拓展成为这个LEB128数值的最终有符号信息，1表示负数，0表示正数。对于一个无符号的LEB128数值(`uleb128`)，最后一个字节的最高位，无论是1还是0，始终看做0，以此表示无符号。

`uleb128p1` 是 `uleb128`的变种，规则是：uleb128p1 + 1 = uleb128

既然是 小端数据格式，以 2字节举例: 

![](/img/dex/leb128_art.jpg)


回到Dex 文件的布局

#### DexHeader

在 `DexFile` 中 第一块数据结构就是 `DexHeader`

```c++
/*
 * Direct-mapped "header_item" struct.
 */
struct DexHeader {
    u1  magic[8];           /* includes version number */
    u4  checksum;           /* adler32 checksum */
    u1  signature[kSHA1DigestLen]; /* SHA-1 hash */
    u4  fileSize;           /* length of entire file */
    u4  headerSize;         /* offset to start of next section */
    u4  endianTag;
    u4  linkSize;
    u4  linkOff;
    u4  mapOff;
    u4  stringIdsSize;
    u4  stringIdsOff;
    u4  typeIdsSize;
    u4  typeIdsOff;
    u4  protoIdsSize;
    u4  protoIdsOff;
    u4  fieldIdsSize;
    u4  fieldIdsOff;
    u4  methodIdsSize;
    u4  methodIdsOff;
    u4  classDefsSize;
    u4  classDefsOff;
    u4  dataSize;
    u4  dataOff;
};
```

__`magic[8]`__ 存储8字节的magic，用来标记 这是一个 Dex 文件。

```
ubyte[8] DEX_FILE_MAGIC = { 0x64 0x65 0x78 0x0a 0x30 0x33 0x38 0x00 }
                        = "dex\n038\0"
```

`038`就是 Dex的版本号  添加的 `\n`和`\0`就是来区分 字符串 dex 和版本号的,`038`版本是Android 8.0 开始支持的。

__checksum__ 用于校验Dex 文件的，参与计算的数据是除了 magic 和checksum 两个字段外的所有数据  

__signature__ 除了 magic和 checksum 和 signature 外的部分的 SHA-1签名，用于唯一标识文件

__file_size__ 整个文件的大小

__header_size__ Header 部分的大小

__endian_tag__ 字节序

```
uint ENDIAN_CONSTANT = 0x12345678; //大端序
uint REVERSE_ENDIAN_CONSTANT = 0x78563412;//小端序，如果是小断序，需要反转成正常的大端序
```
__link_size__ 链接部分的大小，如果此文件未静态链接，则为0

__link_off__ 从文件的开头到链接部分的偏移量，或者如果link_size == 0则为0.偏移量（如果非零）应该是link_data部分的偏移量

__map_off__ 从文件的开头到 map item 的偏移量。偏移量必须为非零，应该是数据部分的偏移量，data 应采用下面“map_list”指定的格式。

__string_ids_size__ 字符串标识符列表中的字符串数
__string_ids_off__ 从文件的开头到字符串标识符列表的偏移量，如果string_ids_size == 0，则为0（无可否认是一个奇怪的边缘情况）。偏移量（如果非零）应该是string_ids部分的开头。

__type_ids_size__ 类型标识符列表中的元素数，最多为65535

__type_ids_off__ 从文件开头到类型标识符列表的偏移量，如果type_ids_size == 0，则为0（无可否认是一个奇怪的边缘情况）。偏移量（如果非零）应该是type_ids部分的开头。

__proto_ids_size__ 原型标识符列表中的元素数，最多为65535
__proto_ids_off__ 从文件的开头到原型标识符列表的偏移量，如果proto_ids_size == 0，则为0（无可否认是一个奇怪的边缘情况）。偏移量（如果非零）应该是proto_ids部分的开头。

类似的 `field_ids_size `,`method_ids_size `,`class_defs_size `,`data_size ` 分别对应着Dex File 不同的区域的大小。相应的`*_off`对应着从文件头到各自区域的偏移量

_提一句，Android上常见的 方法数 超过 65535 问题就是因为 `method_ids_size`使用无符号int来标识，而无符号int的最大范围是 65535。所以就限制了一个 Dex 文件中最多只能索引 65535 个方法。所以 后期的版本，Google 采用了分割 Dex 的方式解决这个问题。_

#### DexStringId Section

该区域存储 字符串标识符列表。存储着使用的所有字符串的标识符，如代码引用的常量对象。

#### DexTypeId Section

存储着 类型标识符列表。
这些是此文件引用的所有类型（类，数组或基本类型）的标识符，无论是否在文件中定义。
此列表必须按字符串索引排序，并且不得包含任何重复的条目

#### DexFieldId Section
存储字段标识符列表。
这些是此文件引用的所有字段的标识符，无论是否在文件中定义。
必须对此列表进行排序，其中定义类型（通过type_id索引）是主要顺序，字段名称（通过string_id索引）是中间顺序，类型（通过type_id索引）是次要顺序。
该列表不得包含任何重复的条目。

#### DexMethodId Section
方法标识符列表。
这些是此文件引用的所有方法的标识符，无论是否在文件中定义。
必须对此列表进行排序，其中定义类型（通过type_id索引）是主要顺序，方法名称（通过string_id索引）是中间顺序，方法原型（通过proto_id索引）是次要顺序。
该列表不得包含任何重复的条目。

#### DexProtoId Section
方法原型标识符列表。
这些是此文件引用的所有原型的标识符。
此列表必须按返回类型（按type_id索引）主要顺序排序，然后按参数列表排序（词典排序，按type_id索引排序的各个参数）。
该列表不得包含任何重复的条目。

proto 的意思是 method prototype 代表 java 语言里的一个 method 的原型 。proto_ids 里的元素为 proto_id_item，结构如下:
```
/*
 * Direct-mapped "proto_id_item".
 */
struct DexProtoId {
    u4  shortyIdx;          /* index into stringIds for shorty descriptor */
    u4  returnTypeIdx;      /* index into typeIds list for return type */
    u4  parametersOff;      /* file offset to type_list for parameter types */
};
```
包含 方法名称，方法的返回值，参数。

#### DexClassDef Section

类定义列表。
必须对类进行排序，使得给定类的超类和实现的接口在引用类之前出现在列表中。

#### Data Section

数据区，包含上面列出的表的所有支持数据。

#### Link Data
静态链接文件中使用的数据。

### 分析 Hi.dex

使用编辑器打开 `Hi.dex`文件

```
6465 780a 3033 3500 *(magic) 4e7d 2ba4 *(checksum) b499 ca35
ee34 b9d6 94de 96e8 a865 9759 ae06 7dee *(signature)
bc03 0000 *(file_size) 7000 0000 *(header_size) 7856 3412 *(endian_tag) 0000 0000 *(link_size)
0000 0000 *(link_off) 1003 0000 *(map_off) 1300 0000 *(string_ids_size) 7000 0000 *(string_ids_off)
0700 0000 *(type_ids_size) bc00 0000 *(type_ids_off) 0300 0000 d800 0000
0200 0000 fc00 0000 0600 0000 0c01 0000
0100 0000 3c01 0000 6002 0000 5c01 0000
0602 0000 1002 0000 1802 0000 2202 0000
2b02 0000 3102 0000 4802 0000 5c02 0000
7002 0000 8402 0000 8702 0000 8b02 0000
a002 0000 a602 0000 b002 0000 b502 0000
bc02 0000 c502 0000 cb02 0000 0400 0000
0500 0000 0600 0000 0700 0000 0800 0000
0900 0000 0b00 0000 0900 0000 0500 0000
0000 0000 0a00 0000 0500 0000 f801 0000
0a00 0000 0500 0000 0002 0000 0000 0300
1100 0000 0400 0100 0e00 0000 0000 0000
0000 0000 0000 0000 0100 0000 0000 0200
0c00 0000 0000 0100 0f00 0000 0100 0100
1000 0000 0200 0000 0100 0000 0000 0000
0100 0000 0200 0000 0000 0000 0300 0000
0000 0000 f202 0000 0000 0000 0100 0000
0000 0000 ec01 0000 0500 0000 1a00 0200
6900 0000 0e00 0000 0100 0100 0100 0000
e001 0000 0400 0000 7010 0500 0000 0e00
0200 0100 0200 0000 e401 0000 1200 0000
6201 0100 1a00 0d00 6e20 0400 0100 2201
0000 7010 0100 0100 6200 0000 6e20 0300
0100 0e00 0300 0200 0200 0000 f001 0000
0600 0000 6200 0100 6e20 0400 2000 0e00
0100 0e00 0601 000e 785a 5a00 0300 0e00
0c01 000e 5a00 0000 0100 0000 0300 0000
0100 0000 0600 083c 636c 696e 6974 3e00
063c 696e 6974 3e00 0848 6920 436c 6173
7300 0748 692e 6a61 7661 0004 4c48 693b
0015 4c6a 6176 612f 696f 2f50 7269 6e74
5374 7265 616d 3b00 124c 6a61 7661 2f6c
616e 672f 4f62 6a65 6374 3b00 124c 6a61
7661 2f6c 616e 672f 5374 7269 6e67 3b00
124c 6a61 7661 2f6c 616e 672f 5379 7374
656d 3b00 0156 0002 564c 0013 5b4c 6a61
7661 2f6c 616e 672f 5374 7269 6e67 3b00
046d 6169 6e00 086d 6169 6e20 2d2d 3e00
036f 7574 0005 7072 696e 7400 0770 7269
6e74 6c6e 0004 7465 7874 0025 7e7e 4438
7b22 6d69 6e2d 6170 6922 3a31 2c22 7665
7273 696f 6e22 3a22 7631 2e30 2e33 3522
7d00 0100 0301 000a 0088 8004 dc02 0181
8004 f802 0109 9003 0301 c403 0000 0000
0e00 0000 0000 0000 0100 0000 0000 0000
0100 0000 1300 0000 7000 0000 0200 0000
0700 0000 bc00 0000 0300 0000 0300 0000
d800 0000 0400 0000 0200 0000 fc00 0000
0500 0000 0600 0000 0c01 0000 0600 0000
0100 0000 3c01 0000 0120 0000 0400 0000
5c01 0000 0320 0000 0400 0000 e001 0000
0110 0000 0200 0000 f801 0000 0220 0000
1300 0000 0602 0000 0020 0000 0100 0000
f202 0000 0310 0000 0100 0000 0c03 0000
0010 0000 0100 0000 1003 0000 
```

`6465 780a 3033 3500` 对应的 `0x64 0x65 0x78 0x0a 0x30 0x33 0x35 0x00`
 = `"dex\n035\0"` 说明 我使用 的 `dx` 编译器 使用的Dex 版本是 `035`

在 以上数据中，我使用*(name)做了部分分割，只有明确 Dex 文件的结构，读懂 Dex 文件也不是那么的难了，有了 这个基础之后，以后就可以明白 各种热修复框架的实现思路了。

















