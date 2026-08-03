网络抓包
========================================

系统代理
----------------------------------------
+ 原理：设置系统代理，将流量转发给burp/fiddler等抓包工具。

透明代理
----------------------------------------
+ Proxifier + burp/fiddler

证书问题
----------------------------------------
+ 问题： 报 net::ERR_INSECURE_RESPONSE
	::

		//主进程增加以下代码
		app.on('certificate-error', (event, webContents, url, error, certificate, callback) => {
			event.preventDefault();
			callback(true); // 信任该证书
		});