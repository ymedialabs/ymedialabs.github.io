---
layout: post
title: "App Thinning"
date: 2015-07-20 11:36:32 +0530
comments: true
author: Karthik Mitta
categories: iOS
---
Hi guys, I hope lot of people come across with this situation, where you will say OOPS...!!!!! My phone is running out of storage space. Let me delete some apps.

<center><img src="/images/posts/2015-07-20/iphone-storage-almost-full.png" align="center"/></center>

If you have a 16 GB iPhone or iPad, you quickly come to regret it, as app sizes become bigger. If you download a modern universal game, where it has 32 bit & 64 bit code, assets for various sized 
devices, full graphical content etc will take more space as a form of app bundle in your phone. This means your phone is storing bunch of data that you will never ever need.

Now all your worries are gone. Here apple come up a with brilliant solution for this, with a new concept App Thinning. Apple always tries to give a better experience for user and address the issues 
with new features.

Actually, apple has been doing this in all it's new software releases by reducing the package size. Previously storage space required by Mac OSX and xcode is huge, now if you observe apple reduced it 
to reasonable amount. Now, it extends same to application level as well.

### What is App Thinning ?

The App Store and operating system optimize the installation of iOS and watchOS apps by tailoring app delivery to the capabilities of the user’s particular device, with minimal footprint. This 
optimization, called app thinning, lets you create apps that use the most device features, occupy minimum disk space, and accommodate future updates that can be applied by Apple. Faster downloads and 
more space for other apps and content provides a better user experience.

Basically, app thinning is that it will exclude the data that you don't need from download. As every app you download will contain assets to support different devices. Consider iPhone 4 that cannot 
capable of some features, still app will download all the data. So, with app thinning your app size will reduce to minimal amount.This means that now apps will download quickly and will consume less 
storage space as required for that particular device.

App Thinning is the combination of slicing, bitcode, and on-demand resources

### Slicing:

Slicing is the process of creating and delivering variants of the app bundle for different target devices. A variant contains only the executable architecture and resources that are needed for the 
target device. You continue to develop and upload full versions of your app to iTunes Connect. The App Store will create and deliver different variants based on the devices your app supports. Image 
resources are sliced according to their resolution and device family. GPU resources are sliced according to device capabilities. When the user installs an app, a variant for the user’s device is 
downloaded and installed.


<center><img src="/images/posts/2015-07-20/app_thinning_2x.png" align="center"/></center>

### What developer need to do ?

* Must use assets catalog for all project resources in order to slice the resources by xcode while building.
* Specify target devices & provide multiple resolutions of images in asset catalog.
* Tag your resources for on demand resource download. 
* Test all variants on respected devices to discover configuration issues at early stage.

Xcode simulates slicing during development so you can create and test variants locally. Xcode slices your app when you build and run your app on a device. When you create an archive, Xcode includes the 
full version of your app but allows you to export variants from the archive.


<center><img src="/images/posts/2015-07-20/Xcode_Archive.png" align="center"/></center>

### Bitcode :

Bitcode refers to to the type of code: "LLVM Bitcode" that is sent to iTunes Connect. This allows Apple to use certain calculations to re-optimize apps further and downsize executable sizes. It will 
allow the store to add support for new CPU architecture introduced at some point down the road. Specifically, after an app has already been submitted for release.All of that, without having the 
developers to resubmit the application in direct response to new hardware being utilized by Apple.

<center><img src="/images/posts/2015-07-20/Xcode_Enable_BitCode.png" align="center"/></center>

Activating this setting indicates that the target should generate bitcode during compilation for platforms and architectures for which it supports. For archive builds bitcode will be generated in the 
linked binary for submission to the appstore. For other builds the compiler with the requirements for bitcode generation, but will not generate actual bitcode.

By default it's Yes for iOS app, but it is optional. For watch OS it is mandatory. 

### On Demand Resources:

These resources are like images & sound files, that you can tag with some keyboard and also you can create a group of assets with one tag. All these resources will store in apple servers and OS will 
manage downloads for you. It will enable faster downloads, smaller app sizes and improves the first launch experience of an app. After download, these resources will not store in app bundle or app 
container like documents directory. OS will store them in device memory and it purges the resources when they are no longer needed and disk space is low. For example, you have a game application with 
multiple levels. Each level need different assets & GPU code, then you can tag all of them and download them based on levels by using NSBundleResourceRequest class api's. OS will intelligently removes 
least priority on demand already downloaded resources and it will download new resources based on request by developer. You can add the tags in asset catalog like below

<center><img src="/images/posts/2015-07-20/Xcode_ODR_Tag.png" align="center"/></center>

If you export your app for testing or distribution outside of the appstore by using your own webservers, you must host the on-demand resources yourself. Note that executable on-demand resources are not 
supported.
Xcode build and run automatically thins resources for the active run destination. Supported for all simulators and device run destinations.

<center><img src="/images/posts/2015-07-20/Xcode_AutoSlice_Setting.png" align="center"/></center>

App thinning supports for all kind of distributions like App store, TestFlight, Ad-hoc/Enterprise & Xcode Server.For over the air installation of app from your own web servers, you need to consider one 
thing while archiving the build select the option include manifest file for over the air installation. Xcode generates manifest plist containing Urls for each app variant.

<center><img src="/images/posts/2015-07-20/Xcode_Manifest.png" align="center"/></center>

Please find more details about [ODR](https://developer.apple.com/library/prerelease/ios/documentation/FileManagement/Conceptual/On_Demand_Resources_Guide/index.html#//apple_ref/doc/uid/TP40015083-CH2-SW1) and [NSBundleResourceRequest](https://developer.apple.com/library/prerelease/ios/documentation/Foundation/Reference/NSBundleResourceRequest_Class/index.html#//apple_ref/doc/uid/TP40015084).

