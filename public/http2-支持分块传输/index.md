# HTTP2 下的 Transfer-Encoding: chunked

在 <span style="color:#FF7F50"> HTTP </span> 中传输数据有一个 <span style="color:#FF7F50">chunked</span> 的方式, 又称“分块传输”。在响应报文里用头字段<span style="color:#FF7F50">Transfer-Encoding: chunked</span> 来表示。意思是报文里的 body 部分不是一次性发过来的，而是分成了许多的块（<span style="color:#FF7F50">chunk</span>）逐个发送。而 <span style="color:#FF7F50">HTTP2.0</span> 协议作为 <span style="color:#FF7F50">HTTP</span>协议的升级，自然是对<span style="color:#FF7F50">chunked</span>模式做支持？不然！

**HTTP2 是没有 chunked 的！**

<!--more-->


分块传输也可以用于“流式数据”，例如由数据库动态生成的表单页面，这种情况下 <span style="color:#FF7F50">body</span> 数据的长度是未知的，无法在头字段“<span style="color:#FF7F50">Content-Length</span>”里给出确切的长度，所以也只能用 <span style="color:#FF7F50">chunked</span> 方式分块发送。

# chunked 的编码规则

- 每个分块包含两个部分，长度头和数据块；
- 长度头是以 CRLF（回车换行，即\r\n）结尾的一行明文，用 16 进制数字表示长度；
- 数据块紧跟在长度头后，最后也用 CRLF 结尾，但数据不包含 CRLF；
- 最后用一个长度为 0 的块表示结束，即“0\r\n\r\n”

<img src="https://img1.kiosk007.top/static/images/network/HTTP2/http2_chunked.webp">


# HTTP2 下的分块传输
先说结论，<span style="color:#FF7F50">HTTP2</span> 是不支持 <span style="color:#FF7F50">HTTP1</span> 语义下的 <span style="color:#FF7F50">chunked</span> 模式的。因为H2的 <span style="color:#FF7F50">Data</span> 帧是纯天然的Chunked模式。



这也是最容易出bug的地方，一些实现不完全的HTTP2开源库经常在这里出问题。因为我们线上都是客户端请求基本都是 <span style="color:#FF7F50">chunked </span>模式，升级到 HTTP2 之后，经常访问一些三方链接访问卡死，最终 <span style="color:#FF7F50">debug</span> 后的原因发现出在服务端对 <span style="color:#FF7F50">HTTP2 chunked </span> 的支持上。


最常见的反向代理实现 <span style="color:#FF7F50">Nginx</span> 就是最容易有这种 <span style="color:#FF7F50">bug</span>的，很多企业维护的<span style="color:#FF7F50">Nginx</span>经常不更新，而低版本的<span style="color:#FF7F50">Nginx</span>在<span style="color:#FF7F50">HTTP2</span>上的这个<span style="color:#FF7F50">bug</span>就被我们遇到过。



具体现象是，客户端使用<span style="color:#FF7F50">chunked</span>模式上传，但是服务端开启了 <span style="color:#FF7F50">HTTP2</span>，自然客户端也就升级到 <span style="color:#FF7F50">HTTP2</span> 。但是请求总是卡主，服务端无响应。最终定位到如果 <span style="color:#FF7F50">DATA</span>帧不携带内容，只携带一个 <span style="color:#FF7F50">End Stream</span> 标志，服务端无法识别流式传输结束。我拿 `www.bing.com` 做了对比，其他网站都是正常的。


<img src="https://img1.kiosk007.top/static/images/network/HTTP2/http2_chunked_bug.png">


其实 <a href="https://datatracker.ietf.org/doc/html/rfc7540#section-8.1" style="color:#556B2F"> RFC 7540 </a> 早有规定，HTTP2 的传输是不支持 <span style="color:#FF7F50">Transfer-Encoding: chunked</span> 的。

<img src="https://img1.kiosk007.top/static/images/network/HTTP2/http2_chunked_rfc.png">

一般的网络库(如 <span style="color:#FF7F50">Cronet</span>、<span style="color:#FF7F50">OKHTTP</span> )底层都给我们做了兼容，如果上层使用 <span style="color:#FF7F50"> chunked </span> 模式传输，而实际使用的是 HTTP2 ,网络库会帮我们自动隐藏掉 <span style="color:#FF7F50">Transfer-Encoding: chunked</span> 这个 <span style="color:#FF7F50"> Header </span> 。


# 使用 HTTP2 发送 chunked
这里我们以 Golang 的 HTTP2 官方的客户端库做一个测试.

``` golang
func main() {
	rd, wr := io.Pipe()
	u, _ := url.Parse("https://httpbin.org/post")


	req := &http.Request{
		Method:           "POST",
		ProtoMajor:       1,
		ProtoMinor:       1,
		URL:              u,
		TransferEncoding: []string{"chunked"},
		Body:             rd,
		Header:           make(map[string][]string),
	}
	req.Header.Set("Content-Type", "application/x-protobuf")
	var client *http.Client
	var transport *http2.Transport

	sslkeylogfile, err := os.OpenFile("/tmp/sslkey.log", os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0666)
	if err != nil {
		panic(err)
	}
	defer sslkeylogfile.Close()

	transport = &http2.Transport{}
	var config *tls.Config = &tls.Config{
		InsecureSkipVerify: true,
		KeyLogWriter:       sslkeylogfile,
	}
	transport.TLSClientConfig = config
	client = &http.Client{Transport: transport}

	go func() {
		buf := make([]byte, 8000)
		str := ""
		for i := 0; i < 100000; i++ {
			str += "0"
		}

		f := strings.NewReader(str)
		for {
			n, _ := f.Read(buf)
			if 0 == n {
				break
			}
			wr.Write(buf)
		}
		wr.Close()

	}()

	resp, err := client.Do(req)
	if nil != err {
		fmt.Println("error =>", err.Error())
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if nil != err {
		fmt.Println("error =>", err.Error())
	} else {
		fmt.Println(string(body))
	}
}

```
通过导入 <span style="color:#FF7F50">SSLKEYLOG</span> 从 <span style="color:#FF7F50">Wireshark</span> 抓包可以看到，每个 <span style="color:#FF7F50">Data Frame</span> 大小为 8000 。


<img src="https://img1.kiosk007.top/static/images/network/HTTP2/http2_chunked_wireshark.png">

