---
layout: cnblog_post
title:  "通读SDWebImage②--视图分类"
date:   2016-02-22 13:50:39
categories: iOS
---
<div><a name="labelTop"></a></div>
<!--Category-->
<div id="navCategory">
	<b>本文目录</b>
	<ul>
		<li><a href="#anchor1_0">UIView+WebCacheOperation</a></li>
        <li><a href="#anchor2_0">UIImageView+WebCache、UIImageView+HighlightedWebCache、MKAnnotationView+WebCache</a>
        </li>
		<li><a href="#anchor3_0">UIButton+WebCache</a></li>
	</ul>
</div>
<!--Category结束-->
对于视图分类，我们最熟悉的当属`UIImageView+WebCache`这个分类了。通常在为一个UIImageView设置一张网络图片并让SD自动缓存起来就会使用这个分类下的`- (void)sd_setImageWithURL:(NSURL *)url;`方法，如果想要设置占位图，则使用了可以传递占位图的方法。本文会从这个方法入手介绍一些视图分类的使用。首先我们要看一下SD中有关视图的几个分类：

```objectivec
UIView+WebCacheOperation // 将操作与视图绑定和取消绑定
UIImageView+WebCache // 对UIImageView设置网络图片，实现异步下载、显示、同时实现缓存
UIImageView+HighlightedWebCache // 与UIImageView+WebCache的功能完全一致，只是将image设置给UIImageView的highlightedImage属性而不是image属性
MKAnnotationView+WebCache // 与UIImageView+WebCache的功能完全一致，只是将image设置给了MKAnnotationView的image属性
UIButton+WebCache // 功能很强大，可以设置不同的state的BackgroundImage或者Image
```
下面我们先看一下所有的视图分类都依赖的UIView的分类：`UIView+WebCacheOperation `
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor1_0">回到顶部</a></div>

### UIView+WebCacheOperation
为方便找到和管理视图的正在进行的一些操作，SD将每一个视图的实例和它正在进行的操作(下载和缓存的组合操作)绑定起来，实现操作和视图的一一对应关系，以便可以随时拿到视图正在进行的操作，控制其取消等。

具体的实现是使用runtime给UIView绑定了一个属性，这个属性的key是`static char loadOperationKey`的地址,这个属性是NSMutableDictionary类型，value为操作，key是针对不同类型的视图和不同类型的操作设定的字符串

为什么要使用`static char loadOperationKey`的地址作为属性的key，实际上很多第三方框架在给类绑定属性的时候都会使用这种方案(如AFN)，这样做有以下几个好处：

>1.占用空间小，只有一个字节。<br/>
>2.静态变量，地址不会改变，使用地址作为key总是唯一的且不变的。<br/>
>3.避免和其他框架定义的key重复，或者其他key将其覆盖的情况。比如在其他文件(仍然是UIView的分类)中定义了同名同值的key，使用objc_setAssociatedObject进行设置绑定的属性的时候，可能会将在别的文件中设置的属性值覆盖。

`UIView+WebCacheOperation`这个分类提供了三个方法，用于操作绑定关系。

```objectivec
// 返回绑定的属性
- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key;

// 对绑定的字典属性setObject
- (void)sd_cancelImageLoadOperationWithKey:(NSString *)key;

// 对绑定的字典属性removeObject
- (void)sd_removeImageLoadOperationWithKey:(NSString *)key;
```
需要注意对绑定值setObject的时候的一些细节：

```objectivec
- (void)sd_setImageLoadOperation:(id)operation forKey:(NSString *)key {
    // 若这个key对应的操作本来就有且正在执行，那么先将这个操作取消，并将它移除。
    [self sd_cancelImageLoadOperationWithKey:key];
    // 然后设置新的操作
    NSMutableDictionary *operationDictionary = [self operationDictionary];
    [operationDictionary setObject:operation forKey:key];
}
```
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor2_0">回到顶部</a></div>

### UIImageView+WebCache、UIImageView+HighlightedWebCache、MKAnnotationView+WebCache
这一部虽然标题设置了三个分类，但是我们主要讲解UIImageView+WebCache，在本文的开始就说到，另外两个分类的实现是完全一致的，而且代码重复度为99%(丝毫没有夸张)。
UIImageView+WebCache,最熟悉的就是以下几个为UIImage设置图片网络的方法：

```objectivec
- sd_setImageWithURL:
- sd_setImageWithURL: placeholderImage:
- sd_setImageWithURL: placeholderImage: options:

- sd_setImageWithURL: completed:
- sd_setImageWithURL: placeholderImage: completed:
- sd_setImageWithURL: placeholderImage: options: completed:
- sd_setImageWithURL: placeholderImage: options: progress: completed:

- sd_setImageWithPreviousCachedImageWithURL: placeholderImage: options: progress: completed:
```
但无论是使用哪个方法，它们的实现上都是调用了`- sd_setImageWithURL: placeholderImage: options: progress: completed:`方法，只是传递的参数不同。(插语：带方法描述的语言就是麻烦，省略参数做起来都复杂)。

下面我们看一下`- sd_setImageWithURL: placeholderImage: options: progress: completed:`方法的实现：

```objectivec
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock {
    [self sd_cancelCurrentImageLoad]; // 移除UIImageView当前绑定的操作。这一句非常关键，当在TableView的cell包含了的UIImageView被重用时，首先调用这一行代码，保证这个ImageView的下载和缓存组合操作都被取消。如果①上次赋值的图片正在下载，则下载不再进行；②下载完成了，但还没有执行到调用回调(回调包含wself.image = image) ，由于操作被取消,因而不会显示和重用的cell相同的图片；③以上两种情况只有在网速极慢和手机处理速度极慢的情况下才会发生，实际上发生的概率非常小，大多数是这种情况：操作已经进行到下载完成了，这次使用的cell是一个重用的cell，而且保留着imageView的image，对于这种情况SD会用下面的设置占位图的语句，将image暂时设置为占位图，如果占位图为空，就意味着先暂时清空image。
    objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC); // 将传入的url与self绑定
    
    // 如果没有设置延迟加载占位图，设置image为占位图
    // 这句代码要结合上面的理解，实际上在这个地方SD埋了一个bug，如果设置了SDWebImageDelayPlaceholder选项，会忽略占位图，而如果imageView在重用的cell中，这时会显示重用着的image。
    // 我建议将下面的两句改为
    /*
        if (!(options & SDWebImageDelayPlaceholder)) {
            dispatch_main_async_safe(^{
                self.image = placeholder;
            });
        } else {
            dispatch_main_async_safe(^{
                self.image = nil;
            });
        }
    */    
    if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
            self.image = placeholder;
        });
    }
    if (url) {
        // 检查是否通过`setShowActivityIndicatorView:`方法设置了显示正在加载指示器。如果设置了，使用`addActivityIndicator`方法向self添加指示器
        if ([self showActivityIndicatorView]) {
            [self addActivityIndicator];
        }
        
        __weak __typeof(self) wself = self;
        id <SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadImageWithURL:url options:options progress:progressBlock completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            [wself removeActivityIndicator]; // 移除加载指示器
            if (!wself) return;
            dispatch_main_sync_safe(^{
                if (!wself) return;
                if (image && (options & SDWebImageAvoidAutoSetImage) && completedBlock)
                { // 如果设置了禁止自动设置image选项，则不会执行`wself.image = image;`,而是直接执行完成回调，有用户自己决定如何处理。
                    completedBlock(image, error, cacheType, url);
                    return;
                }
                else if (image) {
                    // 设置image
                    wself.image = image;
                    [wself setNeedsLayout];
                } else { // image为空，并且设置了延迟设置占位图，会将占位图设置为最终的image
                    if ((options & SDWebImageDelayPlaceholder)) {
                        wself.image = placeholder;
                        [wself setNeedsLayout];
                    }
                }
                if (completedBlock && finished) { 
                    completedBlock(image, error, cacheType, url);
                }
            });
        }];
        // 为UIImageView绑定新的操作
        [self sd_setImageLoadOperation:operation forKey:@"UIImageViewImageLoad"];
    } else { // 判断url不存在，移除加载指示器，执行完成回调，传递错误信息。
        dispatch_main_async_safe(^{
            [self removeActivityIndicator];
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                completedBlock(nil, error, SDImageCacheTypeNone, url);
            }
        });
    }
}
```
这个就是完整的加载网络图片的过程，而具体的如何实现下载细节、网络访问验证、在下载完成之后如何进行内存和磁盘缓存的，请参照上一篇文章的内容。

上面的所有的为UIImageView设置网络图片的方法中有一个和其他稍微不同的`- sd_setImageWithPreviousCachedImageWithURL: placeholderImage: options: progress: completed:`,其实也就是张的有点不同，它的实现是这样的：

```objectivec
- (void)sd_setImageWithPreviousCachedImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock {
    NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:url];
    UIImage *lastPreviousCachedImage = [[SDImageCache sharedImageCache] imageFromDiskCacheForKey:key];
    
    [self sd_setImageWithURL:url placeholderImage:lastPreviousCachedImage ?: placeholder options:options progress:progressBlock completed:completedBlock];    
}
```
可以看到，它的思路是先取得上次缓存的图片，然后作为占位图的参数再次进行一次图片设置。

在设置图片的过程中，有关如何移除和添加加载指示器的两个方法，我们这里不做讨论，其实是对系统的`UIActivityIndicatorView`视图的使用。

还有一个需要的方法`- (void)sd_setAnimationImagesWithURLs:(NSArray *)arrayOfURLs`，要注意的是这个方法传递的参数是一个由URL组成的数组，这个方法用来设置UIImage的animationImages属性。它的实现思路是：
将遍历URL数组中的元素，根据每个URL创建一个下载操作并执行，在回调里面对imationImages属性值追加下载好的image。它的具体实现如下：

```objectivec
- (void)sd_setAnimationImagesWithURLs:(NSArray *)arrayOfURLs {
    [self sd_cancelCurrentAnimationImagesLoad];
    __weak __typeof(self)wself = self;

    NSMutableArray *operationsArray = [[NSMutableArray alloc] init];

    for (NSURL *logoImageURL in arrayOfURLs) {
        id <SDWebImageOperation> operation = [SDWebImageManager.sharedManager downloadImageWithURL:logoImageURL options:0 progress:nil completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            if (!wself) return;
            dispatch_main_sync_safe(^{
                __strong UIImageView *sself = wself;
                [sself stopAnimating]; // 先动画停止
                if (sself && image) {
                    NSMutableArray *currentImages = [[sself animationImages] mutableCopy];
                    if (!currentImages) {
                        currentImages = [[NSMutableArray alloc] init];
                    }
                    [currentImages addObject:image]; // 追加新下载的image

                    sself.animationImages = currentImages;
                    [sself setNeedsLayout];
                }
                [sself startAnimating];
            });
        }];
        [operationsArray addObject:operation];
    }
    // 注意这里绑定的不是单个操作，而是操作数据。UIView+WebCacheOperation的方法`sd_cancelImageLoadOperationWithKey:`也对操作数组做了适配
    [self sd_setImageLoadOperation:[NSArray arrayWithArray:operationsArray] forKey:@"UIImageViewAnimationImages"];
}
```
<div style="text-align: right;font-size:9pt;"><a href="#labelTop" name="anchor3_0">回到顶部</a></div>

### UIButton+WebCache
有关`UIButton+WebCache`分类中的方功能确实强大：可以为image的不同state(Normal、Highlighted、Disabled、Selected)设置不同的backgoud图片或者image图片，但是它的实现很简单，几乎和上面介绍的UIImageView的设置方法是相同的，只是UIButton多了一个管理不同state下的url的功能。

UIButton管理图片的url其实也是通过runtime绑定属性来实现的，和UIImageView不同的是：UIImageView只需一张图片所以就绑定了NSURL类型值，而UIButton需要多张图片且要区分state，所以使用NSMutableDictionary来存储图片的URL，其中key是@(state)，value是该state对应的图片的url。需要注意的是它只是存储了image的URL，而并没有存储backgroudImage的URL。

```objectivec
- (NSMutableDictionary *)imageURLStorage {
    NSMutableDictionary *storage = objc_getAssociatedObject(self, &imageURLStorageKey);
    if (!storage)
    {
        storage = [NSMutableDictionary dictionary];
        objc_setAssociatedObject(self, &imageURLStorageKey, storage, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }

    return storage;
}
```
在`- sd_setImageWithURL: forState: placeholderImage: options: completed:`对它的调用：

```objectivec
[self.imageURLStorage removeObjectForKey:@(state)];

self.imageURLStorage[@(state)] = url;
```