---
title: 使用telnet模拟HTTP和SMTP协议
date: 2022-04-04 20:34:49
tags:
    - 计算机网络
    - telnet
categories:
    - 计算机网络
---

> 前言：使用telnet模拟HTTP请求访问页面，及其使用telnet发送邮件

# 一、运行环境

- ubuntu 16.04
- telnet

<!--more-->

# 二、模拟HTTP请求

参考[CS144 lab0](https://cs144.github.io/assignments/lab0.pdf)

## 2.1 请在浏览器中先访问http://cs144.keithw.org/hello查看内容

## 2.2 我们可以通过telnet做到同样的事情

1. 在终端中输入`telnet cs144.keithw.org http`, 你能够看到以下内容(需要注意的事情是这里输入的网址并不包含`/hello`路径)。

   ```
   ~$ telnet 148.163.153.234 smtp
   Trying 148.163.153.234...
   Connected to 148.163.153.234.
   Escape character is '^]'.
   ```

2. 在终端输入`GET /hello HTTP/1.1`，并按下回车。

3. 继续在终端输入`Host: cs144.keithw.org`，并按下回车。

4. 继续输入`Connection: close`，并按下回车。

5. **不输入任何内容**，并按下回车。

6. 然后你就会看到服务器发来的页面信息。

![img](http://img.singhe.art/FpJoWxvW8X2d-MVv3f2liOcC___z)

### 三、使用telnet发送邮件

> 以下内容参考https://www.cnblogs.com/LYFer233/p/15215991.html

由于无法使用斯坦福的邮箱，这里使用QQ邮箱作为示例，各个邮箱采用SMTP所以流程都大同小异。

1. 在终端输入`telnet smtp.qq.com smtp`，并按下回车，显示以下内容

   ```
   ➜~ telnet smtp.qq.com smtp
   Trying 14.18.175.202...
   Connected to smtp.qq.com.
   Escape character is '^]'.
   220 newxmesmtplogicsvrszc8.qq.com XMail Esmtp QQ Mail Server.
   ```

2. 在终端输入`HELO qq.com`，并按下回车，给服务器打个招呼，得到以下内容。

   ```
   HELO qq.com
   250-newxmesmtplogicsvrszc8.qq.com-9.46.31.207-101844334
   250-SIZE 73400320
   250 OK
   ```

3. 登录你的QQ邮箱，依次进入：设置->账户->POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务，然后开启IMPA/SMTP选项，生成邮箱授权码。

   ![img](http://img.singhe.art/FmnKKUHbO5bmYODrtxHlwe80Zcft)

4. 接下来进行登录，在终端输入`auth login`，进行登录，需要输入你的邮箱的base64编码按下回车，然后输入你邮箱的授权码的base64编码，按下回车。可以在[https://www.base64encoder.io](https://www.base64encoder.io/)进行base64编码。

   如出现`235 Authentication successful`，则说明登录成功。

5. 接下来输入发信人邮箱。在终端输入`mail from： <xinjiempolde@qq.com>`

6. 然后输入收信人邮箱。在终端输入`rcpt to： <xinjiempolde@qq.com`,这里以自己给自己发送邮件作为示例。

7. 然后在终端输入`data`，告诉服务器我们要输入内容了，并收到以下提示。

   ```
   data
   354 End data with <CR><LF>.<CR><LF>.
   ```

8. 输入邮箱的头信息、主题、内容等。注意以`.`和一行空行结束。

   ```
   from: xinjiempolde@qq.com
   to: xinjiempolde@qq.com
   subject: telnet learning
   
   this is a test for learning smtp with telnet.
   .
   ```

9. 输入`quit`结束并发送邮件

完整的流程截图如下：

![img](http://img.singhe.art/FlWSiNNbPEevOvqMhsS8QbL48Td4)

# 四、发送带有附件的邮件

> 本节内容参考[telnet 收发邮件，带附件](https://zhuanlan.zhihu.com/p/46619234)

古老的`SMTP`，也就是`Simple Mail Transfer Protocol`，只支持传输ASCII文本，不能支持其他附件的传输。如果要传输附件的话，需要在SMTP的基础上使用`MINE`。

其他邮箱可能稍有不同，这里以QQ邮箱为例，完整流程如下：

```
telnet smtp.qq.com smtp
helo qq.com
auth login
发送方邮件名的base64编码
邮箱授权码的base64编码
mail from:<邮箱地址> 注意有<>
rcpt to:<接收方邮箱地址>
data
from: <你的邮箱地址>
to: <接收方邮箱地址>
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="a"
subject: test for transferring file 注意下方一定要有空行

--a
this is a test mail
--a
Content-Transfer-Encoding: base64
Content-type:application/octet-stream; name="back.jpg"

(这里是back.jpg的base64编码，可以通过base64 back.jpg > back.txt获取base64编码)
--a--
.
quit
```

# 五、结语

HTTP和SMTP都是在应用层，使用TCP协议进行传输，且都是通过ASCII码的形式进行请求和响应，因此，我们能够通过telnet这一工具进行实验。
