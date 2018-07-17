## iOS 网络监控

## 前言

最近公司业务需要，对网络进行监控，本文主要记录下在网络流量监控实现过程中遇到的一些问题的解决方法。

现在网上都有很多网络监控的文章，本文主要是参考了

 [iOS 流量监控分析 ](http://zhoulingyu.com/2018/05/30/ios-network-traffic/)

[iOS 性能监控方案 Wedjat](https://www.jianshu.com/p/f244fb25d870)

[移动端监控体系之技术原理剖析](https://www.jianshu.com/p/8123fc17fe0e)

[Http 请求头中的 Proxy-Connection](https://imququ.com/post/the-proxy-connection-header-in-http-request.html)

[Hypertext Transfer Protocol -- HTTP/1.1](https://www.w3.org/Protocols/rfc2616/rfc2616.html)



## 流量计算

### Response 流量计算

HTTP 网络的 Response 主要由 3 部分构成，分布是 status line，header 和 body，以下主要是分别分析了这 3 个部分流量的计算方法。

#### Status-Line

通过  [iOS 流量监控分析 ](http://zhoulingyu.com/2018/05/30/ios-network-traffic/) 作者提供的方法，我们可以获取到 Response 的 Status-Line 

``` objective-c
typedef CFHTTPMessageRef (*DMURLResponseGetHTTPResponse)(CFURLRef response);

- (NSString *)statusLineFromCF {
    NSURLResponse *response = self;
    NSString *statusLine = @"";
    // 获取CFURLResponseGetHTTPResponse的函数实现
    NSString *funName = @"CFURLResponseGetHTTPResponse";
    DMURLResponseGetHTTPResponse originURLResponseGetHTTPResponse =
    dlsym(RTLD_DEFAULT, [funName UTF8String]);

    SEL theSelector = NSSelectorFromString(@"_CFURLResponse");
    if ([response respondsToSelector:theSelector] &&
        NULL != originURLResponseGetHTTPResponse) {
        // 获取NSURLResponse的_CFURLResponse
        CFTypeRef cfResponse = CFBridgingRetain([response performSelector:theSelector]);
        if (NULL != cfResponse) {
            // 将CFURLResponseRef转化为CFHTTPMessageRef
            CFHTTPMessageRef messageRef = originURLResponseGetHTTPResponse(cfResponse);
            statusLine = (__bridge_transfer NSString *)CFHTTPMessageCopyResponseStatusLine(messageRef);
            CFRelease(cfResponse);
        }
    }
    return statusLine;
}

```

这个方法获取到的 Response 的 Status-Line 是可以的，对于 Status-Line 的计算我做了一点小补充。

在这里先简单阐述下 Status-Line 在 [rfc - HTTP/1.1 ](https://www.w3.org/Protocols/rfc2616/rfc2616-sec6.html)中的定义

> The first line of a Response message is the Status-Line, consisting of the protocol version followed by a numeric status code and its associated textual phrase, with each element separated by SP characters. No CR or LF is allowed except in the final CRLF sequence.

```
       Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
```

> The Status-Code element is a 3-digit integer result code of the attempt to understand and satisfy the request. These codes are fully defined in section 10. The Reason-Phrase is intended to give a short textual description of the Status-Code. The Status-Code is intended for use by automata and the Reason-Phrase is intended for the human user. **The client is not required to examine or display the Reason- Phrase**.

 

 上面的公式中 SP 代表空格， CR 代表回车，LF代表换行，也就是说，一个完整的 Status-Line 如下所示

 ```
 Status-Line = HTTP-Version 空格 Status-Code 空格  Reason-Phrase /r/n
 ```

而上面定义的最后一句话 **The client is not required to examine or display the Reason- Phrase** 表示并没有强制要求客户端去解析或是展示 Reason- Phrase 字段，也就是说 Reason- Phrase 这个值是可以不存在的。而用 wireShark 进行网络捉包时，也的确发现了有些 Response 的 Stattus-Line 是没有带上 Reason-Phrase， w：

![带有 Reason-Phrase 的图片](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180713223028634.png)

  

![不带有 Reason-Phrase 的图片](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180713223501088.png)



通过对比，会发现虽然有些 Status-Line 中没有 Reason-Phrase，但是在 Status-Code 后面还是会带有一个空格，而在 iOS 中，通过上面方法拿到的 Status-Line，如果是不带有 Reason-Phrase 时，在字符串的最后面是没有添加一个空格的，所以对于这种情况需要做下特殊处理。另外，Status-Line 的数据长度计算，是需要加上 Status-Line 尾部携带的 CRLF。最终，Status-Line 计算的修改结果如下:

  ```objective-c
- (NSUInteger)ep_getStatusLineLengths:(NSString *)statusLine {
    NSMutableString *lineStr = @"".mutableCopy;
    [lineStr appendString: statusLine];
    NSArray *statusLineArr = [statusLine componentsSeparatedByString:@" "];
    
    // 如果 Status-Line 只包含了 HTTP-Version 和 Status-Code 时，判断下字符串的尾部是否添加了空格，如果没有，则人为添加一个空格
    if (statusLineArr.count == 2 && ![statusLine hasSuffix:@" "]) {
        [lineStr appendString:@" "];
    }
    // 判断 Status-Line 是否有 \r\n 后缀，如果没有，也人为添加上 \r\n 后缀
    if (![lineStr hasSuffix:@"\r\n"]) {
        [lineStr appendString:@"\r\n"];
    }
    // Status-Line 进行 Utf-8 编码后，获取到其长度
    NSData *lineData = [lineStr dataUsingEncoding:NSUTF8StringEncoding];
    return lineData.length;
}
  ```

#### Response Header

同样的，在计算 Response Header 之前，我们先去了解下 HTPP/1.1 中关于 [Message Headers](https://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html#sec4.5) 的定义，如下引用所示：

> Each header field consists of a name followed by a colon (":") and the field value. Field names are case-insensitive. **The field value MAY be preceded by any amount of LWS, though a single SP is preferred**. Header fields can be extended over multiple lines by preceding each extra line with at least one SP or HT. Applications ought to follow "common form", where one is known or indicated, when generating HTTP constructs, since there might exist some implementations that fail to accept anything
>
> beyond the common forms.
>
> ```
>        message-header = field-name ":" [ field-value ]
>        field-name     = token
>        field-value    = *( field-content | LWS )
>        field-content  = <the OCTETs making up the field-value
>                         and consisting of either *TEXT or combinations
>                         of token, separators, and quoted-string>
> ```

文档中提示到 **The field value MAY be preceded by any amount of LWS, though a single SP is preferred** ，说明了在每一个 filed-value 的开头，有可能有若干个 LWS，也就是所谓的空格，文档中也强调了最好只有一个空格，虽然没有强制要求，但是使用 wireShark 进行捉包时，发现基本上 HTTP/1.1 请求的 field-value 的前面都带有一个空格。如下图所示：

![field-value首部带有一个空格](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180714101216815.png)

另外，在文档的 Message Types 章节中，也对多个 message-header 的格式进行了定义：

> Both types of message consist of a start-line, zero or more header fields (also known as "headers"), an empty line (i.e., a line with nothing preceding the CRLF) indicating the end of the header fields, and possibly a message-body.
>
> ```
>         generic-message = start-line
>                           *(message-header CRLF)
>                           CRLF
>                           [ message-body ]
>         start-line      = Request-Line | Status-Line
> ```

每一个 message-header 的末尾都会跟着一个 CRLF，而且会有一行空的 CRLF 来用于标识 header fileds 的结束；根据 message-headers 的格式定义，假设我们有以下的 headers 内容

```json
{
    "Connection" = "keep-alive";
    "Content-Type" = "application/json;charset=UTF-8";
    "Date" = "Sat, 14 Jul 2018 02:31:00 GMT";
    "Server" = "openresty/1.13.6.1";
    "Transfer-Encoding" = "Identity";
}
```

在 HTTP/1.1 的header field 中的格式应该是如下所示：

```
Connection: keep-alive\r\nConnect-Type: application/json;charset=UTF-8\r\nData: Sat, 14 Jul 2018 02:31:00 GMT\r\nServer: openresty/1.13.6.1\r\nTransfer-Encoding: Identity\r\n\r\n
```

因此，根据定义最终生成的 message headers 的计算方法如下:

```objective-c
- (NSUInteger)ep_getHeadersLength:(NSDictionary *)headers {
    NSUInteger headersLength = 0;
    NSDictionary<NSString *, NSString *> *headerFields = headers;
    NSString *headerStr = @"";
    for (NSString *key in headerFields.allKeys) {
        headerStr = [headerStr stringByAppendingString:key];
        headerStr = [headerStr stringByAppendingString:@": "];
        if ([headerFields objectForKey:key]) {
            headerStr = [headerStr stringByAppendingString:headerFields[key]];
        }
        headerStr = [headerStr stringByAppendingString:@"\r\n"];
    }
    headerStr = [headerStr stringByAppendingString:@"\r\n"];
    NSData *headerData = [headerStr dataUsingEncoding:NSUTF8StringEncoding];
    headersLength = headerData.length;
    return headersLength;
}
```

但是，当你高兴的使用上面的方法去计算 Response Headers 的长度时，你会发现无论你怎么计算，获取到的 headers 的长度都会比你在 wireShark 中看到不一致。如下图所示：

![image-20180714104643087](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180714104643087.png)

点击 wireShark 的每一行 message header 在红色圆圈部分可以看到这个 message header 的长度，将所有 message header 的长度信息加起来（记得将最后一行的 \r\n 的长度也添加进去），计算后得到的长度为（28 + 37 + 46 + 28 + 30 + 2）171，而自己在程序你们计算出来的长度是 172，如下图所示：

![image-20180714105220771](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180714105220771.png)

通过对比程序获取到的 Response headers 的数据内容和 wireShark 获取到的 headers 内容，可以发现实际上，这2个 headers 的内容是不一致的:

![image-20180714105959120](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180714105959120.png)

![image-20180714110405904](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180714110405904.png)

对比可以发现，在 wireShark 中 Transfer-Encoding 的值为 chunked，但是在我们的应用程序中获取到的却是 Identity，为什么会被修改掉笔者找了很久，暂时没有找到比较有效的数据证明，但是猜测应该是苹果在 CFNetwork 层帮我们对收到的数据进行了解码，所以对于 CFNetwork 的上一层来说，body 中的数据其实已经不是 chunked 编码。

虽然不知道具体原因，但是我们可以根据 HTTP/1.1 的协议定义，自己来判断 Transfer-Encoding 的值，在 [stackoverflow](https://stackoverflow.com/questions/12921300/cfnetwork-reads-http-header-transfer-encoding-identity-but-wireshark-shows-chunk) 已经有人给出了答案。

根据 [rfc2616-sec3](https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.5) 上面的定义：

> Whenever a transfer-coding is applied to a message-body, the set of transfer-codings MUST include "chunked", unless the message is terminated by closing the connection. When the "chunked" transfer- coding is used, it MUST be the last transfer-coding applied to the message-body. The "chunked" transfer-coding MUST NOT be applied more than once to a message-body. These rules allow the recipient to determine the transfer-length of the message

在 [rfc7230](https://tools.ietf.org/html/rfc7230#section-3.3.1) 也有类似的描述

>If any transfer coding other than chunked is applied to a response payload body, the sender MUST either apply chunked as the final transfer coding or terminate the message by closing the connection.
>
>```
>For example,
>
>     Transfer-Encoding: gzip, chunked
>```
>
>indicates that the payload body has been compressed using the gzip coding and then chunked using the chunked coding while forming the message body.

对 message-body 采用任意的传输编码类型，这些编码类型中都必须包括 chunked 传输编码方式，除非连接被关闭(连接被关闭可以理解成 headers 中的 "connect" 或 "proxy-connect" 的值为 close)，而且 message-body 最外面的一层编码必须是 chunked 编码。

在  [rfc2616-sec4](https://www.w3.org/Protocols/rfc2616/rfc2616-sec4.html#sec4.3) 中有一段关于 Message Length 长度计算的定义：

> 2.If a Transfer-Encoding header field (section [14.41](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.41)) is present and has any value other than "identity", then the transfer-length is defined by use of the "chunked" transfer-coding (section [3.6](https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.6)), unless the message is terminated by closing the connection.
>
> 3.If a Content-Length header field (section [14.13](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.13)) is present, its decimal value in OCTETs represents both the entity-length and the transfer-length. The Content-Length header field MUST NOT be sent if these two lengths are different(i.e., if a Transfer-Encoding header field is present).
>
> ```
>      If a message is received with both a
>      Transfer-Encoding header field and a Content-Length header field,
>      the latter MUST be ignored.
> ```

如果 header filed 中存在 Transfer-Encoding 并且其值不是 “identity” ，那么，传输数据的长度是由 “chunked” 编码决定的。

如果 header field 中存在 Content-Length，这个8位字节的十进制数值代表了实体长度和传输数据长度，也就是说，此时实体长度是等于传输数据长度，而当实体长度和传输数据长度2个值不一致时，Content-Length 可以不用传输。（例如：Transfer-Encoding header field 存在时，即使 Content-Length 存在，也会被忽略掉）

所以综合上面的所有定义，可以得出以下结论：

1. 只要是 Transfer-Encoding header field 存在时，那么 Content-Length 这个值就失去了意义，因为传输数据最终一定会进行 chunked 编码。
2. 只有当 Transfer-Encoding header field 不存在，并且 Content-Length header field 存在时，此时证明实体长度和传输数据长度是相同的。说明此时并没有使用 chunked 编码，应为采用 chunked 会导致实体长度和传输数据长度不对等。
3. 如果连接被关闭掉了，那么数据传输也会别终止掉了，所以此时也就无法通过 chunked 来计算传输长度。因为使用 chunked 编码方式传输的前提条件就是必须使用 HTTP 常连接。
4. 如果 Transfer-Encoding header field 存在，HTTP 是常连接，并且值 identity，那么在网络上进行传输的 Transfer-Encoding header field 的值一定是 chunked 
5. 如果 Transfer-Encoding header field 存在，HTTP 是常连接，并且值不是 identity（例如: gzip）,那么在网络上 Transfer-Encoding 的值最后面一定会带有 chunked （这部分是我通过文档推导过来的，暂时没有在捉包的时候看到 Transfer-Encoding 的值不是 chunked 的情况，如果苹果在 CFNetwork 层时，无论 Transfer-Encoding 是什么值，都会帮我们进行处理，然后将传给上层的 Transfer-Encoding 的值设置为 Identity，上层便那么无法判断是否除了采用 chunked 外，还用到了别的传输编码）

所以，最终的结论和 [stackoverflow](https://stackoverflow.com/questions/12921300/cfnetwork-reads-http-header-transfer-encoding-identity-but-wireshark-shows-chunk)  给出的判断条件不是很一致，当 Response Header 中 Transfer-Encoding header field 存在，并且 HTTP 连接是常连接时，那么，在网络中一定是采用了 chunked 编码传输。（判断的是要忽略大小写）

```objective-c
  BOOL headerKeysContainsTransferEncodingHeaderField = [self stringArray:[headers allKeys] containsStringCaseInsensitive:@"Transfer-Encoding"];
        BOOL headerValuseContainsKeepAliveString = [self stringArray:[headers allValues] containsStringCaseInsensitive:@"Keep-alive"] ;
        NSString *tranferEncodingKey = [self getStringFromArray:[headers allKeys] caseInsensitiveString:@"Transfer-Encoding"];
        BOOL headerTransferEncodingValueIsIdentify = [((NSString *)headers[tranferEncodingKey]).lowercaseString isEqualToString:@"identity"];
        if (headerKeysContainsTransferEncodingHeaderField &&
            headerValuseContainsKeepAliveString ) {
            if (headerTransferEncodingValueIsIdentify) {
                headers[tranferEncodingKey] = @"chunked";
            }else {
                NSString *transferEncodingvalue = headers[[self getStringFromArray:[headers allKeys] caseInsensitiveString:@"Transfer-Encoding"]];
                NSString *newTransferEncodingvalue = [transferEncodingvalue stringByAppendingString:@", chunked"];
                headers[tranferEncodingKey] = newTransferEncodingvalue;
            }
        }   

- (BOOL)stringArray:(NSArray <NSString *>*)stringArray containsStringCaseInsensitive:(NSString *)string {
    for (NSString *value in stringArray) {
        if ([value.lowercaseString isEqualToString:string.lowercaseString]) {
            return YES;
        }
    }
    return NO;
}

- (NSString *)getStringFromArray:(NSArray <NSString *>*)stringArray caseInsensitiveString:(NSString *)string {
    for (NSString *value in stringArray) {
        if ([value.lowercaseString isEqualToString:string.lowercaseString]) {
            return value;
        }
    }
    return NULL;
}
```

在学习的过程中，看到了 header filed 中的 Proxy-Connection 和 Connection，具体可以看这篇文章[Http 请求头中的 Proxy-Connection](https://imququ.com/post/the-proxy-connection-header-in-http-request.html) ，挺有趣的。

#### Response Body

关于 Response Body 的获取，网上有很多资料都提及到了 content-Length 不准确的问题，而且也提及到了需要考虑到 Content-Encoding 值对数据大小的影响。因为从 `- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data` 获取到的回调数据 data 是已经经过了 CFNetwork 层的解压，所以按照 data 的长度计算获取到的值都会比真实的值要大一点。所以需要在应用层自己模拟一次 zip 的压缩，可以获取到更加接近真实传输的数据长度。

除了 Content-Encoding 对流量的大小计算有影响，Transfer-Encoding 对流量的计算其实也是有一定的影响的。先简单了解下 Transfer-Encoding 和 Content-Encoding 直接的区别。摘抄至 [rfc2616-sec3](https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.5) :

> Content coding values indicate an encoding transformation that has been or can be applied to an entity.
>
> Transfer-coding values are used to indicate an encoding transformation that has been, can be, or may need to be applied to an entity-body in order to ensure "safe transport" through the network.This differs from a content coding in that the transfer-coding is a property of the message, not of the original entity

Content coding 是针对实体的编码，在整个网络传输过程中，这个值一般是不会变的。Transfer-coding 是作用在两个节点直接的数据传输编码，当数据从服务器发送到客户端时，可能经过了很多个节点，不同节点之间是可以采用不同的  Transfer-coding。例如：假设现在收到一个 HTTP 回复，读取头部信息时，获取到的 content-coding 是 zip，transfer-coding 是 chunked，那么，接受者需要先将收到的数据进行 chunked 解码，然后再进行 zip 解码，才能拿到服务器真正传输的数据。

在 HTTP/1.1 协议定义中，只要连接时常连接，那么就一定会使用 Transfer-coding 中一定会有 chunked 编码，而且传输数据的最外的一层编码方式一定是 chunked 编码。所以，我们在计算 response body 的数据长度是，还需要考虑下 Transfer-coding 对数据的影响。

#### Chunked Transfer Coding

chunked 编码主要是将消息的body转换一组 chunk，每一个 chunk 拥有一个自己大小的标志，在 chunk 最后有可能会有一个包含实体头部字段的尾部。允许动态生成内容跟随必要的信息传递给接受者，接受者可以它来验证是否已经接受到完整的消息。chunked-Body 的格式如下：

```
  Chunked-Body   = *chunk
                        last-chunk
                        trailer
                        CRLF
       chunk          = chunk-size [ chunk-extension ] CRLF
                        chunk-data CRLF
       chunk-size     = 1*HEX
       last-chunk     = 1*("0") [ chunk-extension ] CRLF
       chunk-extension= *( ";" chunk-ext-name [ "=" chunk-ext-val ] )
       chunk-ext-name = token
       chunk-ext-val  = token | quoted-string
       chunk-data     = chunk-size(OCTET)
       trailer        = *(entity-header CRLF)
```

chunk-size 是一个 16进制的字符串，表明了 chunk 的大小。在 chunked 编码的最后，会有一个 chunk-size 值为0 的字符串，在该字符串后面还有一行空行。（感觉理解成一个空的 chunk 就可以了）

trailer 允许用户在消息的尾部添加一些额外的 header 字段。

服务器使用 chunked transfer-coding 进行回复时，除了下面列出的情况，不允许使用使用 trailder：

1. 请求中包含了的 TE header field 明确表明，允许在回复的 transfer-codding 使用 trailers。
2. 服务器是响应的源服务器，拖车字段完全由可选的元数据组成，接收方可以使用该消息(以原始服务器可以接受的方式)，而不接收此元数据。换句话说，源服务器愿意接受这样一种可能性，即拖车字段可能沿着通向客户端的路径被静静地丢弃。

一般来说，客户端收到的回复都是不会有 trailer，所以可以不用考虑 trailer 对流量计算的影响。

根据定义，我们采用 wireShare 捉一个 HTTP 的网络包，了解下实际情况和理论是否一致：

![image-20180714171220923](/var/folders/k9/83_xv8dj5kn9p4bc7wtfpq_m0000gn/T/abnerworks.Typora/image-20180714171220923.png)

可惜的时，在 iOS 中，暂时没有找到获取 chunk 个数 和 chunk 大小的方法，只能简单的将 `-(void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data ` 的一次回调，就当做是收到了一次 chunk（这其实是很不准确的，后面会后续会尝试下 hook CFNetwork 的一些方法，看能不能监听到真实的 chunk 信息）。将每次获取到的 data 的大小记录一些，然后在方向根据协议的定义模拟整个编码过程，计算出编码后数据的长度信息。具体代码如下：

```objective-c
- (NSUInteger)ep_bodyLength:(NSData *)body headers:(NSDictionary *)headers chunkedSizes:(NSArray *)chunkedSizes {
    NSString *transferEncodingKey = [EPUtils getStringFromArray:headers.allKeys caseInsensitiveString:@"Transfer-Encoding"];
    NSString *contentEncodingKey = [EPUtils getStringFromArray:headers.allKeys caseInsensitiveString:@"Content-Encoding"];
    NSString *tranferEncoding = [headers[transferEncodingKey] lowercaseString];
    NSString *contentEncoding = [headers[contentEncodingKey] lowercaseString];
    NSArray *transferEncodingArr = [tranferEncoding componentsSeparatedByString:@", "];
    NSUInteger length = body.length;
    NSData *bodyData = body;
    if ([contentEncoding isEqualToString:@"gzip"]) {
        // 需要模拟 gzip 编码，获取编码后的数据长度
    }
    if (transferEncodingArr.count > 1) {
        for (NSString *transferEncoding in transferEncodingArr) {
            // 按照顺便对数据进行编码
            if ([contentEncoding isEqualToString:@"gzip"]) {
                // 需要模拟 gzip 编码，获取编码后的数据长度
            }else if ([transferEncoding isEqualToString:@"chunked"]) {
                // 长度值独占一行，不包括它结尾的CRLF
                NSUInteger CRLFSize = [@"/r/n" dataUsingEncoding:NSUTF8StringEncoding].length;
                NSUInteger totalChunkedheaderAndFooterSize = 0;
                for (NSString *chunkedSizeHex in chunkedSizes) {
                    NSUInteger chunkedHeaderSize = [chunkedSizeHex dataUsingEncoding:NSUTF8StringEncoding].length + CRLFSize;
                    totalChunkedheaderAndFooterSize += chunkedHeaderSize;
                    totalChunkedheaderAndFooterSize += CRLFSize;
                }
                // 最最有一个 chunked Size 大小为 0 的 chunked
                NSUInteger endChunkedHeaderSize = [@"0" dataUsingEncoding:NSUTF8StringEncoding].length + CRLFSize;
                NSUInteger endChunkedFooterSize = CRLFSize;
                totalChunkedheaderAndFooterSize += endChunkedHeaderSize;
                totalChunkedheaderAndFooterSize += endChunkedFooterSize;
                length = length + totalChunkedheaderAndFooterSize;
            }
        }
    }

    return length;
}
```

这里只是模拟了整个 content-encoding 和 transfer-encoding 的编码过程，最后的结果只是能够更加贴近真实传输数据的长度，但是其实还是有很多误差的。这里虽然在 Content-Encoding 和 TransferEncoding 都进行了 gzip 的编码判断，但是在实际情况下，如果实体已经用 gzip 压缩过了，transfer-encoding 一般时不会进行 gzip 的压缩的，一般只会采用 chunked 编码。至于数据压缩过程，可以放到子线程去进行，子线程完成数据压缩后在存入数据库就可以了。

### Request 流量计算

请求流量可以直接参考   [iOS 流量监控分析 ](http://zhoulingyu.com/2018/05/30/ios-network-traffic/) 中计算。下面主要是记录下计算流量过程中的几个知识点。

#### Request-Line

由于缺乏相关的接口，所以 reuqest 的 line 部分只能用一个经验值来计算，一般来说，我们都会以下面代码进行计算：

```objective-c
- (NSUInteger)ep_getLineLength {
    NSString *lineStr = [NSString stringWithFormat:@"%@ %@ %@\r\n", self.HTTPMethod, self.URL.path, @"HTTP/1.1"];
    NSData *lineData = [lineStr dataUsingEncoding:NSUTF8StringEncoding];
    return lineData.length;
}
```

这个计算方法其实已经和我们平时看到的 line 的格式很一致，但是在某些情况下，还是有一些的误差的，下面主要记录下误差的原因。

在 [rfc2616-sec5](https://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html) 定义了 Request-Line 的格式和内容：

```
        Request-Line   = Method SP Request-URI SP HTTP-Version CRLF
```

其中，Method 和 HTTP-Version 的计算其实都没有误差，HTTP-Version 虽然我们写的是 HTTP/1.1，但是由于 HTTP/1.0 和 HTTP/2.0 进行 UTF-8 编码出来长度是一致的，所以没有误差。所以误差的主要来源在于 Request-URI。

在  [rfc2616-sec5](https://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html) 中，对于 Request-URI 的定义如下：

```
       Request-URI    = "*" | absoluteURI | abs_path | authority
```

由此可以，上面代码中的 self.URL.path 只是 Request-URI 4 种选项中的第3种，所以当 Request-URI 的值是其他3个选项，而我们以第3个选项进行计算时，误差就产生了。

`*`： 表示请求不应用于特定的资源，而是应用于服务器本身，并且只在 method 不一定应用于资源时才被允许。

`absoluteURI` 当请求发给代理时，需要完整的请求地址。

`abs_path` 普遍情况下都是采用相对地址。

因为 `abs_path` 是最为普遍的情况，所以一般来说，直接用这种方式进行计算就可以了。

#### Header

通过 Cocoa 层创建的 request 中，header 的字段缺少几个，包括但不限于：

```
1. Accept
2. Connection / Proxy-Connection
3. Host
```

如果对于流量有比较高的要求，可以自己补上一些经验值。

就 iOS 客户端自己发送的请求， Accept 大部分情况下都是  `*/*` ， 如果没有设置代理，虽然 HTTP/1.1 默认常连接，但是一般在 header 里面还是会设置下 `Connection: keep-alive `，如果设置了代码，那么应该是` Proxy-Connection: keep-alive`。Host 就用 request 中的 Host 就可以了。

#### Cookier

request 中时没有 cookier 信息的，所以自己手动获取下，然后添加到 header 中进行计算就可以了。

```objective-c
- (NSDictionary<NSString *, NSString *> *)dgm_getCookiesByUrl:(NSURL *)url {
    NSDictionary<NSString *, NSString *> *cookiesHeader;
    NSHTTPCookieStorage *cookieStorage = [NSHTTPCookieStorage sharedHTTPCookieStorage];
    NSArray<NSHTTPCookie *> *cookies = [cookieStorage cookiesForURL:url];
    if (cookies.count) {
        cookiesHeader = [NSHTTPCookie requestHeaderFieldsWithCookies:cookies];
    }
    return cookiesHeader;
}
```

#### Body

由于我是使用 NSURLSession 进行网络请求，而且 request 也不会像 response 一样，回去 body 的传输进行编码，所以直接用 NSURLSessionDelegate 的回调就可以获取到精确值了

```objective-c
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend;
```



## 时间统计 

在进行网络请求时，除了流量外，网络请求的相关时间统计也是很重要的一部分。需要统计的时间包括：

1. DNS 解析时间
2. TCP 连接建立时间
3. SSL 认证时间
4. 请求开始时间
5. 收到回复时间

在 iOS 10 即以上的版本，可以直接用 NSURLSession 的回调，具体资料网上一大堆，就不讲了。

```objc
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didFinishCollectingMetrics:(NSURLSessionTaskMetrics *)metrics 
```

如果要求是 iOS 9 的这几个时间的监控，在网上找到的所有方法都是使用 fishhook 去 hook BSDSocket 的connect 方法，但是就本人测试，就没有成功过，在 fishhook 中也找到了相关的解释 [How to hook socket or connect ](https://github.com/facebook/fishhook/issues/40)

另外在 [iOS 性能监控方案 Wedjat](https://www.jianshu.com/p/f244fb25d870) 中提到了去 hook CFNetwork 的 CFReadStreamCreateForHTTPRequest 的方法，返回一个自己的 Proxy 的方式，我尝试了很久，也一直没有成功。通过使用 fishhook 我确定是 hook 到了相关的方法了，但是在使用 NSURLSession 进行网络请求时，并不会调用被我 hook 的方法。当如果是自己去调用  CFReadStreamCreateForHTTPRequest 方法，是会调用 hook 的方法。原因应该是和 fishhook 没办法 hook connect 方法是一致的。

所以在时间统计方法，iOS 9 的手机目前只能获取到 应用层请求开始时间，应用层收到回复时间