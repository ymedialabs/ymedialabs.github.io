---
layout: post
title: "How do we communicate between Watch and iPhone App ?"
date: 2015-06-30 13:11:39 +0530
comments: true
author: Karthik Mitta
categories: iOS
---
Apple Watch uses Wi-Fi and Bluetooth to communicate with your paired iPhone, switching between connections as needed:  

* Apple Watch uses Bluetooth when your iPhone is near, which conserves power.  
* If Bluetooth isn’t available, Apple Watch will try to use Wi-Fi. For example, if compatible Wi-Fi is available and your iPhone isn't in Bluetooth range,Apple Watch uses Wi-Fi.

<center><img src="/images/posts/2015-06-30/Distance-between-iPhone-and-Apple-Watch.jpg" align="center" width="400" height="200" /></center>

For pairing iPhone and apple watch, you need to enable Wi-Fi and Bluetooth
on your paired iPhone. Apple Watch compatible with Wi-Fi 802.11b/g and Bluetooth 4.0.

The communication process between both devices is entirely different in watchOS 2 compared from watchOS 1. In this article, I am going to explain in detail, that how communication works in both OS versions.

### <u>WatchOS 1:</u>

The basic communiation types are:  
1. Message transfer.    
2. Sharing Data or files with Your Containing iOS App.

### Message transfer:

It's a way to communicate directly from watch to iPhone app and get reply back, using **openParentApplication:** communication API in watch kit framework.

```objc
+(BOOL)openParentApplication:(NSDictionary * nonnull)userInfo reply:(void (^ nullable)(NSDictionary * nonnull replyInfo, NSError * nullable error))reply;
```

userInfo:  A dictionary containing data to pass to the iOS app. Use this dictionary to pass
information to the iOS app so that it can perform any needed tasks. The contents of the dictionary must be
serializable to a property list file. This parameter must not be nil.

reply:  It's a block with response data (NSDictionary) & error object for the request send by watch app.

This api belongs to **WKInterfaceController** class, as because every interface controller in watch extension may need to communicate parent iOS app at certain time. If iPhone app is not running, then this api call will wake up the app in background and do the needful.

The otherside in iPhone app, the **handleWatchKitExtensionRequest:** delegate method will get called in appDelegate.

```objc
- (void)application:(UIApplication *)application handleWatchKitExtensionRequest:(NSDictionary *)userInfo reply:(void(^)(NSDictionary *replyInfo))reply
```
#### Usage

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

By using above api, a watch app can communicate with iPhone app. But to communicate from iPhone to watch, there is no direct api provided by apple. For that you need to
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

like NSNotificationCenter you need to add & remove observer.

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

### Sharing Data or files with Your Containing iOS App :

In order to transfer data/files between watch & iPhone app, use a shared app group to store that data. An app group is a secure container that multiple processes can access. Because your WatchKit extension and iOS app run in separate sandbox environments, they normally do not share files or communicate directly with one another. An app group lets the two processes share files or user defaults.

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

After storing the data in shared container, use **openParentApplication:** api or **CFNotificationCenterGetDarwinNotifyCenter** to notify the receiver app for access data.

Find out the details of Communication between new WatchOS 2 and iOS here
[WacthOS2]({% post_url 2015-07-03-communication-in-apple-watchos-2 %})


