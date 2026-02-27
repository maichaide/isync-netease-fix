# isync-netease-fix
修改isync源码后使aerc等邮件客户端可以正常使用网易邮箱IMAP服务
isync官方不支持网易邮箱的ID 命令，本项目为修改isync 1.5.1源码并在termux上实际使用后上传分享。

# 项目简介
说明这个补丁解决了网易邮箱 IMAP 连接时报 BYE IMAP4rev1 Server logging out 或握手失败的问题。

在手机 termux 上编译，由于在 Termux 下的C语言库 Bionic Libc 为手机而生，Android 系统为了减小体积，剔除了许多 glibc 的非必要扩展，在 Termux (Android) 环境下编译 isync 会大量报错。

**说明** 如果要手机 termux 下编译，请执行以替换支持函数操作，如果是在手机以外的电脑系统上使用，则不需要替换支持函数，可以正常编译
### 替换支持函数
把 Termux 使用的 Android Bionic C 库不支持的 fwrite_unlocked 等函数（这是 GNU 扩展，通常在标准的 Linux glibc 中提供，但在轻量级或非标准 C 库中不存在）进行替换就可以顺利编译了。

```
运行以下 sed 命令一键替换当前目录下所有的非标准函数：
sed -i 's/fwrite_unlocked/fwrite/g' src/util.c
sed -i 's/fputs_unlocked/fputs/g' src/util.c
sed -i 's/putc_unlocked/putc/g' src/util.c
```

### 也可以下载isync官方最新的isync-1.5.1.tar.gz源码包后，使用.patch文件补丁后编译
[点击查看aerc + isync-netease 收发网易邮件折腾笔记](./aerc-netease.md)
