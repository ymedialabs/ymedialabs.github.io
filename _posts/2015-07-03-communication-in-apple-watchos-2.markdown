---
layout: post
title: "Communication between Apple Watch with new WatchOS-2 and iPhone"
date: 2015-07-03 10:46:36 +0530
comments: true
author: Karthik Mitta
categories: iOS
---
In Watch OS 2, apple moved the watch extension bundle & data source to watch itself, previously it was in iPhone only. No more **openParentApplication:** api call, **CFNotificationCenterGetDarwinNotifyCenter** & **sharedGroups**. Apple itself given new framework for all sorts of communication named WatchConnectivity, supports for NSURLSession & codedata.


Before using WatchConnectivity framework, you need some setup. This will ensure that device and Watch OS are supported to use WatchConnectivity api's. It's worth to setup this at the earlier stage of your app like in iOS app appDelegate & watch app extensionDelegate class, as becuase receiver app may need some data at early stage.

``` objc
// Always set your apps up to receive incoming WatchConnectivity content
if ([WCSession isSupported])
{
self.session = [WCSession defaultSession];
self.session.delegate = self; // conforms to WCSessionDelegate
[self.session activateSession];
}
```
After activating the watchconnectivy session, you need to check whether your iPhone is paired with any watch device or not.
``` objc
self.session.paired == false;
```
At first it will be false, as watch is not paired. Once iPhone pair with any watch device, iPhone app will get notify about the state change of pair by calling **sessionWatchStateDidChange:** delegare method, if iPhone app is running.

``` objc
- (void)sessionWatchStateDidChange:(WCSession *)session;
```
next step is watch app installation validation. If user install the watch app later after pairing, then iPhone app will get notify about the installion state change by invoking a above same delegare method, if iPhone app is running.

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
These validations are only for iPhone app, because doing this from watch app means it's already paired and installed. When ever watchAppInstalled become true, apple will create a directory in your container, few potins need to consider before storing some data in that container:

* Directory and its contents’ lifetime is tied to the watchAppInstalled property, it means when ever watch app uninstalled all content will get delete.
* As apple will clear all of your data while uninstalling the app or unpair the device. Store only the data relevant to the specific instance of your Watch app like 
1. Last queued item marker
2. Preferences
3. Files queued for transfer    

you can get that container path from the api.

```objc
self.session.watchDirectoryURL
```
Now you are good enough to communicate between both apps. Apple divided the communication into two categories:

* Background transfers
* Interactive messaging

### 1) Background transfers

This meant for content not needed immediately at receiver side, OS intelligently transfer the content. If
container app contains some content and fetching some more content from server & receiver app is not in active
state, that means receiver not needed the content immediately. So, it will queue up content in system. And 
system will validate the right conditions intelligently to transfer the content across, like it will consider
power, performance & app state etc. System will transfer content, if conditions satisfied and then it will
delivered to receiver app when it launches.It divided into three types:     
1. Application context  
2. Userinfo transfer    
3. file transfer

#### Application context

This will offer a single set of information to other side app. For example take FoodPanda iPhone application, which shows the restaurants based on your current of choosed
location. If you want to show the set of restaurants in apple watch as well. Then store them in application context, when ever user launches the watch app the content will be there to show. There are two main
properties **applicationContext** & **receivedApplicationContext** which stores data(dictionary) in sender & receiver apps respectively.

FoodPanda iPhone app will call an api **updateApplicationContext:**, which will store the data in **applicationContext** property, if iPhone app pushing latest context before
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

#### Userinfo transfer

This is similar to application context, but the only difference is, it will transfer multiple set of content by queue up them. It is very usefull to do some userinfo inmemory data transfer. For example, take some of the best fitness & health related apps like Nike+ Running, Hello heart & Map my run. These apps will collect some data related to user from watch and show the progress in iPhone app by graphical representation. In this case you can use userinfoTransfer to send multiple set of userinfo data like running time, heart beat data, walking time etc to iPhone app.

Use **transferUserInfo:** to send userinfo, WCSession will store it in Outstanding userInfo tranfers Queue. This api will return an object of **WCSessionUserInfoTransfer** class.

``` objc
// Sender side code
-(void)sendTheUserinfo
{
NSDictionary *userInfo = @{}; // Create dictionary of userInfo
WCSessionUserInfoTransfer *userInfoTransfer = [[WCSession defaultSession] transferUserInfo:userInfo];
}
```
WCSessionUserInfoTransfer class will give ability to fetch the userinfo, cancel the transfer & to check whether transferring is happening. If you want to cancel any one of the userinfo transfer, then use 
**outstandingUserInfoTransfers** api to get list of current non-transfer userinfo's from queue and cancel the specific transfer.

``` objc
// Will get list of WCSessionUserInfoTransfer objects.
NSArray<WCSessionUserInfoTransfer *> *transfers = [[WCSession defaultSession] outstandingUserInfoTransfers];
```

System will transfer one by one userinfo content to receiver app and will intimate to receiver by calling **didReceiveUserInfo** delegate method.

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

#### File Transfer

This is similar to userinfo transfer, instead of userinfo dictionary, you will be transfering files. For example consider you have an iPhone app, where you have list of photos, 
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
If you want to cancel any one of the file transfer, then use **outstandingFileTransfers** api to get list of current non-transfer files from ueue and cancel the transfer.

``` objc
// Will get list of WCSessionUserInfoTransfer objects.
NSArray<WCSessionFileTransfer *> *transfers = [[WCSession defaultSession] outstandingFileTransfers];
```
System will transfer one by one file to receiver app and will intimate to receiver by calling **didReceiveFile** delegate method.

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

### 2) Interactive messaging

The interactive messaging is meant for live communciation, where both apps are running and communicate back & forth. Before sending message you need to check whether other app is available and required for messaging by using the api.

``` objc
[[WCSession defaultSession]reachable];
```
The reachable property will be true based on different conditions.   
**Iphone:**  
1) Devices should connect   
2) watch app should in foreground. 

**Apple watch:**     
1) Devices should connect   
2) Even though you are invoking from watch app, there may be cases where watch extension will be in background. So, watch app should be in foreground.  
3) No need of iPhone app should run in foreground, system will automatically open the iPhone app in background.

There are two types of interactive messaging. One is sending a dictionary as a message or sending data as a message. The data can be custom or serialized data.

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
These are the different ways of communication between iPhone and apple watch app.

Find out the details of Communication between WatchOS 1 and iOS here
[WacthOS1]({% post_url 2015-06-30-apple-watch-connectivity %})

