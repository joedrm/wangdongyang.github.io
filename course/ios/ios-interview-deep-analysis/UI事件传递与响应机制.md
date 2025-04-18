### `UIView` 和 `CALayer`

二者关系：
- `UIView` 为其提供内容，以及负责处理触摸等事件，参与响应链。
- `CALayer` 负责显示内容 `contents`.

为什么要这么设计？

体现了系统设计 `UIView` 和 `CALayer` 当中所运用的一个设计原则，就是单一职责原则。`UIView` 只负责处理触摸事件，参与视图响应链，而 `CALayer` 只负责内容的显示，体现了职责上的分工。

### 事件传递

关于事件传递，主要和两个方法有关。

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
```

`hitTest:withEvent:` 返回的是 `UIView`，最终哪个视图它响应这个事件就把哪个视图给返回。`pointInside:withEvent:`是用来判断某一个点击的位置是否在当前视图范围内，如果在的话呢，就会返回 `yes`。

事件传递的流程：当我点击了屏幕中的某一个位置的时候，这个事件首先会传递给 `UIApplication`，再由 `UIApplication` 传递给当前的 `UIWindow`。`UIWindow` 里面会判断 `hitTest:withEvent:` 来返回最终的响应的这个视图。内部会调用一个 `pointInside:withEvent:` 的方法来判断当前点击的这个 `point` 是否在 `UIApplication` 范围内，如果在范围内，会遍历它的子视图来查找最终响应这个事件的视图。这里遍历的方式是以倒序的方式，也就是最后添加到 `UIApplication` 上面的视图最优先被便利到。每一个 `UIView` 当中都会去调用它们对应的 `hitTest:withEvent:` 方法。而子视图的内部又会调用它的所有的子视图的方法，最终返回一个响应的视图 `hit`。如果 `hit` 有值的话，这个视图 `hit` 就作为最终的事件，响应的视图就结束了。如果说没有的话，假如它在当前U`UIWindow`范围内呢？那`UIWindow`本身就作为事件的一个响应的视图。

![-min.png](https://s2.loli.net/2025/04/17/by3ATrPxzXSqdOl.png)

一幅流程图来看一下关于 `hitTest:withEvent:` 这个方法的系统内部实现。

![-min.png](https://s2.loli.net/2025/04/17/MTpWNalxIc2Qg7s.png)

案例：方形按钮指定区域接收事件响应，需求是只让它的这个圆形区域可以响应点击事件，而四角红色区域不响应点击事件，

![image.png](https://s2.loli.net/2025/04/17/62MUTHZI9Fzu5EL.png)

创建一个 `UIButton` 的子类 `CustomButton`，以下是 `.m` 文件的代码实现：

```objc
#import "CustomButton.h"

@implementation CustomButton

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
    if (!self.userInteractionEnabled ||
        [self isHidden] ||
        self.alpha <= 0.01) {
        return nil;
    }
    
    if ([self pointInside:point withEvent:event]) {
        //遍历当前对象的子视图，注意是倒序遍历：NSEnumerationReverse
        __block UIView *hit = nil;
        [self.subviews enumerateObjectsWithOptions:NSEnumerationReverse usingBlock:^(__kindof UIView * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            // 坐标转换
            CGPoint vonvertPoint = [self convertPoint:point toView:obj];
            //调用子视图的hittest方法
            hit = [obj hitTest:vonvertPoint withEvent:event];
            // 如果找到了接受事件的对象，则停止遍历
            if (hit) {
                *stop = YES;
            }
        }];
        
        if (hit) {
            return hit;
        }
        else{
            return self;
        }
    }
    else{
        return nil;
    }
}

- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
{
    CGFloat x1 = point.x;
    CGFloat y1 = point.y;
    
    CGFloat x2 = self.frame.size.width / 2;
    CGFloat y2 = self.frame.size.height / 2;
    
    // 通过平方差公式来计算点击的点到圆心的距离。
    double dis = sqrt((x1 - x2) * (x1 - x2) + (y1 - y2) * (y1 - y2));
    // 67.923
    if (dis <= self.frame.size.width / 2) {
        return YES;
    }
    else{
        return NO;
    }
}

@end
```


### 视图响应链

上面是事件的传递的流程。那传递了之后，最终这个事件由谁来响应呢？这就涉及到了视图的一个响应链，包括它的这个响应链的机制和流程，响应有三个方法：

```objc
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{}
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{}
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{}
```
这三个方法是 `UIResponder` 的，而 `UIView` 是继承自 `UIResponder`，所以`UIView`也有这三个方法。当我们点击下图的C2视图的白色圆圈时，最先有C2来接收这个事件，如果此时C2不处理的话，会把这个事件传递给它的下一个响应者B2。也就是C2的直接父视图，如果说B2也不响应这个事件的话，会由B2把这个事件传递给它的下一个响应者，也就是B2的父视图A。视图A如果仍然不响应这个事件的话，就会沿着响应链儿一直向上传递，直到传递到 `UIApplicationDelegate`。

![-min.png](https://s2.loli.net/2025/04/18/spVUl9PqfgT1Zrm.png)

**如果最终仍然没有任何视图去处理这个事件，这个事件就会被忽略掉，当什么都没有发生一样，不会造成崩溃。（面试题）**