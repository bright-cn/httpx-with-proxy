# 在 HTTPX 中使用代理

[![宣传图](https://github.com/luminati-io/Rotating-Residential-Proxies/blob/main/50%25%20off%20promo.png)](https://www.bright.cn/proxy-types/residential-proxies) 

本指南将演示如何在 HTTPX 中使用代理，包括无需身份验证、有身份验证、轮换以及后备代理的示例。

## 目录
1. [使用无需身份验证的代理](#使用无需身份验证的代理)
2. [使用需要身份验证的代理](#使用需要身份验证的代理)
3. [使用轮换代理](#使用轮换代理)
4. [创建后备代理连接](#创建后备代理连接)
5. [总结](#总结)

## 使用无需身份验证的代理

对于无需身份验证的代理，我们不使用用户名或密码，所有请求都将转发到一个 `proxy_url`。下面的示例代码展示了如何使用无需身份验证的代理：

```python
import httpx

proxy_url = "http://localhost:8030"

with httpx.Client(proxy=proxy_url) as client:
    ip_info = client.get("https://geo.brdtest.com/mygeo.json")
    print(ip_info.text)
```

## 使用需要身份验证的代理

需要身份验证的代理需要提供用户名和密码。只要提供正确的凭据，就可以连接到代理。

在进行身份验证时，`proxy_url` 类似这样：`http://<username>:<password>@<proxy_url>:<port_number>`。下面的示例展示了如何将 `zone` 和 `username` 组合到身份验证字符串中，并使用 [Bright Data 的数据中心代理](https://www.bright.cn/proxy-types/datacenter-proxies) 作为连接基础：

```python
import httpx

username = "your-username"
zone = "your-zone-name"
password = "your-password"

proxy_url = "http://brd-customer-{username}-zone-{zone}:{password}@brd.superproxy.io:33335"

ip_info = httpx.get("https://geo.brdtest.com/mygeo.json", proxy=proxy_url)

print(ip_info.text)
```

代码说明如下：

- 首先创建了三个配置变量：`username`、`zone` 和 `password`。  
- 用它们组合出 `proxy_url`：`"http://brd-customer-{username}-zone-{zone}:{password}@brd.superproxy.io:33335"`。  
- 向所示 API 发送请求，以获取我们代理连接的一般信息。  

返回结果应该类似如下示例：

```json
{"country":"US","asn":{"asnum":20473,"org_name":"AS-VULTR"},"geo":{"city":"","region":"","region_name":"","postal_code":"","latitude":37.751,"longitude":-97.822,"tz":"America/Chicago"}}
```

## 使用轮换代理

“轮换代理”指的是创建一个代理列表，并在请求之间随机选择使用哪个代理。下面的示例代码先创建了一个由国家/地区构成的列表 `countries`，然后在每次发送请求时，用 `random.choice()` 随机选择列表中的某个国家。我们的 `proxy_url` 会被格式化为指定国家的代理。这里使用的 [轮换代理](https://www.bright.cn/solutions/rotating-proxies) 列表来自 Bright Data。

```python
import httpx
import asyncio
import random

countries = ["us", "gb", "au", "ca"]
username = "your-username"
proxy_url = "brd.superproxy.io:33335"

datacenter_zone = "your-zone"
datacenter_pass = "your-password"

for random_proxy in countries:
    print("----------connection info-------------")
    datacenter_proxy = f"http://brd-customer-{username}-zone-{datacenter_zone}-country-{random.choice(countries)}:{datacenter_pass}@{proxy_url}"

    ip_info = httpx.get("https://geo.brdtest.com/mygeo.json", proxy=datacenter_proxy)

    print(ip_info.text)
```

与需要身份验证的代理示例非常相似，主要区别在于：

- 这里创建了一个国家/地区数组：`["us", "gb", "au", "ca"]`。  
- 我们将发送多次请求。每次请求都会使用 `random.choice(countries)` 来随机选择一个国家，并将其注入到 `proxy_url` 中。  

## 创建后备代理连接

上面的示例均使用数据中心代理或免费代理。前者经常被网站屏蔽，后者则稳定性较差。因此，为保证可用性，理想的做法是添加一个对住宅代理的后备机制。

为此，我们编写了一个名为 `safe_get()` 的函数。调用时，它会先通过数据中心代理尝试访问 URL，如果失败，就_后备_到住宅代理。

```python
import httpx
from bs4 import BeautifulSoup
import asyncio

country = "us"
username = "your-username"
proxy_url = "brd.superproxy.io:33335"

datacenter_zone = "datacenter_proxy1"
datacenter_pass = "datacenter-password"

residential_zone = "residential_proxy1"
residential_pass = "residential-password"

cert_path = "/home/path/to/brightdata_proxy_ca/New SSL certifcate - MUST BE USED WITH PORT 33335/BrightData SSL certificate (port 33335).crt"

datacenter_proxy = f"http://brd-customer-{username}-zone-{datacenter_zone}-country-{country}:{datacenter_pass}@{proxy_url}"
residential_proxy = f"http://brd-customer-{username}-zone-{residential_zone}-country-{country}:{residential_pass}@{proxy_url}"

async def safe_get(url: str):
    async with httpx.AsyncClient(proxy=datacenter_proxy) as client:
        print("trying with datacenter")
        response = await client.get(url)
        if response.status_code == 200:
            soup = BeautifulSoup(response.text, "html.parser")
            if not soup.select_one("form[action='/errors/validateCaptcha']"):
                print("response successful")
                return response
    print("response failed")
    async with httpx.AsyncClient(proxy=residential_proxy, verify=cert_path) as client:
        print("trying with residential")
        response = await client.get(url)
        print("response successful")
        return response

async def main():
    url = "https://www.amazon.com"
    response = await safe_get(url)
    with open("out.html", "w") as file:
        file.write(response.text)

asyncio.run(main())
```

代码说明如下：

- 我们有两组配置变量：一组用于数据中心代理连接，另一组用于住宅代理连接。  
- 此示例使用 `AsyncClient()`，展示 HTTPX 更为高级的功能。  
- 首先尝试使用 `datacenter_proxy` 发出请求。  
- 如果未能获得预期的响应，就退回使用 `residential_proxy`。注意代码中的 `verify` 参数。当使用 Bright Data 的住宅代理时，需要下载并使用 [SSL 证书](https://docs.brightdata.com/general/account/ssl-certificate)。  
- 一旦获得合格的响应，我们会将页面写入本地 HTML 文件，我们可以用浏览器查看代理实际请求并返回的页面。  

运行上述示例后，输出和生成的 HTML 文件应类似如下所示：

```
trying with datacenter
response failed
trying with residential
response successful
```

![亚马逊首页截图](https://github.com/bright-cn/httpx-with-proxy/blob/main/Images/image.png)

## 总结

当你将 HTTPX 与 [Bright Data 的高质量代理服务](https://www.bright.cn/proxy-types) 结合使用时，你能更私密、更高效、更稳定地进行网络抓取。立即开始试用 Bright Data 的代理，体验它带来的优势吧！
