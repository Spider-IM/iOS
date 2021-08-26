### 集成步骤

#### 1、导入 SDK 
   引入 ```Spider-lib.framework```,如果您使用 SDK 中定义的语音消息，您还需要引入 ```lame.framework```
#### 2、初始化
```
SPIMConfig *config = [[SPIMConfig alloc]init];//你可以在 config 里设置您自己部署好的 Spider Server 地址 
[[SPIMManager sharedClient]initWithAppkey:@"appkey" config:config];
```
#### 3、设置消息和连接状态回调的代理
```
[[SPIMManager sharedClient]addConnectionStatusListener:self];
[[SPIMManager sharedClient]addMessageListener:self];
```
您需要设置并实现代理中的方法，来接收消息和连接状态变化的通知

```
@protocol SPConnectionStatusListener <NSObject>
@optional
-(void)onConnecting; 
-(void)onConnected;
-(void)onOpenDB:(int)code;
-(void)onDisconnect:(int)code describe:(NSString *)describe;;
-(void)onKickedOffline:(NSString *)describe;
@end
```

```
@protocol SPReceiveMessageListener <NSObject>
- (void)onReceived:(SPMessage *)message left:(int)nLeft object:(id)object;
@end
```
#### 4、连接
传入用户的 UserId 和他的签名（如何获取签名），你可以在成功或者失败的回调中处理相关的跳转逻辑。注：重连失败之后，如果 uid 和 sign 都争取，SDK 会帮您不断重试。
```
[[SPIMManager sharedClient]connect:uid sign:self.imToken success:^() { 

} error:^(int errorCode) {

}];
 ```
### 会话体系
目前支持一对一的单聊和，一对多的群聊模式

```
typedef NS_ENUM(NSUInteger, SPSessionType) {

    SPSessionType_Single = 1,

    SPSessionType_Group = 2,
};
``` 
### 自定义消息
#### 1、消息体系
* SPMessage 消息的基类 ，包含了消息的 Id 、内容、发送者、状态等等信息。
* SPMessageContent 消息内容的基类，所有的消息都继承这个基类比如 SPTextMessage等，子类需要继承该类并实现 SPMessageContentProtocol 协议。
* SPMessageContentProtocol 消息内容协议，所有消息必须实现该协议。
* SPMessageProfileType 消息属性，定义了消息是否存储计数等信息。

```
/**
 消息存储类型
 
 - SPMessagePersistFlag_NOT_SAVE_NOT_COUNT: 本地不存储 不计数
 - SPMessagePersistFlag_SAVE: 本地存储，不计数
 - SPMessagePersistFlag_SAVE_AND_COUNT: 本地存储，并计入未读计数
 - SPMessagePersistFlag_TRANSPARENT: 透传消息，不多端同步，如果对端不在线，消息会丢弃
 */
typedef NS_ENUM(NSInteger, SPMessageProfileType) {
    SPMessageProfileType_NOT_SAVE_NOT_COUNT = 0,
    SPMessageProfileType_SAVE_NOT_COUNT = 1,
    SPMessageProfileType_SAVE_AND_COUNT = 2,
    SPMessageProfileType_TRANSPARENT = 3,
};
```
 SDK 内置了文本消息 SPTextMessage 、图片消息 SPImageMessage、语音消息 SPVoiceMessage
 
```
SPMessage.h

@interface SPMessage : NSObject
/**
 消息在本地数据库存储的 Id
 */
@property (nonatomic, assign, readonly) long messageId;

/*!
 消息在服务器唯一 Id
 */
@property (nonatomic, copy) NSString *uniqueId;

/**
 消息所属的会话 Id
 */
@property (nonatomic, copy) NSString *sessionId;

/**
消息所属的会话类型
*/
@property (nonatomic, assign) SPSessionType sessionType;

/**
 消息内容
 */
@property (nonatomic, strong, readonly, nullable) SPMessageContent *content;

/**
 消息发送时的属性
 */
@property (nonatomic, strong, readonly) SPMessageSendProfile *profile;

/**
 消息类型
 */
@property (nonatomic, copy, readonly) NSString *messageType;

/**
 消息发送者的用户ID
 */
@property (nonatomic, copy, readonly) NSString *senderId;

/**
 消息方向
 */
@property (nonatomic, assign, readonly) SPMessageDirection direction;

/*!
 消息的发送状态
 */
@property (nonatomic, assign) SPMessageSentStatus sentStatus;

/**
 消息的发送时间
 */
@property (nonatomic, assign) long long sentTime;

/**
 消息的接收时间
 */
@property (nonatomic, assign, readonly) long long receivedTime;

/**
 消息的接收时间
 */
@property (nonatomic, copy, readonly) NSString *digest;
@end
```

```
**
 消息内容，自定义消息可以继承此类
 */
@interface SPMessageContent : NSObject <SPMessageContentProtocol>

/**
 推送内容
*/
@property (nonatomic, copy)NSString *pushContent;

/**
 消息原始内容
 */
@property (nonatomic, strong)NSData *originalData;

/**
 附加信息
 */
@property (nonatomic, strong)NSDictionary *extra;

/**
 摘要信息，在会话列表最后一条消息处显示
 */
@property (nonatomic, strong)NSString *digest;
@end
```

```
/**
 消息协议，所有消息(包括自定义消息均需要实现此协议)
 */
@protocol SPMessageContentProtocol <NSObject>

/**
 消息编码

 @return 消息的持久化内容
 */
- (NSData *)encode;

/**
 消息解码

 @param payload 消息的持久化内容
 */
- (void)decode:(NSData *)payload;

/**
 消息类型，必须全局唯一。1000及以下为系统内置类型，自定义消息需要使用1000以上。

 @return 消息类型的唯一值
 */
+ (NSString *)getMessageType;

/**
 消息的存储策略

 @return 存储策略
 */
+ (SPMessageProfileType)getMessageProfileType;

/**
 消息的简短信息

 @return 消息的简短信息，主要用于通知提示和会话列表等需要简略信息的地方。
 */
- (NSString *)getMessageDigest;
@end
```

以 SPTextMessage 为例：

SPTextMessage.h 定义如下： 

```
#define SPTextMessageName @"SP:TxtMsg"

@interface SPTextMessage:SPMessageContent
/**
 构造方法

 @param text 文本
 @return 文本消息
 */
+ (instancetype)initWithString:(NSString *)text;

/**
 文本内容
 */
@property (nonatomic, copy)NSString *text;

@end
```

```
@implementation SPTextMessage
- (NSData *)encode {
    
    NSMutableDictionary *dataDict = [NSMutableDictionary dictionary];
    if (self.extra) {
        [dataDict setObject:self.extra forKey:@"extra"];
    }
    if (self.digest) {
        [dataDict setObject:self.digest forKey:@"digest"];
    }
    if (self.text) {
        [dataDict setObject:self.text forKey:@"text"];
    }else {
        [dataDict setObject:@"" forKey:@"text"];
    }
    NSData *data = [NSJSONSerialization dataWithJSONObject:dataDict options:kNilOptions error:nil];
    return data;
}

- (void)decode:(NSData *)data {
    __autoreleasing NSError *__error = nil;
    if (!data) {
        return;
    }
    NSDictionary *dictionary = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&__error];
    SPSafeDictionary *json = [[SPSafeDictionary alloc] initWithDictionary:dictionary];

    if (json) {
        self.text = [json stringObjectForKey:@"text"];
        self.extra = [json objectForKey:@"extra"];
        self.digest = [json stringObjectForKey:@"digest"];
    }
}

+ (NSString *)getMessageType {
    return SPTextMessageName;
}

+ (SPMessageProfileType)getMessageProfileType {
    return SPMessageProfileType_SAVE_AND_COUNT;
}


+ (instancetype)initWithString:(NSString *)text {
    SPTextMessage *content = [[SPTextMessage alloc] init];
    content.text = text;
    return content;
}

- (NSString *)getMessageDigest {
    if (self.digest && self.digest.length > 0) {
        return self.digest;
    }
    return self.text;
}
```
#### 2、消息注册

所有的消息都有在初始化之后调用 SPIMManager 中的 ```registerMessageType``` 方法注册

```
/*
注册消息类型

@param messageClass 消息类
*/
- (void)registerMessageType:(Class)messageClass;
``` 
#### 3、消息发送

```
/*
发送消息

@param message  发送的 message ,message 对象的 和 content 以及 profile
不能为空，session 确定消息发送的目标，content 为消息的内容，profile 是消息属性
@param session  要发送到的 session
@param successBlock 成功回调
@param errorBlock 失败回调
@return SPMessage 更新 messageId
*/
- (SPMessage *)sendMessage:(SPMessage *)message
                 toSession:(SPSessionModel *)session
                   success:(void (^)(long messageId,NSString *uniqueId))successBlock
                     error:(void (^)(int errorCode, long messageId))errorBlock;
```
#### 4、接收消息
当收到消息的时候会触发您设置的消息接收的监听的代理方法
```
- (void)onReceived:(SPMessage *)message left:(int)nLeft object:(id)object;
```

### 插件机制
Spider-lib 支持插件的方式开发其他功能模块，方便您做功能模块的拆分，只需要在一个地方维护连接 Spider 的逻辑，其他功能模块不用关心和维护连接逻辑，模块只需要实现 SpPlugin 协议，当 Spider 连接或者收到消息的时候都会回调给扩展模块。

```
@protocol SPPlugin <NSObject>

+ (instancetype)loadPlugin;

-(void)onLogin:(NSString *)appkey userId:(NSString *)userId;

- (BOOL)onReceivedMessage:(SPMessage *)message;

- (void)onConnectionStatusChanged:(SPConnectionStatus)status;

@end
```

SPIMManager 里有注册和卸载插件的方法

```
/*
注册插件，需在 connect 之前注册

 @param SPPlugin
*/
- (BOOL)registerPlugin:(id<SPPlugin>)plugin;

/*
卸载插件

 @param SPPlugin
*/
- (BOOL)unRegisterPlugin:(id<SPPlugin>)plugin;
```

#### 消息相关接口

```
/*
获取本地存储的消息

@param session  要获取消息所属的会话
@param messageTypes   消息的类型集合
@param messageId 起始
@param isForward 获取消息的方向
@param count     获取的条数
@result  message 集合
*/
- (NSArray<SPMessage *> *)getMessages:(SPSessionModel *)session
                         messageTypes:(NSArray *)messageTypes
                            messageId:(long)messageId
                            direction:(BOOL)isForward
                                count:(int)count;
```
```
/*
获取本地存储的消息

@param messageId messageId
@result  message 消息
*/
- (SPMessage *)getMessage:(long)messageId;
```
```
/*!
 删除本地数据库存储的消息

 @param messageIds  消息 ID 的列表，元素需要为 NSNumber 类型
 @return            是否删除成功

 @remarks 消息操作
 */
- (BOOL)deleteMessages:(NSArray<NSNumber *> *)messageIds;
```
```
/*!
 删除本地数据库存储该会话的消息

 @param session  会话
 @return            是否删除成功

 @remarks 消息操作
 */
- (BOOL)deleteSessionMessages:(SPSessionModel *)session;
```
```
/*
获取本地的会话列表

@param sessionTypes  会话类型的集合
@result session  集合
*/
- (NSArray<SPSessionModel *> *)getSessionList:(NSArray *)sessionTypes;
```
```
/*
获取本地的会话

@param sessionId  会话Id
@param sessionType 会话类型
@result SPSessionModel 会话 model
*/
- (SPSessionModel *)getSession:(NSString *)sessionId sessionType:(SPSessionType)sessionType;
```
```
/*
删除本地的会话

@param sessionId  会话Id
@param sessionType 会话类型
@result result 成功失败
*/
- (BOOL)deleteSession:(SPSessionModel *)session;
```
```
/*
更新本地的会话制定状态

@param sessionId  会话Id
@param sessionType 会话类型
@param isTop 是否制定
*/
-(BOOL)updateSessionTopStatus:(SPSessionModel *)session isTop:(BOOL)isTop;
```
```
/*
 清理会话的未读状态
 */
-(BOOL)clearSessionUnreadStatus:(SPSessionModel *)session;
```
```
/*
 更新消息的发送状态
 */
- (BOOL)updateMessageSentStatus:(long)messageId sentStatus:(SPMessageSentStatus)sentStatus;
```
```
/**
 获取某些类型的会话中所有的未读消息数

 @param sessionTypes   会话类型的数组
 @return             该类型的会话中所有的未读消息数
 */
- (int)getUnreadCount:(NSArray *)sessionTypes;
```
