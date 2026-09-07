CEF
=========================================================

基础
---------------------------
- CEF（Chromium Embedded Framework）：把 Chromium 引擎嵌入原生（C++/Qt 等）桌面应用的框架。典型用户：游戏客户端、银行/支付类、企业 IM、桌面工具。
- CEF 2P / 3P
    + 2P（Two-Party）：CEF 二进制随应用编译分发， **CEF 的漏洞 = 应用的漏洞** ，由应用厂商负责升级。桌面应用几乎都是 2P，因此"CEF/Chromium 旧版漏洞"属于应用漏洞。
    + 3P（Three-Party）：CEF 作为独立组件。
- 进程模型
    + Browser 进程：应用主 ``<app>.exe``，负责窗口/输入/渲染调度/原生 API。
    + Renderer 进程：执行页面 JS/DOM。Windows 上常以 **同一可执行文件self-exec** 为子进程，或加载独立子进程二进制。
    + GPU / Utility 进程：GPU、媒体、网络。
    + 单进程模式（ ``--single-process`` / ``CefSettings::single_process``）：所有角色挤一进程，隔离消失，利用门槛更低。
- CEF vs Electron
    + Electron = Chromium + **Node.js** （关注 ``nodeIntegration`` / ``contextIsolation`` / ``preload``）。
    + CEF = 纯 Chromium 嵌入 + **C++ 应用** ,JS↔原生走 **CefV8 桥** （非 Node）。
    + 二者都嵌 Chromium，故"内核 CVE 复用"都适用；但"JS→原生"机制不同，审计手法不同。

识别CEF 应用
--------------------

**静态签名**

- 关键库：Windows ``libcef.dll``；macOS ``libcef.dylib``；Linux ``libcef.so``。
- 强信号 DLL：``cef.dll``、``chrome_elf.dll``、``v8.dll``、``d3dcompiler_47.dll``。
- 进程名：主进程常为应用名；子进程多为"同一 .exe 带参数"或 ``*_renderer`` / ``*helper``。
- 字符串：``libcef``、``cef_``、``CefBrowser``、``CefClient``、``CefV8*``、
  V8 扩展注册相关字符串。

**动态确认**

- 任务管理器看进程树：主进程 + 若干子进程（renderer/GPU/utility）。
- 查看进程模块是否含 ``libcef``。
- 查看完整命令行：CEF 应用命令行通常带一串 Chromium 开关，既"自证身份"又直接暴露安全配置。

**确认命令（Windows）**

.. code-block:: powershell

   # 确认是否 CEF（看是否加载 libcef）
   (Get-Process -Name <app>.exe).Modules | Where-Object { $_.ModuleName -like '*cef*' } | Select-Object ModuleName,FileName

   # 看完整命令行（直接暴露 --remote-debugging-port 等开关）
   Get-CimInstance Win32_Process -Filter "name='<app>.exe'" | Select-Object -ExpandProperty CommandLine

攻击面总览
-------------
- **#1 CDP 远程调试端口（``remote_debugging_port``）** — 前置：应用开启 / 命令行注入；
  影响：任意 JS、窃会话、内部 API、绕鉴权，绑 0.0.0.0→远程；优先级：★★★★★
- **#2 JS↔原生桥（CefV8）危险方法 + JS 执行** — 前置：XSS / 调试端口 / 内容加载；
  影响：本地 RCE；优先级：★★★★★
- **#3 命令行开关注入** — 前置：可影响 CEF 命令行；影响：开调试、读数据、加载任意内容；
  优先级：★★★★
- **#4 ``browser_subprocess_path`` / 子进程二进制** — 前置：本地可写 / 弱 ACL；
  影响：本地 RCE；优先级：★★★★
- **#5 IPC 通道（命名管道/共享内存）ACL** — 前置：本地；影响：消息注入/控制；优先级：★★★
- **#6 本地 HTTP API 未鉴权 / CORS 错** — 前置：本地 / 同机页面；影响：本地提权；优先级：★★★
- **#7 ``file://`` 放宽 + JS 执行** — 前置：JS 执行；影响：本地文件读；优先级：★★★
- **#8 自动接受证书错误（``ignore_certificate_errors``）** — 前置：中间人；影响：MitM /
  劫持会话；优先级：★★★
- **#9 单进程模式 / 沙箱禁用** — 前置：配置；影响：降低利用门槛；优先级：★★
- **#10 自定义 scheme（``app://`` 等）** — 前置：其他应用/页面触发；影响：深链滥用；优先级：★★
- **#11 旧 Chromium/CEF 内核 CVE** — 前置：版本；影响：直接 RCE/逃逸；优先级：视版本
- **#12 CSP / 安全头缺失** — 前置：内容加载；影响：放大 XSS/注入；优先级：★★

核心漏洞
----------------------------

CDP远程调试端口
~~~~~~~~~~~~~~~~~~~~~~~~~~~
- DevTools 后端由 ``CefSettings::remote_debugging_port`` 或 ``--remote-debugging-port`` 启用。

JS↔原生桥（CefV8）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- CEF 允许 C++ 应用把 **原生函数暴露给页面 JS**
  - ``CefV8Handler`` / ``CefV8ContextHandler`` / ``CefV8Accessor`` / ``CefV8PropertyHandler`` 注册 JS 对象（如 ``window.NativeBridge``），方法回调到 C++
  - 应用用 ``CefFrame::ExecuteJavaScript`` 注入一段 JS shim 定义桥对象
- 页面 JS 里出现形如：

  .. code-block:: javascript

     NativeBridge.readFile(path)
     NativeBridge.writeFile(path, data)
     NativeBridge.exec(cmd)
     NativeBridge.getAuthToken()
     NativeBridge.openExternal(url)

命令行开关注入
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- CEF 启动 = 一串 Chromium 开关。应用（或其上游输入）拼接开关时，若存在 **可控字符串** （启动参数、配置文件、"打开 URL"、剪贴板深链、拖入文件），即可 **注入额外开关** 
    + ``--remote-debugging-port=9222`` （直接触发 §5.1）
    + ``--remote-debugging-address=0.0.0.0:9222`` （远程调试）
    + ``--user-data-dir=<path>`` （指向/读另一用户数据目录 → 会话/凭证）
    + ``--app=<url>`` （加载任意内容到应用上下文）
    + ``--allow-file-access-from-files`` （放大文件读）
    + ``--disable-web-security`` （弱化同源，放大内部 API 访问）
    + ``--no-sandbox`` / ``--single-process`` （降隔离，放大内核类利用）
    + ``--js-flags=...`` （改 V8 行为）、 ``--enable-features=...`` / ``--disable-features=...``
- 注入点通常在"应用构造 CEF 命令行"或"把外部字符串直接拼进命令行"的代码路径。

子进程路径劫持
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- CEF 启动 renderer/子进程：默认 **self-exec** （用主 ``.exe`` 再启一次带标志），或用 ``CefSettings::browser_subprocess_path`` 指向独立子进程二进制。
- 风险点
    + ``browser_subprocess_path`` 指向 **可写目录 / 弱 ACL / 相对路径**  → 本地非特权用户可 **替换/植入恶意可执行文件**  → 应用启动时加载 →  **以应用权限执行恶意代码**
    + self-exec 模式下主 ``.exe`` 若位于可写目录，同理。
- 属 **本地提权 / 代码执行** ，常见于多用户/共享终端场景。

IPC 通道（命名管道 / 共享内存）ACL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- CEF browser 与 renderer 进程间走 Chromium IPC
    + Windows： **命名管道** （ ``\\.\pipe\...`` ）
    + POSIX： **Unix domain socket**
    + **共享内存** 区域
- 若通道 **名称可预测** 、 **ACL 过宽** （任意用户可连/可读写共享内存），本地进程可连接命名管道注入/读取 IPC 消息、读写共享内存 → 干扰/控制应用状态。
- 属 **本地攻击** ，用于 **本地提权/横向**

**挖掘手法**

1. 枚举 CEF 进程打开的命名管道/共享内存（Sysinternals ``handle64.exe -a``、 ``listdlls`` 、 ``Get-Process`` + ``handle.exe`` ）
2. 检查各通道 ACL（``icacls``、管道对象 ACL 用 ``accesschk`` / 专门工具）。
3. 授权验证 ACL 过宽即可（不需真正注入）。
4. 结合 CEF IPC 消息格式，评估"注入某消息能否触发某行为"（点到为止，遵守 Safe Payload）。


``file://`` 放宽 + JS 执行 → 本地文件读
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- 若应用 **放宽 ``file://`` 同源/读取限制** （ ``--allow-file-access-from-files`` 、自定义scheme 对本地文件放行），且攻击者 **能在渲染器执行 JS** （§5.1/XSS），即可读本地文件：

  .. code-block:: javascript

     fetch('file:///C:/Users/<u>/AppData/...').then(r=>r.text())

- 常与桥方法（ ``readFile`` ）二选一作为"文件读"证据；有桥方法时优先用桥（更直接）。

自定义 scheme
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- CEF 应用常注册自定义 URL scheme 用于深链 / 跨应用通信（ ``myapp://...`` 、 ``cef://...`` ）。
- 风险：
    + 恶意网页 / 其他应用触发该 scheme → 本地应用**跳转/执行动作**（深链滥用）；
    + scheme 处理器若带权限（读数据、改状态、触发内部 API）→ 被诱导触发即本地提权；
    + scheme 与本地 API 交叉（scheme 参数进 API）→ 参数注入。

逆向分析技术（静态 + 动态）
------------------------------

静态分析
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **确认 CEF + 版本**：DLL 版本资源、字符串（ ``libcef`` ``cef_`` ``CefV8*`` 、V8 扩展注册）。
- **找 CefSettings 配置**：定位赋值，读出
    + ``remote_debugging_port`` （是否 > 0）
    + ``ignore_certificate_errors``
    + ``single_process``
    + ``browser_subprocess_path`` （指向哪、是否可写）
    + ``command_line_args`` （额外开关）
- **找 JS 桥（CefV8）**：定位 ``CefV8Handler``/``CefV8Accessor`` 注册点，读出 **方法名→C++ 回调** 映射，判断每个方法危险性（文件/进程/密钥/网络）
- **找命令行构造点**：搜拼接开关的字符串（ ``--`` 前缀），看是否有外部输入进入
- **找本地 API / scheme**：搜 ``127.0.0.1``、端口、scheme 注册、 ``app://``
- **工具**：``strings``、``rg``/``grep``、反汇编（``IDA Pro`` / ``Ghidra`` / ``Binary Ninja`` ）、 ``listdlls``

工具与命令速查
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**进程 / 命令行 / 模块（Windows）**

.. code-block:: powershell

   # 进程 + 完整命令行
   Get-CimInstance Win32_Process -Filter "name='<app>.exe'" |
     Select-Object ProcessId,Name,CommandLine,ExecutablePath

   # 加载的 CEF 模块
   (Get-Process -Name <app>.exe).Modules | Where-Object { $_.ModuleName -like '*cef*' } | Select-Object ModuleName,FileName

   # CEF 进程的监听端口
   Get-NetTCPConnection -State Listen | Where-Object { $_.OwningProcess -in (Get-CimInstance Win32_Process -Filter "name='<app>.exe").ProcessId } | Select-Object LocalAddress,LocalPort,OwningProcess

**CDP 探测（确认调试端口）**

.. code-block:: bash

   curl -s http://127.0.0.1:<port>/json/version
   curl -s http://127.0.0.1:<port>/json

连上后用 CDP 客户端调 ``Runtime.evaluate`` / ``Page.getResourceTree`` /
``Network.getAllCookies`` 等。

- CDP 客户端选项：``puppeteer`` / ``playwright`` / ``pyppeteer`` / Node
  ``chrome-remote-interface`` / 裸 ``websocket-client`` + CDP 协议。

**ACL / 命名管道 / 共享内存**

.. code-block:: bash

   icacls "C:\path\to\subprocess.exe"
   accesschk.exe -uw "C:\path\to\subprocess.exe"
   handle64.exe -a            # 列出命名管道 + 属主（Sysinternals）
   listdlls.exe -m <app>.exe

**跨平台（Python）**

- ``psutil``：进程、命令行、连接、模块（``psutil.Process().memory_maps()``）
- ``socket`` / ``scapy``：端口/协议探测
- CDP：``websocket-client`` + CDP，或 ``pyppeteer``
