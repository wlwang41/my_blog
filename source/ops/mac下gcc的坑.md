---
title: "mac下gcc的坑"
date: 2014-07-11 17:17
---

某一天我更新了OSX和xcode之后，突然发现装PIL的时候出错了.

查了半天在苹果官网上找到下面这样的说明:

> As of Apple LLVM compiler version 5.1 (clang-502) and later, the optimization level -O4 no longer implies link time optimization (LTO). In order to build with LTO explicitly use the-flto option in addition to the optimization level flag. (15633276)

> The Apple LLVM compiler in Xcode 5.1 treats unrecognized command-line options as errors. This issue has been seen when building both Python native extensions and Ruby Gems, where some invalid compiler options are currently specified.
Projects using invalid compiler options will need to be changed to remove those options. To help ease that transition, the compiler will temporarily accept an option to downgrade the error to a warning:
-Wno-error=unused-command-line-argument-hard-error-in-future

> Note: This option will not be supported in the future.

> To workaround this issue, set the ARCHFLAGS environment variable to downgrade the error to a warning. For example, you can install a Python native extension with:
$ ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future easy_install ExtensionName
Similarly, you can install a Ruby Gem with:
$ ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future gem install GemName

这就意味着, 以后装的库只要带gcc make的都需要加上`ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future`
