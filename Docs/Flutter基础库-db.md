# 数据库
上边我们提到数据库，其实它是一个plugin，最终是通过原生这边实现数据库增删改查的。
比如 查询操作：
flutter定义查询方法:
```dart
/* 查询
* @querySql:sql语句
* @return:Error为null表示成功，SqliteRecordSet包含结果,否则表示失败
*/
Future<Tuple2<Error?, SqliteRecordSet?>> query(String querySql) async {
  var queryResult = await _sqliteMethodChannel.invokeMethod(CHANNEL_SQLITE_METHOD_QUERY, {'ident':_ident, 'sql':querySql});
  SqliteRecordSet? recordSet;
  final Error? err = _resultToError(queryResult);
  if (null == err) {
    final String? recordSetIdent = queryResult['recordSetIdent'];
          recordSet = SqliteRecordSet(_ident, recordSetIdent, _sqliteMethodChannel);
  }
  return Tuple2(err, recordSet);
}
```

原生注册channel的方法调用监听，并处理真正的逻辑：

```kotlin
_runInThreadExecutor(ident, {
    var errorInfo = "";
    var resultCount = 0;
    var cursorIdent = "${UUID.randomUUID()}";
    try {
        val cursor = database.rawQuery(sql, arrayOf<String>());
        resultCount = cursor.count;
        _cursorInfoMap.set(cursorIdent, cursor);
    }
    catch (ex:Exception){
        errorInfo = Trim.trimString(ex.message);
    }

    Triple(errorInfo, cursorIdent, resultCount);
}) {arg:Any->
    val (errorInfo, cursorIdent, resultCount) = arg as Triple<String, String, Int>;
    val isSuccess = errorInfo.isEmpty();
    result.success(mapOf("ret" to isSuccess, "info" to errorInfo, "recordSetIdent" to cursorIdent, "count" to resultCount));
}
```

注意，由于`Cursor`对象没办法传递给dart端，query方法其实是拿到了一个`Cursor`的标识符。这里传了一个`cursorIdent`标识符并把它保存在一个map中`_cursorInfoMap.set(cursorIdent, cursor)`，当需要批量查询数据的时候，需要把这个标识符传过来:
```kotlin
private fun _handleMoveRecordSetNextMethodCall(call:MethodCall, result:Result) {

    val arguments = call.arguments as Map<String,String>;
    val ident = Trim.trimString(arguments.get("ident"));
    val maxCount = Trim.trimString(arguments.get("maxCount")).toInt();
    val recordSetIdent = Trim.trimString(arguments.get("recordSetIdent"));

    val resultInfoList = mutableListOf<Map<String, Any?>>();
    _runInThreadExecutor(ident, {
        val cursor = _cursorInfoMap.get(recordSetIdent);
        var count = 0;
        while (count++ < maxCount && cursor?.moveToNext() == true) {
            val columnCount = cursor.columnCount;
            val resultInfo = mutableMapOf<String, Any?>();
            for (index in 0 until columnCount) {
                val valueType = cursor.getType(index);
                val columnName = cursor.getColumnName(index);
                val columnValue = when (valueType) {
                    Cursor.FIELD_TYPE_NULL -> null;
                    Cursor.FIELD_TYPE_FLOAT -> cursor.getDouble(index);
                    Cursor.FIELD_TYPE_INTEGER -> cursor.getLong(index);
                    Cursor.FIELD_TYPE_BLOB -> cursor.getBlob(index);
                    else -> cursor.getString(index);
                }
                resultInfo.set(columnName, columnValue);
            }
            resultInfoList.add(resultInfo);
        }
    }) {
        result.success(resultInfoList);
    };
}
```

当`Cursor`使用完毕后，需要手动关闭它:
```kotlin
 private fun _handleCloseRecordSetMethodCall(call:MethodCall, result:Result) {
        val arguments = call.arguments as Map<String,String>;
        val ident = Trim.trimString(arguments.get("ident"));
        val recordSetIdent = Trim.trimString(arguments.get("recordSetIdent"));
        _runInThreadExecutor(ident, {
            val cursor = _cursorInfoMap.remove(recordSetIdent);
            cursor?.close();
            "";
        }) {
            result.success(null);
        };
}
```

# SharePreference
sp其实就是对原生的封装。`getInstance`传入`spName`，会创建`_SCSharedPreferecePlugin`对象：
```dart
static SCSharedPreferencesHelper getInstance(String spName) => _getInstance(spName: spName);

static SCSharedPreferencesHelper _getInstance({String? spName}) {
    return SCSharedPreferencesHelper._internal(spName ?? 'flutter_shared');
}

SCSharedPreferencesHelper._internal(String spName) {
    _pluginShared = _SP_PLUGIN_MAP[spName];
    _cacheShared = _SP_CACHE_MAP[spName];
    if (null == _pluginShared) {
      _pluginShared = new _SCSharedPreferecePlugin(spName);
      _SP_PLUGIN_MAP[spName] = _pluginShared;
    }
    if (null == _cacheShared) {
      final Map<String, Object> _shareCache = new Map();
      _cacheShared = new _SCSharedPrefereceCache(_shareCache);
      _SP_CACHE_MAP[spName] = _cacheShared;
    }
}
```
可见，这里做了优化。`_cacheShared`是内存缓存，其实就是一个`Map`，存值的时候顺便在map中存一份，取值的时候也优先从内存中取值，如果内存中没有，则通过channel调用分发到平台取值。

以`getString`为例：
```dart
Future<String?> getString(String key, {String? defaultValue}) async {
    String? cacheValue = await _cacheShared?.getString(key, defaultValue);
    if (null == cacheValue) {
      String? pluginValue = await _pluginShared?.getString(key, defaultValue);
      return pluginValue;
    }
    return cacheValue;
}
```
`_cacheShared?.getString`如下：

```dart
@override
Future<String?> getString(String key, String? defaultValue) {
    String? value = _preferenceCache[key] as String?;
    return Future.value(value);
}
```
`_preferenceCache`本身是个Map： `final Map<String, Object> _preferenceCache;`，所以`_cacheShared`就是一个快速存取的作用。

`_pluginShared?.getString`如下：
```dart
  @override
  Future<String?> getString(String key, String? defaultValue) async {
    Map<String, String?> map = Map();
    map['key'] = key;
    map['defaultValue'] = defaultValue;
    map['spName'] = spName;
    String? value = await utilChannel.invokeMethod('getString', map);
    return Future.value(value);
  }
```

原生监听flutter的调用：
```dart
channel = MethodChannel(binding.binaryMessenger, "utils")
channel.setMethodCallHandler(this)

 "getString" -> {
    val key = arguments["key"] as String
    val spName = arguments["spName"] as String
    val defaultValue = if (null == arguments["defaultValue"]) {
        null
    } else {
        arguments["defaultValue"] as String
    }
    val value = SCSharePreferenceHelper.getInstance(spName).getString(key, defaultValue)
    Handler(Looper.getMainLooper()).post {
        if (TextUtils.isEmpty(value)) {
            result.success(null)
        } else {
            result.success(value)
        }
    }
 }
```
所以，`_pluginShared`是通过原生平台实现的。
