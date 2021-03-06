# 第七节 IO

## No.1 什么是IO,什么又是IO密集型业务呢?

IO称为输入/输出,分为IO设备和IO接口两个部分.
IO密集型指的是系统的CPU性能相对磁盘、内存要好很多,此时,系统运作大部分的状态是CPU在等I/O(硬盘/内存)的读/写操作,此时CPU loading并不高.

## No.2 什么是Stream,Stream有什么好处,Stream的应用场景是什么?

Node.js内置的Stream模块是多个核心模块的基础,但是流(stream)是一种很早之前流行的编程方式,可以用大家比较熟悉的C语言来看这种流式操作:

```js
int copy(const char *src, const char *dest)
{
    FILE *fpSrc, *fpDest;
    char buf[BUF_SIZE] = {0};
    int lenSrc, lenDest;、
    // 打开要src的文件
    if ((fpSrc = fopen(src, "r")) === NULL) {
        printf("文件 '%s' 无法打开\n", src);
        return FAILURE;
    }
    // 打开dest的文件
    if ((fpDest == fopen(dest, "w")) == NULL) {
        printf("文件 '%s' 无法打开\n", dest);
        fclose(fpSrc);
        return FAILURE;
    }

    // 从src中读取BUF_SIZE长的数据到buf中
    while ((lenSrc = fread(buf, 1, BUF_SIZE, fpSrc)) > 0) {
        // 将buf中的数据写入dest中
        if ((lenDest = fwrite(buf, 1, lenSrc, fpDest)) != lenSrc) {
            printf("写入文件 '%s' 失败\n", dest);
            fclose(fpSrc);
            fclose(fpDest);
            return FAILURE;
        }
        // 写入成功后清空buf
        memset(buf, 0, BUF_SIZE);
    }
    // 关闭文件
    fclose(fpSrc);
    fclose(fpDest);
    return SUCCESS;
}
```

Stream是基于事件EventEmitter的数据管理模式,由各种不同的抽象接口组成,主要包括可写,可读,可读写,可转换等几种类型.
好处是:非阻塞式数据处理提升效率,片段处理节省内存,管道处理方便可扩展等.
应用场景:文件、网络、数据转换、音频、视频等
针对大文件的使用场景:
比如拷贝一个20G大的文件,如果你一次性将20G的数据读入到内存,你的内存可能会被撑爆,或者严重影响性能.但是你如果使用一个1MB大小的缓存(buf)每次读取1Mb,然后写入1Mb,那么不论这个文件多大都只会占用1Mb的内存.

## No.3 如何捕获Stream的错误事件?

只要监听error事件,方法与EventEmitter相同.

## No.4 有哪些常用的Stream流,分别什么时候使用?

|    类    |   使用场景   |    重写方法    |
| ------ | ------ | ------ |
| Readable | 只读,为可被读流,在作为输入数据源时使用,比如process.stdin | 重写_read方法 |
| Writable | 只写,为可被写流,在作为输出源时使用,比如process.stdout或process.stderr | 重写_write方法 |
| Duplex   | 读写,为可读写流,它作为输出源接受被写入,同时又作为输入源被后面的流读出. | 重写_read和_write |
| Transform | 与Duplex相似,操作被写入数据, 然后读出结果 | 重新_transform和_flush |

注: 

* Transform机制和Duplex一样,都是双向流,区别是Transform只需要实现一个函数 _transform(chunk,encoding,callback);而Duplex需要分别实现_read(size)函数和_write(chunk,encoding,callback)函数.
* Duplex流和Transform流都是同时可读写的,他们会在内部维持两个缓冲区,分别对应读取和写入,这样就可以允许两边同时独立操作,维持高效的数据流.
* net.Socket是一个Duplex流,Readable端允许从socket获取、消耗数据,Writable端允许向socket写入数据.数据写入的速度很有可能与消耗的速度有差距,所以两端可以独立操作和缓冲是很重要的.

## No.5 如何实现一个Writable Stream?

主要思路:

* 创建一个构造函数,使用Writable进行call调用;
* 构造函数继承Writable类;
* 构造函数原型实现_write(chunk,encoding,callback)函数

```js
const Writable = require("stream").Writable;
const util = require("util");
function MyWritable(options) {
    Writable.call(this, options);
}
util.inherits(MyWritable, Writable); // 继承自Writable
// 重写_write方法
MyWritable.prototype._write = function(chunk, encoding, callback) {
    console.log("被写入的数据是:", chunk.toString()); // 此处可对写入的数据进行处理
    callback();
};
process.stdin.pipe(new MyWritable()); // stdin作为输入源, MyWritable作为输出源
```

## No.6 Stream的highWaterMark与drain事件是什么?二者之间的关系是?

Stream缓冲区的大小,由构造Stream的highWaterMark标志来指定容纳的byte大小,对于objectMode的stream,该标志表示可以容纳的对象个数.在一个可写实例上不停地调用writable.write(chunk)的时候数据会被写入到可写流的缓冲区,如果当前缓冲区的缓冲数据量低于highWaterMark设定的值,调用writable.write()方法会返回true,否则当缓冲的数据量达到了阈值,就需要触发drain事件之后才能继续调用write写入.

```js
function writeOneMillionTimes(writer, data, encoding, callback) {
    let i = 1000000;
    write();
    function write() {
        var ok = true;
        do {
            i--;
            if (i === 0) {
                // 最后一次
                writer.write(data, encoding, callback);
            } else {
                // 如果ok为true,继续写入,否则跳出循环
                ok = writer.write(data, encoding);
            }
        } while(i > 0 && ok);
        if (i > 0) {
            // 当无法写入并且还有需要写入的数据时,执行一次drain事件
            writer.once('drain', write);
        }
    }
}
```

## No.7 Stream的pipe作用是什么?在pipe的过程中数据是引用传递还是拷贝传递?

Stream的.pipe()是将一个可写流附到可读流上,同时将可写流切换到流模式,并把所有数据推给可写流.在pipe传递数据的过程中,objectMode是传递引用,非objectMode则是拷贝一份数据传递下去.
pipe方法最主要的目的就是将数据的流动缓冲到一个可接受的水平,不让不同速度的数据源之间的差异导致内存被占满.

* 通过pipe实现一个导流的示例

```js
'use strict'
import {createReadStream, createWriteStream} from 'fs';
createReadStream('/path/to/a/big/file').pipe(createWriteStream('/path/to/the/dest'))
```

* 功能分解:

  * 不断从来源可读流中获得一个指定长度的数据;
  * 将获取到的数据写入目标可写流;
  * 平衡读取和写入速度,防止读取速度大大超过写入速度时,出现大量滞留数据.

## No.8 Buffer一般用于处理什么数据?其长度能否动态变化?

Buffer是Node.js中用于处理二进制数据的类,其中与IO相关的操作(网络/文件等)均基于Buffer.Buffer类的实例非常类似整数数组,但其大小是固定不变的,并且其内存在V8堆栈外分配原始内存空间.Buffer类的实例创建之后,其所占用的内存大小就不能再进行调整.

## No.9 Node.js在Buffer上做了哪些改进和优化?

Node.js的Buffer在ES6增加了TypedArray类型之后,修改了原来的Buffer的实现,选择基于TypedArray中Uint8Array来实现,从而提升了一波性能.

|  接口  | 用途 |
| ------ | ------ |
| Buffer.from() | 根据已有数据生成一个Buffer对象 |
| Buffer.alloc() | 创建一个初始化后的Bufer对象 |
| Buffer.allocUnsafe() | 创建一个未初始化的Buffer对象 |

用法示例代码:

```js
const arr = new Uint16Array(2);
arr[0] = 5000;
arr[1] = 4000;
const buf1 = Buffer.from(arr); // 拷贝了该Buffer
const buf2 = Buffer.from(arr.buffer); // 与该数组共享了内存
console.log(buf1); // 输出:<Buffer 88 a0>,拷贝的buffer只有两个元素
console.log(buf2); // 输出:<Buffer 88 13 a0 0f>

arr[1] = 6000;
console.log(buf1); // 输出:<Buffer 88 a0>
console.log(buf2); // 输出:<Buffer 88 13 70 17>
```

## No.10 什么是文件描述符?输入流/输出流/错误流是什么?

unix/linux一切皆是文件,所有操作对象均为fd(文件描述符),都可以通过同一套system call来读写,在linux中可以通过ulimit来对fd资源进行一定程度的管理限制.
stdio(standard input output)标准的输入输出流,即输入流(stdin),输出流(stdout),错误流(stderr)三者.
在Node.js中分别对应process.stdin(Readable),process.stdout(Writable)以及 process.stderr(Writable)三个 stream.

## No.11 child_process和process的stdin,stdout,stderror是一样的吗?

从概念上都是一样的,输入,输出,错误都是流,区别是在父进程眼里,子程序的stdout是输入流,stdin是输出流.

## No.12 console.log是同步还是异步?如何实现一个console.log?

正常情况下是异步的,除非使用new Console().
具体看一下源码的实现片段,其中this._stdout默认是process.stdout

```js
Console.prototype.log = function (...arg) {
    this._stdout.write(`${util.format.apply(null, args)}\n`);
};
```

* 自己实现一个console.log方法,示例代码如下:

```js
let print = (str) => process.stdout.write(str + ‘\n’);
print(‘hello world’);
```
注意:该代码并没有处理多参数,也没有处理占位符,util.format 提供这个功能

* console.log.bind(console)的作用

我们平时使用控制台输出代码用console.log()即可,如果需要多次输出,需要不断的console.log()有点麻烦;
可以使用bind来简化功能:

```js
let log = console.log.bind(console);
```

## No.13 如何同步的获取用户的输入?

获取用户的输入其实就是读取Node.js进程中的输入流(即process.stdin这个stream)的数据.
而要同步读取,则是不用异步的read接口,而是用同步的readSync接口去读取stdin的数据即可实现.
看一段实现的示例代码:

```js
var fs = require("fs");
var buf = new Buffer(256); // 申请一块buffer内存空间
module.exports = function() {
    // 获取当前输入流的fd
    var fd = ('win32' === process.platform) ? process.stdin.fd : fs.openSync('/dev/stdin', 'rs');
    try {
        bytesRead = fs.readSync(fd, buf, 0, BUFSIZE); // 同步读取输入流
    }
    // ...
    var content = buf.toString(null, 0, bytesRead - 1); // 进行流的拼接
};

```

## No.14 Readline 是如何实现的?

readline模块提供了一个用于从Readable的stream(例如process.stdin)中一次读取一行的接口.当然你也可以用来读取文件或者net,http的stream,比如:

```js
const readline = require("readline");
const fs = require("fs");
const rl = readline.createInterface({
    input: fs.createReadStream('sample.txt')
});
rl.on('line', (line) => {
    console.log(`Line from file: ${line}`);
});
```

实现上,readline在读取TTY的数据时,是通过input.on('keypress', onkeypress)时发现用户按下了回车键来判断是新的line的,而读取一般的stream时,则是通过缓存数据然后用正则.test来判断是否为new line的.

如果在编写脚本时,不习惯异步获取输入,而是要同步获取用户的输入可以使用scanf模块.

## 附录

* [Node.js中流操作实践](https://zhuanlan.zhihu.com/p/44120527)