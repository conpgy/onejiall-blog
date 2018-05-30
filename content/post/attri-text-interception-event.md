---
title: "属性文本的截取与事件处理"
date: 2014-07-24T18:57:49+08:00
draft: false
tags: [iOS]
categories: [
    "iOS",
]
auther: "Onejiall"
---
上一篇[实现新浪微博图文混排](http://blog.onejiall.com/post/weibo-emotion-text/)谈到了表情与文本的混合排列。属性文本可以有效的设置一段文本中的文字可以有不同的属性。比方微博文本中有话题，超链接，@某人等等。对这些文本需要进行高亮和事件处理。跳转到相应的微博或者链接等等。要实现这个功能，可以分成四个步骤：

1. 查找出所有的链接（用一个数组存放所有的链接）
2. 在touchesBegan方法中，根据触摸点找出被点击的的链接。
3. 在被点击的链接的边框范围内添加高亮背景
4. 在touchesEnd中，移除高亮背景。并发出通知，通知相应的控制器进行事件处理。
<!-- more -->

查看完整代码，请猛击[这里](https://github.com/conpgy/weibo)。

####查找出所有的链接####

通过遍历属性文本，找出对应的链接。然后用一个`GYLink`模型保存起来。模型总有三个属性：

	/** 链接文字 */
	@property (nonatomic, copy) NSString *text;
	
	/** 链接范围 */
	@property (nonatomic, assign) NSRange range;
	
	/** 链接边框 */
	@property (nonatomic, strong) NSArray *rects;

然后将所有的链接保存在一个数组中。代码如下：

    -(NSMutableArray *)links
    {
        if (!_links) {
            NSMutableArray *links = [NSMutableArray array];
            
            // 搜索所有的链接
            [self.attributedText enumerateAttribute:GYLinkText inRange:NSMakeRange(0, self.attributedText.length) options:0 usingBlock:^(id value, NSRange range, BOOL *stop) {
                
                if (value == nil) return;
                
                GYLink *link = [[GYLink alloc] init];
                link.text = value;
                link.range = range;

                // 处理矩形框
                NSMutableArray *rects = [NSMutableArray array];
                // 设置字符选中的范围
                self.textView.selectedRange = range;
                // 计算选中字符范围的边框
                NSArray *selectionRects = [self.textView selectionRectsForRange:self.textView.selectedTextRange];
                for (UITextSelectionRect *selectionRect in selectionRects) {
                    if (selectionRect.rect.size.width == 0 || selectionRect.rect.size.height == 0) continue;
                    
                    [rects addObject:selectionRect];
                }
                link.rects = rects;
                
                [links addObject:link];
            }];
            _links = links;
        }

        return _links;
    }
    
这里，取出链接在属性文本中的范围，设置为textView的选中范围。然后会自动计算出textView中文本字符的范围。根据计算出边框，放在数组中`selectionRects`,然后保存到`link`对象的rects中。这样链接的文本边框就被保存起来了。接下来当点击链接的时候就可以高亮链接的边框了。

####在touchesBegan方法中，根据触摸点找出被点击的的链接。####

在touchesBegan方法中，根据被触摸的点计算出哪个链接被点击了。

    __block GYLink *touchingLink = nil;

    [self.links enumerateObjectsUsingBlock:^(GYLink *link, NSUInteger idx, BOOL *stop) {
        for (UITextSelectionRect *selectionRect in link.rects) {
            if (CGRectContainsPoint(selectionRect.rect, point)) {
                touchingLink = link;
                break;
            }
        }
    }];
    
这样，被点击的链接`touchingLink`就计算出来了。然后，高亮选中的链接边框。
    
####在被点击的链接的边框范围内添加高亮背景####

通过一个for循环，遍历的得到链接字符串所在的边框范围。

    for (UITextSelectionRect *selectionRect in link.rects) {
        UIView *view = [[UIView alloc] init];
        view.tag = GYLinkBackgroundTag;
        view.alpha = 0.5;
        view.layer.cornerRadius = 3;

        view.frame = selectionRect.rect;
        
        [self insertSubview:view atIndex:0];
        view.backgroundColor = GYColor(83, 148, 255);
    }

然后在`touchCancelled`和`touchEnded`方法中将高亮的背景view给删除。
 
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.25 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        for (UIView *view in self.subviews) {
            if (view.tag == GYLinkBackgroundTag) {
                [view removeFromSuperview];
            }
        }
    });
    
为了能够获得这个背景view，前面给这个view添加了一个tag。最后，发送一个通知给控制器，让控制器决定链接点击事件进行处理。做出相应的跳转。

        [[NSNotificationCenter defaultCenter] postNotificationName:GYLinkDidSelectedNotification object:nil userInfo:@{GYLinkText: touchingLink.text}];
