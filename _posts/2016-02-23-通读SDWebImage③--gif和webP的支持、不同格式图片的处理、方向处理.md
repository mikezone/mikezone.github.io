---
layout: cnblog_post
title:  "通读SDWebImage③--gif和webP的支持、不同格式图片的处理、方向处理"
date:   2016-02-23 13:50:39
categories: iOS
---
<div><a name="labelTop"></a></div>
<!--Category-->
<div id="navCategory">
	<b>本文目录</b>
	<ul>
		<li><a href="#anchor1_0">NSData+ImageContentType: 根据NSData获取MIME</a></li>
        <li><a href="#anchor2_0">UIImage+GIF</a></li>        	
        <li><a href="#anchor3_0">UIImage+WebP</a></li>
        <li><a href="#anchor4_0">UIImage+MultiFormat:根据NSData相应的MIME将NSData转为UIImage</a></li>
        <li><a href="#anchor5_0">方向处理</a></li>
	</ul>
</div>
<!--Category结束-->
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor1_0">回到顶部</a></div>

### NSData+ImageContentType: 根据NSData获取MIME
正如标题`NSData+ImageContentType`的唯一方法`+ (NSString *)sd_contentTypeForImageData:(NSData *)data;`就是根据图片的二进制数据返回其对应的MIME类型。它的具体实现如下：

```objectivec
+ (NSString *)sd_contentTypeForImageData:(NSData *)data {
    uint8_t c;
    [data getBytes:&c length:1];
    switch (c) {
        case 0xFF:
            return @"image/jpeg";
        case 0x89:
            return @"image/png";
        case 0x47:
            return @"image/gif";
        case 0x49:
        case 0x4D:
            return @"image/tiff";
        case 0x52:
            if ([data length] < 12) {
                return nil;
            }

            NSString *testString = [[NSString alloc] initWithData:[data subdataWithRange:NSMakeRange(0, 12)] encoding:NSASCIIStringEncoding];
            if ([testString hasPrefix:@"RIFF"] && [testString hasSuffix:@"WEBP"]) {
                return @"image/webp";
            }

            return nil;
    }
    return nil;
}
```
实际上每个文件的前几个字节都标识着文件的类型，对于一般的图片文件，通过第一个字节(WebP需要12字节)可以辨识出文件类型。

这个方法的实现思路是这样的：<br/>
1.取data的第一个字节的数据，辨识出JPG/JPEG、PNG、GIF、TIFF这几种图片格式，返回其对应的MIME类型。<br/>
2.如果第一个字节是数据为`0x52`，需要进一步检测，因为以`0x52`为文件头的文件也可能会是rar等类型(可以在<a href="http://baike.baidu.com/link?url=_PP3WE8Xx_j8lEFuiO-MfBmI-TskdkeF1ZQE6CUUUGtQbPE6RB1SGTal7-oUZJVWK0n9AkpnQpQ2E4YRScA1q_" target='blank'>文件头</a>查看)，而webp的前12字节有着固定的数据：<br/>
<img src="http://7vim0m.com1.z0.glb.clouddn.com/img%2FKVZ%7D50%60FID%5BX1%5DJ%5D%5D4J%5DRRU.jpg" width="400" alt="WebP文件前12字节数据"/>
因此前12字节数据有前缀`RIFF`和后缀`WEBP`的就是WebP格式。
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor2_0">回到顶部</a></div>

### UIImage+GIF
在介绍这个分类之前，我们要弄清一个问题，iOS展示gif图的原理：<br/>
1.将gif图的每一帧导出为一个UIImage，将所有导出的UIImage放置到一个数组<br/>
2.用上面的数组作为构造参数，使用animatedImage开头的方法创建UIImage，此时创建的UIImage的images属性值就是刚才的数组，duration值是它的一次播放时长。<br/>
3.将UIImageView的image设置为上面的UIImage时，gif图会自动显示出来。(也就是说关键是那个数组，用尺寸相同的图片创建UIImage组成数组也是可以的)

这个分类下有三个方法:

```objectivec
// 指定在main bundle中gif的文件名，读取文件的二进制，然后调用下面的方法
+ (UIImage *)sd_animatedGIFNamed:(NSString *)name; 

// 将gif文件的二进制转为animatedImage
+ (UIImage *)sd_animatedGIFWithData:(NSData *)data; 

// 将self.images数组中的图片按照指定的尺寸缩放，返回一个animatedImage，一次播放的时间是self.duration
- (UIImage *)sd_animatedImageByScalingAndCroppingToSize:(CGSize)size; 
```
可以说共有两个功能：<br/>
1.`+sd_animatedGIFNamed:`和`+ sd_animatedGIFWithData:`将文件(二进制)转为animatedImage。<br/>
2.`-sd_animatedImageByScalingAndCroppingToSize:`负责gif图的缩放。

方法`+ sd_animatedGIFWithData:`的实现细节是这样的：

```objectivec
+ (UIImage *)sd_animatedGIFWithData:(NSData *)data {
    if (!data) {
        return nil;
    }
    CGImageSourceRef source = CGImageSourceCreateWithData((__bridge CFDataRef)data, NULL);
    // 获取图片数量(如果传入的是gif图的二进制，那么获取的是图片帧数)
    size_t count = CGImageSourceGetCount(source);
    UIImage *animatedImage;
    if (count <= 1) {
        animatedImage = [[UIImage alloc] initWithData:data];
    }
    else {
        NSMutableArray *images = [NSMutableArray array];
        NSTimeInterval duration = 0.0f;
        // 遍历每张图，通过`sd_frameDurationAtIndex:source:`获取每张图需要的播放时间，用duration累加，将图到出为UIImage，依次放到数组imges中
        for (size_t i = 0; i < count; i++) {
            CGImageRef image = CGImageSourceCreateImageAtIndex(source, i, NULL);
            if (!image) {
                continue;
            }
            duration += [self sd_frameDurationAtIndex:i source:source];
            [images addObject:[UIImage imageWithCGImage:image scale:[UIScreen mainScreen].scale orientation:UIImageOrientationUp]];
            CGImageRelease(image);
        }
        // 如果上面的计算播放时间方法没有成功，就按照下面方法计算
        // 计算一次播放的总时间：每张图播放1/10秒 * 图片总数
        if (!duration) {
            duration = (1.0f / 10.0f) * count;
        }        
        animatedImage = [UIImage animatedImageWithImages:images duration:duration];
    }
    CFRelease(source);
    return animatedImage;
}
```
计算每帧需要播放的时间的方法实现为：

```objectivec
+ (float)sd_frameDurationAtIndex:(NSUInteger)index source:(CGImageSourceRef)source {
    float frameDuration = 0.1f;
    // 获取这一帧的属性字典
    CFDictionaryRef cfFrameProperties = CGImageSourceCopyPropertiesAtIndex(source, index, nil);
    NSDictionary *frameProperties = (__bridge NSDictionary *)cfFrameProperties;
    NSDictionary *gifProperties = frameProperties[(NSString *)kCGImagePropertyGIFDictionary];
    // 从字典中获取这一帧持续的时间
    NSNumber *delayTimeUnclampedProp = gifProperties[(NSString *)kCGImagePropertyGIFUnclampedDelayTime];
    if (delayTimeUnclampedProp) {
        frameDuration = [delayTimeUnclampedProp floatValue];
    } else {
        NSNumber *delayTimeProp = gifProperties[(NSString *)kCGImagePropertyGIFDelayTime];
        if (delayTimeProp) {
            frameDuration = [delayTimeProp floatValue];
        }
    }    
    // 许多烦人的广告指定duration为0来让图像尽可能快地闪过。
    // 我们遵循Firefox的做法：对于指定duration小于<= 10 ms的帧设置duration值为100ms
    // 详见<rdar://problem/7689300>和<http://webkit.org/b/36082>
    if (frameDuration < 0.011f) {
        frameDuration = 0.100f;
    }
    CFRelease(cfFrameProperties);
    return frameDuration;
}
```
`+ (UIImage *)sd_animatedGIFNamed:(NSString *)name`方法的实现比较简单，对retina的屏幕做了一点适配，只需将文件的name传入即可，不需传入文件后面的`@"2x"`或者`.gif`文件后缀。这个方法内部会根据当前屏幕的scale决定时候添加`@"2x"`,然后添加文件后缀，在mainBundle中找到这个文件读取出二进制然后调用方法`+ (UIImage *)sd_animatedGIFWithData:(NSData *)data`。

对gif图进行缩放的方法`- sd_animatedImageByScalingAndCroppingToSize:`的实现思路为：<br/>
1.取较大的缩放比例值，用这个值让宽高等比缩放<br/>
2.调整位置，使缩放后的图居中<br/>
3.遍历self.images， 将每张图缩放后导出，放到数组中<br/>
4.使用上面的数组创建animatedImage并返回
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor3_0">回到顶部</a></div>

### UIImage+WebP
首先了解一下WebP
>WebP格式，谷歌（google）开发的一种旨在加快图片加载速度的图片格式。图片压缩体积大约只有JPEG的2/3，并能节省大量的服务器带宽资源和数据空间。Facebook Ebay等知名网站已经开始测试并使用WebP格式。<br/>
>但WebP是一种有损压缩。相较编码JPEG文件，编码同样质量的WebP文件需要占用更多的计算资源。<br/>
>桌面版Chrome可打开WebP格式。

题外话：google还开发了音/视频格式WebM，针对Web平台的音视频传输格式

>WebM由Google提出，是一个开放、免费的媒体文件格式。WebM 影片格式其实是以 Matroska（即 MKV）容器格式为基础开发的新容器格式，里面包括了VP8影片轨和 Ogg Vorbis 音轨，其中Google将其拥有的VP8视频编码技术以类似BSD授权开源，Ogg Vorbis 本来就是开放格式。 WebM标准的网络视频更加偏向于开源并且是基于HTML5标准的，WebM 项目旨在为对每个人都开放的网络开发高质量、开放的视频格式，其重点是解决视频服务这一核心的网络用户体验。Google 说 WebM 的格式相当有效率，应该可以在 netbook、tablet、手持式装置等上面顺畅地使用。

`UIImage+WebP`提供了一个WebP图片的二进制数据转为UIImage的方法`+ (UIImage *)sd_imageWithWebPData:(NSData *)data;`，但是想要使用它，还必须先在项目中导入WebP的解析器libwebp，需要在google code相应的页面clone下来<a href="https://developers.google.com/speed/webp/download" target='blank'>https://developers.google.com/speed/webp/download</a>,但是没翻墙就哭了。这里提供了一个github上的一个mirror：<a href="https://github.com/webmproject/libwebp" target='blank'>https://github.com/webmproject/libwebp</a>, SD对libwebp的版本也有要求，我下载是用的是0.4.3版本。

下面我们看一下`+ (UIImage *)sd_imageWithWebPData:(NSData *)data;`方法的实现：

```objectivec
+ (UIImage *)sd_imageWithWebPData:(NSData *)data {
    WebPDecoderConfig config;
    // 初始化解码器结构体(WebPDecoderConfig类型)变量config
    if (!WebPInitDecoderConfig(&config)) {
        return nil;
    }
    // 将WebP图片的二进制数据传递给config
    if (WebPGetFeatures(data.bytes, data.length, &config.input) != VP8_STATUS_OK) {
        return nil;
    }

    config.output.colorspace = config.input.has_alpha ? MODE_rgbA : MODE_RGB;
    config.options.use_threads = 1;

    // 将WebP图片数据解码为RGBA值数组，保存在config中
    if (WebPDecode(data.bytes, data.length, &config) != VP8_STATUS_OK) {
        return nil;
    }
    // 从config中读取出图片的宽高信息
    int width = config.input.width;
    int height = config.input.height;
    if (config.options.use_scaling) {
        width = config.options.scaled_width;
        height = config.options.scaled_height;
    }

    // 根据解码后的RGBA值数组创建UIImage.
    // 1.创建数据提供者，参数指定了RGBA值数组的开始地址`config.output.u.RGBA.rgba`和长度`config.output.u.RGBA.size`，用于释放数据的回调`FreeImageData`
    CGDataProviderRef provider = CGDataProviderCreateWithData(NULL, config.output.u.RGBA.rgba, config.output.u.RGBA.size, FreeImageData);
    CGColorSpaceRef colorSpaceRef = CGColorSpaceCreateDeviceRGB();
    CGBitmapInfo bitmapInfo = config.input.has_alpha ? kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast : 0;
    size_t components = config.input.has_alpha ? 4 : 3;
    CGColorRenderingIntent renderingIntent = kCGRenderingIntentDefault;
    // 2.使用数据提供者和其他信息创建CGImageRef
    CGImageRef imageRef = CGImageCreate(width, height, 8, components * 8, components * width, colorSpaceRef, bitmapInfo, provider, NULL, NO, renderingIntent);

    CGColorSpaceRelease(colorSpaceRef);
    CGDataProviderRelease(provider);
    // 3.将CGImageRef转为UIImage
    UIImage *image = [[UIImage alloc] initWithCGImage:imageRef];
    CGImageRelease(imageRef);

    return image;
}
```
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor4_0">回到顶部</a></div>

### UIImage+MultiFormat:根据NSData相应的MIME将NSData转为UIImage
这个分类提供了一个通用的方法，的当不知道图片是什么格式的时候，可以使用这个方法将二进制直接传递过来，这个方法的内部会检测图片的类型，并根据相应的方法创建UIImage。

```objectivec
+ (UIImage *)sd_imageWithData:(NSData *)data {
    if (!data) {
        return nil;
    }
    
    UIImage *image;
    NSString *imageContentType = [NSData sd_contentTypeForImageData:data];
    if ([imageContentType isEqualToString:@"image/gif"]) {
        // 1.如果是gif，使用gif的data转UIImage方法
        image = [UIImage sd_animatedGIFWithData:data];
    }
#ifdef SD_WEBP
    else if ([imageContentType isEqualToString:@"image/webp"]) {
        // 2.如果是WebP，使用WebP的data转UIImage方法
        image = [UIImage sd_imageWithWebPData:data];
    }
#endif
    else {// 3.其他情况
        image = [[UIImage alloc] initWithData:data];
        // 获取图片的方向
        UIImageOrientation orientation = [self sd_imageOrientationFromImageData:data];
        // 如果方向不是向上，则使用方向重新创建图片
        if (orientation != UIImageOrientationUp) {
            image = [UIImage imageWithCGImage:image.CGImage
                                        scale:image.scale
                                  orientation:orientation];
        }
    }
    return image;
}
```
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor5_0">回到顶部</a></div>
需要说明的是：在其他情况的处理上的一些细节，
SD会先获取到图片的原始方向，如果方向不是UIImageOrientationUp，使用UIImage的`-imageWithCGImage:scale:orientation:`方法创建图片，这个方法内部会按照传递的方向值将图片还原为正常的显示效果。 举例来说，如果拍摄时相机摆放角度为`逆时针`旋转90度(对应着的EXIF值为8)，拍摄出来的图片显示效果为`顺时针`旋转了90度(这就好比在查看时相机又摆正了，实际上在windows下的图片查看器显示为顺时针旋转了90度，而mac由于会自动处理则正向显示)，而如果使用UIImage的`-imageWithCGImage:scale:orientation:`方法创建图片，则会正向显示也就是实际拍摄时的效果。

至于相机摆放的角度如何与EXIF值对应，请参照这篇文章<a href="http://www.cocoachina.com/ios/20150605/12021.html" target='blank'>《如何处理iOS中照片的方向》</a>，注意的就是iphone的初始方向是横屏home键在后侧的情况。

图片的EXIF信息会记录拍摄的角度，SD会从图片数据中读取出EXIF信息，由于EXIF值与方向一一对应(EXIF值-1 = 方向)，那么就使用`+ sd_exifOrientationToiOSOrientation:`方法通过传递EXIF值获取到方向值。最后就是通过UIImage的`-imageWithCGImage:scale:orientation:`方法创建图片。

在网上有很多介绍如何获取正向图片的方法，它们的思路大多是这样：根据图片的方向值来逆向旋转图片。殊不知，apple早就为你提供好了`-imageWithCGImage:scale:orientation:`方法来直接创建出一个可正常显示的图片。
