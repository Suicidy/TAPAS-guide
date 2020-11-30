# TAPAS

[TAPAS](https://ieeexplore.ieee.org/abstract/document/8574545) is a toolchain for generating application-specific accelerator from Simon Fraser University. First, it uses Tapir Clang-based compiler compiling C code into LLVM IR. TAPAS, then, translates generated LLVM IR with task-level parallelism into hardware code based on dandelion library. The dandelion library is built on top of Chisel3 which can be used to generate Verilog code ready to synthesize and implement on FPGA.

This guide is based on Ubuntu LTS 18.04

## Installing muir(TAPAS) and muir-lib(dandelion-lib)
#### 1. muir installation

the guide provided in the repository is sufficient to understand what are the dependencies of this tool but running the script (muir-setup.sh) might cause errors. Thus, i will provide a detailed guide to install the tool and its dependencies.

```markdown
# Install dependencies from apt-get
sudo apt-get install build-essential cmake libjsoncpp-dev  libncurses5-dev graphviz binutils-dev
sudo apt-get install gcc-8-multilib g++-8-multilib
```
```markdown
# Install verilator v4.016
git clone "http://git.veripool.org/git/verilator"
cd verilator
git pull
git checkout v4.016
unset VERILATOR_ROOT
autoconf
./configure
sudo make install
```
```markdown
# Install Tapir compiler
git clone https://github.com/sfu-arch/Tapir-Meta.git
cd Tapir-Meta
./build.sh release
```
```markdown
# Install muir
# Don't forget to change <path to Tapir compiler> into the Tapir instalation path
git clone https://github.com/sfu-arch/muir
cd muir
mkdir build
cd build
cmake -DLLVM_DIR=<path to Tapir compiler>/Tapir-Meta/tapir/build/lib/cmake/llvm/ -DTAPIR=ON ..
make
```

#### 2. muir-lib installation

muir-lib is ready to use and does not need to perform any installation. Below is my forked version with additional Posit computing unit. 
```markdown 
https://github.com/Suicidy/muir-lib.git
```
## Get started
#### Compilation flow
C --> LLVM IR --> Chisel3 --> Verilog

#### From C to LLVM IR
```markdown 
# Don't forget to change <path to Tapir compiler> into the Tapir instalation path
<path to Tapir compiler>/Tapir-Meta/install/bin/clang -DUSE_CILK_API -DTAPIR -m32 -O1 -fno-unroll-loops \
  -fno-vectorize -fno-slp-vectorize -emit-llvm -S -o - bgemm.c -o - | \
  <path to Tapir compiler>/Tapir-Meta/install/bin/opt -mem2reg -loop-simplify \
  -loop-simplifycfg -simplifycfg -disable-loop-vectorization -dce -dot-cfg -o bgemm.bc
<path to Tapir compiler>/Tapir-Meta/install/bin/llvm-dis bgemm.bc    
```
#### From LLVM IR to Chisel3 (.scala)
```markdown 
# Don't forget to change <path> into the Tapir instalation path or muir installation path
<path to muir>/muir/build/bin/tapir-extract -fn-name=bgemm bgemm.bc -o final
<path to Tapir compiler>/Tapir-Meta/install/bin/llvm-dis bgemm_detach.bc
<path to muir>/muir/build/bin/dandelion -fn-name=bgemm -config=/home/parallels/muir/build/scripts/config.json bgemm_detach.ll -o bgemm_nine.scala
```

#### From Chisel3 to Verilog
After complete the previous step, you will get hardware code (.scala) for each function and each child task. Here is a very breif instruction need to be done
- Manually connect between the generated file in the root file
- The generated Chisel3 code might not compatible with muir-lib. Thus, you have to manually make it compatible. 
- After finishing the root file, you have to create a fore file consisted of FSA. The FSA is handling computing behavior based on control input.
- After finishing the core file, you have to wrap Cache, SimpleRegister and Core module into accelerator file which is ready to be built as an accelerator.  

The bellow code will create an accelerator for bgemm.
```markdown
cd muir-lib
sbt
# Verilog generation code define in generateVerilog.scala
# Float version 
runMain verilogmain.accelerator_edited_Main
# Posit version
runMain verilogmain.accelerator_edited_Main_posit
```

## Change datasize from 32 bit to 16 bits
- Change dataLen indicated in DandelionAccelParams class in Configs.scala (32->16)
- For each file generated from muir, there are three things need to be change
  1. Data size of memory controller (32 -> 16) 
  2. GEP node need to change elementsize (4 -> 2)
  3. Type of data in ld and store node (Typ = MT_W -> Typ = MT_H)
- For posit computing unit, users need to edit paremeter of PositU (PositALU(32, 3, opCode) -> PositALU(16, 2, opCode))

## Hierachy of BGEMM accelerator
```markdown
- Accelerator_test.scala
|
|--- bgemm_core.scala
|   |
|   --- bgemm_root.scala
|       |
|       --- bgemm.scala (Parent)
|       |
|       --- bgemm_detach1.scala (Child)
|
|--- AXICache.scala
|
|--- SimpleReg_edited.scala
```
