UITableView 重用机制

当我们使用 `[tableView dequeueReusableCellWithIdentifier:identifier]` 的时候，实际上就使用到了 `UITableView` 的重用机制。

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    static NSString *identifier = @"reuseId";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:identifier];
    //如果重用池当中没有可重用的cell，那么创建一个cell
    if (cell == nil) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:identifier];
    }
    //返回一个cell
    return cell;
}
```

如下图所示，虚线部分为 `UITableView` 在屏幕上所显示的区域，对于A3、A4和A5这三个 `cell` 是全部显示在当前屏幕当中，而A2和A6只有一部分显示在当前屏幕当中。假如当前 `UITableView` 处在向上滑动之后的一个中间状态，那么A1就会被加入到重用池当中，因为它已经滚出到屏幕之外了。如果我们继续向上滑动这个 `UITableView` 的话，A7就会从重用池当中根据指定的 `identifier` 标识符取出一个可重用的 `cell`。假如说A1到A7这些 `cell` 都是用同一个标识符的话呢？这个A7就可以复用A1所创建的那个 `cell` 的内存，达到了 `cell` 一个复用的目的。

![UITableView.png](https://s2.loli.net/2025/04/17/twyVdeacE5IJxnp.png)


字母索引条的案例

定义一个 `ViewReusePool` 类作为重用池，继承自 `NSObject`，代码如下：

`ViewReusePool.h` 文件

```objc
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>
// 实现重用机制的类
@interface ViewReusePool : NSObject

// 从重用池当中取出一个可重用的view
- (UIView *)dequeueReusableView;

// 向重用池当中添加一个视图
- (void)addUsingView:(UIView *)view;

// 重置方法，将当前使用中的视图移动到可重用队列当中
- (void)reset;

@end
```

`ViewReusePool.m` 文件

```objc
#import "ViewReusePool.h"

@interface ViewReusePool ()
// 等待使用的队列
@property (nonatomic, strong) NSMutableSet *waitUsedQueue;
// 使用中的队列
@property (nonatomic, strong) NSMutableSet *usingQueue;
@end

@implementation ViewReusePool

- (id)init{
    self = [super init];
    if (self) {
        _waitUsedQueue = [NSMutableSet set];
        _usingQueue = [NSMutableSet set];
    }
    return self;
}

- (UIView *)dequeueReusableView{
    UIView *view = [_waitUsedQueue anyObject];
    if (view == nil) {
        return nil;
    }
    else{
        // 进行队列移动
        [_waitUsedQueue removeObject:view];
        [_usingQueue addObject:view];
        return view;
    }
}

- (void)addUsingView:(UIView *)view
{
    if (view == nil) {
        return;
    }
    
    // 添加视图到使用中的队列
    [_usingQueue addObject:view];
}

- (void)reset{
    UIView *view = nil;
    while ((view = [_usingQueue anyObject])) {
        // 从使用中队列移除
        [_usingQueue removeObject:view];
        // 加入等待使用的队列
        [_waitUsedQueue addObject:view];
    }
}

@end
```

现在来使用重用池 `ViewReusePool`，这里实现一个继承自 `UITableView` 的自定义子类 `IndexedTableView`，通过懒加载的方式来创建一个字母索引条容器`view`，并把它添加到 `IndexedTableView` 的 `superview` 上.`IndexedTableViewDataSource` 协议用来获取字母索引条数据，后面会在 `Controller` 实现，下面是 `IndexedTableView` 类的代码实现：

`IndexedTableView.h` 文件

```objc
#import <UIKit/UIKit.h>

@protocol IndexedTableViewDataSource <NSObject>

// 获取一个tableView的字母索引条数据的方法
- (NSArray <NSString *> *)indexTitlesForIndexTableView:(UITableView *)tableView;

@end

@interface IndexedTableView : UITableView
@property (nonatomic, weak) id <IndexedTableViewDataSource> indexedDataSource;
@end
```

`IndexedTableView.m` 文件

```objc
#import "IndexedTableView.h"
#import "ViewReusePool.h"
@interface IndexedTableView ()
{
    UIView *containerView;
    ViewReusePool *reusePool;
}
@end

@implementation IndexedTableView

- (void)reloadData{
    [super reloadData];
    
    // 懒加载
    if (containerView == nil) {
        containerView = [[UIView alloc] initWithFrame:CGRectZero];
        containerView.backgroundColor = [UIColor whiteColor];
        
        //避免索引条随着table滚动
        [self.superview insertSubview:containerView aboveSubview:self];
    }
    
    if (reusePool == nil) {
        reusePool = [[ViewReusePool alloc] init];
    }
    
    // 标记所有视图为可重用状态
    [reusePool reset];
    
    // reload字母索引条
    [self reloadIndexedBar];
}

- (void)reloadIndexedBar
{
    // 获取字母索引条的显示内容
    NSArray <NSString *> *arrayTitles = nil;
    if ([self.indexedDataSource respondsToSelector:@selector(indexTitlesForIndexTableView:)]) {
        arrayTitles = [self.indexedDataSource indexTitlesForIndexTableView:self];
    }
    
    // 判断字母索引条是否为空
    if (!arrayTitles || arrayTitles.count <= 0) {
        [containerView setHidden:YES];
        return;
    }
    
    NSUInteger count = arrayTitles.count;
    CGFloat buttonWidth = 60;
    CGFloat buttonHeight = self.frame.size.height / count;
    
    for (int i = 0; i < [arrayTitles count]; i++) {
        NSString *title = [arrayTitles objectAtIndex:i];
        
        // 从重用池当中取一个Button出来
        UIButton *button = (UIButton *)[reusePool dequeueReusableView];
        // 如果没有可重用的Button重新创建一个
        if (button == nil) {
            button = [[UIButton alloc] initWithFrame:CGRectZero];
            button.backgroundColor = [UIColor whiteColor];
            
            // 注册button到重用池当中
            [reusePool addUsingView:button];
            NSLog(@"新创建一个Button");
        }
        else{
            NSLog(@"Button 重用了");
        }
        
        // 添加button到父视图控件
        [containerView addSubview:button];
        [button setTitle:title forState:UIControlStateNormal];
        [button setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
        
        // 设置button的坐标
        [button setFrame:CGRectMake(0, i * buttonHeight, buttonWidth, buttonHeight)];
    }
    
    [containerView setHidden:NO];
    containerView.frame = CGRectMake(self.frame.origin.x + self.frame.size.width - buttonWidth, self.frame.origin.y, buttonWidth, self.frame.size.height);
}

@end
```

调用重用池的 `reset` 方法的目的就是把所有重用中的所有 `Button` 全部移动到可重用的队列当中。然后在 `reloadIndexedBar` 中调用重用池的 `dequeueReusableView` 方法再去从重用池当中取一个 `Button` 出来，当然首次使用时，没有可重用的 `Button` 就重新创建一个，并注册 `button` 到重用池当中。最后来看看在 `ViewController` 中如何使用？

```objc
#import "ViewController.h"
#import "IndexedTableView.h"
@interface ViewController ()<UITableViewDataSource,UITableViewDelegate,IndexedTableViewDataSource>
{
    IndexedTableView *tableView;//带有索引条的tableview
    UIButton *button;
    NSMutableArray *dataSource;
}
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //创建一个Tableview
    tableView = [[IndexedTableView alloc] initWithFrame:CGRectMake(0, 60, self.view.frame.size.width, self.view.frame.size.height - 60) style:UITableViewStylePlain];
    tableView.delegate = self;
    tableView.dataSource = self;
    
    // 设置table的索引数据源
    tableView.indexedDataSource = self;
    
    [self.view addSubview:tableView];
    
    //创建一个按钮
    button = [[UIButton alloc] initWithFrame:CGRectMake(0, 40, self.view.frame.size.width, 40)];
    button.backgroundColor = [UIColor redColor];
    [button setTitle:@"reloadTable" forState:UIControlStateNormal];
    [button addTarget:self action:@selector(doAction:) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:button];
    
    // 数据源
    dataSource = [NSMutableArray array];
    for (int i = 0; i < 100; i++) {
        [dataSource addObject:@(i+1)];
    }
    // Do any additional setup after loading the view, typically from a nib.
    
}

#pragma mark IndexedTableViewDataSource

- (NSArray <NSString *> *)indexTitlesForIndexTableView:(UITableView *)tableView{
    
    //奇数次调用返回6个字母，偶数次调用返回11个
    static BOOL change = NO;
    
    if (change) {
        change = NO;
        return @[@"A",@"B",@"C",@"D",@"E",@"F",@"G",@"H",@"I",@"J",@"K",@"L",@"M",@"N",@"O",@"P",@"Q"];
    }
    else{
        change = YES;
        return @[@"A",@"B",@"C",@"D",@"E",@"F"];
    }
    
}

#pragma mark UITableViewDataSource

- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
{
    return 1;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    return [dataSource count];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    static NSString *identifier = @"reuseId";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:identifier];
    //如果重用池当中没有可重用的cell，那么创建一个cell
    if (cell == nil) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:identifier];
    }
    // 文案设置
    cell.textLabel.text = [[dataSource objectAtIndex:indexPath.row] stringValue];
    
    //返回一个cell
    return cell;
}

#pragma mark - UITableViewDelegate

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    return 40;
}

- (void)doAction:(id)sender{
    NSLog(@"reloadData");
    [tableView reloadData];
}

- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}


@end
```
运行起来之后，右侧的索引条内容显示有A、B、C、D、E六个字母，当点击顶部的按钮刷新后，变成了11个按钮，实际上这11个中前面6个重用了刷新前的6个按钮。如下图所示：

![-min.png](https://s2.loli.net/2025/04/17/xOX3afGvc5nREpK.png)

而当我们第二次点击顶部刷新按钮刷新后，显示的5个按钮全部发生了重用。第三次点击刷新按钮后，就没有再去新建按钮了，显示的11个按钮也全部发生了重用。控制台打印结果如下：

```
新创建一个Button
新创建一个Button
新创建一个Button
新创建一个Button
新创建一个Button
新创建一个Button
reloadData
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
新创建一个Button
新创建一个Button
新创建一个Button
新创建一个Button
新创建一个Button
reloadData
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
reloadData
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
Button 重用了
```