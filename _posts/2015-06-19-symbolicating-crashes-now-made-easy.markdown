---
layout: post
title: "Symbolicating crashes now made easy"
date: 2015-06-19 09:30:15 +0530
comments: true
author: Mahesh Shanbhag
categories: iOS
---

###About dSYM###
If you have issues with symbolicating crashes in iOS by dragging the .crash file into the organiser tab, there is a `symbolicatecrash.pl` script provide by Xcode as well as `atos` tool to symbolicate the crash and a memory reference respectively. The script allows us to symbolicate the whole crash file where as using atos we can symbolicate single memory reference.

Just for posterity when Xcode is building the application lldb debugger copies the symbols and the memory print to .dSYM debug symbol file. This file is created every time the application is built and no two .dSYM files are the same. This is the reason the crash is symbolicated on the machine on which the version of the application was built.

###Using dSYM for symbolicating crashes###
To symbolicate a crash file you need the original .dSYM file for that build. Xcode comes with a build in perl script to symbolicate the crash file using the .dSYM file.To do this we need to open terminal, set up an env variable DEVELOPER_DIR and add an alias to 'symbolicatecrash.pl' script.

### symbolication project (A Xcode plugin) ###
Symbolication project enables *symbolication-plugin* an Xcode plugin. This plugin is available in Product menu and is named `Symbolicate`. This is now available on [YML Github Account](https://github.com/ymedialabs/symbolication-plugin).

* Symbolicate Plugin is used to symbolicate crashes. If `dSYM file` and  `crash file` is available, the crash can be symbolicated using the Plugin. 
* 1.0 (0.1)

![Product Menu](https://raw.githubusercontent.com/ymedialabs/symbolication-plugin/master/screenshots/product_menu.png "Produce Menu")

### How do I get set up? ##
* Open the SymbolicationPlugin project
* cmd + k (clean the project)
* cmd + b (build the project)
* ta-da! the plugin is installed

### How to check if the plugin is installed
* Open Terminal
* Navigate to ~/Library/Application Support/Developer/Shared/Xcode/Plug-ins directory
* Type ls -l
* `SymbolicationPlugin.xcplugin` should be listed in the installed plug-in's list.

### Requirements
To use the complete features of the Plugin the following files are required.

* The application bundle (application.app file).
* The dSYM file associated with the build.
* The application unix executable file for the available inside the application.app bundle (application.app/application).

### How to use the Plugin?
Below are the details of using the three sections of the plugin `Symbolicate`, `Details` and `Memory`

#### Symbolicating crash file
Use the `Symbolicate` tab to symbolicate the crash log.

* Select the dSYM file from the disk.
* Select the the crash file from the disk.
* Select Symbolicate. The plugin begins symbolicating the crash.

Note: Additionally you can save the crash file by clicking at the down arrow at the bottom left of the screen.

![Symbolicate](https://raw.githubusercontent.com/ymedialabs/symbolication-plugin/master/screenshots/Symbolicate_screen.png "Symbolicate Section")

#### Checking the application details
Used the `Details` tab to get the build information.

* Select the application executable file (Unix executable file) available inside the application.app bundle (application.app/application).
* The details of the application like the build UUID, the build architecture is displayed.

![Details](https://raw.githubusercontent.com/ymedialabs/symbolication-plugin/master/screenshots/Details_screen.png "Details Section")

#### Symbolicating memory references available in the crash file 
Used the `Memory` tab to symbolicate memory references

* Select the application executable file (Unix executable file) available inside the application.app bundle (application.app/application).
* List down the memory addresses a single space saperated list.
* Select the architecture (This can be found using the above 'Details' section).
* Select Symbolicate. The Plugin displayes the symbolicated memory reference.

![Memory](https://raw.githubusercontent.com/ymedialabs/symbolication-plugin/master/screenshots/Memory_screen.png "Memory Section")

This is now also available on [Alcatraz Package Manager](http://alcatraz.io) for Xcode. 

