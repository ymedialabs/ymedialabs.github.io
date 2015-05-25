---
layout: post
title: "Possible abuse of dispatch_after"
date: 2015-05-25 17:36:08 +0530
comments: true
author: Mahesh Shanbhag
categories: iOS
---

I often come across code which uses dispatch_after GCD API to fire some process when an animation completes. I often feel this an abuse on using **dispatch_after** API. I would more like to use callbacks which notifies the completion of animation.

iOS provides these call backs via **CATransaction**. CATransaction groups several mono animation transaction into a atomic trasaction. All the changes to the rendering tree in the new run loop must be applied here.

Say there are two mono animations A (say translates a view in time t1) and B (say say scales a view in a time t2: t2>t1) and some process P needs to be fired after all the animation is complete i.e after time t2. Usually dispatch_after API is used to fire P after t2.

The usage of dispatch_after can be eliminate as follows:
```objc
    [CATransaction begin];
    [CATransaction setCompletionBlock:^{
        // call process P
    }];
    
    // Animation A;
    // Animation B;
    [CATransaction commit];
```

when animation A and B are performerd in between [CATransaction begin] and [CATransaction commit] calls, the animations are considered as an atomic change and are applied to the render tree the same time and the transaction completion block is called when the atomic animation completes. 

This way we can eliminate the usage of dispatch_after API reducing the overhead of changing the adjusting the time if the animation duration changes.
