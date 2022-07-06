# Chapter1-环境配置

## 写在前面

以下步骤一般情况下只在用户能够正常访问外网时成立。 大致来说，您需要为自己的设备和 git 配置代理，然后才能够顺利完成以下步骤，但出于某些原因，笔者不便在这里过多赘述配置代理的步骤，还望读者见谅

## 运行环境

### step 0

似乎现在去 clone 那个 depot\_tools 的仓库里自带了 ninja，有需要这一步的师傅可以在后续步骤中遇到报错时再回来补

```bash
git clone https://github.com/ninja-build/ninja.git
cd ninja && ./configure.py --bootstrap && cd ..
echo 'export PATH=$PATH:"/path/to/ninja"' >> ~/.bashrc
# /path/to/ninja改成ninja的目录
```

### step 1

安装依赖并克隆仓库，设置环境变量后拉取 v8 的代码 但考虑到中英文问题和一些网络代理问题，这里不安装字体依赖，有需要的师傅可以试着去掉该参数

```bash
sudo apt install bison cdbs curl flex g++ git python vim pkg-config
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
echo 'export PATH=$PATH:"/path/to/depot_tools"' >> ~/.bashrc
# /path/to/depot_tools改成depot_tools的目录
fetch v8
./v8/build/install-build-deps.sh --no-chromeos-fonts
```

### step 2

写一个脚本去跑编译，方便以后直接换版本编译：

```shell
#!/bin/bash
VER=$1 
if [ -z $2 ]; then
        NAME=$VER
else
        NAME=$2
fi
cd /path/depot_tools/v8
# /path/depot_tools/v8 换成自己的路径
git reset --hard $VER
gclient sync -D
gn gen out/x64_$NAME.release --args='v8_monolithic=true v8_use_external_startup_data=false is_component_build=false is_debug=false target_cpu ="x64" use_goma=false goma_dir="None" v8_enable_backtrace=true v8_enable_disassembler=true v8_enable_object_print=true v8_enable_verify_heap=true'
ninja -C out/x64_$NAME.release d8
```

```bash
time ./build.sh "9.6.180.6"
```

编译效率一般取决于自己的设备性能

## 调试环境

将 “v8/tools/gdbinit” 保存到自己惯用的目录下，这里称之为 “path”，然后将路径写到 .gdbinit 下：

```shell
cp v8/tools/gdbinit /path/gdbinit_v8
cat ~/.gdbinit
#source /home/tokameine/Desktop/env/pwndbg/gdbinit.py
#source /path/gdbinit_v8
```

做完以后，就能够在源代码中插入如下代码进行调试了：

```javascript
%DebugPrint(x); 打印变量 x 的相关信息
%SystemBreak(); 抛出中断，令 gdb 在此处断点
```

> 但这两条代码并非原有的语法，在执行时需添加参数 “--allow-natives-syntax”， 否则会提示 “SyntaxError: Unexpected token '%'”

## 调试样本

就用一个简单的 demo 测试一下调试能够正常进行：

```javascript
//demo.js
%SystemBreak();
var wasmCode = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11]);

var wasmModule = new WebAssembly.Module(wasmCode);
var wasmInstance = new WebAssembly.Instance(wasmModule, {});
var f = wasmInstance.exports.main;
%DebugPrint(f);
%DebugPrint(wasmInstance);
%SystemBreak();
```

> 我们暂时不用在意这段代码在做什么，这无关紧要，我们现在只想知道调试环境是否能够正常工作而已，所以读者只需要知道有这么个变量名为 f 的变量即可

在 v8/out/x64\_$name.release 目录下可以找到二进制程序 d8，它才是解析执行 js 代码的引擎，通过 gdb 去调试该程序，并将 demo.js 作为参数传给它

```
$ gdb d8
pwndbg> r --allow-natives-syntax /home/tokameine/Desktop/demo/test.js 
pwndbg> c
```

可以看到 gdb 正常发生了中断，但由于我们调试的并非 js 脚本，所以自然不可能顺着脚本中断，而是在 d8 的某行机器码处中断了，此时它会打印出数组 f 的数据：

```
pwndbg> c
Continuing.
DebugPrint: 0x2bdb081d370d: [Function] in OldSpace
 - map: 0x2bdb08204919 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x2bdb081c3b4d <JSFunction (sfi = 0x2bdb08144165)>
 - elements: 0x2bdb0800222d <FixedArray[0]> [HOLEY_ELEMENTS]
 - function prototype: <no-prototype-slot>
 - shared_info: 0x2bdb081d36e9 <SharedFunctionInfo js-to-wasm::i>
 - name: 0x2bdb080051cd <String[1]: #0>
 - builtin: GenericJSToWasmWrapper
 - formal_parameter_count: 0
 - kind: NormalFunction
 - context: 0x2bdb081c3649 <NativeContext[252]>
 - code: 0x2bdb0018d801 <Code BUILTIN GenericJSToWasmWrapper>
 - Wasm instance: 0x2bdb081d35b9 <Instance map = 0x2bdb08207399>
 - Wasm function index: 0
 - properties: 0x2bdb0800222d <FixedArray[0]>
 - All own properties (excluding elements): {
    0x2bdb080048f1: [String] in ReadOnlySpace: #length: 0x2bdb08142339 <AccessorInfo> (const accessor descriptor), location: descriptor
    0x2bdb08004a21: [String] in ReadOnlySpace: #name: 0x2bdb081422f5 <AccessorInfo> (const accessor descriptor), location: descriptor
    0x2bdb08004029: [String] in ReadOnlySpace: #arguments: 0x2bdb0814226d <AccessorInfo> (const accessor descriptor), location: descriptor
    0x2bdb08004245: [String] in ReadOnlySpace: #caller: 0x2bdb081422b1 <AccessorInfo> (const accessor descriptor), location: descriptor
 }
 - feedback vector: feedback metadata is not available in SFI
0x2bdb08204919: [Map]
 - type: JS_FUNCTION_TYPE
 - instance size: 28
 - inobject properties: 0
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - callable
 - back pointer: 0x2bdb080023b5 <undefined>
 - prototype_validity cell: 0x2bdb08142405 <Cell value= 1>
 - instance descriptors (own) #4: 0x2bdb081d0445 <DescriptorArray[4]>
 - prototype: 0x2bdb081c3b4d <JSFunction (sfi = 0x2bdb08144165)>
 - constructor: 0x2bdb08002235 <null>
 - dependent code: 0x2bdb080021b9 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0

DebugPrint: 0x2bdb081d35b9: [WasmInstanceObject] in OldSpace
 - map: 0x2bdb08207399 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x2bdb08048079 <Object map = 0x2bdb08207af1>
 - elements: 0x2bdb0800222d <FixedArray[0]> [HOLEY_ELEMENTS]
 - module_object: 0x2bdb08049cbd <Module map = 0x2bdb08207231>
 - exports_object: 0x2bdb08049e71 <Object map = 0x2bdb08207bb9>
 - native_context: 0x2bdb081c3649 <NativeContext[252]>
 - memory_object: 0x2bdb081d35a1 <Memory map = 0x2bdb08207641>
 - table 0: 0x2bdb08049e41 <Table map = 0x2bdb082074b1>
 - imported_function_refs: 0x2bdb0800222d <FixedArray[0]>
 - indirect_function_table_refs: 0x2bdb0800222d <FixedArray[0]>
 - managed_native_allocations: 0x2bdb08049df9 <Foreign>
 - memory_start: 0x7f8f28000000
 - memory_size: 65536
 - imported_function_targets: 0x55b1281580e0
 - globals_start: (nil)
 - imported_mutable_globals: 0x55b128158210
 - indirect_function_table_size: 0
 - indirect_function_table_sig_ids: (nil)
 - indirect_function_table_targets: (nil)
 - properties: 0x2bdb0800222d <FixedArray[0]>
 - All own properties (excluding elements): {}

0x2bdb08207399: [Map]
 - type: WASM_INSTANCE_OBJECT_TYPE
 - instance size: 240
 - inobject properties: 0
 - elements kind: HOLEY_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - stable_map
 - back pointer: 0x2bdb080023b5 <undefined>
 - prototype_validity cell: 0x2bdb08142405 <Cell value= 1>
 - instance descriptors (own) #0: 0x2bdb080021c1 <Other heap object (STRONG_DESCRIPTOR_ARRAY_TYPE)>
 - prototype: 0x2bdb08048079 <Object map = 0x2bdb08207af1>
 - constructor: 0x2bdb081d242d <JSFunction Instance (sfi = 0x2bdb081d2409)>
 - dependent code: 0x2bdb080021b9 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0
```

另外，v8提供的gdbinit中额外支持了一条 “job” 命令，它可以用来打印对象的相关信息，这里我们可以用数组 a 进行测试：

```
pwndbg> job 0x2bdb081d370d
0x2bdb081d370d: [Function] in OldSpace
 - map: 0x2bdb08204919 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x2bdb081c3b4d <JSFunction (sfi = 0x2bdb08144165)>
 - elements: 0x2bdb0800222d <FixedArray[0]> [HOLEY_ELEMENTS]
 - function prototype: <no-prototype-slot>
 - shared_info: 0x2bdb081d36e9 <SharedFunctionInfo js-to-wasm::i>
 - name: 0x2bdb080051cd <String[1]: #0>
 - builtin: GenericJSToWasmWrapper
 - formal_parameter_count: 0
 - kind: NormalFunction
 - context: 0x2bdb081c3649 <NativeContext[252]>
 - code: 0x2bdb0018d801 <Code BUILTIN GenericJSToWasmWrapper>
 - Wasm instance: 0x2bdb081d35b9 <Instance map = 0x2bdb08207399>
 - Wasm function index: 0
 - properties: 0x2bdb0800222d <FixedArray[0]>
 - All own properties (excluding elements): {
    0x2bdb080048f1: [String] in ReadOnlySpace: #length: 0x2bdb08142339 <AccessorInfo> (const accessor descriptor), location: descriptor
    0x2bdb08004a21: [String] in ReadOnlySpace: #name: 0x2bdb081422f5 <AccessorInfo> (const accessor descriptor), location: descriptor
    0x2bdb08004029: [String] in ReadOnlySpace: #arguments: 0x2bdb0814226d <AccessorInfo> (const accessor descriptor), location: descriptor
    0x2bdb08004245: [String] in ReadOnlySpace: #caller: 0x2bdb081422b1 <AccessorInfo> (const accessor descriptor), location: descriptor
 }
 - feedback vector: feedback metadata is not available in SFI
```

其参数是之前 DebugPrint 打印出的地址，可以看见，该指令将对象的各个信息都打印出来了，但我们可以注意到，这个地址的最低位似乎没有四字节对齐，其真实地址是 0x2bdb081d370d-1，但使用 job 时需要将地址加一来区分对象类型和数字类型。如果给出的参数是真实地址，大致会像下面这样：

```shell
pwndbg> job 0x2bdb081d370d-1
Smi: 0x40e9b86 (68066182)
# 0x40e9b86 * 2 = 81D370D-1
```

> v8 储存数据的方式有些特别，它会让这些整数都乘以二，也包括数组的长度，因此当 job 认为该地址是一个数字类型时，会将其除以二后的值当作本来的值

可以通过其他查看真正的内存数据：

```
pwndbg> x/20xw 0x2bdb081d370d-1
0x2bdb081d370c:	0x08204919	0x0800222d	0x0800222d	0x081d36e9
0x2bdb081d371c:	0x081c3649	0x0814244d	0x0018d801	0x080026c1
0x2bdb081d372c:	0x00000008	0x00000000	0x00000002	0x0800528d
0x2bdb081d373c:	0x08207bbb	0x00000000	0x00000000	0x00000000
0x2bdb081d374c:	0x00000000	0x00000000	0x00000000	0x00000000
```

可以发现，v8 对地址数据进行了压缩储存，由于高 32bit 的地址完全相同，每个地址只会存放其低 32bit 的数据

