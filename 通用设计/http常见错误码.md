## 4xx
4xx错误码代表客户端错误，比如参数有问题、请求方法不支持等。

499：并非是http标准里的错误码，而是nginx层返回的。原因是nginx代理请求打到后端，后端处理是将较久，拿到响应返回给客户端时，客户端已经断开了连接。
所以这是一种在反向代码场景下的错误码。原因一般是后端处理时间较久，例如服务升级未预热等情况。

## 5xx
5xx错误码代表服务端错误，请求打到了后端实例。

502：代表upstream处理失败，也即nginx将请求打到后端，后端响应失败，一般是后端服务不可用。

## 参考
- [https://datatracker.ietf.org/doc/html/rfc7231#section-6.6.3](https://datatracker.ietf.org/doc/html/rfc7231#section-6.6.3)
- [Nginx 499 状态码](https://imajinyun.xyz/2019/11/15/nginx-499-faq/)
- [stack overflow提问](https://stackoverflow.com/questions/39636795/http-status-code-4xx-vs-5xx)