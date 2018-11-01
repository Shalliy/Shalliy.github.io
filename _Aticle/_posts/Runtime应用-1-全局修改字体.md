---
title: Runtime应用(1)-全局修改字体
date: 2018-11-01 11:55:52
tags: iOS, Runtime
---
### 我们首先来说下一般情况下更换字体的步骤
1. 我们从网上down下来一个ttf格式的字体，拖入到项目中，如图所示

<!--More-->

<img src="/img/Runtime01/runtime_01.png" width="50%">
2. 在info.plist文件中添加`Fonts provided by application`，`item`写上文件名，我这里是`蔡云汉隶书书法字体.ttf`
3. 这时候需要检查一下`Build Phases` --> `Copy Bundle Resources`里面有没有我们拖进去的文件。如果没有，把它Add进去。    
<img src="/img/Runtime01/runtime_02.png" width="50%">
4. **但是，我们现在还不能使用，因为我们需要找到这个字体的`PostScript`名称**
那到底要怎么找呢？我们先在电脑上安装这个字体，这时候会在字体册看到它，然后如图操作就可以了。
<img src="/img/Runtime01/runtime_03.png" width="50%">
5. 最后我们在项目中使用如下代码，就可以看到效果了
```objc
    label.font = [UIFont fontWithName:@"CaiYunHanMaoBi-LiShu" size:11];
```
### Runtime更换全局字体
上面只是简单回顾了一下更换字体的步骤，如果我们在开发过程中，每次更换字体都这样，未免有点麻烦。如果说我们项目已经完成了，老板突然来一句，嗯，苹果字体太丑了，我们换个字体，你是不是要哭晕在厕所了。
下面就祭出我们的大杀器Runtime。我们想到每次更换字体都会调`systemFontOfSize:`方法，那么是不是只要我们交换这个方法，在方法里面换成我们自己的字体不就行了吗。说干就干。我们添加一个`UIFont`的分类`XX`
```objc
    #import "UIFont+XX.h"
    
    @implementation UIFont (XX)
    
    + (void)load {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            Method systemMethod = class_getClassMethod([UIFont class], @selector(systemFontOfSize:));
            Method swizzMethod = class_getClassMethod([UIFont class], @selector(my_systemFontOfSize:));
            method_exchangeImplementations(systemMethod, swizzMethod);
        });
    }
    
    + (UIFont *)my_systemFontOfSize:(CGFloat)size {
        [UIFont my_systemFontOfSize:size];
        return [UIFont fontWithName:@"CaiYunHanMaoBi-LiShu" size:size];
    }
```
然后运行查看效果
<img src="/img/Runtime01/runtime_04.GIF">

我们发现，还有很大一部分控件的字体是没有改变的。是不是有好多控件是不会调用`systemFontOfSize:`方法的。那怎么办？兵来将挡，水来土掩，袖子一撸就是干。一般情况下，显示text的控件只有一个，就是UILabel。就算是什么UIButton，UITextField的用于显示Text的子控件，也是继承于UILabel。想到这点，我们就去打UILabel的主意，只要我们改变所有的UILabel的字体不就可以了吗？

那么，我们要交换哪个方法呢？答案是: `willMoveToSuperview:`。
看方法名我们知道，这个方法会在控件将要加载到父view上时调用，这时候控件的各种属性基本已经确定了，我们就可以偷偷得修改了。
```objc
    #import "UILabel+FontChange.h"
    #import <objc/runtime.h>
    
    #define CustomFontName @"CaiYunHanMaoBi-LiShu"
    
    @implementation UILabel (FontChange)
    
    + (void)load {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            SEL systemSel = @selector(willMoveToSuperview:);
            SEL swizzSel = @selector(myWillMoveToSuperview:);
            Method systemMethod = class_getInstanceMethod([self class], systemSel);
            Method swizzMethod = class_getInstanceMethod([self class], swizzSel);
        });
        method_exchangeImplementations(systemMethod, swizzMethod);
    }
    
    - (void)myWillMoveToSuperview:(UIView *)newSuperview {
        self.font = [UIFont fontWithName:@"CaiYunHanMaoBi-LiShu" size:self.font.pointSize];
        [self myWillMoveToSuperview:newSuperview];
    }
```

然后我们运行项目。
WTF，居然崩溃了，崩溃信息如下：
```objc
-[_UIWebViewScrollView font]: unrecognized selector sent to instance 0x10c810a00
2018-11-01 17:05:23.843594+0800 AnXinCaiFu_XSD[18446:4169433] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[_UIWebViewScrollView font]: unrecognized selector sent to instance 0x10c810a00'
```

崩溃的主要原因是`_UIWebViewScrollView`没有`getFont`这个方法（它当然没有）。那么为什么它会调到`getFont`方法呢？我们写的是`UIFont`的分类，`self.font`应该调用的是`UILabel`的`getFont`方法呀？

答案当然是因为消息转发，由于UIFont没有实现`willMoveToSuperview:`方法，然后就被转发到`_UIWebViewScrollView`那里了，但是它并没有`getFont`方法，所以就Crash了。（可以参考iOS的消息转发机制），这里不细说（细说我也不会啊😂）。

那怎么解决呢？很简单，判断一下就好了：
```objc
    /**
         *  我们在这里使用class_addMethod()函数对Method Swizzling做了一层验证，如果self没有实现被交换的方法，会导致失败。
         而且self没有交换的方法实现，但是父类有这个方法，这样就会调用父类的方法，结果就不是我们想要的结果了。
         所以我们在这里通过class_addMethod()的验证，如果self实现了这个方法，class_addMethod()函数将会返回NO，我们就可以对其进行交换了。
         */
    BOOL isAdd = class_addMethod([self class], systemSel, method_getImplementation(swizzMethod), method_getTypeEncoding(swizzMethod));
    if (isAdd) {
        class_replaceMethod([self class], swizzSel, method_getImplementation(systemMethod), method_getTypeEncoding(systemMethod));
    } else {
        //否则，交换两个方法的实现
        method_exchangeImplementations(systemMethod, swizzMethod);
    }
```
然后我们**再一次**运行项目，基本上是接近完美了。
<img src="/img/Runtime01/runtime_05.jpeg" width="50%">
但是我们发现Tabbar的字体还是顽固的没有变化😓。**因为我们完全可以在它加载到父View上之后再次改变它的字体**
我们再想，每次UILabel设置字体是不是要调`setFont`方法，就像上面说的，那么我们就交换它的`setFont`方法，让它设置字体的时候偷偷调换掉它的字体。

```objc
    //交换Set方法
    SEL sysSetFont = @selector(setFont:);
    SEL swizzSetFont = @selector(mySetFont:);

    Method sysSetFontMethod = class_getInstanceMethod([self class], sysSetFont);
    Method swizzSetFontMethod = class_getInstanceMethod([self class], swizzSetFont);

    method_exchangeImplementations(sysSetFontMethod, swizzSetFontMethod);
    
- (void)mySetFont:(UIFont *)font {
    UIFont *swizzFont = [UIFont fontWithName:CustomFontName size:font.pointSize];
    [self mySetFont:swizzFont];
}
```
至此，我们终于完成了全局更换字体，虽然代码不多，但是也耗费了我们一段时间，这其中思想重于代码。
我们要实现什么效果，怎么做才能实现我们的目的，这其中会造成什么样的后果，或者为什么没有实现我们想要的效果。这是我们要思考的问题，这些为什么，也就是促进我们继续学习的动力。

示例代码我放在Github上了，地址：https://github.com/Shalliy/changeFont.git