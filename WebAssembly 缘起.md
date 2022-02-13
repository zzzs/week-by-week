# WebAssembly 缘起

# tag
`javascript` `WebAssembly 原生` `AssemblyScript` `Emscripten 编译器`

# 为什么需要 WebAssembly ？
JS 代码的执行过程
![](https://mmbiz.qpic.cn/mmbiz_png/ndgH50E7pIpoC4GCCpgaSlCwdbzUhSsK3ia0gOrlTU0cWeV9VRicVNpt8tfbLL6FicqaxKpzP3ZbC9L0hpf5vbs8w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

整体的流程就是：

- 拿到了 JS 源代码，交给 Parser，生成 AST
- ByteCode Compiler 将 AST 编译为字节码（ByteCode）
- ByteCode 进入翻译器，翻译器将字节码一行一行翻译（Interpreter）为机器码（Machine Code），然后执行

我们平时写的代码会多次执行同一个函数，那么可以将这个函数生成的 Machine Code 标记可优化，然后打包送到 JIT Compiler（Just-In-Time），下次再执行这个函数的时候，就不需要经过 Parser-Compiler-Interpreter 这个过程，可以直接执行这份准备好的 Machine Code，大大提高的代码的执行效率。

但是上述的 **JIT 优化只能针对静态类型的变量**，如我们要优化的函数，它只有两个参数，每个参数的类型是确定的，而 JavaScript 却是一门动态类型的语言，这也意味着，函数在执行过程中，可能类型会动态变化，参数可能变成三个，第一个参数的类型可能从对象变为数组，这就会导致 JIT 失效，需要重新进行 Parser-Compiler-Interpreter-Execuation，而 Parser-Compiler 这两步是整个代码执行过程中最耗费时间的两步，这也是为什么 JavaScript 语言背景下，Web 无法执行一些高性能应用，如大型游戏、视频剪辑等。

## 静态语言优化
如果我们能够为 JS 引入静态特性，那么可以保持有效的 JIT，势必会加快 JS 的执行速度，这个时候 asm.js 出现了。

asm.js 只提供两种数据类型：

- 32 位带符号整数
- 64 位带符号浮点数

其他类似如字符串、布尔值或对象都是以数值的形式保存在内存中，通过 TypedArray 调用。
```javascript
var a = 1;

var x = a | 0;  // x 是32位整数

var y = +a;  // y 是64位浮点数

function add(x, y) {

  x = x | 0;

  y = y | 0;

  return (x + y) | 0;

}
```
上述的函数参数及返回值都需要声明类型，这里都是 32 位整数。

而且 asm.js 也不提供垃圾回收机制，内存操作都是由开发者自己控制，通过 TypedArray 直接读写内存：
```javascript
var buffer = new ArrayBuffer(32768); // 申请 32 MB 内存

var HEAP8 = new Int8Array(buffer); // 每次读 1 个字节的视图 HEAP8

function compiledCode(ptr) {

  HEAP[ptr] = 12;

  return HEAP[ptr + 4];

}
```
> asm.js 是一个严格的 JavaScript 子集要求变量的类型在运行时确定且不可改变，且去除了 JavaScript 拥有的垃圾回收机制，需要开发者手动管理内存。这样 JS 引擎就可以基于 asm.js 的代码进行大量的 JIT 优化，据统计 asm.js 在浏览器里面的运行速度，大约是原生代码（机器码）的 50% 左右。

# asm 还不够
不管 asm.js 再怎么静态化，干掉一些需要耗时的上层抽象（垃圾收集等），也还是属于 JavaScript 的范畴，代码执行也需要 Parser-Compiler 这两个过程，而这两个过程也是代码执行中最耗时的。

为了极致的性能，Web 的前沿开发者们抛弃 JavaScript，创造了一门可以直接和 Machine Code 打交道的汇编语言 WebAssembly，直接干掉 Parser-Compiler，同时 WebAssembly 是一门强类型的静态语言，能够进行最大限度的 JIT 优化，使得 WebAssembly 的速度能够无限逼近 C/C++ 等原生代码。

![](https://mmbiz.qpic.cn/mmbiz_png/ndgH50E7pIpoC4GCCpgaSlCwdbzUhSsKIG9sZ5xpUSsibibicZmDVzZ8WCJxK9lSHFibePMwEKO1NJYZtxTOFO2oeA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## WebAssembly 来了
![](https://mmbiz.qpic.cn/mmbiz_png/ndgH50E7pIpoC4GCCpgaSlCwdbzUhSsK6ZDF5oRDa8dHBomgDXTZicpy8fQ27REIeOuic0Pk4aRhqRazqAYic2MCw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

WebAssembly（也称为 WASM），是一种可在 Web 中运行的全新语言格式，同时兼具体积小、性能高、可移植性强等特点，在底层上类似 Web 中的 JavaScript，同时也是 W3C 承认的 Web 中的第 4 门语言。

为什么说在底层上类似 JavaScript，主要有以下几个理由：

- 和 JavaScript 在同一个层次执行：JS Engine，如 Chrome 的 V8
- 和 JavaScript 一样可以操作各种 Web API

同时 WASM 也可以运行在 Node.js 或其他 WASM Runtime 中。

## WebAssembly 文本格式

实际上 WASM 是一堆可以直接执行二进制格式，但是为了易于在文本编辑器或开发者工具里面展示，WASM 也设计了一种 “中间态” 的文本格式，以 .wat 或 .wast 为扩展命名，然后通过 `wabt` 等工具，将文本格式下的 WASM 转为二进制格式的可执行代码，以 .wasm 为扩展的格式。

例子:
```javascript
(module

  (func $i (import "imports" "imported_func") (param i32))

  (func (export "exported_func")

    i32.const 42

    call $i

  )

)
```

我们通过 wabt 将上述文本格式转为二进制代码：

- 将上述代码复制到一个新建的，名为 simple.wat 的文件中保存
- 使用 wabt[5] 进行编译转换

当你安装好 wabt 之后，运行如下命令进行编译：
```shell
wat2wasm simple.wat -o simple.wasm
```

可以看到，WebAssembly 其实是二进制格式的代码，即使其提供了稍为易读的文本格式，也很难真正用于实际的编码，更别提开发效率了。

## WebAssembly 作为编程语言的一种尝试
为了突破这个限制，`AssemblyScript` 走到台前，`AssemblyScript` 是 TypeScript 的一种变体，为 JavaScript 添加了 WebAssembly 类型， 可以使用 `Binaryen` 将其编译成 WebAssembly。

> WebAssembly 类型大致如下：
> - i32、u32、i64、v128 等
> - 小整数类型：i8、u8 等
> - 变量整数类型：isize、usize 等

Binaryen 会前置将 AssemblyScript 静态编译成强类型的 WebAssembly 二进制，然后才会交给 JS 引擎去执行，所以说虽然 AssemblyScript 带来了一层抽象，但是实际用于生产的代码依然是 WebAssembly，保有 WebAssembly 的性能优势。AssemblyScript 被设计的和 TypeScript 非常相似，提供了一组内建的函数可以直接操作 WebAssembly 以及编译器的特性.

可以看到 AssemblyScript 在为 JavaScript 添加类似 TypeScript 那样的语法，然后在使用上需要保持和 C/C++ 等静态强类型的要求，如不初始化，进行内存分配就访问就会报错。

# 参考
- [为什么说 WebAssembly 是 Web 的未来？](https://mp.weixin.qq.com/s/dEOIArtK6DIfewIva2zLKw)
