# Report
report库是用来上报日志、异常或者打点的，目前可以分为以下几种：
- Reportor ：上报数据
- TrackReportor : 上报页面展示、生命周期、用户的行为
- NetworkReportor ： 上报网络信息
`TrackReportor`以及`NetworkReportor`其实是`Reportor`的装饰，他们都进一步规范了具体数据的字段。

```dart
 ReportMgr.defaultTrackReportor().onDataStatistics(
    '1028', '点击隐私号码保护', SecurityCenterConfig.ReportVersionName);
```
以这行代码为例，分析调用流程。

## 配置

在混合工程的main.dart执行时，会调用`_initBaseLibs`，report的配置工作就是在这里完成的。

`ReportMgr`是一个门户类，它提供了使用json配置功能的方法`setCommonReportConfigJsonInfo`，可配置项包括：
- header：请求头
- isDisable：是否可用
- server: 上报地址
- isDebug: debug模式下会打印日志
- common: 通用信息，比方说设备信息、应用信息等
- reportLimit: 上传数据大小限制
- reportStrategy: 上报策略，maxCount-达到一定数量上报；maxSeconds-达到一定时间上报
- storeStrategy: 存储策略，memory-内存；sqlite-数据库
虽然这里定义了storeStrategy，但默认是有memory+sqlite共同实现的。
同时它也是一个工厂类，提供了这几种Reportor的对象获取，这里调用的是`defaultTrackReportor`，获取的对象是`TrackReportor`,调用它的`onDataStatistics`:

`setCommonReportConfigJsonInfo`方法中设置了`eventName`和`Strategy`的对应关系，不同的上报业务可以有不同的上报策略，也就有不同的上报管理与实现(`ReportItemMgr`):

```dart
 ReportStrategy? defaultReportStrategy = _parseReportStrategy(
    Convert.convertMap<int>(configMapInfo['reportStrategy']));
if (null != defaultReportStrategy) {
    _defaultReportor.setReportStrategy(defaultReportStrategy);
}

final customItemList = configMapInfo['custom'];
if (null != customItemList) {
    for (var costomItem in customItemList) {
    final eventNameList = costomItem['eventNameList'];
    final reportStrategy = _parseReportStrategy(
        Convert.convertMap<int>(costomItem['reportStrategy']));
    if (null != eventNameList && null != reportStrategy) {
        _defaultReportor.setReportStrategy(reportStrategy,
            eventNames: Convert.convertList<String>(eventNameList));
    }
    }
}
```
## 添加数据

`onDataStatistics`方法如下：

```dart
void onDataStatistics(String dataId, String dataDesc, String dataVersion, [Map<String, dynamic>? extraInfo]) {
    final reportInfo = Map<String, dynamic>();
    reportInfo['dataId'] = dataId;
    reportInfo['dataDesc'] = dataDesc;
    reportInfo['dataVersion'] = dataVersion;
    reportInfo.addAll(Trim.trimMap<String, dynamic>(extraInfo));
    reportor.addReportData(DataStatisticsEventName, reportInfo);
}
```
除了`onDataStatistics`外，还定义了其他方法，目的是规范上报的数据格式，比方说`onPageEvent`、`onAppEvent`、`onLoggerEvent`等，他们都有"定制化"的字段。
这个方法必要的参数是`dataId`、`dataDesc`以及`dataVersion`。然后调用`reportor.addReportData`:

```dart
void addReportData(String eventName, Map<String, dynamic> reportData) {
    if (_flutterLogDisable) {
      // 日志禁用
      return;
    }
    _addReportData(
        ReportItemType.Data, eventName, null, reportData, DateTime.now().millisecondsSinceEpoch, UUID.uuid());
}
```
```dart
void _addReportData(ReportItemType reportItemType, String? eventName, String? filePath,
      Map<String, dynamic>? reportData, int? timestamp, String? ident,
      [bool reAdd = false]) {
        //相当于把switch语句转为表达式
    final reportItem = Value.fromLambda<ReportItem?>(() {
      switch (reportItemType) {
        case ReportItemType.Data:
          {
            final reportItem = ReportItem.data(eventName, reportData, timestamp, ident);
            _findReportItemMgr(eventName, _reportDataItemMgrList).addReportItem(reportItem);
            return reportItem;
          }
        case ReportItemType.File:
          {
            final reportItem = ReportItem.file(eventName, filePath, reportData, timestamp, ident);
            _findReportItemMgr(eventName, _reportFileItemMgrList).addReportItem(reportItem);
            return reportItem;
          }
        default:
          {
            return null;
          }
      }
    });
    //打印日志 0-add 1-success 2-failed
    _printReportInfo(reportItemType, [reportItem], 0);
    if (_isInitialized) {
        if (reAdd) {
            _reportDao.reAddReportItem(reportItemType, reportItem);
        } else {
            _reportDao.addReportItem(reportItemType, reportItem!);
        }
    }
}
```
上报的类型`ReportItemType`分为`data`和`file`，如果是`file`类型，则需要提供`filePath`；`ReportItem`代表数据上报的基本单位：
```dart
class ReportItem {
  final String? ident;
  final String? eventName;
  final String? filePath;
  final int? timestamp;
  final Map<String, dynamic>? reportData;
}
```
`_findReportItemMgr`这个方法是为了获取`ReportItemMgr`对象，`ReportItemMgr`内部使用List管理`ReportItem`），它提供了`check`方法和`take`方法来检查和获取待上报的数据:

```dart
bool check() {
    bool checkSeconds = false;
    bool ckeckMaxCount = false;
    if (0 != _reportStrategy!.maxSeconds) {
      checkSeconds = ++_seconds >= _reportStrategy!.maxSeconds;
    }
    if (0 != _reportStrategy!.maxCount) {
      ckeckMaxCount = _reportItemList.length >= _reportStrategy!.maxCount;
    }
    return ((checkSeconds || ckeckMaxCount) && _reportItemList.length > 0);
}
```

```dart
List<ReportItem> takeReportItemList() {
    List<ReportItem> repotItemListRet;
    int endRange = _reportItemList.length;
    if (0 != _reportStrategy!.maxCount && _reportItemList.length >= _reportStrategy!.maxCount) {
      endRange = _reportStrategy!.maxCount;
    }
    repotItemListRet = _reportItemList.sublist(0, endRange);

    //重置
    _seconds = 0;
    _reportItemList.removeRange(0, endRange);

    return repotItemListRet;
}
```

`_findReportItemMgr`这个方法先从`_eventReportStrategy`中获取`eventName`对应的`ReportStrategy`，然后遍历`reportItemMgrList`比较`reportItemMgr.reportStrategy!.hashKey == reportStrategy!.hashKey`获取到目标`ReportItemMgr`，也就是说，`eventName`与`ReportStrategy`有一一对应关系，`ReportStrategy`与`ReportItemMgr`又有一一对应关系。如果最终没有拿到，则会新建一个`ReportItemMgr`然后添加到`reportItemMgrList`中:

```dart
ReportItemMgr _findReportItemMgr(String? eventName, List<ReportItemMgr> reportItemMgrList) {
    ReportStrategy? reportStrategy = _defaultReportStrategy;
    if (_eventReportStrategy.containsKey(eventName)) {
      reportStrategy = _eventReportStrategy[eventName!];
    }

    ReportItemMgr? reportItemMgr;
    for (ReportItemMgr reportItemMgrTmp in reportItemMgrList) {
      if (reportItemMgrTmp.reportStrategy!.hashKey == reportStrategy!.hashKey) {
        reportItemMgr = reportItemMgrTmp;
        break;
      }
    }
    if (null == reportItemMgr) {
      reportItemMgr = ReportItemMgr(reportStrategy);
      reportItemMgrList.add(reportItemMgr);
    }
    return reportItemMgr;
}
```

`_isInitialized`代表的是是否已经开始上报，当调用`start`方法后，这个值会被置为true，这时会多执行一步操作，把数据插入数据库中:
```dart
Future<void> addReportItem(ReportItemType reportItemType, ReportItem reportItem) async {
		final reportDataStr = trimSqlFieldValue(jsonEncode(reportItem.reportData));
		final insertSql = buildInsertReportItemSql(reportItemType.value, reportItem.eventName, reportDataStr, trimSqlFieldValue(reportItem.filePath), reportItem.timestamp, reportItem.ident);
		final insertErr = await _sqlite!.exec(insertSql);
		if (null != insertErr) {
			print('[ReportDao] addReportItem failed sql:$insertSql error:$insertErr');
		}
}
```

## 上报
上报逻辑是`Reportor`调用`start`方法开始的：
```dart
  //开启上报逻辑
  Future<void> start() async {
    if (_isInitialized) {
      return;
    }
    //加载缓存中数据
    await _loadReportInfo();
    //开启上报检测定时器
    _timer = Timer.periodic(Duration(seconds: 1), onCheckReportTimer);
  }
```

## 收集数据

`_loadReportInfo`这个方法是一次性上报历史数据，包括数据库中的数据和当前内存中的数据：
1. 将数据库中的历史数据取出来
2. 获取所有内存中的数据(`ReportItemMgr`)
3. 将所有的数据合并
4. 将内存中的数据存到数据库并清理掉清理掉
5. 上报合并后的数据

开启一个定时器，每`1s`都去检测`_reportDataItemMgrList`中`ReportItemMgr`的数据，具体检测的方式上面已经提到过。

## 空包检测
上报数据之前，会进行空包检测，`_couldUpload`标志位标志了空包检测/上报数据是否成功，如果出现过失败，会先进行空包检测，目的是为了节省处理数据的成本：

```dart
Future<void> _doUploadEmptyPacket() async {
    final timestamp = DateTime.now().millisecondsSinceEpoch;
    final rquestUrl = '$_reportServerUrl?reportTs=$timestamp';
    final header = {'appId': _appId, 'Content-Encoding': 'gzip', 'Content-Type': 'application/json; charset=UTF-8'};

    final request = RequestBuilder.httpRequest(rquestUrl, header: header, useDio: false, useCommonHeaders: false);
    final response = await request.start(useThread: false);
    _lastUploadTime = DateTime.now().millisecondsSinceEpoch;
    if (null != response.succed() && response.succed()!) {
      _doUploadSuccess();
    } else {
      _doUploadFailure();
    }
}

```
```dart
void _doUploadSuccess() {
    _couldUpload = true;
    _failedCount = 0;
    _reportInterval = 0;
}
```
```dart
void _doUploadFailure() {
    _couldUpload = false;
    _failedCount++;
    if (_failedCount < 7) {
      _reportInterval =  pow(2, _failedCount) * 1000;
    }
}
```
我们发现，每次网络失败都以指数时间延长上报间隔。

## 上报间隔
每次上报数据前，会校验上报间隔
```dart
timestamp - _lastUploadTime >= min(_reportInterval, 60000)
```
`_reportInterval`默认是0，只有在上报失败后，会指数增长，但最大上报间隔是1分钟。

## 发起上报
通过http发起上报，上报成功后，会删除在数据库中对应的上报数据。
```dart
Future<void> removeReportItems(ReportItemType reportItemType, List<ReportItem> reportItemList) async {
		var identIdList = reportItemList.map<String?>((item)=>item.ident).toList();
		final removeSql = buildRemoveReportItemSql(reportItemType.value, identIdList);
		final removeErr = await _sqlite!.exec(removeSql);
		if (null != removeErr) {
			print('[ReportDao] reAddReportItem failed sql:$removeSql error:$removeErr');
		}
	}
```

