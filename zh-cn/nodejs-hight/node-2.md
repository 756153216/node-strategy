# 第二节 node-eyes进程管理工具核心设计

> 为了让Node.js应用很好的运行在TMA框架中,node-eyes将作为启动器来启动应用,同时打通TMA框架与Node.js间的差异,并提供生产环境所需的服务特性.

## 一、功能设计

### 1.1 主要功能

* 内置负载均衡(通过cluster模块实现);
* 异常退出的监控与管理;
* 日志搜集与处理;
* 支持TMA运营平台的管理命令;
* 支持HTTP(s)服务监控上报(在TMA平台上运行)
* 支持服务用量上报(在TMA平台上运行)

### 1.2 安装与用法

* 安装

```js
npm install node-eyes -g
```

由于node-eyes是一个CLI程序,所以一般需要使用-g参数来安装;

* 用法

```js
node-eyes index.js [options]
```

* index.js: 为程序的入口脚本;
* [options]可选配置.

* 常见的启动方式:

  * 直接运行入口index.js入口文件; $ node-eyes index.js
  * 以TMA服务的配置文件来启动; $ node-eyes index.js --config TMA.Video.conf
  * 启动并命名应用为TMA.Video; $ node-eyes index.js --name TMA.Video
  * 定义日志输出路径; $ node-eyes index.js --log ./logs/
  * 传递子进程node的启动参数; $ node-eyes index.js --node-args="--debug=9001"
  * 定义子进程数量: $ node-eyes index.js -i 8

## 二、核心代码逻辑设计

### 2.1 入口/bin/node-eyes如何设计?

```js
/**
 * 解析配置文件:启动脚本中的-c后面跟随的文件
 * 其中可以指定worker进程数量
 */
const parseConfig = function(script, file) {
    // ...
    // 在配置文件中读取client.asyncthread的数值
    // 在配置文件中读取server.instances的数值,优先级更高
    // ...
};

/**
 * 使用commander来解析命令行参数
 */
commander.version(pkg.version)...

/**
 * 在start.sh的启动命令中,对启动文件的解析
 * 如果第二个参数是文件,则直接指定为启动文件
 * 如果第二个参数是文件夹,通过package.json执行文件入口
 */
if (scriptStat.isFile() && path.extname(commander.args[0]).toLowerCase() ===".js") {
    entryPoint = path.resolve(commander.args[0]);
} else if (scriptStat.isDirectory()) {
    // 在package.json中查找nodeEye.main
    // 在package.json查找scripts.start
    // 在package.json中查找main
}

/**
 * 真正的执行入口
 */
CLI.start(entryPoint, commander);
```

### 2.2 CLI.js都需要哪些核心逻辑?

```js
/**
 * 初始化Node-PM2组件
 */
const initComponent = (args, opts) => {
    reportVersion(); // 上报服务版本
    report.keepAlive(); // 上报心跳
    // 初始化Admin进程服务,用于下发和接收命令
    // 初始化上报服务
}
/**
 * 启动入口
 */
exports.start = (script, opts) => {
    // 获取worker进程的设置参数
    // 根据读取配置文件来设置常量
    // 初始化日志
    // 将所有启动日志打印至node-eyes.log中
    // 读取package.json中依赖的模块
    /**
     * CPU初始化之后,获取机器的CPU信息
     */
    bindEvents(); // 绑定cluster事件,这里的事件监听主要用于日志打印和上报使用,并对接Node运营平台
    God.prepare(args); // 初始化集群
    initComponent(args, opts); // 初始化组件
    startWorker(opts); // 启动worker进程.实际上由God模块来启动
    notify.report.info("restart"); // 上报
};
```

### 2.3 CLI.js中如何通过God.js模块来管理进程?

```js
const setCluster = (args, execArgv) => {
    // 设置执行worker进程的执行文件
    const setting = {
        exec: path.resolve(path.dirname(module.filename), "ProcessContainer.js") // 设置执行worker进程的执行文件
    };
    // 监听worker进程的退出,并尝试再启动一个worker进程
    // 在10s内连续重启2次还失败则杀死全部进程,在Node运营管理平台显示进程重启异常
    // 监听worker进程的通信消息,worker进程的通信仍然是通过cluster事件来实现的
    // 通知所有子进程,不同的worker进程进行通信
    // worker进程给master进程上报心跳
    /**
     * 设置对僵死进程的监控
     */
    // Linux 提供了这样一个参数min_free_kbytes，用来确定系统开始回收内存的阀值，控制系统的空闲内存。值越高，内核越早开始回收内存，空闲内存越高。设定这个参数时请小心，因为该值过低和过高都有问题。
    // min_free_kbytes 太低可防止系统重新利用内存。这可导致系统挂起并让 OOM 杀死多个进程。
    // 但将这个参数值设定太高（占系统总内存的 5-10%）会让您的系统很快会内存不足。Linux 的设计是使用所有可用 RAM 缓存文件系统数据。
    // 设定高 min_free_kbytes 值的结果是在该系统中花费太多时间重新利用内存。
    // 通知master进程对应的僵死进程
};
```

### 2.4 God模块如何使用ProcessContainer来管理业务进程?

在上面的God模块中,fork出来的worker进程并不是直接运行业务代码的入口文件,而是通过ProcessContainer.js来对业务代码的执行入口文件做进一步的封装.包括:日志打印、给master进程上报心跳,给Node运营平台下发命令,上报网络情况和流量情况到Node运营平台(暂且称特性监控)

ProgressContainer.js最后调用方法:

```js
require('module')._load(exec_script, null, true); // 执行业务代码的入口文件
```