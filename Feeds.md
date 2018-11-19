目录

1 整体设计

1.1 首页模块化框架背景介绍

1.2 首页模块化框架整体设计

1.3 首页Feed流整体设计

2 首页Feed流体验优化



### 1 整体设计

#### 1.1 首页模块化框架背景介绍

大众点评V9首页主要内容包括搜索栏、金刚位、大牌抢购、本地运营位、情景感知、垂直频道和猜你喜欢等

大众点评V10首页主要内容包括搜索栏、二楼、金刚位、本地运营位、首页Feed流等

大众点评V9和V10主要内容变化，如下图所示：

![img](https://km.meituan.net/111102343.png?contentId=100821831&attachmentId=111102344&originUrl=https://km.meituan.net/111102343.png&contentType=2&isDownload=false&token=e0656090dd*141168fcbc2a4463bf461&isNewContent=false&isViewPage=true)



其中，首页Feed流由V9单列表改版为V10嵌套列表，包括内层嵌套Tab和多个可横向滑动双列Feed流列表，如下图所示：

首页单列Feed流：

![img](https://p1.meituan.net/dpgroup/b75ef5df8b9184ae499d43f9204e717e577229.jpg) 

首页双列Feed流：

![img](https://p0.meituan.net/dpgroup/482345c0ba2426a37b335201eace282c128319.jpg)

#### 1.2 首页模块化框架整体设计

首页模块化框架包括页面和模块

页面主要负责模块配置，整个页面的视图主体部分是一个UITableView，所有模块共用这个UITableView来渲染各自的视觉。

每个模块为UITableViewSection，其根据Mapi接口返回业务数据，绘制模块UITableViewCell，最后展现在页面上，如下图所示：

![img](https://km.meituan.net/111160756.png?contentId=100821831&attachmentId=111160757&originUrl=https://km.meituan.net/111160756.png&contentType=2&isDownload=false&token=e0656090dd*141168fcbc2a4463bf461&isNewContent=false&isViewPage=true)

首页模块化框架功能特性包括缓存、精细化打点、模块生命周期事件、模块显示或隐藏逻辑、模块排序，如下图所示：

![img](https://km.meituan.net/111124353.png?contentId=100821831&attachmentId=111124354&originUrl=https://km.meituan.net/111124353.png&contentType=2&isDownload=false&token=e0656090dd*141168fcbc2a4463bf461&isNewContent=false&isViewPage=true)

进过V10更新后，我们将Picasso动态化框架融合到首页模块化框架中，提供了两方面能力：

1. 运行时约定页面行为，可动态修改模块样式
2. 布局范式可定制，面向交互友好

V10首页模块化框架主要组成部分为：页面、PicassoModule以及PicassoCell，如下图所示：

![img](https://raw.githubusercontent.com/nisacy/legendary/master/PageModules.png)

PicassoModule将后端Mapi接口返回的JS和业务数据，传递给Cell的PicassoView，通过PicassoView预计算，并根据计算结果渲染模块视图，最后通知页面加载模块

与此同时，每个模块的功能特性，也需要相应的改造

1. 自动化打点：开发在TS侧预先埋点，首页模块化框架将需要打点的View注册到精细化打点框架中
2. 模块缓存：原先首页模块化框架只需要缓存Mapi数据，现在需要缓存Picasso预计算中间结果，即PModel数据
3. 模块生命周期事件：原先页面可以直接将Appear或者Disappear事件传递给模块，现在需要通过PicassoJS桥，将生命周期事件传递给TS侧



#### 1.3 首页Feed流整体设计

首页Feed流主要包括横向Tab组件和横向滑动ScrollView，其中ScrollView为多个纵向双列Feed流列表容器，即首页Feed流为二级双向分离式联动页面，如下图所示：

![img](https://p0.meituan.net/dpgroup/eb3a074101d9c5c9d503cad701625fe867376.png)

其实现细节为：

首页Feed流为首页模块化框架一个子模块，WaterfallCell为具体展现的UITableViewCell，该Cell包括TabControl和ScrollView，TabControl用于不同Feed流栏目切换，ScrollView用于承载每个Feed流栏目以及栏目切换翻页效果，每个栏目为一个CollectionView和多个卡片Cell，根据Layout的不同设置，动态适配单or双列Feed流。每个卡片为一个PicassoView，根据Mapi接口返回的JS和数据，接着进行Picasso预计算，最后渲染到卡片视图，如下图所示：

![img](https://km.meituan.net/111232653.png?contentId=100821831&attachmentId=111232654&originUrl=https://km.meituan.net/111232653.png&contentType=2&isDownload=false&token=e0656090dd*141168fcbc2a4463bf461&isNewContent=false&isViewPage=true)



同时，首页Feed流列表采用MVC软件架构模式，将首页Feed流列表分为三个基本部分：模型（Model）、视图（View）和控制器（Controller），如下图所示：

![img](https://p1.meituan.net/dpgroup/c829e9286336b0c65a3cc11a8461d2c538974.png)

模型（Model）用于封装首页Feed流MApi请求返回业务逻辑相关数据和对数据的处理方法，其中接口返回业务逻辑数据包括Picasso JS数据、JSON数据、Tab列表数据以及翻页逻辑数据等等，如下图所示：

![img](https://p0.meituan.net/dpgroup/47c514a982f9e30484e196720215765063994.png)

其中，首页Feed流数据合法性校验包括：

- 是否包含图片宽高



控制器（Controller）包含主控制器（GuessLike VC）和多个子控制器（Feed VC）

主控制器（GuessLike VC）起到不同View之间的组织作用，包括控制请求、打点、监控、构建视觉、打点等等

子控制器（Feed VC）控制单页Feed流事件并作出响应，包括上拉刷新、下拉翻页、打点、监控、缓存、Gif播放控制等等



视图（View）能够根据PModel数据，展现Feed流双列卡片视觉



### 2 首页Feed流体验优化

##### 2.1 优化首页Feed流空窗体验

当App冷启动时，需要预先展示上一次缓存的PModel数据，避免首页Feed流全白的空窗状态，提升用户体验，演示视频如下所示：

[演示视频](https://pan.baidu.com/s/1vHis5Mz13utyb9812NYong)





##### 2.2 优化首页Feed流首帧图片加载体验

首页Feed流首帧图片加载优化：首页Feed流可以利用Picasso预计算的时间差，并行下载首帧2～4张图片，优化首页Feed流首帧用户体验。其具体流程为：首页Feed流请求成功后，开始Picasso预计算工作，与此同时，并行下载首帧图片，等Picasso预计算完毕时，根据首帧图片预下载超时时间来判断是否还需要继续等待一段时间，如果已经超时，则立即展现首页Feed流，演示视频如下所示：

![img](https://p1.meituan.net/dpgroup/bc7b26bf6bb36d7859642abd455f30ad44783.png)

演示视频如下所示：



[演示视频](https://pan.baidu.com/s/1IUqGMvl_2MFbovwOh43LZA)





##### 2.3 首页Feed流Gif展现优化

首页Feed流Gif展示优化：一般来说，静图比Gif动图小，可以优先展示静图，等待Gif动图下载完毕后再展示动图，改善首页Feed流Gif图片Loading态用户体验，演示视频如下所示：



[演示视频](https://pan.baidu.com/s/1vHis5Mz13utyb9812NYong)





##### 2.4 首页Feed流Gif播放控制

点评首页除了Feed流外，其他模块也可以投放Gif动图，比如517或者广告中通，其中517或者广告中通的Gif播放优先级比首页Feed流高，当两者同时出现时，需要全局控制Gif播放，避免分散用户注意力，演示视频如下所示：



[演示视频](https://pan.baidu.com/s/1Ba8OvGKegKKRx9ADxHigDQ)





##### 2.5 首页Feed流卡片占位图优化

首页Feed流将原先白色的占位图，改进为淡色系占位图，同时结合后端算法计算色值，优化首页Feed流卡片图片Loading态用户体验，演示视频如下所示：



[演示视频](https://pan.baidu.com/s/1w6TxzeV-vkz2eA-7DwrHng)





##### 2.6 首页Feed流请求定位参数精度优化

首页Feed流通过以下方式来获取相对稳定和精确的定位数据：点评App启动会开启系统定位，并返回持续定位结果，客户端比较当前定位和上一次定位的距离，距离结果处在阀值范围内，同时连续维持一定次数，则可以大体认为此时定位处在一个相对稳定状态，接着设置整体计算过程超时时间，避免计算过程耗时过长，如下图所示：

![img](https://p0.meituan.net/dpgroup/cb61685a2cc0cab833c2136c9af3c88562290.png)