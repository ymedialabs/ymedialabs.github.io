---
layout: post
author: Mahesh Shanbhag
title: "Shaping up with CAShapeLayer"
date: 2015-05-14 12:19:13 +0530
comments: true
categories: iOS
---

What out stands iOS mobile platform from others is the power of animations. For animations iOS provides several classes with animatable properties that makes animations look easier. However some animations like the *Apple's application circular loader*, there are no special classes that readily available for animation.

This can be made possible with a special class **CAShapeLayer**. CAShapeLayer shapes itself to the path provided to it through the *path* property. The *path* property is a *CGPath* instance. We can use UIBezierPath API's to get the required shape and retrive the CGPaths.   

Lets get started to design out custom **Apple application circular loader** using key frame animations

**Create a CAShapeLayer**
```objc
self.animatinglayer = [CAShapeLayer layer];
self.animatingLayer.strokeColor = <#stroke color#>;
self.animatingLayer.fillColor = <#fill color#>;
self.animatingLayer.lineWidth = <#line width#>;
self.animatingLayer.frame = self.bounds;
[self.layer addSublayer:self.animatingLayer];

```

**Create a Initial BezierPath and assign it to the above layer above**
```objc
UIBezierPath *initialPath = [UIBezierPath bezierPath]; //empty path

// get the radius of the inner path
CGFloat radius = MIN(CGRectGetWidth(self.bounds), CGRectGetHeight(self.bounds))/2;

// add the arc
[initialPath addArcWithCenter:center radius:radius startAngle:degreeToRadian(-90) endAngle:degreeToRadian(-90) clockwise:YES];

// set the path
self.animatinglayer.path = initialPath.CGPath;
```

once we have the shape and the initial path all we need are the array of paths for key frame animation

**Create a Initial BezierPath and assign it to the above layer above**

```objc
- (NSArray *)keyframePathsWithDuration:(CGFloat)duration lastUpdatedAngle:(CGFloat)lastUpdatedAngle newAngle:(CGFloat)newAngle radius:(CGFloat)radius type:(RMIndicatorType)type
{
    NSUInteger frameCount = ceil(duration * 60);
    NSMutableArray *array = [NSMutableArray arrayWithCapacity:frameCount + 1];
    for (int frame = 0; frame <= frameCount; frame++)
    {
        CGFloat startAngle = degreeToRadian(-90);
        CGFloat endAngle = lastUpdatedAngle + (((newAngle - lastUpdatedAngle) * frame) / frameCount);
        
        [array addObject:(id)([self pathWithStartAngle:startAngle endAngle:endAngle radius:radius type:type].CGPath)];
    }
    
    return [NSArray arrayWithArray:array];
}

// Method to return the bezier path with required start end angle and radius 
- (UIBezierPath *)pathWithStartAngle:(CGFloat)startAngle endAngle:(CGFloat)endAngle radius:(CGFloat)radius type:(RMIndicatorType)type
{
    BOOL clockwise = startAngle < endAngle;
    
    UIBezierPath *path = [UIBezierPath bezierPath];
    CGPoint center = CGPointMake(self.bounds.size.width / 2, self.bounds.size.height / 2);
    
    if(type == kRMClosedIndicator)
    {
        [path addArcWithCenter:center radius:radius startAngle:startAngle endAngle:endAngle clockwise:clockwise];
    }
    else
    {
        [path moveToPoint:center];
        [path addArcWithCenter:center radius:radius startAngle:startAngle endAngle:endAngle clockwise:clockwise];
        [path closePath];
    }
    return path;
}

// Function to convert degree to radians
float degreeToRadian(float degree)
{
    return ((degree * M_PI)/180.0f);
}

```

once we have the array of paths, we can perform the animation
```objc
 CAKeyframeAnimation *pathAnimation = [CAKeyframeAnimation animationWithKeyPath:@"path"];
    [pathAnimation setValues:self.paths];
    [pathAnimation setDuration:self.animationDuration];
    [pathAnimation setTimingFunction:[CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseOut]];
    [pathAnimation setRemovedOnCompletion:YES];
    [self.animatingLayer addAnimation:pathAnimation forKey:@"path"];
```



ta-da that is all need to create Apple's loader animation. The complete source code is available [here on github]. 

{% img /assets/images/posts/2015-05-14/RMDownload_Indicator.gif "Animation Gif" %}


[here on github]:https://github.com/MaheshRS/Download-Indicator
