# 背景

从用户行为日志中进行分析可以产出PV，UV，CTR等关键的业务数据，通过这些数据，可以更加清楚用户的行为习惯，利于发现产品运营和推广中的问题，为产品指定运营策略提供标准。

因此如何正确并且精准的进行用户行为日志上报（客户端埋点），是整个产品关键的一环。

# 客户端埋点

我们对客户端埋点进行了定义：记录用户在客户端的*用户行为*，并结合当前*环境上下文*进行快照**采集**与**上报**。

整个客户端埋点问题可以总结为：

1. 客户端埋点=用户行为采集+上报

2. 用户行为采集=捕捉用户行为+上下文关联

如下图：

![客户端埋点](https://raw.githubusercontent.com/nisacy/legendary/master/img1.png)

**其中上报，业务方无感知，并且已经有了集团统一的解决方案：灵犀。**

**整个客户端埋点的难点，都集中在了用户行为的采集部分，因此如何解决这一部分的难点成为了解决客户端埋点问题的痛点。**

# 原状

打点主要针对一些关键的节点，这里主要介绍一下关键点和原先打点方式。

## 关键点

1. PV/PD

页面展现触发Page View

页面消失触发Page Disappear

2. MV/MC

模块展现触发Module View

模块点机触发Module Click

**MV/MC都是建立于PV之上的，只有有了PV，才有MV/MC的价值。**

## 命令式采集方式

命令式打点采集方案都是在代码执行路径上插桩式打点。

如MV点：

* 在模块被加到视图上时，插入展示打点代码。

* 在模块被点击时，在点击回调中，插入点击打点代码。

以iOS内代码为MV例子：
```

//模块展示逻辑对应的方法
- (void)moduleShow {
  //xxxxx
  //xxxxx
  //在模块展示的最后要插入
    [[NVStatisticsCenter defaultCenter] eventWithCategory:@"shopinfo5" 
                                                   action:@"shopinfo5_info" 
                                                     label:nil
                                                     value:0 
                                                    extras:@[@"shopid",
                                                            [NSString stringWithFormat:@"%@",@(self.shop.uid)]]];
}
```
这样的方式，使得业务方会有很多的困扰

1. 打点代码耦和业务代码

2. 业务方需要找到每个View的打点时机（比如滑停还是露出1px）

3. 业务方需要对打点数据过滤，去重。

4. 埋点只记录瞬时状态，点与点之间没有关联性，容易造成上下文环境缺失

**这种打点方式，使得整个App内打点方式五花八门，需要有一个统一解决这些问题的解决方案，这个方案就是DiTing。**

# DiTing

## 介绍

DiTing是一套客户端声明式的打点框架。

主要解决用户行为采集的问题：

![用户行为采集](https://raw.githubusercontent.com/nisacy/legendary/master/img2.png)

业务方只需要在对应的模块上，绑定相关的上下文，DiTing就会在合适的打点时机，进行相应的埋点操作。

## 例子

DiTing是声明式的框架，业务方使用的时候，只要在对应的UI组件初始化时，声明式的定义相关的上下文信息，就可以做到自动化埋点。

上下文信息也统一采用了静态模型的方式，确保声明的正确性。

iOS:
```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    //cell初始化代码等
    //xxxxxx
    //声明式绑定UI相关上下文
    cell.element_id = @"element_id";
    cell.user_info.title = @"title";
    cell.user_info.index = [NSString stringWithFormat:@"%@",@(indexPath.row)];
    cell.user_info.shop_id = @"shop_id";
    return cell;
}
```
Android:
```
//上下文声明
DTUserInfo userinfo = DTUserInfo();
userinfo.title = "";
userinfo.index = "0";
userinfo.shop_id = "shop_id";
//注册相关的view和对应参数
DTAnalytics.registerModuleEvent(view, "element_id" , userinfo, DTAnalytics.EVENT_VIEW);
```
## 优势

根据上方例子可以看出，使用DiTing有十分强大的优势：

1. 打点代码与业务代码分离

只需要在对应的UI组件上面去绑定相应的上下文，而不需要在业务代码过程中进行插入。

2. 业务方只需要在对应的模块上进行声明式的上下文信息的绑定，不需要再关心埋点的时机和去重等逻辑

所有的打点实际的捕获，去重都由DiTing统一进行管理，确保每个使用DiTing的业务方的打点口径都是一致的。

3. 接入方便，统一管控

接入成本十分的低，由DiTing统一接入灵犀进行上报，方便之后的扩展。

## 原理

下面介绍一下DiTing的原理：

### iOS

PV：对应的ViewController触发-(void)viewDidAppear时候，进行PV打点

PD：对应的ViewController触发-(void)viewDidDisappear时候，进行PD打点

MV：从UI组件触发，在UI组件的时机回调中，增加时机的捕获，整合成合适的打点时机，触发相应的MV打点

MC：截取UIButton点击事件，增加MC打点

### Android

PV：对应的Activity触发protected void onResume()时候，进行PV打点

PD：对应的Activity触发protected void onDestroy()时候，进行PD打点

MV：在Activity监听的所有注册View的滑动事件，滑动停止后，判断对应view是否在屏幕上，触发相应的MV打点

MC：截取View.AccessibilityDelegate打点事件，增加MC打点

### 原理总结
1. PV与PD打点都是在客户端对应的页面生命周期中进行打点。
2. MC统一都是截取点击事件进行打点。
3. MV的实现会有所不同：
* iOS是自下而上：在UI组件级别进行事件的捕获，再将信息整合，筛选出合适的打点时机
* Android是自上而下：在页面级别进行整体页面滑动事件的监控，在滑停的时机去判断合适的打点时机

## 实现

我们以伪代码的方式进行实现的解释。

### iOS

#### 基类Category

`UIResponder+log`
```
@interface UIResponder(log)
//MC/MC的模块信息
@property (nonatomic, strong) LogViewInfo *viewInfo;
//PV/PD的页面信息
@property (nonatomic, strong) LogViewInfo *pageInfo;

//ViewController触发会打PV
- (void)uploadPV;
//ViewController触发会打PD
- (void)uploadPD;
//View触发MV
- (void)uploadMV;
//View触发MC
- (void)uploadMC;
  
@end
```

#### PV/PD
```
@implementation LogBaseController
  
- (void)viewDidAppear:(BOOL)animated {
    //xxxx
    //在viewDidAppear的时候触发打点
    [self logPV];
}

- (void)viewDidDisappear:(BOOL)animated {
    //xxxx
    //在viewDidDisappear的时候触发打点
    [self logPD];
}

//PV方法获取当前页面的上下文，进行打点
- (void)logPV {
    //判断是否需要自动打点
    if ([self needAutoPV]) {
        //触发上报接口，pageinfo是当前页面信息
        [self uploadPV];
    }
}

//PD方法获取当前页面的上下文，进行打点
- (void)logPD {
    //判断是否需要自动打点
    if ([self needAutoPV]) {
        //触发上报接口，pageinfo是当前页面信息
        [self uploadPD];
    }
}

@end
```

#### MC

增加UIButton的category方法，`UIButton+log`，在设置打点参数的时候，进行增加点击点事件
```
@implementation UIButton (informer)

- (void)setViewInfo:(LogViewInfo *)viewInfo {
    //如果没有设置过这个参数，就增加点击方法
    if (self.viewInfo == nil) {
        [self addTarget:self action:@selector(logButtonTouched:) forControlEvents:UIControlEventTouchUpInside];
    }
    [super setViewInfo:viewInfo];
}

- (void)logButtonTouched:(id)sender {
    //触发上报接口，
     [self uploadMC];
}

@end
```
#### MV

mv需要基于UI事件的回调，我们以最常见的UITableView作为例子，LogTableViewDelegate实现`UITableViewDelegate`与`UITableViewDataSource`
```
@interface LogTableViewDelegate() <UITableViewDelegate, UITableViewDataSource>
@property (nonatomic, strong) LogTableView *tableView; 
@end

@implementation LogTableViewDelegate

- (void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath {
    //reloadData时候触发展示某个cell
    if (!tableView.isDecelerating && !tableView.isDragging) {
        [self.tableView logCell:cell];
    }
}

- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    //滑停时候触发打所有展示cells
    [self.tableView logAllCells];
}

- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate
{
    //滑停时候触发打所有展示cells
    if (!decelerate) {
        [self.tableView logAllCells];
    }
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    //点击时候触发cell点击点
    UITableViewCell *cell = [tableView cellForRowAtIndexPath:indexPath];
    [cell uploadMC];
}
 
@end
  
@implementation LogTableView
  
- (void)ga_logVisibleCell:(UITableViewCell *)cell {
    //进行去重和判断cell在屏幕上操作
    if ([self isLoggedAndOnScreen:cell]) {
        [cell uploadMV];
    }
}

- (void)logAllCells {
    //全遍历所有的显示cell
    for (UITableViewCell *cell in [self visibleCells]) {
        [self ga_logVisibleCell:cell];
    }
}
  
  
@end
```

## Android

### PV/PD

```
public class MainActivity extends Activity {

    @Override
    protected void onResume() {
        super.onResume();
       //自动pv
        DTAnalytics.autoPD(this);
     
      //关闭自动PV/PD  一个activity只需调用一次
        DTAnalytics.enableAutoExpose(this, false);
       //手动PV
        DTUserInfo userInfo = new DTUserInfo();
        userInfo.put(DTInfoKeys.AD_ID, "3434343");
        userInfo.put(DTInfoKeys.INDEX, "1");
        DTAnalytics.forcePV(this, "test", userInfo);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
      //自动PD
        DTAnalytics.autoPD(this);
      
      //手动PD
        DTUserInfo userInfo = new DTUserInfo();
        userInfo.put(DTInfoKeys.AD_ID, "3434343");
        userInfo.put(DTInfoKeys.INDEX, "1");
        ........
        DTAnalytics.forcePD(this, "test", userInfo)
    }
}
```

### MV

基于ViewTreeObserver监听的ExposeManager自动曝光。
```
 private static class ViewTreeEventHandler implements ViewTreeObserver.OnGlobalLayoutListener, ViewTreeObserver.OnScrollChangedListener{

        private static final int SCROLL_STOP = 1;

        private Activity mActivity;

        private int mDelay;

        private Handler mHandler = new Handler(Looper.getMainLooper()) {
            @Override
            public void handleMessage(Message msg) {
                if (msg.what == SCROLL_STOP) {
                   //滑停
                }
            }
        };

        public ViewTreeEventHandler(Activity activity, int delay) {
            mActivity = activity;
            mDelay = delay;
        }

        @Override
        public void onGlobalLayout() {
           //曝光
        }

        @Override
        public void onScrollChanged() {
         //滑动
          if (mDelay > 0) {
                mHandler.removeMessages(SCROLL_STOP);
                Message msg = Message.obtain(mHandler, SCROLL_STOP);
                mHandler.sendMessageDelayed(msg, mDelay);
            } else {
                //滑停
            }
        }
    }
```
### MC

基于View.AccessibilityDelegate的自动事件打点
```
//添加辅助监听
host.setAccessibilityDelegate(new DTAccessibilityDelegate())

public class DTAccessibilityDelegate extends View.AccessibilityDelegate {

    
    @Override
    public void sendAccessibilityEvent(View host, int eventType) {
        handleEvent(host, eventType);
        super.sendAccessibilityEvent(host, eventType);
    }

    private void handleEvent(View host, int eventType) {
        switch (eventType) {
            case AccessibilityEvent.TYPE_VIEW_CLICKED: {
               //自动发点击时机
                break;
            }
            default:
        }
    }
}

```
