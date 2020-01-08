---
 layout:     post
 title:      Unity AssetBundle
 subtitle:   文件格式解析 
 date:       2020-01-08
 author:     Bob
 header-img: img/post-bg-unity.jpg
 catalog: true
 tags:
     - Unity
---

Unity 生产资源内部文件格式说明：

- [Asset Bundle Internal Structure](https://docs.unity3d.com/530/Documentation/Manual/AssetBundleInternalStructure.html)

# 文件类型

## 序列化文件(serializedFile)

 序列化文件是AssetBundle的重要组成，本质上是对象和元数据的集合.

### 文件头(header)

文件以0x14字节开头.  文件头中数据都是以big-endian(大端模式:数据的低位保存在内存的高地址中，而数据的高位保存在内存的低地址中。小端模式：数据的低位保存在内存的低地址中，而数据的高位保存在内存的高地址中存储.

```c
struct SerializedFileHeader {
  int32 metadataSize;
  int32 fileSize;
  int32 version;
  int32 objectDataOffset;
  bool bigEndian;
  char _padding[3];
};
```

- *`metadataSize`*: 元数据大小.
- *`fileSize`*: 整个序列化文件文件大小.
- *`version`*: 版本号
- *`objectDataOffset`*: 从文件开头到序列化对象数据开头的字节偏移量.
- *`bigEndian`*: 当为true, 文件剩余数据用big-endian存储. 当为false,  文件剩余数据用little-endian存储. 文件前0x14字节的文件头始终用big-endian存储.

当version < 9 时, `bigEndian`字段不存在, 文件全部用big-endian存储, 文件头仅为0x10字节.

### 元数据(metadata)

元数据包含一些可变成字符串，空字符结束，比如"5.3.6p1" 将写入 `35 2E 33 2E 36 70 31 00` ，8个字节包含空字符。


格式排布说明:

- `string generatorVersion`: 生成文件使用的Unity软件版本, 例如 "2018.2.20f1".
- `int32 platform`: 平台的枚举值，可参考 [RuntimePlatform enum](https://docs.unity3d.com/ScriptReference/RuntimePlatform.html).
- 当 version >= 13:
  - `bool hasTypeTrees`: TypeTree数据是否包含在metadata中
  - `int32 numTypes`: 序列化文件中对象的个数
  - 遍历numTypes个Type:
    - 当 version  >= 17:
      - `int32 classID`: 0x72 (114: MonoBehaviour)是脚本类型
      - `int8 ???`
      - `int16 ???`
    - 否则 (version < 17):
      - `int32 classID`: Negative for script types
    - 如果 [classID](https://docs.unity3d.com/Manual/ClassIDReference.html) 表示为脚本类型: 
      - `char scriptHash[16]`
    - `char typeHash[16]`
    - 如果 hasTypeTrees:
      - `TypeTree typeTree`
- 否则 (version < 13):
  - `int32 numTypes`
  - 遍历numTypes个Type:
    - `int32 classID`: Negative for script types
    - `TypeTree typeTree`
- 当 7 <= version 且 version <= 13:
  - `int32`: (unused)
- `int32 numObjectInfos`
- 遍历numObjectInfos个objectInfo:
  - 当 version >= 14:
    - 将流与下一个4字节边界对齐 (相对于元数据部分的开头)
    - `int64 objectID`
  - 否则:
    - `int32 objectID`
  - `int32 dataOffset`: This is added to the `objectDataOffset` value in the header to determine the file offset to the object data.
  - `int32 dataSize`: 序列化对象的大小（以字节为单位）.
  - 当 version < 17:
    - `int32 typeID`
    - `int16 classID`
  - 否则:
    - `int32 typeIndex`: an index in to the array of type information given by looping `numTypes` above.
  - 当 version <= 10:
    - `int16 isDestroyed`: (未使用)
  - 当 11 <= version 且 version <= 16:
    - `int16`: 未知.  Starting in version 17 this is the `int16` read in the loop over `numTypes` above.  默认为0xffff.
  - 当 15 <= version 且 version <= 16:
    - `int8`: 未知.  Starting in version 17 this is the `int8` read in the loop over `numTypes` above.  默认为0.
- 当 version >= 11:
  - `int32 numAdds`
  - 遍历numAdds个Add:
    (Serialize a PPtr)
    - `int32 fileID`
    - 当 version >= 14:
      - 将流与下一个4字节边界对齐
      - `int64 pathID`
    - 否则:
      - `int32 pathID`
- `int32 numExternalFiles`
- 遍历numExternalFiles个externalFile:
  - 当 version >= 6
    - `string assetName`
  - 当 version >= 5
    - `int32 guid[4]`
    - `int32 type`
  - `string fileName`
- 当 version >= 5
  `string`: 未知, 始终为空.

#### TypeTree

TypeTree描述了如何序列化单个对象中的值。它是一个树形结构，其中每个节点都是一个struct字段。可以通过深度优先遍历此树并根据每个节点上的信息进行读取或写入来执行对象序列化。

较新的TypeTree格式的类型和字段名称不是直接存储为字符串，而是存储为两个字符串缓冲区之一的偏移量：全局字符串缓冲区，在引擎中定义为常量字符串；和本地字符串缓冲区，包括在TypeTree中。这些字符串缓冲区包含顺序存储的以空终止的字符串。当偏移量值设置了第31位（即为负）时，该位被屏蔽以在全局字符串缓冲区中获取偏移量。如果未设置位31，则偏移量位于本地字符串缓冲区中。

- 当 version == 10 或 version >= 12:  
  (新的紧凑型 blob 格式)
  - `int32 numNodes`: 树中的节点数
  - `int32 stringBufferSize`: 本地字符串缓冲区的大小（以字节为单位）
  - 遍历numNodes个node:
    - `int16 version`: .
    - `int8 depth`: The depth in the tree of the current node.  Nodes of the tree are serialized depth-first, so this number will increase when the current node is a child of the previous node, will stay the same when the current node is a sibling of the previous node, and will decrease when the current node is a sibling of one of the previous node's parents.
    - `bool array`: When true, this node is a special array node -- its first child (`size`) in the tree is the size in elements of the array, and its next child (`data`) is serialized in a loop for each element of the array.
    - `int32 type`: 此节点的类型名称的字符串缓冲区偏移量.
    - `int32 name`: 此节点的字段名称的字符串缓冲区偏移量.
    - `int32 size`: The expected size in bytes when this node (including children) is serialized.  This is -1 for variable-sized fields, such as arrays or structs that have arrays as children.
    - `int32 index`: This is just an index of the node in the flat depth-first list of nodes.
    - `int32 flags`: 序列化和其他信息的标志.
      - 0x4000: 序列化此字段后，应对齐流
  - `char stringBuffer[stringBufferSize]`: 本地字符串缓冲区
- 否则 (version != 10 且 version < 12):  
  (老格式; 上面描述了字段)
  - `string type`
  - `string name`
  - `int32 size`
  - `int32 index`
  - `int32 array`: 这仍然是布尔值（0或1），只有4个字节，而blob格式为1个字节
  - `int32 version`
  - `int32 flags`
  - `int32 numChildren`
    - For each child in numChildren, recurse starting at `string type`  
      Note that this is explicitly listing children, as opposed to the blob format which just uses `depth` to determine child/parent relationships.

### Objects

元数据中找到的每个`ObjectInfo`的值都对应于组成其余部分的`SerializedFile`中的一个序列化对象
这些对象的类型可以在TypeTree中找到, 类型可以是native Unity Engine类型, 例如 Texture2D 或 TextAsset, 也可以是源自MonoBehaviour的serialized script 类型.  Native Unity Engine types根据native code序列化, 通常会镜像native structure layout; 一个非常早期的TypeTrees的native types快照可参见该[链接文档](https://gist.githubusercontent.com/Mischanix/7db0145e809b692b63f2/raw/0ae1905171cc38dbfb68994c3cb679c3b8bf9e0c/structs.dump).  Script types通过迭代class's serializable fields--serializable fields来序列化 ，由所有公用字段和未标记为 `[NonSerialized]` 类的基类 和 标记为`[SerializeField]`所有其他字段组成.

当序列化字段是对另一个Object的引用时, 它被序列化为 `PPtr<T>` T 继承Object的位置.  PPtr由 `fileID` 和`pathID`组成.  当 `fileID` 为0, 它引用当前文件.  否则, PPtr 引用在metadata中 externalFiles数组中索引为 `fileID - 1`的文件.  `pathID` corresponds to the `objectID` found in the ObjectInfos section of the referenced SerializedFile.  从version 14开始, `pathID` 变成  64-bit integer,并且在将每个`pathID`列化之前，将流与下一个32位边界对齐;version 14之前的版本, `pathID` 是  32-bit integer, 并且在序列化流之前未对齐流.  另外, 64-bit pathIDs 通常是哈希, 而32-bit pathIDs 通常是 sequential indices.

当对变长类型（如数组或字符串）进行序列化时，它将以`int32 size`开始序列化，用于确定数组的个数，然后按顺序序列化所有数组元素。dictionary 或 map序列化为成对的数组，例如 `[int32 size] [[K first] [V second]] [[K first] [V second]] ...`.

## Asset Bundles

AssetBundle文件包含一个SerializedFile。 AssetBundle具体使用参见文档 [Unity documentation](https://docs.unity3d.com/ScriptReference/AssetBundleBuild.html).

### Asset Bundles文件格式构成

除非另有说明，否则此头中的值均为big-endian

- `string signature`:  AssetBundle签名字符串，以空字符结束, 取值为 `UnityFS`, `UnityWeb`, `UnityRaw`, `UnityArchive`.  剩余内容格式依赖于此签名.

- 当 signature 是 `UnityFS`: 
    - `uint32 formatVersion`: 1~6
    - `string targetVersion`  5.x.x
    - `string generatorVersion` 2018.2.20f1
    - 当 formatVersion < 6:
        - `uint32 MinimumStreamedBytes`
        - `uint32 HeaderSize`
        - `uint32 TotalChunkCount`
        - 当 formatVersion >= 2:
            - `uint32 bundleSize`
        - 当 formatVersion >= 3:
            - `uint32 metaDatauncompressedSize`
    - 否则:
        - `uint64 bundleSize`
        - `uint32 metaDatacompressedSize`
        - `uint32 metaDatauncompressedSize`
        - `uint32 flag` 

- 当 signature 是 `UnityArchive`:  
  
  - `int64 headerOffset`: Seek to this offset to begin reading the header:
    - `uint32 formatVersion`: 5、6等值.
    - `string targetVersion` 
    - `string generatorVersion` 
    - `char guid[16]`
    - `uint32`
    - `uint32`
    - `uint32 storageInfoOffset`: Add to header offset and seek to begin reading the storage information:
      - `uint32 dataOffset`
      - `uint32 numNodes`
      -  遍历numNodes个node:
        - `uint64 dataOffset`
        - `uint64 dataSize`
        - `uint32 status`
        - `string name`
      - `uint32 numBlocks`
      - `uint64 firstBlockOffset`
      - 遍历numBlocks个block:
        - `uint64 nextBlockOffset`:解压缩的块大小是通过从该偏移量减去先前的偏移量来计算的。
      - 遍历numBlocks+1个block:
        - `uint64 fileOffset`: 通过从此偏移量减去前一个偏移量来计算前一个块的压缩块大小。对于此循环中的第一次迭代，将跳过此计算.
        - `uint32 compressionType`
        - `uint32 flag_40`: &1 等价于StorageBlock.flags & 0x40; 没有其他意义
- 当 signature 是 `UnityWeb` 或 `UnityRaw`:
  - `uint32 formatVersion`
  - `string targetVersion`
  - `string generatorVersion`
  - 当 formatVersion >= 4:
    - `char guid[16]`
    - `uint32`
  - `uint32 fileSize`
  - `uint32 headerSize`: aka dataOffset
  - `uint32 unkCount0`
  - `uint32 unkCount1`
  - for each i in unkCount1:
    - `uint32 uncompressedSize`
    - `uint32 compressedSize`
  - 当 formatVersion >= 2:
    - `uint32`
  - 当 formatVersion >= 3:
    - `uint32`


## Flat resource

*`*.resource` and `*.resS`* 文件是 flat resource files, 通常是音频或纹理数据, and are viewed 并且被Unity视为字节序列。  对于音频数据，字节是FMOD声音库文件，可以直接传递给FMOD以创建可播放的声音.  对于texture数据, 字节是texture image 数据。音频或图像数据的每个段的位置和长度在单个AudioClip或Texture2D对象内的同名asset文件中。