---
title: "EMFILE Issues Analysis"
date: 2025-06-15
permalink: /posts/2025/EMFILE Issues Analysis/
tags:
  - Node.js
  - 资源管理
  - macOS
---

在 macOS 环境下开启后端服务器时，开发者可能会遇到一个常见错误：**Internal watch failed: EMFILE: too many open files, watch**。本文旨在深入分析此错误的根本原因，并提供一个标准化的最佳实践解决方案。

- [1. 错误分析](#1-错误分析)
    - [查看资源限制](#查看资源限制)
- [2. 问题诊断](#2-问题诊断)
    - [launchd 的资源策略](#launchd-的资源策略)
    - [问题根本原因](#问题根本原因)
- [3. 解决方案：优化 nodemon 配置](#3-解决方案优化-nodemon-配置)
    - [nodemon.json 示例配置：](#nodemonjson-示例配置)
- [4. 总结](#4-总结)


# 1. 错误分析

**EMFILE** 是 C 语言标准错误码的缩写，全称为 "**Error, Maximum FILEs open**"。这是操作系统内核发出的信号，表明某个进程尝试打开的文件数超过了系统的限制。**watch** 这个词出现在错误信息中，是 Node.js 的 `fs.watch` 模块提供的上下文，它表明这个错误是在 `nodemon` 尝试监视文件时发生的。

可以初步得出结论：当 `nodemon` 执行“监视文件”这个动作时，向操作系统申请“打开并监视”一个新文件。但操作系统发现该进程持有的文件描述符数量达到了上限，于是拒绝了新的申请，并抛出 EMFILE 错误，导致程序崩溃。

### 查看资源限制

为验证此假设，先使用 `ulimit` 命令，显示当前用户（在当前这个终端会话中）的各项资源限制：

```bash  
ulimit -a  
```


终端输出：

![alt text](https://l1anch1.github.io/assets/my_blog_images/20250615-1.png)

其中，“-n: file descriptors 1048575”说明文件描述符限制为 1048575，也就是单个进程理论上可以打开的最大文件数。看着这个数值，我不禁感到疑惑。

# 2. 问题诊断

### launchd 的资源策略

难道我的项目中真的有超过一百万个文件吗？
其实，在 macOS 系统中，所有进程都派生自一个父进程 `launchd`，而正是这个 `launchd` 的最大文件数限制了用户可申请的文件数。也就是说 `launchd` 进程的限制优先于终端子进程的限制。终端实际采用的限制或许并不是 `ulimit` 命令查询出来的结果。

真相逐渐明了，通过下面这个指令直接向 `launchd` 查询它设定的权威限制：

```bash  
launchctl limit maxfiles  
```

发现输出为：


![alt text](https://l1anch1.github.io/assets/my_blog_images/20250615-2.png)

此处，256 代表软限制只有 256 个，即任何进程在未主动提升权限的情况下，其可持有的文件描述符数量不能超过 256。`unlimited` 代表硬限制（软限制可以被提升到的理论上限）是无限的。

这才是 `nodemon` 真正遵守的规则！

### 问题根本原因

综上所述，问题的根本原因就是：`nodemon` 默认会递归扫描并尝试监视项目目录下的所有文件，其中 `node_modules` 目录通常包含数万个文件。这一行为导致请求的文件描述符数量迅速超过由 `launchd` 设定的 256 的软限制，从而触发错误。

# 3. 解决方案：优化 nodemon 配置

为了解决这个 bug，最好的方法是优化工具自身的配置，让其忽略掉那些不需要监视的文件。通过在项目根目录创建 `nodemon.json` 配置文件，可以精确控制 `nodemon` 的监视范围。

### nodemon.json 示例配置：

```json  
{  
  "watch": [  
    "app.js",  
    "routes/",  
    "controllers/",  
    "common/"  
  ],  
  "ignore": [  
    "node_modules/",  
    "build/",  
    "public/",  
    "src/",  
    ".git/"  
  ],  
  "ext": "js,json",  
  "verbose": true,  
  "delay": "500"  
}  
```

- **"watch"** （白名单）：明确告诉 `nodemon` 只需要监视这几个核心的后端目录和文件。
- **"ignore"** （黑名单）：作为双重保险，强制 `nodemon` 忽略所有前端代码、依赖包和构建产物。
- **"ext"**：只关心 `.js` 和 `.json` 文件的变化。
- **"verbose": true**：让 `nodemon` 在启动和重启时打印出更详细的信息，方便调试。
- **"delay": 500**：在文件变动后，延迟 500 毫秒再重启。这可以有效防止因编辑器连续保存等行为导致的频繁重启。

应用此配置后，`nodemon` 的文件描述符使用量将大幅降低，从根本上解决了问题。再次运行后端服务器，问题得以解决。

# 4. 总结

一个看似简单的错误背后，可能隐藏着操作系统的底层设计哲学。作为开发者，遇到问题时，除了找到解决方案，更应该去追寻“为什么”。深入理解应用程序、工具链与操作系统之间的相互作用，是构建和维护健壮、可靠的软件系统的关键。