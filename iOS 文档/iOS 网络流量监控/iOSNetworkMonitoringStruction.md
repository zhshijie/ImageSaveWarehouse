# iOS 网络监控整体架构

## 基本原则

- 不入侵现有业务
- 简单接口调用，一键开启



## 整体架构设计

![WX20180725-094443@2x](/Users/zhangshijie/Desktop/WX20180725-094443@2x.png)

- Catch： 网络请求捕获模块，主要是利用苹果的 NSURLProtocl 抽象协议来实现。捕获并生产网络监控记录
- DataManager： 数据处理模块，包括记录的存储，删除，读取等。
- Store：数据库模块，用于存取捕获到的记录
- Report：数据上报模块，将数据库中的数据上报
- Manager：核心管理模块，用于控制 Catch 和 Report 模块的启动和关闭，暴露了对外调用的接口，包括网络监控的开启关闭，监控数据的获取等。
- Display：展示模块，将数据库中的数据展示到界面上



## 整体运行流程

### 数据上报整体流程

![数据上报整体流程](/Users/zhangshijie/Desktop/ddd.png)

- 开启网络监控功能
- 开启网络数据捕获和网络监控数据上报功能
- 后台线程捕获网络请求，并生成网络监控记录
- 将网络监控记录保存到数据库
- 每保存一条记录，通知所以的数据监听着
- 数据上报模块收到数据插入回调，查询当前数据库中的记录数据
- 如果数据库中的记录数据超过阀值，则将数据库中的所有数据上报
- 删除已上报的数据

### 数据展示整体流程

![数据展示整体流程](/Users/zhangshijie/Desktop/守护家园 (4).png)

- 开启网络监控功能
- 开启网络数据捕获和网络监控数据上报功能
- 后台线程捕获网络请求，并生成网络监控记录
- 将网络监控记录保存到数据库
- 调用者打开网络监控界面
- 网络监控显示模块调用 Manager 中获取（清理）数据的接口
- 显示获取到的数据



##如何集成

- 支持 CocoasPod，导入方式如下

```
pod 'EPNetworkMonitoring', :git => 'git@gitlab.gz.cvte.cn:zhangshijie/EPNetworkMonitoring.git', :branch => '0.1.0'
```

- 导入 EPNetworkMonitoring.h 头文件

```
#import <EPNetworkMonitoring.h>
```

- 必要设置

```
[EPNetworkMonitoringManager setAppId:@"seewocareapm"]; // 数据上报需要个 appid，这个是和业务相关的，启动前必须设置
```

- 开启网络监控上报 

```
#ifdef RELEASE
        [EPNetworkMonitoringManager start]; // 不会将 header，body 等数据保存在 debug 数据库
#else
        [EPNetworkMonitoringManager startWithDebug]; //以 Debug模式开启， 会将 header，body 等数据保存在 debug 数据库
#endif
```

- 打开网络监控界面

```
// 直接将  EPNetworkMonitoringViewController 实例后 push 出来就可以了,网络监控只有在 Debug 模式下才会有数据
EPNetworkMonitoringViewController *viewController = [[EPNetworkMonitoringViewController alloc] init];
[self.navigationController pushViewController:viewController animated:YES];
```

