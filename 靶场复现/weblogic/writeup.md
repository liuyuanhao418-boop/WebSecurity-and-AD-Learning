cve-2018-2894

搭建靶场

![示例图片](images/启动靶场.png)

以给的密码登录后台，点击base_domain -> 高级 -> 启用Web服务测试页 -> 保存

![示例图片](images/开启web服务.png)

访问/ws_utc/config.do（一个未授权访问路径）

设置 Work Home Dir 为：

/u01/oracle/user_projects/domains/base_domain/servers/AdminServer/tmp/_WL_internal/com.oracle.webservices.wls.ws-testclient-app-wls/4mcj4y/war/css

使用哥斯拉生成一个shell

![示例图片](images/生成shell.png)

依次点击安全->添加，来上传这个shell

![示例图片](images/上传成功.png)

使用浏览器检测源码，获取到一个时间戳

![示例图片](images/时间戳.png)

上传路径为 http://you-ip/ws_utc/css/config/keystore/[时间戳]_[文件名]

访问url测试上传成功没

![示例图片](images/测试上传.png)

上传成功，开启哥斯拉建立连接

![示例图片](images/建立连接.png)

成功

![示例图片](images/查看连接.png)













