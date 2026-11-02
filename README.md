# 通过 HTTPX 使用代理

[![推广](https://github.com/bright-cn/Rotating-Residential-Proxies/blob/main/50%25%20off%20promo%20%281%29.png)](https://www.bright.cn/proxy-types/residential-proxies) 

本指南介绍如何在 HTTPX 中使用代理，并提供未经身份验证的代理、经过身份验证的代理、动态代理和回退代理的示例。

## 使用未经身份验证的代理

使用未经身份验证的代理时，无需用户名或密码，所有请求都会发送到 `proxy_url`。以下是使用未经身份验证代理的代码片段：

```python
import httpx

proxy_url = "http://localhost:8030"


with httpx.Client(proxy=proxy_url) as client:
    ip_info = client.get("https://geo.brdtest.com/mygeo.json")
    print(ip_info.text)
```

## 使用经过身份验证的代理

经过身份验证的代理需要用户名和密码。提交正确的凭据后，你便可以连接到代理。

使用身份验证时，`proxy_url` 如下所示：`http://<username>:<password>@<proxy_url>:<port_number>`。以下示例展示了如何同时使用 `zone` 和 `username` 构建身份验证字符串中的用户部分。它还使用 [Bright Data 的数据中心代理](https://www.bright.cn/proxy-types/datacenter-proxies)作为基础连接。

```python
import httpx

username = "your-username"
zone = "your-zone-name"
password = "your-password"

proxy_url = "http://brd-customer-{username}-zone-{zone}:{password}@brd.superproxy.io:33335"

ip_info = httpx.get("https://geo.brdtest.com/mygeo.json", proxy=proxy_url)

print(ip_info.text)
```

以下是上述代码的详解：

- 首先创建配置变量：`username`、`zone` 和 `password`。
- 我们使用这些变量创建 `proxy_url`：`"http://brd-customer-{username}-zone-{zone}:{password}@brd.superproxy.io:33335"`。
- 我们向 API 发送请求，以获取代理连接的常规信息。

响应应如下所示。

```json
{"country":"US","asn":{"asnum":20473,"org_name":"AS-VULTR"},"geo":{"city":"","region":"","region_name":"","postal_code":"","latitude":37.751,"longitude":-97.822,"tz":"America/Chicago"}}

```

## 使用动态代理

动态代理是指创建代理列表并从中随机选择代理。以下代码片段创建了一个 `countries` 列表，然后在每次请求时使用 `random.choice()` 从列表中随机选择一个国家/地区。我们的 `proxy_url` 会根据该国家/地区进行格式化。代码中的[动态代理](https://www.bright.cn/solutions/rotating-proxies)列表来自 Bright Data。

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

这段代码与经过身份验证的代理所用代码非常相似。主要区别如下：

- 我们创建了一个国家/地区数组：`["us", "gb", "au", "ca"]`。
- 我们会发出多个请求，而不是单个请求。对于每个请求，每次创建 `proxy_url` 时都会使用 `random.choice(countries)` 随机选择一个国家/地区。

## 创建回退代理连接

以上所有示例均使用数据中心代理和免费代理。前者经常会被网站屏蔽，后者则不太可靠。为确保整个流程正常运行，应将住宅代理作为回退方案。

为此，我们创建一个名为 `safe_get()` 的函数。调用该函数时，首先尝试通过数据中心连接获取 URL。如果失败，则_回退_到住宅代理连接。

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

以下是代码详解：

- 现在有两组配置变量：一组用于数据中心连接，另一组用于住宅代理连接。
- 这一次，我们使用 `AsyncClient()` 会话来引入 HTTPX 的一些高级功能。
- 首先，我们尝试使用 `datacenter_proxy` 发出请求。
- 如果未能获得正常响应，则使用 `residential_proxy` 重试请求。还需注意代码中的 `verify` 标志。使用 Bright Data 的住宅代理时，你需要下载并使用 [SSL 证书](https://docs.brightdata.com/general/account/ssl-certificate)。
- 获得有效响应后，我们将页面写入 HTML 文件。可以在浏览器中打开此页面，查看代理实际访问的内容及其返回给我们的结果。

运行上述代码后，输出和生成的 HTML 文件应如下所示。

```
trying with datacenter
response failed
trying with residential
response successful
```

![Amazon 首页截图](https://github.com/bright-cn/httpx-with-proxy/blob/main/Images/image.png)

## 总结

将 HTTPX 与 [Bright Data 的顶级代理服务](https://www.bright.cn/proxy-types)结合使用，即可通过私密、高效且可靠的方式抓取网页。立即开始免费试用 Bright Data 的代理！
