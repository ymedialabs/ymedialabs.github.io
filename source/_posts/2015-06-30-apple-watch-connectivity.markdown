---
layout: post
title: "Apple Watch Connectivity"
date: 2015-06-30 13:11:39 +0530
comments: true
author: Karthik Mitta
categories: iOS
---
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Apple Watch uses Wi-Fi and Bluetooth to communicate with your paired iPhone, switching between connections as needed:  

* Apple Watch uses Bluetooth when your iPhone is near, which conserves power.  
* If Bluetooth isn’t available, Apple Watch will try to use Wi-Fi. For example, if compatible Wi-Fi is available and your iPhone isn't in Bluetooth range,Apple Watch uses Wi-Fi.

<center><img src="/images/posts/2015-06-30/Distance-between-iPhone-and-Apple-Watch.jpg" align="center" width="400" height="200" /></center>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;To enjoy every feature on your Apple Watch, you'll need to enable Wi-Fi and Bluetooth on your paired iPhone.Apple Watch compatible with Wi-Fi 802.11b/g and Bluetooth 4.0.

### How do we communicate between watch and iPhone app ?

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The communication process between both devices is entirely different in watchOS 2 compared from watchOS 1. In this article, I am going to explain in detail, that how communication works in both OS versions.

**<u>WatchOS 1:</u>**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Basic communiation types are listed below:  
1. Message transfer.    
2. Sharing Data or files with Your Containing iOS App.

**Message transfer:**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; It's a way to communicate directly from watch to iPhone app and get reply back, using below communication API in watch kit framework.

```objc
+(BOOL)openParentApplication:(NSDictionary * nonnull)userInfo reply:(void (^ nullable)(NSDictionary * nonnull replyInfo, NSError * nullable error))reply;
```

userInfo:&nbsp;&nbsp;&nbsp;A dictionary containing data to pass to the iOS app. Use this                     dictionary to pass information to the iOS app so that it can perform any needed tasks. The contents of the dictionary must be serializable to a property list file. This parameter must not be nil.

reply:&nbsp;&nbsp;&nbsp;It's a block with response data (NSDictionary) & error object for the request send by watch app.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; This api belongs to **WKInterfaceController** class, as because every interface controller in watch extension may need to communicate parent iOS app at certain time. If iPhone app is not running, then this api call will wake up the app in background and do the needful.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; The otherside in iPhone app, the below delegate method will get called in appDelegate.

```objc
- (void)application:(UIApplication *)application handleWatchKitExtensionRequest:(NSDictionary *)userInfo reply:(void(^)(NSDictionary *replyInfo))reply
```
**Usage**

```objc
// Method in apple watch app
-(NSInteger)getTheiPhoneAppStatus
{
    NSInteger loginStatus = 0;
    [WKInterfaceController openParentApplication:@{@"requestType":@"getLoginStatus"} reply:^(NSDictionary *replyInfo, NSError *error) 
    {
        if (error == nil)
        {
            NSLog(@"Error %@",error.localizedDescription);
        }
        else if(replyInfo != nil)
        {
            loginStatus = [[replyInfo valueForKey:@"loginStatus"] integerValue];
        }
    }];
    return loginStatus;
}

// Delegate method in iPhone app
- (void)application:(UIApplication *)application handleWatchKitExtensionRequest:(NSDictionary *)userInfo reply:(void(^)(NSDictionary *replyInfo))reply
{
    NSDictionary *responseDict = nil;
    NSString *requestType = [userInfo valueForKey:@"requestType"];
    if([requestType isEqualToString:@"getLoginStatus"])
    {
        responseDict = @{@"loginStatus":[NSNumber numberWithInteger:self.loginStatus]};
    }
    reply(responseDict);
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; By using above api, a watch app can communicate with iPhone app. But to communicate from iPhone to watch, there is no direct api provided by apple. For that you need to
use notification center. NSNotificationCenter API's will work only in same bundle. As you know, iPhone & watch app are two differnt bundles, you can use corefoundation framework api **CFNotificationCenterGetDarwinNotifyCenter** to send notification from iphone to watch app & viceversa.

```objc
// Posting notification
- (void)sendNotificationForMessageWithIdentifier:(NSString *)identifier
{
    CFNotificationCenterRef const Notificationcenter = CFNotificationCenterGetDarwinNotifyCenter();
    CFDictionaryRef const userInfo = NULL;
    BOOL const deliverImmediately = YES;
    CFStringRef str = (__bridge CFStringRef)identifier;
    CFNotificationCenterPostNotification(Notificationcenter, str, NULL, userInfo, deliverImmediately);
}
```

like NSNotificationCenter you need to add & remove observer like below.

```objc
- (void)registerNotificationsWithIdentifier:(NSString *)identifier 
{
    CFNotificationCenterRef const center = CFNotificationCenterGetDarwinNotifyCenter();
    CFStringRef str = (__bridge CFStringRef)identifier;
    CFNotificationCenterAddObserver(center,
    (__bridge const void *)(self),
    WatchNotificationCallback,
    str,
    NULL,
    CFNotificationSuspensionBehaviorDeliverImmediately);
}

- (void)unregisterNotificationsWithIdentifier:(NSString *)identifier 
{
    CFNotificationCenterRef const center = CFNotificationCenterGetDarwinNotifyCenter();
    CFStringRef str = (__bridge CFStringRef)identifier;
    CFNotificationCenterRemoveObserver(center,
    (__bridge const void *)(self),
    str,
    NULL);
}

void WatchNotificationCallback(CFNotificationCenterRef center,
void * observer,
CFStringRef name,
void const * object,
CFDictionaryRef userInfo) 
{
    NSString *identifier = (__bridge NSString *)name;
    [[NSNotificationCenter defaultCenter] postNotificationName:@"WatchDataSynNotificationName"
    object:nil
    userInfo:@{@"WatchIdentifier" : identifier}];
}
```
in order to handle the above notifications, app should be in active state.

**Sharing Data or files with Your Containing iOS App :**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; In order to transfer data/files between watch & iPhone app, use a shared app group to store that data. An app group is a secure container that multiple processes can access. Because your WatchKit extension and iOS app run in separate sandbox environments, they normally do not share files or communicate directly with one another. An app group lets the two processes share files or user defaults.

```objc
// Sharing files
- (NSString *)getSharedDirectoryPath 
{
    NSURL *appGroupContainer = [self.fileManager containerURLForSecurityApplicationGroupIdentifier:@""group.com.companyname.productname];
    NSString *appGroupContainerPath = [appGroupContainer path];
    NSString *directoryPath = appGroupContainerPath;
    if (self.dirName != nil) 
    {
        directoryPath = [appGroupContainerPath stringByAppendingPathComponent:self.dirName];
    }
    [self.fileManager createDirectoryAtPath:directoryPath withIntermediateDirectories:YES attributes:nil error:NULL];
    return directoryPath;
}
```
```objc
// Shared Userdefaults
- (void)storeDataInSharedDefaults:(id)data 
{
    NSUserDefaults *sharedUserDefaults = [[NSUserDefaults alloc] initWithSuiteName:@"group.com.companyname.productname"];
    [sharedUserDefaults setObject:data forKey:@"watchIdentifier"];
}
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; After storing the data in shared container, use **openParentApplication:** api or **CFNotificationCenterGetDarwinNotifyCenter** to notify the receiver app for access data.

**<u>WatchOS 2:</u>**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; In Watch OS 2, apple moved the watch extension bundle & data source to watch itself, previously it was in iPhone only. No more **openParentApplication:** api call, **CFNotificationCenterGetDarwinNotifyCenter** & **sharedGroups**. Apple itself given new framework for all sorts of communication named WatchConnectivity, supports for NSURLSession & codedata.


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Before using WatchConnectivity framework, you need some setup like below. This will ensure that device and Watch OS are supported to use WatchConnectivity api's. It's worth to setup this at the earlier stage of your app like in iOS app appDelegate & watch app extensionDelegate class, as becuase receiver app may need some data at early stage.

``` objc
// Always set your apps up to receive incoming WatchConnectivity content
if ([WCSession isSupported])
{
    self.session = [WCSession defaultSession];
    self.session.delegate = self; // conforms to WCSessionDelegate
    [self.session activateSession];
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; After activating the watchconnectivy session, you need to check whether your iPhone is paired with any watch device or not by usng below api.
``` objc
self.session.paired == false;
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; At first it will be false, as watch is not paired. Once iPhone pair with any watch device, iPhone app will get notify about the state change of pair by invoking a below delegare method, if iPhone app is running.

``` objc
- (void)sessionWatchStateDidChange:(WCSession *)session;
```
next step is watch app installation validation. If user install the watch app later after pairing, then iPhone app will get notify about the installion state change by invoking a above same delegare method, if iPhone app is running. The below validations are only for iPhone app, because doing this from watch app means it's already paired and installed.

``` objc
// Watch connectivity session delegate method
- (void)sessionWatchStateDidChange:(WCSession *)session
{
    if(self.session.paired = YES && self.session.watchAppInstalled = YES)
    {
    // Implement need full task here.
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; When ever watchAppInstalled become true, apple will create a directory in your container, few potins need to consider before storing some data in that container:

* Directory and its contents’ lifetime is tied to the watchAppInstalled property, it means when ever watch app uninstalled all content will get delete.
* Store only for data relevant to the specific instance of your Watch app like below, as apple will clear all of your data while uninstalling the app or unpair the device.

1. Last queued item marker
2. Preferences
3. Files queued for transfer

you can get that container path by using below api.

```objc
self.session.watchDirectoryURL
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Now you are good enough to communicate between both apps. Apple divided the communication into two categories:

* Background transfers
* Interactive messaging

###**1) Background transfers**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; This meant for content not needed immediately at receiver side, OS intelligently transfer the content. If container app contains some content and fetching some more content from server & receiver app is not in active state, that means receiver not needed the content immediately. So, it will queue up content in system. And system will validate the right conditions intelligently to transfer the content across, like it will consider power, performance & app state etc. System will transfer content, if conditions satisfied and then it will delivered to receiver app when it launches.It divided into three types:

1. Application context
2. Userinfo transfer
3. file transfer

**Application context**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; This will offer a single set of information to other side app. For example take FoodPanda iPhone application, which shows the restaurants based on your current of choosed
location. If you want to show the set of restaurants in apple watch as well. Then store them in application context, when ever user launches the watch app the content will be there to show. There are two main
properties **applicationContext** & **receivedApplicationContext** which stores data(dictionary) in sender & receiver apps respectively.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; FoodPanda iPhone app will call an api **updateApplicationContext:**, which will store the data in **applicationContext** property, if iPhone app pushing latest context before
transfer, system will replace the existing data in **applicationContext** property. Always latest set of content will be there in applicationContext. And you can use this context to show subset of data in watch glance
as well.

``` objc
// Sender side code
-(void)updateTheLatestContext
{
    NSDictionary *context = // Create context dictionary with latest state;
    NSError *error = nil;
    [[WCSession defaultSession] updateApplicationContext:context error:&error];
    if (error != nil) {
        // Handle any errors
    }
}
```
``` objc
// Receiver side delegate method to handle it.
-(void)session:(nonnull WCSession *)session didReceiveApplicationContext:(nonnull NSDictionary<NSString *,id> *)applicationContext
{
    // Handle application context dictionary
    dispatch_async(dispatch_get_main_queue(), ^{
        //Handle UI operations.
    });
}
```
**NOTE:** All delegate methods of this framework will call in secondary thread, if you have any UI operations, you need to use dispatch_async with mainQueue.

**Userinfo transfer**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; This is similar to application context, but the only difference is, it will transfer multiple set of content by queue up them. It is very usefull to do some userinfo inmemory data transfer. For example, take some of the best fitness & health related apps like Nike+ Running, Hello heart & Map my run. These apps will collect some data related to user from watch and show the progress in iPhone app by graphical representation. In this case you can use userinfoTransfer to send multiple set of userinfo data like running time, heart beat data, walking time etc to iPhone app.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Use **transferUserInfo:** to send userinfo, WCSession will store it in Outstanding userInfo tranfers Queue. This api will return an object of **WCSessionUserInfoTransfer** class.

``` objc
// Sender side code
-(void)sendTheUserinfo
{
    NSDictionary *userInfo = @{}; // Create dictionary of userInfo
    WCSessionUserInfoTransfer *userInfoTransfer = [[WCSession defaultSession] transferUserInfo:userInfo];
}
```
WCSessionUserInfoTransfer class will give ability to fetch the userinfo, cancel the transfer & to check whether transferring is happening. If you want to cancel any one of the userinfo transfer, then use below api to 
get list of current non-transfer userinfo which are stored in Outstanding userInfo tranfers Queue and cancel the transfer.

``` objc
// Will get list of WCSessionUserInfoTransfer objects.
NSArray<WCSessionUserInfoTransfer *> *transfers = [[WCSession defaultSession] outstandingUserInfoTransfers];
```

System will transfer one by one userinfo content to receiver app and will intimate to receiver by invoking below delegate method.

``` objc
// Receiver side delegate method to handle the userinfo.Will be called on startup if the user info finished transferring when the receiver was not running.
-(void)session:(nonnull WCSession *)session didReceiveUserInfo:(nonnull NSDictionary<NSString *,id> *)userInfo
{
    // Handle incoming user info dictionary
    dispatch_async(dispatch_get_main_queue(), ^{
        //Handle UI operations.
    });
}

// Called on sending side when a data transfer operation finished successfully or because of an error.
- (void)session:(WCSession * __nonnull)session didFinishUserInfoTransfer:(WCSessionUserInfoTransfer *)userInfoTransfer error:(nullable NSError *)error
{
}
```

**File Transfer**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; This is similar to userinfo transfer, instead of userinfo dictionary, you will be transfering files. For example consider you have an iPhone app, where you have list of photos, 
which you can edit & morph etc and you want to show the photos in watch app always updated. In this case you can use file transfer to transfer multiple image files with some metadata as well. All the files transfered 
will be queue up into Outstanding File Transfers queue and system will transfer them based on conditions and store them in **~/Documents/Inbox/** path at receiver side app. Based on file size, transfer may take time.

``` objc
// Sender side code to transfer file
-(void)transferUpdatedImage
{   
    NSURL *url = // Retrieve local stored URL of file
    NSDictionary *metadata = // Create dictionary of metadata
    WCSessionFileTransfer *fileTransfer = [[WCSession defaultSession] transferFile:url metadata:metadata];
}
```
If you want to cancel any one of the file transfer, then use below api to get list of current non-transfer files which are stored in Outstanding file tranfers Queue and cancel the transfer.

``` objc
// Will get list of WCSessionUserInfoTransfer objects.
NSArray<WCSessionFileTransfer *> *transfers = [[WCSession defaultSession] outstandingFileTransfers];
```
System will transfer one by one file to receiver app and will intimate to receiver by invoking below delegate method.

``` objc
// Called on the delegate of the receiver. Will be called on startup if the file finished transferring when the receiver was not running.
- (void)session:(WCSession *)session didReceiveFile:(WCSessionFile *)file
{
    // Handle incoming files
    dispatch_async(dispatch_get_main_queue(), ^{
        //Handle UI operations.
    });
}

// Called on the sending side after the file transfer has successfully completed or failed with an error. Will be called on next launch if the sender was not running when the transfer finished.
- (void)session:(WCSession *)session didFinishFileTransfer:(WCSessionFileTransfer *)fileTransfer error:(nullable NSError *)error
{
}
```
The incoming file will be located in the Documents/Inbox/ folder when being delivered. The receiver must take ownership of the file by moving it to another location. The system will remove any content that has not been
moved when this delegate method returns.

###**2) Interactive messaging**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; The interactive messaging is meant for live communciation, where both apps are running and communicate back & forth. Before sending message you need to check whether other app is available and required for messaging by using below api.

``` objc
[[WCSession defaultSession]reachable];
```
The reachable property will be true based on conditions differently in iPhone & watch like below.   
**Iphone:**  
1) Devices should connect   
2) watch app should in foreground. 

**Apple watch:**     
1) Devices should connect   
2) Even though you are invoking from watch app, there may be cases where watch extension will be in background. So, watch app should be in foreground.  
3) No need of iPhone app should run in foreground, system will automatically open the iPhone app in background.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; There are two types of interactive messaging. One is sending a dictionary as a message or sending data as a message. The data can be custom or serialized data.

``` objc
-(void)sendMessage
{
    if ([[WCSession defaultSession]reachable] == YES)
    {
        // Sending a dictionary as message

        NSDictionary *message = // Create dictionary of content
        [[WCSession defaultSession] sendMessage:message replyHandler:^(NSDictionary<NSString *,id> * __nonnull replyMessage) {
            // Handle reply
        } errorHandler:^(NSError * __nonnull error) {
            // Handle error
        }]


        // Sending a data as message

        NSData * data = // get your content data.
        [[WCSession defaultSession] sendMessageData:data replyHandler:^(NSData * __nonnull replyMessageData) {
            // Handle reply
        } errorHandler:^(NSError * __nonnull error) {
            // Handle error
        }
    }
}
```
Reply handler is optional. So, based on the replyhandler two different delegate methods will invoke at receiver side.

``` objc
// Delegate invoke, if reply handler is not passed.
-(void)session:(nonnull WCSession *)session didReceiveMessageData:(nonnull NSData *)messageData
{
    // Handle message
}

// Delegate invoke, if reply handler is  passed.
-(void)session:(nonnull WCSession *)session didReceiveMessageData:(nonnull NSData *)messageData replyHandler:(nonnull void (^)(NSData * __nonnull))replyHandler
{
    // Handle message, return reply
}
```

These are differnt ways of communication between iPhone and apple watch app. 