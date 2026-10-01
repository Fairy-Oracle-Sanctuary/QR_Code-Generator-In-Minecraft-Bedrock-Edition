[English](./README_EN.md) | [中文](./README.md)

# QR Code Generator in Minecraft

在 Minecraft 基岩版中，用 `mcfunction`、计分板、实体和方块实现的二维码生成器。输入编码、Reed–Solomon 纠错计算、数据分块与交织、矩阵填充和掩码异或都在游戏内执行。

本项目以类似汇编的方式组织计算：**计分板当寄存器，实体当指针，`qr_prg` 当执行状态，方块当二进制存储。** 可以在世界中观察数据如何被写入、搬移和参与运算。

## 支持范围

- Minecraft 基岩版；行为包声明的最低引擎版本为 **1.21.0**。
- 二维码 **版本 1–40，L 级纠错**。
- 使用字节模式编码，配套输入板提供 ASCII 字符输入。
- 版本 1、2 每次直接生成 **8 个掩码结果**；版本 3–40 每次生成 **1 个结果**。
- **需要配套世界才能完整运行。** 世界还提供输入交互、命令方块调度和其他结构数据。

## 如何使用

1. 从项目的 [Releases 页面](https://github.com/baby20162016/QR_Code-Generator-in-Minecraft/releases) 下载发布文件并解压。
2. 将 `QR_Code Generator.mcpack` 和 `QR_Code Generator.mcworld` 导入 Minecraft 基岩版。
3. 进入配套世界，通过输入板按钮输入内容，选择二维码版本，再点击生成按钮。
4. 等待编码、纠错计算和矩阵填充完成。版本 1、2 会同时输出 8 个二维码，后续版本输出 1 个。

`structures/main.mcstructure` 是输入板。单独导入行为包并执行 `/function QR/main`，不能替代配套世界的完整运行环境。

## 实现原理

### 1. 用命令搭建计算模型

| Minecraft 机制 | 在本项目中的作用 |
| --- | --- |
| 计分板分数 | 保存数值、计数器和运算中间结果，类似寄存器 |
| `qr_prg` | 保存执行状态，通过 `execute ... scores=...` 选择当前执行的命令 |
| 盔甲架的位置 | 指向当前读写的方块，移动实体即移动指针 |
| 实体的名称、编号和标签 | 区分主流程、输入字符、字节运算与读写指针 |
| 黑白混凝土 | 保存位值：黑色为 1，白色为 0 |
| `structure save/load` | 复制、备份和搬移方块数据，用于移位和切换工作区 |
| 路径标记方块 | 为矩阵写入指针编码移动方向与跳转距离 |

这里的“类似汇编”指这些底层机制的分工。函数按执行状态推进计算；同一次函数调用中，状态改变后，后面的命令也可能继续执行下一阶段。部分函数通过重复调用，在一次调度中完成多步运算。

### 2. 从字符到位流

`ASCII.mcfunction` 根据输入值创建代表字符的盔甲架，用 `qr_uid` 记录顺序，用 `qr_encode` 保存字符值。生成时，`data_code.mcfunction` 写入字节模式标识、字符数量和字符数据；字符数量字段在版本 1–9 使用 8 位，在版本 10–40 使用 16 位。

`encode.mcfunction` 用 `% 2` 取最低位、`/ 2` 推进到下一位，从右向左写入黑白混凝土。这样逐步拆出的最低位，最终形成从左向右读取的高位在前的位流。数据区每行容纳 64 位，指针到达边界后转入上一层。

输入结束后，补上终止位并对齐到字节边界，再由 `pad.mcfunction` 交替写入 `0xEC` 和 `0x11`，补齐所选版本的数据码容量。

### 3. 在游戏内计算纠错码

Reed–Solomon 纠错需要 GF(256) 有限域运算。本项目将生成多项式系数和对数/反对数映射预先写在函数中，实际数据的纠错计算在运行时完成：

1. `decode.mcfunction` 读取 8 个方块，按 `1、2、4、8、16、32、64、128` 还原字节。
2. `GF_2.mcfunction` 将非零字节转换为指数。与生成多项式系数相乘时，使用指数相加并模 255。
3. `GF_1.mcfunction` 将结果指数转换回字节，`encode_sub.mcfunction` 再把字节写成方块。
4. `xor.mcfunction` 对数据区和乘积区逐位异或，得到本轮余数，再搬移数据继续计算。头部零字节由 `sup.mcfunction` 负责移除。

异或也通过方块实现：两位相同输出白色，两位不同输出黑色。`xor.mcfunction` 利用两侧实体朝向和多级局部坐标偏移，展开一行中的多个运算位置；重复调用时沿高度方向推进。

### 4. 分块和交织

版本 1–5 使用单个数据块，版本 6–40 按版本配置分块。`config_split.mcfunction` 提供块长度，`split.mcfunction` 逐字节搬移数据；主流程分别计算每块的纠错码，并备份、恢复生成多项式系数。

`read_high.mcfunction` 在块之间逐字节交织读取，先读取数据码，再读取纠错码。对于具有两组不同数据块长度的版本，额外指针标记负责读取较长块剩余的数据。版本 1–5 使用 `read_low.mcfunction` 顺序读取。

### 5. 生成矩阵、填充与掩码

二维码边长按 `4 × 版本 + 17` 计算。版本 1–12 通过 `qr_mode_*` 结构加载框架和填充路线；版本 13–40 由 `mode_summon.mcfunction` 生成框架与路线，结合定位图案、校正图案和版本信息结构完成布局。

填充时，`qr_fill` 指针读取路径层上的彩色羊毛或其他标记方块，据此转向、跳转并绕过功能图案，将交织后的位流写入矩阵。

版本 1、2 将填充结果复制到 8 个工作区，分别叠加 8 个掩码结构；版本 3–12 使用单个掩码结构；版本 13–40 由 `matrix_summon.mcfunction` 按坐标和的奇偶生成棋盘式掩码。最后 `matrix.mcfunction` 将数据层与掩码层异或，保留功能图案，形成最终二维码。

## 源码导览

建议先读 `QR.mcfunction` 了解状态切换，再按编码、纠错、填充的顺序阅读各子函数。

| 文件或目录 | 作用 |
| --- | --- |
| [manifest.json](./manifest.json) | 行为包元信息及最低引擎版本 |
| [functions/QR/main.mcfunction](./functions/QR/main.mcfunction) | 入口，调用 QR/QR |
| [functions/QR/QR.mcfunction](./functions/QR/QR.mcfunction) | 初始化、纠错循环、分块切换及填充的主状态流程 |
| [functions/QR/ASCII.mcfunction](./functions/QR/ASCII.mcfunction) | 字符输入、编号、删除/清空及生成触发 |
| [functions/QR/version.mcfunction](./functions/QR/version.mcfunction) | 版本 1–40 的容量、纠错长度和块数配置 |
| [functions/QR/data_code.mcfunction](./functions/QR/data_code.mcfunction) | 字符数量、字符数据、终止与分块阶段 |
| [functions/QR/encode.mcfunction](./functions/QR/encode.mcfunction) | 数值转方块位流；encode_sub 为纠错运算写入字节 |
| [functions/QR/encode_sub.mcfunction](./functions/QR/encode_sub.mcfunction) | 纠错运算中并行写入字节 |
| [functions/QR/decode.mcfunction](./functions/QR/decode.mcfunction) | 从 8 个方块还原字节 |
| [functions/QR/pad.mcfunction](./functions/QR/pad.mcfunction) | 交替补齐数据码 |
| [functions/QR/GF/](./functions/QR/GF/) | GF(256) 指数与字节的查表映射 |
| [functions/QR/generator/](./functions/QR/generator/) | 按纠错码长度组织的预置生成多项式系数 |
| [functions/QR/xor.mcfunction](./functions/QR/xor.mcfunction) | 方块异或 |
| [functions/QR/sup.mcfunction](./functions/QR/sup.mcfunction) | 移除头部零字节 |
| [functions/QR/split.mcfunction](./functions/QR/split.mcfunction) | 数据码分块与搬移 |
| [functions/QR/read_low.mcfunction](./functions/QR/read_low.mcfunction) | 单块顺序读取 |
| [functions/QR/read_high.mcfunction](./functions/QR/read_high.mcfunction) | 多块交织读取 |
| [functions/QR/summon.mcfunction](./functions/QR/summon.mcfunction) | 填充指针、路径解释、掩码加载与最终运算调度 |
| [functions/QR/main_sub.mcfunction](./functions/QR/main_sub.mcfunction) | 重复调用 QR/summon，推进填充和最终运算 |
| [functions/QR/mode_summon.mcfunction](./functions/QR/mode_summon.mcfunction) | 版本 13–40 的框架、校正图案与填充路线生成 |
| [functions/QR/matrix_summon.mcfunction](./functions/QR/matrix_summon.mcfunction) | 棋盘式掩码生成 |
| [functions/QR/matrix.mcfunction](./functions/QR/matrix.mcfunction) | 数据与掩码层的最终异或 |
| [functions/QR/config/](./functions/QR/config/) | 多项式选择、工作区补零、框架选择、分块长度和校正图案位置 |
| [functions/math/NUM.mcfunction](./functions/math/NUM.mcfunction) | 计分板常数；同目录还包含其他数学函数 |
| [structures/](./structures/) | 输入板 main、框架 qr_mode_*、掩码 qr_matrix*、版本信息 qr_via_*、定位与校正图案 |

世界内的命令方块和结构数据也是维护对象。例如主流程引用的 `a`、`b`、`c` 不在仓库的 `structures/` 目录中；理解调度和工作区布局时，需要同时查看配套世界。

## 许可证

[MIT License](./LICENSE) · By Baby_2016
