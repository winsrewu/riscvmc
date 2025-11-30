English | [简体中文](README_zh_CN.md)  

# riscvmc
This datapack provides a RISC-V emulator for Minecraft.
It supports RV32I instruction sets, except for csr instructions.  
This datapack is built using https://mcbuild.dev.  

Similar repo https://github.com/SuperTails/riscvcraft but with rv32ima. This author also have llvm / wasm support for mc datapack.

# Performance
**1.1k** commands per instruction on average with decode_map cache hit.  
5k commands per instruction on average otherwise.

# Quick Start
### Eliza in Minecraft
Download the datapack from https://storage.jawbts.org/datapack/eliza%40riscvmc-3.0.0.zip and follow the instructions below, start from 'Running the program'.  
This datapack uses a recreation of Eliza, which origins in https://github.com/anthay/ELIZA/blob/master/src/eliza.cpp.

# Usage
### Compiling the datapack
1. Install mc-build, please follow the instructions on https://mcbuild.dev.
2. Clone this repository directly into your Minecraft world's datapacks folder.
3. Run ```mcb build``` in the repository directory.
4. Now the repository directory becomes a datapack. You can now load it into your Minecraft world.

### Compiling the program
1. You may need RISC-V toolchain to compile the program. Good to know that I had create a fork at https://github.com/winsrewu/rv32i-gnu-toolchain, you can directly use that workflow to compile the toolchain.
2. Run c/build.sh {program_name} to compile the program, this should generate a .mem file and copy it to python/ folder, and call a python script to generate a .mcfunction file, which will be copied to data/loader.
Please notice that the global pointer is really easy to fail, you may need to adjust it manually for different programs. I wrote two link scripts in pyriscv, but you may need to modify them on your own or disable global pointer via ``-Wl,--no-relax-gp`` option.
3. If you want to load from a dump file (See [Python emulator](https://github.com/winsrewu/pyriscv)),
you need to use `dump_to_mcf.py` instead of `mem_to_mcf.py`.


### Running the program
1. Load the datapack. Please notice that there's a limit in Minecraft of commands per tick. You can set it via ```/gamerule maxCommandChainLength <>```.
2. If you had runned before, do
```
/function riscvmc:reset
# reset memory, pc, regs, if you are loading from a dump file, don't do this, it will be done in app.mcfunction
```
3. Load the program by doing
```
/function <your app.mcfunction>
# load memory. It will say when it's finished, otherwise there may be an error or the command limit exceeded.

/function <your decode_map>
# optional, if you have generated a decode_map. It will say when it's finished, otherwise there may be an error or the command limit exceeded.

/function riscvmc:set_running
# set running flag to true
```
4. You can now run the program by doing ```function riscvmc_runner:tick50``` multiple times.
This will let the emulator run 50 instructions at one tick. Please notice that there's a limit in Minecraft of commands per tick.
5. If you want to stop the program, do ```function riscvmc:stop_running```.


# Good to know
- Python emulator: https://github.com/winsrewu/pyriscv, i think they have the same behavior.
- The output is buffered until a \n is encountered.
- You can use scanf. The read action is blocking. You can set the ascii tmp like this:
```
/data modify storage riscvmc_ascii_buffer:temp input_buffer set value "Hello_from_input_buffer!\n"
```
- The entry point of the program is hardcoded to 0x0, but you can change it.
- I have provided a systemcalls.c in the python emulator.
- There do have some problems, like invaild file descriptor. I dont know why, i tried it on spike and the result is the same. But most of code works fine.
- I think I won't support privileged architecture.
- You need a really high command limit to run the program, this datapack won't modify this limit. So do ```/gamerule maxCommandChainLength 10000000``` yourself.
- memory load (app.mcfunction) and decode_map load (decode_map.mcfunction) will print a message when they are finished.

# Troubleshooting
If your program breaks, you can use the riscvmc_tester, where you can import a correct list of program counter, and the emulator will check it for you when running. And you can also use the emulator based on python or spike to debug your program. The python emulator mentioned above do have some functionalities to help you debug your program, like printing the registers at each step, monitering memory access, etc.

# System Calls Table
| number | function | args | return |
|--------|----------|------|--------|
| 63     | read     | fd, ptr, len | number of bytes read |
| 64     | write    | fd, ptr, len | number of bytes written |
| 93     | exit     | error_code | - |
| 513    | run_command | ptr, len | - |
| 514    | run_if | ptr, len | the result of the if statement, 0 or 1 |
| 515    | place_block | x, y, z, block_type | - |

# Block Types
| id | name |
|----|------|
| 0  | air  |
| 1  | stone |
| 2  | dirt |
| 3  | birch_wood |

## Explanation
- read: read the buffer from the file descriptor (only support 0, which is stdin) to the memory pointer, return the number of bytes read.
- write: write the buffer from the memory pointer to the file descriptor (only support 1, which is stdout), return the number of bytes written.
- exit: exit the emulator with the error code.
- run_command: run the command in the buffer immediately, ignoring all the control characters.
- run_if: run /execute if data storage riscvmc:if $(buffer) run ..., if test passed, return 1, else return 0.

# Submodules
These mcb files are included as submodules, which means you can remove them if you don't need them.
- src/riscvmc_terminal.mcb - A terminal with keyboard to put stuff into input buffer.
- src/riscvmc_runner.mcb - Runner.
- src/riscvmc_tester.mcb - Tester.

## Recommended Usage
- Output screen:
```
/summon minecraft:text_display ~ ~ ~ {alignment: "left", line_width: 400, Tags:["output"]}

# clear the buffer
/data modify storage riscvmc_ascii_buffer:temp buffer set value ""

# set as repeating command
/data modify entity @e[tag=output,limit=1,sort=nearest] text set from storage riscvmc_ascii_buffer:temp buffer
```

- Terminal screen with keyboard (requires terminal module):
```
/summon minecraft:text_display ~ ~ ~ {alignment: "left", line_width: 400, Tags:["riscvmc_terminal"]}

# print keyboard
/function riscvmc_terminal:print_keyboard
```

# TODO
I need help :sob:
- [ ] Optimizing
- [ ] Extended instruction set support