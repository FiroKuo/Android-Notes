# Chat & Message

## 初始化
当首次打开App时是登录状态，或者登录成功后，会进行chat模块的初始化，包括：
 - 用户信息
 - 产品线
 - 功能设置：提示音开关、日志级别、网络监听、音频录制和播放
 - 路径：媒体保存路径、数据库路径、日志保存路径
 - 查询chatId
 - 数据库：数据库及表的创建，包括消息表、用户表、群组表、会话管理表
 - 长连接：连接websocket
 - 配置对外的页面和功能：提供入口页面、未读消息查询等
 - 解析yaml配置：读取pubspec.yaml中的submodule_config
    ```dart
    Future<Map> getSubmoduleConfig() async {
      String valueString = await rootBundle.loadString('res/pubspec.yaml');
      final mapData = loadYaml(valueString);
      var rst = mapData['submodule_config'];
      return rst != null ? rst as Map : {};
    }
    ```

拉取最新消息
 - 根据数据库中记录的最新的`msgId`来获取新消息，根据`messageType`对不同的消息进行一些预处理操作，比方说push类型的聊天系统里不必处理；对于语音消息，把readStatus置为0；对于已读/未读通知消息，会刷新页面状态，并修改数据库中消息的状态；对于撤回，会刷新页面("你/对方撤回了一条消息")，更新数据库撤回状态以及撤回时间
 - 把消息插入到数据库message表中
 - 根据message的fromUserId/toUserId判断user表中是否包含该聊天对象，如果不存在则获取该User，插入到user表中
 - 将收到的消息回调出去，比方说回调到`message`模块，如果正在聊天页面，会直接把消息设为已读，并且插入到列表中，渲染出来；再比如，需要更新上一层的群组状态，展示最新的消息预览/状态

获取未读消息数
 - 直接通过接口查询未读消息数量

检查消息
- 启动消息重发的定时器

## 消息中心
消息中心即展示对话列表页面，设计为一个StatefulWidget + DataBind业务类，把页面和数据分开，页面就是一个列表，当首次进入页面时，会调用`dataBind#queryMessages`查询数据：
1. 调用接口查询未读消息数量
2. 调用接口查询消息列表
3. 根据两个列表判断是否存在未读消息，展示小红点
4. 通过 message db 查询最近的一条消息
5. 通过 conversation db 查询当前用户未读消息数量
6. 查询最近消息的会话user信息(websocket查询后，插入user表)
7. 通过4，5，6构造对话消息并插入到消息列表

对话的消息类型和其他的消息类型数据结构以及ui list item都是不一样的，它们是继承关系。当点击消息后，根据消息的类型进入到不同的二级页面(对话消息`ConversationListPage`，其他奖励、账单、行程、系统等等`SecondaryListPage`)，跳转后会`wait`住，等返回消息中心后，触发数据刷新操作：

```dart
if (widget.onItemSelected != null) {
  bool finished = await widget.onItemSelected!(context, model!);
  if (finished) {
    await _onRefreshData(false);
  }
}
```

## 非对话消息
非对话消息是指非实时聊天类型的消息列表，比如系统通知、行程状态、奖励信息、账单信息等等。
`SecondaryListPage`也是一个支持刷新、分页的list，它同样把数据操作放在另一个类`dataBind`中，进入页面时刷新数据，这里的数据就直接根据`messageType`去请求接口获取。
然后调用`updateUnreadMsg`传入`messageType`通过接口更新已读状态。

点击item的时候，判断当前消息是否可点击(`jumpUrl`是否为null)，然后分发到原生打开本地页面或者webview。

## 对话消息
对话消息是指实时聊天类型的消息列表。`ConversationListPage`其实就是一个List，不支持下拉刷新和分页加载，它仍然使用`dataBind`管理数据model。
- 首先从数据库中加载当前用户的所有的会话记录
- 同时根据这些会话记录的`chatToUserId`通过websocket查询到`UserInfo`并更新到`User`表中，主要是为了获取当前聊天记录的会话名称以及头像
- 查询当前会话对象的最近一条消息，把它记录到会话对象中
- 查询当前会话对象的未读消息数量，把它记录到会话对象中
- 循环完毕后，对列表进行排序(`lastMessage!.time`)

这个页面没有分页，是一次性加载全部聊天记录，数据构建完毕后，将聊天记录渲染到list中。

**性能问题**
其实，我们可以发现这个页面是有性能隐患的。它在一个dart层面的循环中查询了很多个数据库，边构建边查询，数据少的情况下可能没啥问题，但数据如果上百个，加载速度就会很明显了。
可以构建更加复杂的sql语句，把dart层的循环放在sql层面，包括构建对象，也放在sql层面，这样可以提高加载速度，优化用户体验。但复杂的sql语句要小心维护。

会话支持左滑删除，删除会话其实就是先删除该会话下的所有message: 
```dart
@override
String deleteMessageSql(String? messageId, String conversationId) {
  if (messageId == null) {
    return "delete from messages where current_user_id='$currentChatId' and conversion_id='$conversationId'";
  }
  return "delete from messages where current_user_id='$currentChatId' and message_id='$messageId' and conversion_id='$conversationId'";
}
```
然后再删除该会话：
```dart
@override
String deleteSql(String conversationId) {
  return "delete from conversations where conversation_id='${conversationId}'";
}
```
会话删除之后，由于消息中心是有聊天消息的总入口的，所以要刷新一下未读消息。这里通过listener回调到消息中心页，消息中心的`dataBind`会重新查询一下数据库，刷新一下UI。

最后把该会话对象从内存中删除，刷新UI。

**疑问**
没有看到接口调用? 如何同步的？

## 会话
当点击会话列表中的一项，会打开会话页面`ChatPageRoute`，它是一个`StatefulWidget`，支持很多功能，比如：图片、音频、快捷短语等。进入这个页面时，传递了大量的参数，可见这个页面之复杂。

进入到该页面，首先进行一些功能的加载与准备。

### 消息的加载
会触发消息的拉取，也就是`batchMessages`。

等消息获取完，默认会调用一次loadMore，下拉加载更多数据，从数据库中进行分页查询数据。消息之间会展示时间，判断前一条消息和本条消息之间的间隔是否大于5min，是则设置标志位。

然后把数据渲染出来。

### 消息的监听
首先页面会注册一个消息监听，当接收到消息的时候，做一些预处理，然后渲染到页面上：
```dart
widget.chatDatabind.onReciverMessages = (Message msg) {
  if (widget.messagelist.length == 0)
    msg.isShowTimer = true;
  else {
    Message preMsg = widget.messagelist[0];
    msg.isShowTimer = msg.sendTimeStamp! - preMsg.sendTimeStamp! > seconds_show ? true : false;
  }
  if (msg.isMe == true) {
    widget.chatDatabind.setMessageRead(msg);
  } else {
    /// 打开聊天页面，这个时候收到的消息，都是已读
    if (msg.messageType != MessageType.message_type_audio) {
      widget.chatDatabind.setMessageRead(msg);
    }
  }
  if (!widget.messagelist.contains(msg)) {
    widget.messagelist.insert(0, msg);
  }
  if (widget.businessLine == MessageBIZLine.free_ride) {
    isTripAlreadyOver = false;
  }
  /// 如果切换到后台的话，则不通知已读
  if (!widget.isAppBackground) {
    notifyReadMsg(msg);
  }
  updateState();
};
```
这个监听触发的时机，除了几处主动`batchMessage`调用外，还有`websocket`的推送，当长连接消息`isDirection=1`，说明这是一条会话消息，这个时候，先判断一下当前消息的uid是否是正确的，然后触发`batchMessage`拉取消息。

前面说到初始化的时候，也有大致说过`batchMessage`，它是通过最新的一条`message`的`id`作为`req.ackId`，发起`websocket`请求，拿到`msg list`后，如果该`msg`不存在于 `messages db`中，则插入，并且刷新总消息未读数量以及对应会话的消息未读数量、更新用户关系表、会话表。这是一个递归拉取的过程，只要服务端返回的字段表示继续，则会一直拉取，依据其实就是传过去的`req.ackId`。

这个过程中，如果正在会话中(有注册`onReciverMessages`)，则会收到回调，直接渲染页面，把最新的消息展示出来。

### 消息的发送
底部功能区主要是`InputWidget`，它是一个自定义的`StatefulWidget`，是一个自定义组合组件，支持文本、语音、图片、快捷回复等功能。快捷回复组件`QuickPhraseWidget`也是一个`StatefulWidget`，本质上是一个`ListView`。

消息发送过程如下:
```dart
 @override
  Future<SCError?> sendMessage(MessageModel messageModel, void Function(double)? callback) async {
    this.log!.debug("SendMessage------------------", "ChatManger");
    String? chatId = getChatId!();
    if (chatId == null || chatId.length == 0) {
      return IMError.internetError("当前用户chatid 为空");
    }
    String conversionId = StringUtils.formattingP2PConversionId(chatId, messageModel.toUserId);
    messageModel.conversationId = conversionId;

    SCError? error;
    //更新好友关系及会话列表
    SCError? _error = await ConverterUtils.messageDataStorage(
      messageModel: messageModel,
      log: this.log!,
      db: db!,
      cChatid: chatId,
    );

    if (_error != null) return _error;

    //消息入库
    this.log!.debug("send message add message", "ChatManger");
    error = await db!.messageDb()!.addMessage(messageModel);
    if (error != null) {
      this.log!.error("send message add message failed:${error.errorString()}", "ChatManger");
      return error;
    }

    //消息存数据库 消息状态为正在发送
    this.log!.debug("send message update message send status", "ChatManger");
    error = await db!.messageDb()!.updateMessageSendStatus(messageModel.conversationId, messageModel.tempMsgId, 3);
    if (error != null) {
      this.log!.error("send message update message send status:${error.errorString()}", "ChatManger");
      return error;
    } else {
      this.log!.debug("send message update message send status succed", "ChatManger");
    }

    this.log!.debug("send message type:${messageModel.messageType} | 1=picture,4=audio", "ChatManger");
    String? path = messageModel.text;
    //文件类型判断，如果是图片，音视频，上传文件
    if (messageModel.messageType == 2) {
      Tuple2<SCError?, String?> _uploadRst = await _uploadImage(messageModel, path, callback);
      if (_uploadRst.item1 != null) {
        await db!.messageDb()!.updateMessageSendStatus(messageModel.conversationId, messageModel.tempMsgId, 4);
        messageModel.sendStatus = 4;
        this.listenerManager?.messageUpdate(messageModel);
        return _uploadRst.item1;
      }
      messageModel.text = _uploadRst.item2;
    } else if (messageModel.messageType == 4) {
      //判断文件是否存在
      Tuple2<SCError?, String?> _uploadRst = await _uploadAidio(messageModel, path, callback);
      if (_uploadRst.item1 != null) {
        await db!.messageDb()!.updateMessageSendStatus(messageModel.conversationId, messageModel.tempMsgId, 4);
        messageModel.sendStatus = 4;
        this.listenerManager?.messageUpdate(messageModel);
        return _uploadRst.item1;
      }
      messageModel.text = _uploadRst.item2;
    }

    Tuple2<SCError?, MessageModel?> responseSend = await _onSendMessage(messageModel, callback);
    error = responseSend.item1;
    this.listenerManager?.messageUpdate(messageModel);
    if (error != null) {
      this.log!.error("send message end error:${error.errorString()}", "ChatManger");
    } else {
      this.log!.debug("send message end succed", "ChatManger");
    }
    return error;
  }
```

我们可以看到，首先调用`addMessage`将消息添加到数据库，然后更新`message`的状态，首先更新为正在发送-3；如果是音频、图片，要先上传文件，如果上传失败会把上传状态更新为上传失败，如果多媒体文件上传成功，会把连接更新到`messageModel.text`。真正发送消息的方法是`_onSendMessage`:

```dart
 Future<Tuple2<SCError?, MessageModel?>> _onSendMessage(
      MessageModel messageModel, void Function(double)? callback) async {
    //发送消息GtwSendMsgP2PReq
    SCError? erro;
    GtwSendMsgP2PReq req = GtwSendMsgP2PReq.create();
    req.message = messageModel.text!;
    req.toUserId = messageModel.toUserId!;
    req.messageType = messageModel.messageType!;
    req.extraInfo = messageModel.extraInfo!;
    req.sceneId = TimeUtils.currentTimeStamp().toString();
//    req.imProtocolNo = 0;//im协议号 默认为0， 3.0及其以后版本 暂定为1 传该协议号 只会拉取小于等于该协议号的消息
    this.log!.debug("send message p2p", "ChatManger");
    Tuple2<SCError?, GtwSendMsgP2PRes?> result = await rpcSession!.sendP2P(req);
    if (result.item1 != null) {
      messageModel.sendStatus = 4;

      await db!.messageDb()!.updateMessageSendStatus(messageModel.conversationId, messageModel.tempMsgId, 4);

      this.log!.error("send message p2p failed:${result.item1!.errorString()}", "ChatManger");
      return Tuple2(result.item1, messageModel);
    }
    if (result.item2!.retCode == 0) {
      this.log!.debug("send message p2p succed   messageId = ${result.item2!.messageId}", "ChatManger");
      messageModel.msgId = result.item2!.messageId.toString();
      //成功
      messageModel.sendStatus = 1;
      messageModel.time = TimeUtils.convertToTimestamp(result.item2!.messageTime);
      SCError? dbError = await db!.messageDb()!.updateMessageSendStatus(
            messageModel.conversationId,
            messageModel.tempMsgId,
            1,
            msgid: messageModel.msgId,
            messageTime: messageModel.time,
          );
      log?.debug('send message p2p updateMessageSendStatus messageModel = ${messageModel.toString()}', 'ChatManger');
      if (dbError != null) {
        log?.error(
            "send message p2p succed,update message send status failed:'${dbError.errorString()}'", "ChatManger");
      }
    } else {
      //失败
      erro = IMError(result.item2!.retCode, "sendP2P error", true);
      this.log!.debug("send message p2p failed:${erro.errorString()}", "ChatManger");
      messageModel.sendStatus = 4;
      SCError? dberro = await db!.messageDb()!.updateMessageSendStatus(
            messageModel.conversationId,
            messageModel.tempMsgId,
            4,
          );
      if (dberro != null) {
        this.log!.error("sendP2P failed,update message send status failed:'${dberro.errorString()}'", "ChatManger");
      }
    }
    if (callback != null) callback(1.0);
    return Tuple2(erro, messageModel);
  }
```
这个方法里面通过`websocket`发送消息，如果发送失败则会更新`status`-4到`messages db`，发送成功则会更新`status`-1到`message db`。

### 消息重发
前面提到过，当初始化的时候，会创建`MessageHandle`对象，当消息发送失败后，会把消息添加到失败列表：
```dart
if (IMManager.instance.imPtr != null) {
      SCError? error = await IMManager.instance.imPtr!.chatManager()!.sendMessage(model, null);
      if (error != null) {
        //推送失败加入  队队列处理
        msgHandle!.addSendFailedMsg(model);
      }
}
```
会开启一个定时任务，不断重试发送消息：
```dart
addSendFailedMsg(MessageModel messageModel) async {
  msgList.add(SMsgModel(messageModel));
  _startTimer();
}
```
```dart
timer = Timer(timeout, () async {
  //到时回调
  await _resendMsgs();
});
```
`_resendMsgs`的实现和`sendMessage`类似，最终调用`_onSendMessage`发送消息，就不赘述了。











