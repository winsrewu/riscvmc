[English](README.md) | 简体中文 

# riscvmc
这是一个为 Minecraft 设计的 RISC-V 指令集模拟器数据包（Datapack）。  
它支持完整的 RV32I 指令集，**但不包括 CSR 指令**。  
本数据包使用 [MC-Build](https://mcbuild.dev) 构建。  

类似的仓库 https://github.com/SuperTails/riscvcraft 但是支持 rv32ima. 该作者也有对 llvm / wasm 的mc数据包支持。

# Performance
当解码表缓存命中时平均 **1.1k** 命令 (Command) 每指令 (Instruction)。
其他情况下平均 5k 命令每指令。

# 快速开始
### 在《我的世界》中使用Eliza
请从 https://storage.jawbts.org/datapack/eliza%40riscvmc-3.0.0.zip 下载数据包，并按照以下说明操作，从“运行程序”步骤开始。  
该数据包使用了Eliza的复刻版本，其原始代码源自 https://github.com/anthay/ELIZA/blob/master/src/eliza.cpp 。

# 使用方法
### 编译数据包
1. 安装 mc-build，请参考 [https://mcbuild.dev](https://mcbuild.dev) 的安装指南。
2. 将本仓库克隆到你的 Minecraft 世界目录下的 `datapacks` 文件夹中。
3. 在仓库目录下运行 `mcb build` 命令。
4. 此时该目录已生成为一个可加载的数据包，可直接放入 Minecraft 世界中使用。

### 编译程序
1. 你可能需要 RISC-V 工具链来编译程序。我创建了一个分支：https://github.com/winsrewu/rv32i-gnu-toolchain ，你可以直接使用其中的工作流来编译工具链。
2. 运行 c/build.sh {程序名称} 来编译程序，这将生成一个 .mem 文件并复制到 python/ 文件夹，然后调用一个 Python 脚本生成 .mcfunction 文件，该文件将被复制到 data/loader 目录。
请知晓, 全局指针可能很容易失败，你可能需要手动调整它以适应不同的程序。在pyriscv中，我准备了两个版本的连接脚本，当然你也有可能要自己修改或者是使用``-Wl,--no-relax-gp``来禁用全局指针。
3. 如果你想要从一个转储文件加载（参见 [Python 模拟器](https://github.com/winsrewu/pyriscv)），你需要使用 `dump_to_mcf.py` 而不是 `mem_to_mcf.py`。

### 运行程序
1. 加载数据包。请注意 Minecraft 对每 tick 可执行命令数量有限制，您可以通过 ```/gamerule maxCommandChainLength <>``` 进行设置。
2. 若之前已运行过程序，请执行：
```
/function riscvmc:reset
# 重置内存、PC和寄存器。若要从转储文件加载，请勿执行此步骤，该操作将在 app.mcfunction 中完成
```
3. 通过以下命令加载程序：
```
/function <你的 app.mcfunction>
# 加载内存。完成后会提示，否则可能是遇到错误或超出命令限制

/function <你的 decode_map>
# 可选步骤，若已生成 decode_map。完成后会提示，否则可能是遇到错误或超出命令限制

/function riscvmc:set_running
# 将运行标志设置为 true
```
4. 此时可通过多次执行 ```function riscvmc_runner:tick50``` 来运行程序。
每次执行将让模拟器在一个 tick 内运行 50 条指令。请注意 Minecraft 对每 tick 可执行命令数量有限制。
5. 若需停止程序，请执行 ```function riscvmc:stop_running```。

# 注意事项
- Python 模拟器：https://github.com/winsrewu/pyriscv ，我认为它们行为一致。
- 输出内容会被缓冲，直到遇到换行符 `\n` 才会显示。
- 你可以使用 scanf。读取操作是阻塞的。你可以像这样设置 ascii tmp：
```
/data modify storage riscvmc_ascii_buffer:temp input_buffer set value "Hello_from_input_buffer!\n"
```
- 程序入口点在 0x0, 但是你可以更改
- 我在python模拟器内提供了 systemcalls.c
- 存在一些问题，例如无效的文件描述符。我不清楚原因，我在 spike 上尝试过，结果也是一样的。但大部分代码都能正常工作。
- 我想我不会支持特权架构。
- 你需要一个很高的命令限制来运行程序，本数据包不会修改这个限制。所以你需要自己 ```/gamerule maxCommandChainLength 10000000```。
- 内存加载（app.mcfunction）和解码表加载（decode_map.mcfunction）完成后会打印一条消息。

# 故障排查
如果程序运行出错，你可以使用 `riscvmc_tester` 模块：导入正确的程序计数器（PC）序列，模拟器会在运行时自动比对。  
此外，你也可以使用 Python 版模拟器或 Spike 进行调试。上述 Python 模拟器支持打印每步的寄存器状态、监控内存访问等调试功能，便于定位问题。

# 系统调用表
| 编号 | 函数名       | 参数          | 返回值         |
|------|--------------|---------------|----------------|
| 63   | read         | fd, ptr, len  | 读取的字节数   |
| 64   | write        | fd, ptr, len  | 写入的字节数   |
| 93   | exit         | error_code    | 无             |
| 513  | run_command  | ptr, len      | 无             |
| 514  | run_if       | ptr, len      | if 语句结果，0 或 1 |
| 515    | place_block | x, y, z, block_type | - |

# 方块类型
| id | name |
|----|------|
| 0  | air  |
| 1  | stone |
| 2  | dirt |
| 3  | birch_wood |

## 说明
- read: 从文件描述符（仅支持 0，即标准输入）读取数据到内存指针处，返回读取的字节数。
- write: 从内存指针处写入数据到文件描述符（仅支持 1，即标准输出），返回写入的字节数。
- exit: 以错误码退出模拟器。
- run_command: 立即执行缓冲区中的命令，忽略所有控制字符。
- run_if: 执行 /execute if data storage riscvmc:if $(buffer) run ...，如果测试通过返回 1，否则返回 0。

# 子模块
这些 mcb 文件是作为子模块引入的，这意味着如果您不需要它们，可以将其移除。
- src/riscvmc_terminal.mcb - 带键盘的终端，可将内容放入输入缓冲区。
- src/riscvmc_runner.mcb - 运行器。
- src/riscvmc_tester.mcb - 测试器。

## 推荐用法
- 输出屏幕：
```
/summon minecraft:text_display ~ ~ ~ {alignment: "left", line_width: 400, Tags:["output"]}

# 清除缓冲区
/data modify storage riscvmc_ascii_buffer:temp buffer set value ""

# 设置为重复执行命令
/data modify entity @e[tag=output,limit=1,sort=nearest] text set from storage riscvmc_ascii_buffer:temp buffer
```

- 带键盘的终端屏幕（需要终端模块）：
```
/summon minecraft:text_display ~ ~ ~ {alignment: "left", line_width: 400, Tags:["riscvmc_terminal"]}

# 打印键盘布局
/function riscvmc_terminal:print_keyboard
```

# TODO
懒得翻译了自己去看英文版本