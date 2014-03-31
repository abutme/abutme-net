title: gzip分析和php实践
date: 2013-12-28 13:26:55
tags:
- gzip压缩
category: 编码
---

gzip压缩在web开发中为大家熟知，它也是极为重要的速度优化方案，这里分享一下端产品开发中遇到的问题和个人分析。实际开发中发现，不少开发人员并没有真的理解gzip压缩和它的重要性。出现过以下问题：

<!-- more -->

> 1. 服务端rd在接口中手动gzcompress做压缩输出

> 2. 服务端rd只知道有统一gzip压缩，并没有联调确认是否开启

> 3. 客户端rd上传的header不支持gzip的返回

## RFC解读

为此，我读了下gzip的[RFC 1952][1]，gzip格式的基本构成：

```
含义：+--ID1--+--ID2--+--CM--+--FLG--+-------MTIME-------+---XFL--+--OS--+

示例：|  0x1f |   8b  |  08  |   00  |     00000000      |   00   |  03  |  (more-->)

Byte: +-------+-------+------+-------+----+----+----+----+--------+------+
```
前10个字节是header必带的，FLG低五位的置位会影响header可选区。可以认为完整的GZIP组成如下：

```
FIXED(10)+OPT HDR(EXTRA+FNAME+FCOMMENT+CRC16)+ COMPRESSED BLOCKS + OPT FTR(CRC32+ISIZE，8)
```

对比这个RFC，php手册评论中提到的[gzdecode][2]`函数实现有问题`，由于FLG的可配位都为0，还没有发现bad case，后面持续关注。

## GZIP测试

`COMPRESSED BLOCKS`可以用`gzinflate`来decode，可以认为`gzdeflate`函数是不带header和footer的纯净压缩。

比较gzcompress和gzdeflate可以发现，gzcompress是在头部和尾部分别添加了2、4个字节。

测试代码：

```php
<?php

$headers[] = 'Accept-Encoding: gzip';
$ch = curl_init('http://m.baidu.com');

curl_setopt($ch, CURLOPT_BINARYTRANSFER, true);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

$gzRaw = curl_exec($ch);
$fh = fopen('gzrawfile', 'w');
fwrite($fh, $gzRaw);
fclose($fh);

$gzRawHdr = unpack('H20', substr($gzRaw, 0, 10));

$gzCompressedBody = substr($gzRaw, 10);
$gzinflateData = gzinflate($gzCompressedBody);

$gzCompressData = gzcompress($gzinflateData);
$gzCompressHdr = unpack('H8', substr($gzCompressData, 0, 4));
$gzCompressFtr = unpack('H32', substr($gzCompressData, -16));
error_log(print_r($gzCompressFtr,true),3,'/tmp/xxy');

$gzDeflateData = gzdeflate($gzinflateData);
$gzDeflateHdr = unpack('H8', substr($gzDeflateData, 0, 4));
$gzDeflateFtr = unpack('H16', substr($gzDeflateData, -8));
error_log(print_r($gzDeflateFtr,true),3,'/tmp/xxy');

```

实践得出以下结论：

> 1. curl默认是返回普通文本的，需要加header设置和参数`CURLOPT_BINARYTRANSFER`

> 2. gzencode/gzcompress/gzdeflate压缩算法是一致的，即CM=8的DEFLATE算法

> 3. gzdecode函数在php 5.4以后才支持，可以借助gzinflate实现gzdecode

## GZIP的开发关注点

gzip的必要性不多说，说一下个人的看法：

> 1. 在端产品的开发联调中，要检查Accept-Encoding

> 2. 检查web server的压缩能力，尤其是新产品上线。如果有多层web server的，开启压缩的那一层，需要加到开发、测试和联调环境中。

> 3. 没有上传header支持的用户流量，进行分析，是速度优化的一个点

一切为了流量和速度。



[1]: http://www.faqs.org/rfcs/rfc1952.html
[2]: http://cn2.php.net/manual/zh/function.gzdecode.php

