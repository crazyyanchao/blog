---
title: UNSAFE_LEGACY_RENEGOTIATION_DISABLED问题解决
tags: [HTTPS]
author: Yc-Ma
show_author_profile: true
key: 2023-05-23-UNSAFE_LEGACY_RENEGOTIATION_DISABLED问题解决
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

# UNSAFE_LEGACY_RENEGOTIATION_DISABLED问题解决

- 可以降低版本到 OpenSSL 1.1.1 或使用以下代码:

```
import urllib3
from urllib3.util.ssl_ import create_urllib3_context

ctx = create_urllib3_context()
ctx.load_default_certs()
ctx.options |= 0x4  # ssl.OP_LEGACY_SERVER_CONNECT

with urllib3.PoolManager(ssl_context=ctx) as http:
    r = http.request("GET", "https://nomads.ncep.noaa.gov/")
    print(r.status)
```

[GITHUB: ssl.SSLError: [SSL: UNSAFE_LEGACY_RENEGOTIATION_DISABLED] unsafe legacy renegotiation disabled (_ssl.c:1131)](https://github.com/urllib3/urllib3/issues/2653)


