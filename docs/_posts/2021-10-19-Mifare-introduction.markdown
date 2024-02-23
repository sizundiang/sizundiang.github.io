---
layout: post
title: Mifare介绍
author: sizundiang
---

最近研究了下通过卡应用对卡内Mifare内存区域进行操作访问的可行性，顺便对Mifare的结构与机制做了一定的了解。本文将重点梳理Mifare内存区域的结构及安全机制，并结合现有设备，介绍几种对Mifare内存区域进行操作访问的方式.  

Mifare是一个很大的称呼，其下有多个产品分支，粗略分一下，大致有Mifare Classic系列，MifareUltralight系列和Mifare Desfire系列，其中，Mifare Classic系列有1K和4K两种内存大小的产品，篇幅所限，本文只讲述Mifare Classic相关的内容，sim卡使用的也是Mifare Classic的相关技术(基本确认)，文章后续包括标题等，所提到的Mifare，均指**Mifare Classic 1K/4K**，1K/4K表示内存区域大小。  

### 一、Mifare内存结构
Mifare的基本组织结构有扇区(sector)和区块(block)的概念，一个Mifare内存有多个扇区，每个扇区有多个区块，一个区块固定为16个字节。  

对于Mifare Classic 1K产品，其内存有16个扇区，每个扇区固定4个区块，对于Mifare Classic4K产品，其内存有40个扇区，其中前32个扇区固定4个区块，与Mifare Classic 1K的一致，后面8个扇区，每个扇区固定16个区块。  

扇区的索引从0开始，因此对于Mifare Classic 1K，扇区索引为0-15；对于Mifare Classic 4K，扇区索引为0-39。  

区块的索引有两种方式，分别为绝对位置索引和相对位置索引，对于Mifare Classic 1K，绝对位置的区块索引为0-63，相对位置索引下，一个扇区内的区块索引为0-3，其换算关系为
> 扇区索引 * 4 + 相对位置索引 = 绝对位置索引  

对于Mifare Classic 4K，绝对位置的区块索引为0-255，相对位置索引下，前32个扇区的扇区内区块索引为0-3，后8个扇区的扇区内区块索引为0-15，其换算关系为
> 127 + (扇区索引 mod 32) * 16 + 相对位置索引 = 绝对位置索引  

Mifare Classic 1K的内存结构如下：  
![M1 structure.png](/assets/images/2021-10-19-Mifare-introduction/M1-structure.png)

Mifare Classic 4K的内存结构如下：  
![M4 structure.png](/assets/images/2021-10-19-Mifare-introduction/M4-structure.png)

区块是有类型区别的，Mifare的区块共有四种类型，分别是厂商区块(Manufacturer Block)，数据区块(Data Block)，值区块(Value Block)和尾部区块(Sector Trailer Block)，各块的性质及作用介绍如下：  

- **厂商区块**
0扇区的区块0为厂商专用区块，该区块只允许读取(read-only)，不允许修改，该区块存储4字节或7字节的UID(**U**nique **ID**entifier)，UID将在读卡器寻卡和选卡时返回给读卡器；剩余字节用以存储厂商专用数据(Manufacturer Data)。厂商区块的结构如下  

![Manufacturer block 1.png](/assets/images/2021-10-19-Mifare-introduction/manufacturer-block-1.png)

![manufacturer block 2.png](/assets/images/2021-10-19-Mifare-introduction/manufacturer-block-2.png)

- **数据区块**
每个扇区，除了最后一个区块(即尾部区块)，其余区块均为数据区块( 0扇区的区块0为厂商专用区块)，可用以存储数据，对于Mifare 1K，数据区块每个扇区有3个，对于Mifare 4K，数据区块每个扇区有3个或15个。可以对数据区块进行读/写操作，数据在区块内无格式限制。  

- **值区块**
值区块是用于特定应用，如电子钱包，电子票据一类的专用区块，有特定的格式，可以对数据值进行递增/递减的操作，并通过冗余的字段允许进行错误检测，值区块只允许被按照值区块格式编码的数据所操作，值区块的结构如下：  

![value block.png](/assets/images/2021-10-19-Mifare-introduction/value-block.png)

其中，~表示对应值的补码(2's complement)。  

- **尾部区块**
尾部区块是一个扇区的最后一个区块，该区块用以存储鉴权密钥KeyA和KeyB以及访问条件(Access Conditions)字节。  

KeyA和KeyB分别占6个字节，位于高6字节和低六字节，其中KeyA是必须的，KeyB是可选的，如果KeyB不存在，该部分字节用以存储普通数据，也因为有这种可能性，因此对Mifare的介绍中一般不建议采用KeyB进行鉴权的方式(但这只是针对一般Mifare产品来说的，对于智能卡的Mifare内存，有其他规定)。  

访问条件字节共有3个字节，规定了各类区块的访问条件，及是否可以通过鉴权来进行读写操作或不允许读写操作；剩余的一个字节用以存储用户数据。尾部区块的结构如下：  

![trailer block.png](/assets/images/2021-10-19-Mifare-introduction/trailer-block.png)

一般这些字段都有默认值，KeyA和KeyB默认值为0xFFFFFFFFFFFF，访问条件字节默认值为0xFF0780，剩余一个字节的默认值为0x69，当该区块被读取时，KeyA对应的字节将以全0返回，KeyB根据访问条件字节的定义决定是否可读。默认的访问条件字节值是允许KeyB被读取的。  

### 二、权限控制
如前文所述，一个扇区的访问权限由访问条件字节进行管理，共四个区块，每个区块由3位bits标识访问权限，如是否可以通过KeyA/KeyB进行访问，访问条件字节在其本身允许更新的情况下可以通过相关密钥进行修改。其中，各区块对应的访问位如下表所示：  

| Access Bits | Valid Commands | Block | Description |
| --- | --- |--- | --- | 
| C13 C23 C33 | read/write | 3 | Sector Trailer |
| C13 C23 C33 | read/write/increment/decrement | 2 | Data Block |
| C11 C21 C31 | read/write/increment/decrement | 1 | Data Block |
| C11 C21 C31 | read/write/increment/decrement | 0 | Data Block |

其中，第一个数字标识对应哪个半字节，第二个数字标识对应半字节中的哪一位，这也与所控制访问权限的区块数字相同，更具体地位与字节对应关系如下图所示，其中，带横线的表示对应位的补码：  

![bit map.png](/assets/images/2021-10-19-Mifare-introduction/bit-map.png)

1. **区块权限**

下面将罗列各类区块对应的访问条件，其中**Never**表示不允许操作，**KeyA**表示可以通过KeyA鉴权通过后操作，**KeyB**表示可以通过KeyB鉴权通过后操作，**KeyA/KeyB**表示可以通过校验KeyA或校验KeyB通过后操作。  

- **尾部区块(Sector Trailer Block)**
对于尾部区块，其位值组合对应的访问权限如下图所示：  

![Sector Trailer Block.png](/assets/images/2021-10-19-Mifare-introduction/sector-trailer-block.png)

这里应注意，虽然各部分的访问权限不同，但进行读取或写入时是对整个区块进行操作的，不满足条件的情况下，对应部分值会以别的形式，如全0，返回。  

- **数据区块(Data Block)**
对于数据区块，其位值组合对应的访问权限如下图所示：

![data block.png](/assets/images/2021-10-19-Mifare-introduction/data-block.png)

其中，Increment(递增)和Decrement(递减)、Transfer(转移)、Restore(复原)均是针对值区块的操作。  

2. **实例分析**
访问条件字节的默认值为0xFF0780，对应的位值为：  

|  | Block3 Bits | Block2 Bits | Block1 Bits | Block0 Bits | Block3 Bits | Block2 Bits | Block1 Bits | Block0 Bits |
| :---: | --- |--- | --- | --- | --- |--- | --- |  --- |
| **Byte6** | ~C2(3) | ~C2(2) | ~C2(1) | ~C2(0) | ~C1(3) | ~C1(2) | ~C1(1) | ~C1(0) |
| **0xFF** | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |
|  |  |  |  |  |  |  |  |  |
| **Byte7** | C1(3) | C1(2) | C1(1) | C1(0) | ~C3(3) | ~C3(2) | ~C3(1) | ~C3(0) |
| **0x07** | 0 | 0 | 0 | 0 | 0 | 1 | 1 | 1 |
|  |  |  |  |  |  |  |  |  |
| **Byte8** | C3(3) | C3(2) | C3(1) | C3(0) | C2(3) | C2(2) | C2(1) | C2(0) |
| **0x80** | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

对应各区块的访问条件位即为：  

|  | C1 | C2 | C3 | Description |
| :---: | --- |--- | --- | --- |
| **Sector Trailer** | 0 | 0 | 1 | KeyA不允许被读取，但可以通过鉴权KeyA进行更新，其他部分的读取与写入可以通过鉴权KeyA进行 |
| **Data Block 2** | 0 | 0 | 0 | 允许鉴权通过后的读写与针对值区块的特定操作 |
| **Data Block 1** | 0 | 0 | 0 | 同上 |
| **Data Block 0** | 0 | 0 | 0 | 同上 |

### 三、Mifare Password
Mifare Password是将Mifare内存区域的尾部区块存储的KeyA和KeyB按照一定算法进行运算而获取到的8字节密文，一个扇区对应一个Mifare Password，当KeyA和KeyB为默认值时，Mifare Password将相同。由于此处KeyB将参与鉴权密码的运算，因此KeyB理论上说不应用于普通数据存储，且应设置为不可读，在读取尾部区块时，应同KeyA一样，以全0返回，根据这样的规定，访问条件字节的配置也应不能采用默认的。  

Mifare Password的生成规则是，将6字节的KeyA和KeyB按照一定规则组装成两把8字节的密钥DKeyA和DkeyB，然后对8字节的0x00原始数据进行3Des加密运算，最终结果即为Mifare Password(MF Password)，流程演示如下图所示：  

![password generation flow.png](/assets/images/2021-10-19-Mifare-introduction/password-generation-flow.png)

DKeyA和DKeyB将分别由KeyA和KeyB产生，其规则是将KeyA和KeyB的各个位重新排列，并在剩余有效位填充0，同时补充奇校验位，从而得到8字节的密钥，首先，尾部区块的内容分布如下：  

![sector trailer block structure.png](/assets/images/2021-10-19-Mifare-introduction/sector-trailer-block-structure.png)

其中，LSByte表示最低有效字节(Least Significant Byte)，MSByte表示最高有效字节(Most Significant Byte)，可以看到两把密钥在尾部区块内是倒序存储的，尾部区块的最高有效字节存储的是密钥的最低有效字节。确认了其最低与最高有效字节的位置，接下来对各个字节的各位进行编号，按如下规则：  

![sector trailer block encoding rules.png](/assets/images/2021-10-19-Mifare-introduction/sector-trailer block-encoding-rules.png)

比如图中的K57，第一个数字表示其所在的字节，第二个数字表示其对应的位数，其中5字节为最高有效字节，0字节为最低有效字节，7位为最高有效位，0位为最低有效位。确认了各位编号后，分别将KeyA和KeyB按如下规则排列成DKeyA和DKeyB，DKeyB与KeyB的映射规则为：  

![DkeyB.png](/assets/images/2021-10-19-Mifare-introduction/DkeyB.png)

DKeyA与KeyA的映射规则为：  

![DKeyA.png](/assets/images/2021-10-19-Mifare-introduction/DKeyA.png)

其中P表示奇校验位，由于实际3DES运算中并不会用到这一位，因此在生成DKeyA和DKeyB的时候，将该位统一置为0，即不做奇校验。  

按照以上的规则，KeyA和KeyB为默认值0xFFFFFFFFFFFF(这里的排列顺序为最高有效字节→最低有效字节，若按照实际从尾部区块读取出来的顺序，则为最低有效字节→最高有效字节)的情况下，易知DKeyA即为0x007EFEFEFEFEFEFE，DKeyB为0xFEFEFEFEFEFE7E00。  

对初始向量0x0000000000000000进行3DES加密运算，可得最终的MF Password值为0xE73AFE450757540B，这里的排列顺序均为最高有效字节→最低有效字节，若按最低有效字节→最高有效字节，则DKeyA为0xFEFEFEFEFEFE7E00，DKeyB为0x007EFEFEFEFEFEFE，MF Password值为0x0B54570745FE3AE7。  

几个比较常用的密钥组合及对应的MF Password见下表：  

![Mifare Password.png](/assets/images/2021-10-19-Mifare-introduction/Mifare-password.png)

### 四、访问Mifare
本节将介绍两种访问Mifare的方法，一种是借助Javacard API实现对Mifare内存的读写操作，一种是利用读卡器的特定指令，实现对Mifare内存的读写操作，该方式目前只能做KeyA或KeyB的鉴权。  

- Javacard API 方式
Javacard API中提供了读数据与写数据两个方法用以操作Mifare这类扩展内存区域：  

```
short readData(byte[] dest, short dest_off, byte[] auth_key, short auth_key_off, short auth_key_blen, short other_sector, short other_block, short other_len) throws ExternalException
boolean writeData(byte[] src, short src_off, short src_blen, byte[] auth_key, short auth_key_off, short auth_key_blen, short other_sector, short other_block) throws ExternalException
```

在提供了正确的MF Password后，即可对对应扇区进行读写操作，其中厂商区块不支持修改。  

- 读卡器的特定指令
这里使用的读卡器的型号是**Identiv uTrust 4701F**双界面读卡器，支持接触式与非接触式的卡片交互，读卡器的特定指令参照自当前使用的读卡器的用户手册*Reference Manual for uTrust 4701F and uTrust 4711F Readers*。  

读卡器内有专用缓存用以存储鉴权密钥KeyA或KeyB，可以通过非接接口将密钥缓存进读卡器内的内存区域，并用其封装的指令实现对Mifare卡的鉴权与交互，这里使用普通M1卡进行测试，读取卡内的1扇区和7扇区的内容，用到的指令及说明如下：  

> // 将密钥0xFFFFFFFFFFFF装载进读卡器内的缓存中，P2标识密钥版本，0x60表示KeyA，0x61表示KeyB  
> **FF 82 00 60 06 FFFFFFFFFFFF**  
> // 正常响应  
> **9000**  
> 
> // 鉴权指令，数据域为0x0100046001，其中，0x0004标识区块号(绝对位置索引)，对应1扇区区块0，0x60对应KeyA  
> **FF 86 00 00 05 01 00 00 60 01**  
> **9000**  
>
> // 读取1扇区整个扇区的数据  
> **FF B3 00 01 00**  
> // 6982 安全条件不满足，该扇区的KeyA不为0xFFFFFFFFFFFF  
> **6982**  
> 
> // 鉴权指令，数据域为0x 01001C6001，其中，0x001C标识区块号(绝对位置索引)，0x1C对应7扇区区块0，0x60对应KeyA  
> **FF 86 00 00 05 01 00 1C 60 01**  
> **9000**  
>
> // 读取7扇区整个扇区的数据  
> **FF B3 00 07 00**  
> // 整个扇区的数据 + 返回码  
>**00000000000000000000000000000000000000000000000000000000000000  0000000000000000000000000000000000000000000000FF078069FFFFFFFFFFFF9000**  

需声明的是，以上指令仅支持非接界面上的操作。  

### 五、其他
- **参考文献**
[1]  AN0215 Secure Access to MIFARE Memory on Dual Interface Smart Card ICs *by PHILIPS*  
[2]  MIFARE CLASSIC 1K/4K USER MANUAL Release 1.1.0 *by SonMicro Elektronik*  
[3]  Reference Manual for uTrust 4701F Dual Interface Reader and uTrust 4711F Contactless Reader with SAM  
[4]  ISO/IEC 14443-3 Identification cards — Contactless integrated circuit(s) cards — Proximity cards — Part 3: Initialization and anticollision  

- **相关链接**
一款用以计算MF Password的小工具，github地址： [gelvaos/MifarePass: Mifare Password Calculator (github.com)](https://github.com/gelvaos/MifarePass)  