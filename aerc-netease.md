# 网易邮箱人为曲解RFC规范<br>aerc 收发网易邮箱折腾笔记

![](https://fastly.jsdelivr.net/gh/bucketio/img0@main/2026/02/27/1772167731480-1e32e193-ca18-4561-b748-b43e2eb45d8a.png)

在命令行下工作高效便捷，aerc 是一款 TUI 界面的便捷好用的邮件客户端，很方便地配置好了QQ邮箱，但配置网易邮箱时遇到了问题，即使IMAP服务配置正确的情况下，仍然报错 Unsafe Login，无法正常登录收取邮件。

### RFC规范被曲解利用
邮件协议RFC 2971规范中设置了 IMAP ID 命令，被设计为可选的身份声明，通常用于统计或兼容性调试。

然而，网易邮箱将其作为一种“拦截机制”来强制区分客户端。

### 见人下菜
网易邮箱要求客户端在登录后（或选择文件夹前）必须发送 IMAP ID 命令（符合 RFC 2971 规范），以声明客户端的身份。如果客户端不发送此信息，网易服务器会返回 Unsafe Login 错误并拒绝同步邮件（纯粹人为制造问题）。

腾讯邮箱 IMAP 服务不存在这个问题，设置好IMAP服务 aerc 收发邮件一切正常。

![](https://fastly.jsdelivr.net/gh/bucketio/img3@main/2026/02/27/1772167769631-ebae577b-d806-45d6-998a-dc39dbcf05c7.png)

### 解决思路
原因找到了，解决思路就是提供IMAP ID命令响应，并且假装是网易邮箱主流客户端Foxmail。这需要修改邮件IMAP处理逻辑源码。

修改 aerc 邮件客户端源码有些麻烦，发现使用 isync 是个方便稳定易上手的办法。
### 强大的 isync 邮箱同步程序
[isync 官方网址](http://isync.sf.net/See) isync 是项目名称，`mbsync`是当前可执行文件名称；`mbsync`是一个命令行应用程序，用于同步邮箱，支持 Maildir 和 IMAP4 邮箱，新邮件，消息删除和标记更改可以双向传播。

### 编译源码步骤
```
# 首次编译
./autogen.sh (仅从 git 构建时需要)
./configure
make

# 重新编译前需要清理：
make clean
make
```
我是在手机 termux 上编译，由于在 Termux 下的C语言库 Bionic Libc 为手机而生，Android 系统为了减小体积，剔除了许多 glibc 的非必要扩展，在 Termux (Android) 环境下编译 isync 会大量报错。

### 替换支持函数
把 Termux 使用的 Android Bionic C 库不支持的 fwrite_unlocked 等函数（这是 GNU 扩展，通常在标准的 Linux glibc 中提供，但在轻量级或非标准 C 库中不存在）进行替换就可以顺利编译了。

```
运行以下 sed 命令一键替换当前目录下所有的非标准函数：
sed -i 's/fwrite_unlocked/fwrite/g' src/util.c
sed -i 's/fputs_unlocked/fputs/g' src/util.c
sed -i 's/putc_unlocked/putc/g' src/util.c
```
下载最新的isync 1.5.1.tar.gz源码包后，在替换完以上函数后可以顺利编译，但不支持网易邮箱的ID 命令，所以需要修改 isync 源码。

### 源码修改方法
虽然 isync 项目的 mbsync 也不支持发送 ID，但修改起来相对容易。主要检查 drv_imap.c 源码: IMAP 的交互逻辑都在 src/drv_imap.c 中。

```
grep -i "IMAP_CMD_ID" src/drv_imap.c
# 或者搜索字符串
grep -i "\"ID\"" src/drv_imap.c
```
如果搜索结果将为空，说明需要修改源码，添加 ID 命令支持。

==主要修改源码==
```
imap_exec( ctx, NULL, NULL, "ID (\"name\" \"foxmail\" \"version\" \"7.2\" \"vendor\" \"tencent\")" );
```
在源码中搜索` imap_open_store_p2 `函数的具体实现，将其修改为如下逻辑：
```
static void
imap_open_store_p2( imap_store_t *ctx, imap_cmd_t *cmd ATTR_UNUSED, int response )
{
	if (response != OK) {
		imap_open_store_bail( ctx, response );
		return;
	}

	/* --- 核心修改：注入 ID 指令 --- */
	// 增加一个判断，如果服务器支持 ID 且我们还没发过
	if (CAP(ID)) {
		info( "Sending NetEase ID bypass...\n" );
		// 发送 ID，并将回调设为 imap_open_store_authenticate
		// 这样服务器回了 ID OK 后，才会执行下一步的认证
		imap_exec( ctx, NULL, (void(*)(imap_store_t *, imap_cmd_t *, int))imap_open_store_authenticate, 
                   "ID (\"name\" \"Foxmail\" \"version\" \"7.2\" \"vendor\" \"Tencent\")" );
		return; // 关键：在这里中断，等待 ID 回调
	}
	/* --- 修改结束 --- */

	// 原有逻辑：处理 TLS 或 直接认证
	if (CAP(STARTTLS) && !ctx->conn.ssl) {
		imap_open_store_tlsstarted1( 0, ctx );
		return;
	}
	imap_open_store_authenticate( ctx );
}
```
修改好源码后重新编译，顺利得到mbsync，位置在src文件夹中，将它复制到$PREFIX/bin文件夹中，方便termux下使用。

![](https://fastly.jsdelivr.net/gh/bucketio/img3@main/2026/02/27/1772167516851-e832aeb5-0129-413b-bf7d-a703df3c09b8.png)

### 让 mbsync 首先同步网易邮箱
1. **建立 `mbsyncrc` 配置文件**

```
# .mbsyncrc 配置示例
IMAPAccount 126
Host imap.126.com
User 你的账号@126.com
Pass 你的授权码
TLSType IMAPS
AuthMechs LOGIN

IMAPStore 126-remote
Account 126

MaildirStore 126-local
# 建议路径保持一致
Path ~/.local/share/mail/126/
Inbox ~/.local/share/mail/126/INBOX
SubFolders Verbatim

Channel 126-channel
Far :126-remote:
Near :126-local:
# 只同步收件箱，这是最容易成功的
Patterns INBOX
Create Near
SyncState *
```
2. **同步网易邮箱 IMAP 服务**
```
## 查错调试 -V -D 开关显示同步信息
mbsync -V -D -c ~/.mbsyncrc 126-channel
```
必要时可以使用 openssl 手动连接测试：
```
openssl s_client -connect imap.126.com:993 -crlf
连接成功后，按顺序输入（每行回车）：
A1 LOGIN 你的账号@126.com 你的授权码
A2 ID ("name" "Foxmail")
A3 LIST "" "*"
观察 A3 输出的结果：
如果它列出的是 * LIST (\HasNoChildren) "/" "INBOX"，说明路径没问题。
如果它列出的是 * LIST (\HasNoChildren) "/" "收件箱"，就需要在 mbsync 里做中文编码映射。
```

![](https://fastly.jsdelivr.net/gh/bucketio/img19@main/2026/02/27/1772167562073-aaa2ae28-6d2b-4c70-a818-024d1917d131.png)

3. **aerc 的账号配置文件**
aerc 邮件客户端配置文件通常在 `~/.config/aerc/accounts.conf`，在邮箱帐户设置中将 source 修改为刚才在 mbsync 中设定的路径。
```
[126-Mail]
# 这里的路径必须与 mbsyncrc 中的 Path 保持一致
# 使用 maildir:// 协议告诉 aerc 读取本地同步好的邮件
source = maildir://~/.local/share/mail/126

# 发信设置直接连网易 SMTP
outgoing = smtps://你的账号@126.com:你的授权码@smtp.126.com:465
from = 你的名字 <你的账号@126.com>

# 刷新邮箱调用你编译的 mbsync
check-mail-cmd = ~/isync-1.5.1/src/mbsync 126-channel
# 每 5 分钟自动运行一次同步
check-mail-timeout = 5m
```

