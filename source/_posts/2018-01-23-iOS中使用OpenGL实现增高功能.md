---
title: iOS中使用OpenGL实现增高功能
categories: iOS开发
date: 2018-01-23 21:23:49
tags:
  - OpenGL
  - 底层
  - UI
comments:
---
### 功能效果

![demo示例](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6lpd4q64g308u0fqnpe.gif)
<!--more-->

### 功能分析  
* 功能：渲染一张传入的图片 -> 手动选择编辑区域 -> 通过滑块来编辑区域的增高或者缩短

* OpenGL原理：
  * 因为OpenGL 只能绘制三角形，所以在处理图像或者图形的时候我们需要将被处理的对象用三角行来分割转换为三角形和顶点的组成的对象。

  * OpenGL里面坐标是以左下角为原点X轴向上为正，Y轴向右为正

### 功能实现

* 渲染图片拆分图片：
   * 拆分方法1：通过图形看出是一个矩形，而矩形是可以分成两个三角形和四个顶点，通过此可以用GL渲染出图片。

![拆分三角形](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6lpjw6g3j30dw0ee7ft.jpg)

  这样拆分之后虽然可以正常渲染，但是带来的问题是我的四个顶点都是死的，也就是四个顶点必须是画布的四个顶点，改变顶点的坐标后只能导致整张画布的变动，而不是某一个区域的变动，拉伸的话也是整张图片的拉伸，所以想要实现局部处理的话这种分割方式不可行。
  * 拆分方法2：将整张图片先拆分为三个矩形，然后再把每个矩形拆分成两个三角形，得到6个三角形，8个顶点，如下图：

![拆分2](https://cdn.cdnjson.com/tvax3.sinaimg.cn/large/006tNc79gy1fo6lprpvrrj30dw0eddrh.jpg)

这样一来就可以保证中间的矩形的高度可以任意变化而上下两部分的高度不变只改变位置，也就是说我们这个DEMO中所做的任何拉伸操作都是对中间矩形的操作，换而言之就是改变最上面的矩形和最下面的矩形之间的距离来达到对中间区域的拉伸和压缩的目的。根据拆分的方式我们用顶点的坐标创建一个数组
```
//顶点数组
GLfloat vertices[] = {   
    -1.2, -1.2,     //左下
    1.2, -1.2,      //右下
    -1.2, -0.4,     //小矩形左下
    1.2, -0.4,      //小矩形右下
    -1.2,  0.4,     //小矩形左上
    1.2,  0.4,      //小矩形右上
    -1.2,  1.2,     //左上
    1.2,  1.2,      //右上
};

 //填充纹理的数组
GLfloat texCoords[] = {     
    0, 0,        //左下                  //下标为 0 1
    1, 0,        //右下                  //下标为2 3
    0, 1.0/3.0,  //小矩形左下             //下标为4 5
    1, 1.0/3.0,  //小矩形右下             //下标为6 7
    0, 2.0/3.0,  //小矩形左上角           //下标为8 9
    1, 2.0/3.0,  //小矩形右上角           //下标为10 11
    0, 1,        //左上                  //下标为12 13
    1, 1,        //右上                  //下标为14 15
};

```

* 手动选择区域：通过添加带有自定义手势的UIView 来实现拖动修改选择区域。

```
*****************CustomPanView代码******************
#import <UIKit/UIKit.h>

@protocol CustomPanViewDelegate <NSObject>

/**
 *  开始拖拽
 *
 *  @param customPanView 自身
 *  @param centerY       自身所在的y坐标
 */
- (void)beginDragWithCoustomPanView:(UIView *)customPanView centerY:(CGFloat)centerY;

@end

@interface CustomPanView : UIView

@property (nonatomic, assign) id<CustomPanViewDelegate> delegate;

@end

```
```
*****************CustomPanViewDelegate代码******************

#pragma mark -
#pragma mark 拖拽View的代理方法
-(void)beginDragWithCoustomPanView:(UIView *)customPanView centerY:(CGFloat)centerY {

    // 限制范围：裁剪区不能大于图片区域
    if (customPanView.center.y >= imageBottom) {
        customPanView.center = CGPointMake(customPanView.center.x, imageBottom);
    }
    if (customPanView.center.y <= imageTop) {
        customPanView.center = CGPointMake(customPanView.center.x, imageTop);
    }

//    获取两条线的坐标
    CGFloat topY = _topView.center.y;

    CGFloat bottomY = _bottomView.center.y;

//    根据两条线的坐标刷新裁剪区域UI
    [_cutLabel setFrame:CGRectMake(0, topY < bottomY ? topY : bottomY, SCREEN_WIDTH, fabs(bottomY - topY))];

//    算出裁剪起始坐标和结束坐标
    CGFloat fromPoint = topY < bottomY ? (imageBottom - bottomY) / imageHeight : (imageBottom - topY) / imageHeight;

    CGFloat toPoint = topY < bottomY ? (imageBottom - topY) / imageHeight : (imageBottom - bottomY) / imageHeight;

   //将中间的矩形的顶点坐标和坐标联系裁剪区域联系起来。
    [self sendFromePoint:fromPoint endPoint:toPoint];



    if (_cutLabel.frame.size.height < 30) {  //隐藏文字
        _cutLabel.text = @"";
    } else {
        _cutLabel.text = @"编辑区域";
    }

    [self.slider setValue:0.0 animated:YES];

    tmpHeight = 0.0f;

}
```
使用一个Delegate将拖移后的Y坐标返回，因为是竖直运动的所以我们只关心Y轴坐标。

* 改变大小：通过将UISliderBar的ValueChange和顶点坐标关联来实现改变顶点坐标，之后调用GLKView 的display的方法来刷新UI，将变化的过程展现出来。

```
- (void)action:(UISlider *)sender {
    //判断是否是向右滑动
    isRightDirection = sender.value >= judgeDirection ? YES : NO;
    //所改变的高度
    changeHeight = sender.value - tmpHeight;

    //遍历数组
    for (int i = 0; i < 16; i ++) {
        //将Y坐标筛选出来
        if (i % 2 != 0) {
            //下半部分矩形
            if (i <= 7) {
                //下半部分矩形Y轴做减法减去变化的高度
                vertices[i] = verticesCopy[i] - changeHeight;
                //上半部分矩形
            } else if (i >= 9) {
                //上半部分矩形Y轴做加法加上变化的高度
                vertices[i] = verticesCopy[i] + changeHeight;

            }

        }

    }
    //缩小时候如果编辑区域已经成为一条线了就不能在缩小了
    if (vertices[11] > vertices[7]) {

        [self.glView display];

    }

}

```

* 通过glReadPixels来从内存中读取像素数据，GLubyte -> CGImageRef -> UIimage 然后最相关的保存或者其他操作。

```
#pragma mark -
#pragma mark 获取处理后的图片
- (UIImage *) createImage {

    int imageY = 0;

    int imgHeight = 0;

    if (isRightDirection) { // 判断slider滑动方向

        imageY = fabs(imageTop - fabs(changeHeight * perOpengleseCoordinate)) * screenScale;

        imgHeight = fabs(imageHeight + 2 * fabs(changeHeight * perOpengleseCoordinate)) * screenScale;

    } else {

        imageY = fabs(imageTop + fabs(changeHeight * perOpengleseCoordinate)) * screenScale;

        imgHeight = fabs(imageHeight - 2 * fabs(changeHeight * perOpengleseCoordinate)) * screenScale;

    }


    int imageWidth = SCREEN_WIDTH * screenScale;

    int dataLength = imageWidth * imgHeight * 4;

    GLubyte *data = (GLubyte*)malloc(dataLength * sizeof(GLubyte));

    glPixelStorei(GL_PACK_ALIGNMENT, 4);

    glReadPixels(0, imageY, imageWidth, imgHeight, GL_RGBA, GL_UNSIGNED_BYTE, data);  //从内存中读取像素
    CGDataProviderRef ref = CGDataProviderCreateWithData(NULL, data, dataLength, NULL);
    CGColorSpaceRef colorspace = CGColorSpaceCreateDeviceRGB();
    CGImageRef iref = CGImageCreate(imageWidth, imgHeight, 8, 32, imageWidth * 4, colorspace, kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast,ref, NULL, true, kCGRenderingIntentDefault);

    UIGraphicsBeginImageContext(CGSizeMake(imageWidth, imgHeight));
    CGContextRef cgcontext = UIGraphicsGetCurrentContext();
    CGContextSetBlendMode(cgcontext, kCGBlendModeCopy);
    CGContextDrawImage(cgcontext, CGRectMake(0, 0, imageWidth, imgHeight), iref);

    CGImageRef imageMasked = CGBitmapContextCreateImage(cgcontext);
    UIImage * image = [UIImage imageWithCGImage:imageMasked scale:screenScale orientation:UIImageOrientationUp];
    UIGraphicsEndImageContext();

    free(data);
    CFRelease(ref);
    CFRelease(colorspace);
    CGImageRelease(iref);

    return image;

}

```
