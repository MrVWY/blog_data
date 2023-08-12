---
title: "go http.transport"
date: "2022-12-10 15:00:42"
categories:
  - Go
tags:
  - Go
toc: true
math: true
type: about
---

​	理解一下golang标准库里面的net/http包中的client类型，但是其实client类型也是通过实现`RoundTripper`接口来实现一个http客户端，因此如果说不希望默认的client类型帮你预处理一遍的话，那么你可以直接定义一个http.transport去直接`获取返回来的resp`，就不必经过默认的client类型处理了。
### client客户端
```
	c := http.Client{
		Transport:     nil,
		CheckRedirect: nil,
		Jar:           nil,
		Timeout:       0,
	}
	c.Get()
```
​	通过查看源码，可以发现client.get最终也是通过实现RoundTripper来实现http的发送。具体代码流程如下

```
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: defaultTransportDialContext(&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}),
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}

func (c *Client) transport() RoundTripper {
	if c.Transport != nil {
		return c.Transport
	}
	return DefaultTransport
}

func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
	if c.Jar != nil {
...
	resp, didTimeout, err = send(req, c.transport(), deadline)
...
	return resp, nil, nil
}

// send issues an HTTP request.
// Caller should close resp.Body when done reading from it.
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {

...
resp, err = rt.RoundTrip(req)

...
}

```

​	最后的话通过定义http.transport来实现RoundTripper接口（http/transport.go），最终client就能通过DefaultTransport去调用，因此我们可以直接定义一个http.transport去直接`获取返回来的resp`，就不必经过默认的client类型处理了。


### client 类型

```
type Client struct {
    // Transport 指定执行独立、单次 HTTP 请求的机制。
    // 如果 Transport 为 nil，则使用 DefaultTransport 。
    Transport RoundTripper

    // CheckRedirect 指定处理重定向的策略。
    // 如果 CheckRedirect 不为 nil，客户端会在执行重定向之前调用本函数字段。
    // 参数 req 和 via 是将要执行的请求和已经执行的请求（切片，越新的请求越靠后）。
    // 如果 CheckRedirect 返回一个错误，本类型的 Get 方法不会发送请求 req，
    // 而是返回之前得到的最后一个回复和该错误。（包装进 url.Error 类型里）
    //
    // 如果CheckRedirect为nil，会采用默认策略：连续10此请求后停止。
    CheckRedirect func(req *Request, via []*Request) error

    // Jar 指定 cookie 管理器。
    // 如果Jar为nil，请求中不会发送 cookie ，回复中的 cookie 会被忽略。
    Jar CookieJar

    // Timeout 指定本类型的值执行请求的时间限制。
    // 该超时限制包括连接时间、重定向和读取回复主体的时间。
    // 计时器会在 Head 、 Get 、 Post 或 Do 方法返回后继续运作并在超时后中断回复主体的读取。
    //
    // Timeout 为零值表示不设置超时。
    //
    // Client 实例的 Transport 字段必须支持 CancelRequest 方法，
    // 否则 Client 会在试图用 Head 、 Get 、 Post 或 Do 方法执行请求时返回错误。
    // 本类型的 Transport 字段默认值（ DefaultTransport ）支持 CancelRequest 方法。
    Timeout time.Duration
}

```

### RoundTripper interface
```
type RoundTripper interface {
	// RoundTrip executes a single HTTP transaction, returning
	// a Response for the provided Request.
	//
	// RoundTrip should not attempt to interpret the response. In
	// particular, RoundTrip must return err == nil if it obtained
	// a response, regardless of the response's HTTP status code.
	// A non-nil err should be reserved for failure to obtain a
	// response. Similarly, RoundTrip should not attempt to
	// handle higher-level protocol details such as redirects,
	// authentication, or cookies.
	//
	// RoundTrip should not modify the request, except for
	// consuming and closing the Request's Body. RoundTrip may
	// read fields of the request in a separate goroutine. Callers
	// should not mutate or reuse the request until the Response's
	// Body has been closed.
	//
	// RoundTrip must always close the body, including on errors,
	// but depending on the implementation may do so in a separate
	// goroutine even after RoundTrip returns. This means that
	// callers wanting to reuse the body for subsequent requests
	// must arrange to wait for the Close call before doing so.
	//
	// The Request's URL and Header fields must be initialized.
	RoundTrip(*Request) (*Response, error)
}

```