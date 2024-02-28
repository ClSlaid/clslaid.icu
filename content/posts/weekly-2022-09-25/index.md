---
title: "Weekly 2022 09 25"
date: 2022-09-25T19:17:34+08:00
draft: false
---

## Databend

本周感觉全公司都在做 `COPY INTO` 的工作，鄙人忝列期间，摸鱼之际为其中增加了：

1. IPFS 支持
2. FTP/FTPS 支持
3. AWS 的 Temporary Credential 支持
4. 修复了端口号被忽略的问题

   以及其他杂活

### IPFS 支持

只需要文件夹 CID 和具体路径，现在 databend 可以直接从 IPFS 中拷贝数据了。

```mysql
COPY INTO mytable FROM 'ipfs://Qmc1145141919810/yaju.csv' FILE_FORMAT=(TYPE='CSV');
```

### FTP/FTPS 支持

FTP 这个协议虽然挺老，但是有一定历史沉淀的机构比如部分政府机关都会提供 FTP 服务器用于分享文件。目前 databend 可以使用 FTP 和 FTPS 协议了！

```mysql
COPY INTO mytable FROM 'ftps://ftp_server.example.com/yaju.csv' FILE_FORMAT=(TYPE='CSV');
```

### Temporary Credential

AWS 提供了一个在一定时间后会过期的 Temporary Credential 功能，相对于传统的 Credential，由于其具有一定的作用时限，因此即使被泄露，造成的后果也不会那么严重。

```mysql
COPY INTO books FROM "s3://my_bucket/my_path/books.csv" CONNECTION = (AWS_KEY_ID = "aws_key", AWS_SECRET_KEY = "aws_sec", AWS_TOKEN = "AWS_TOKEN") FILE_FORMAT = (TYPE = "CSV" );
```

## 域名续费

晚上和同学分享博客内容的时候发现自己的博客打不开了，仔细一看原来是自己的域名过期了。好一通折腾最终才将博客重新恢复过来。

## 更换编辑器

之前由于 rust-analyzer 的过程宏分析在 rust 1.62 以上版本不可用，intellij-rust 很快修复并能正常展开宏，令人感叹。于是切换到 intellij-rust (CLion) 使用了一段时间，实际上 CLion 最吸引人的不是它的 rust 编程体验，而是一些其他方面。比如方便快捷的 Git 操作界面，让人可以在痛苦的解 resolve 中轻松不少。

不过最主要的是它的 VIM 按键绑定，比 VSCode 显得更加舒服。在 rust-analyzer 修复后的一段时间回到 VSCode 中尝试 rust 编程，结果发现 vscode-nvim 在 rust 编程时候的卡顿显得令人难以接受，在输入法上的表现也有些差强人意。更别说 vscode-vim 了——谁用那劳什子东西。

不过鄙人总觉得被一个一个全家桶“套牢”不是一件什么好事，相比之下更愿意花一些时间尝试一下其他的东西。想来想去不如直接使用 neovim 吧！于是重新打开了尘封已久的 neovim，升级拷贝了由 [@ayamir](github.com/ayamir) 维护的 neovim [配置文件集合](github.com/ayamir/nvimdots.git)。花了亿点时间自己配置了一下，打算作为下周的主力编辑器使用。

## 生活

现在进澡堂还是需要扫码，不过感觉大家都已经接受了。

周六的时候前往丰台参加了无线电执照考试，出地铁站的时候手机电量由 40% 三秒归零，差点没有考试成功。最终感觉还是考砸了。

那种大脑被 Burn Out，想不出写法的状态又慢慢回来了，在周四周五的感觉尤其严重。感觉应该做好 WLB，同时兼顾工作效率和个人休息。
