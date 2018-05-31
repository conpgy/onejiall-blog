---
title: "OC字符串的处理"
date: 2018-05-31T11:20:58+08:00
draft: false
tag: [iOS]
categories: [iOS]
---

### 1.字符串的截取

当字符串还有emoji表情时，emoji的length大于1。所以在调用`-[NSString substringToIndex:]`方法时，有可能截取到emoji表情的一部分而出现乱码,
NSString API中提供有两个方法:

    - (NSRange)rangeOfComposedCharacterSequenceAtIndex:(NSUInteger)index;
    - (NSRange)rangeOfComposedCharacterSequencesForRange:(NSRange)range NS_AVAILABLE(10_5, 2_0);

当index所在是一个emoji时，他会返回这个emoji的range。因此可以根据range进行正确的截取。 可以提供这么一个方法进行截取:

    // NSString+Util.m
    -(NSString *)subStringToIndex:(NSInteger)index {
        NSString *result = self;
        if (result.length > index) {
            NSRange rangeIndex = [result rangeOfComposedCharacterSequenceAtIndex:index];
            result = [result substringToIndex:(rangeIndex.location)];
        }

        return result;
    }

    // 示例：
    NSString *str = @"😯😯😯😯😯";
    NSString *result = [str subStringToIndex:1]; // 结果: 😯


### 2.限制字符串长度

上面的方法其实还是有个问题。我们其实并不太能确定我们所需要传递的index的具体值。比如我们想限制字符串的字符长度为10。
这个时候我们并不能确定index是多少。如果index是10，最终获取的结果并不是10个字符。如😯😯😯😯😯。 这并不符合用户的
一般认知。我们发现苹果的API中有这个方法`- (void)enumerateSubstringsInRange:(NSRange)range options:(NSStringEnumerationOptions)opts usingBlock:(void (^)(NSString * _Nullable substring, NSRange substringRange, NSRange enclosingRange, BOOL *stop))block API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0));`。
他能枚举出每个具体的字符。和他们占用的range。因此我们可以借助这个方法进行字符数限制。

    - (NSString *)subStringWithMaxChacterCount:(NSUInteger)maxCount {
        NSString *result = self;
    
        if (self.length > maxCount) {
            __block NSUInteger count = 0;
            __block NSRange subRange;
            [self enumerateSubstringsInRange:NSMakeRange(0, self.length) options:NSStringEnumerationByComposedCharacterSequences usingBlock:^(NSString * _Nullable substring, NSRange substringRange, NSRange enclosingRange, BOOL * _Nonnull stop) {
                count += 1;
                if (count > maxCount) {
                *stop = YES;
                } else {
                    subRange = substringRange;
                }
            }];
            result = [self substringToIndex:(subRange.location + subRange.length)];
        }
    
        return result;
    }

### 3.输入框中限制字符串的长度

    - (void)textChanged:(UITextField *)textField {
        NSInteger maxCount = 10;

        UITextRange *selectedRange = [textField markedTextRange];
        UITextPosition *position = [textField positionFromPosition:selectedRange.start offset:0];
        if (!position) {
            textField.text = [textField.text subStringWithMaxChacterCount:maxCount];
        }
    }