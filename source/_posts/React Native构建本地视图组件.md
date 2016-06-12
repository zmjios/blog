---
title: React Native构建本地视图组件
date: 2015-05-10
desc: react native, 本地视图, 组件
---

在使用React Native开发App的过程中，我们可能需要调用RN没有实现的原生视图组件或第三方组件。甚至，我们可以把本地模块构造成一个React Native组件，提供给别人使用。由于我自己开发中遇到了这样的问题，于是通过查看源码和一些资料总结出了构建的一个流程。

<!-- more -->

如果是调用本地的Api，那么可以直接使用``RCTBridgeModule``进行访问，目前已经实现了对Swift的支持，详见文档([中文](http://wiki.jikexueyuan.com/project/react-native/native-modules.html)，[英文(较新)](http://facebook.github.io/react-native/docs/nativemodulesios.html#content))。我们这里讲的是如何进行本地视图组件的封装。下面进入正题：

要构建本地组件，我们要继承``RCTViewManager``这个类，以及使用JavaScript进行接口封装。为了不增加教程的篇幅，我们以简单的Swich组件为例。

首先，我们需要使用obj-c或者swift封装好组件的接口。

```obj-c
#import <UIKit/UIKit.h>
@interface RCTSwitch : UISwitch    //继承UISwitch
@property (nonatomic, assign) BOOL wasOn;
@end
```

然后实现

```obj-c
#import "RCTSwitch.h"
#import "RCTEventDispatcher.h"
#import "UIView+React.h"

@implementation RCTSwitch
- (void)setOn:(BOOL)on animated:(BOOL)animated {
  _wasOn = on;
  [super setOn:on animated:animated];
}
@end
```
### RCTSwitchManager.h

```obj-c
#import "RCTViewManager.h"
@interface RCTSwitchManager : RCTViewManager
@end

```
这是一个视图管理器的头文件，命名规范为 视图名称+Manager. 视图名称可以加上自己的前缀，这里最好避免使用RCT前缀，除非你想给官方pull request.

```obj-c
#import "RCTSwitchManager.h"    // 首先导入头文件
#import "RCTBridge.h"    //进行通信的头文件
#import "RCTEventDispatcher.h"    //事件派发，不导入会引起Xcode警告
#import "RCTSwitch.h"    //第三方组件的头文件
#import "UIView+React.h"    //若使用React封装的UIView(例如reactTag)
```
下面是实现的部分：

```obj-c
@implementation RCTSwitchManager

RCT_EXPORT_MODULE()

- (UIView *)view
{
  RCTSwitch *switcher = [[RCTSwitch alloc] init];
  [switcher addTarget:self
               action:@selector(onChange:)
     forControlEvents:UIControlEventValueChanged];
  return switcher;
}

- (void)onChange:(RCTSwitch *)sender
{
  if (sender.wasOn != sender.on) {
    [self.bridge.eventDispatcher sendInputEventWithName:@"topChange" body:@{
       @"target": sender.reactTag,
       @"value": @(sender.on)
     }];

    sender.wasOn = sender.on;
  }
}

RCT_EXPORT_VIEW_PROPERTY(onTintColor, UIColor);
RCT_EXPORT_VIEW_PROPERTY(tintColor, UIColor);
RCT_EXPORT_VIEW_PROPERTY(thumbTintColor, UIColor);
RCT_EXPORT_VIEW_PROPERTY(on, BOOL);
RCT_EXPORT_VIEW_PROPERTY(enabled, BOOL);

@end
```

这里首先调用``ECT_EXPORT_MODULE()``的宏，让模块接口暴露给JavaScript，然后我们必须定义``- (UIView *)view`` 这个方法来创建并返回组件视图，同时我们对视图事件进行监听。
紧接着我们实现了``- (void)onChange:(RCTSwitch *)sender``这个处理回调，``target``就是视图组件的一个实例，最后我们通过事件派发器的``sendInputEventWithName``方法来包装事件，其中"topChange"映射到UIManager中的``onChange``事件，也对应到我们组件的``onChange``属性详见**./React/Modules/RCTUIManager.m**文件。

```obj-c
@"topChange": @{
      @"phasedRegistrationNames": @{
        @"bubbled": @"onChange",
        @"captured": @"onChangeCapture"
      }
}
```
文件的最后的``RCT_EXPORT_VIEW_PROPERTY()``这个宏来设置该组件接受的参数及其类型。js中没有的类型，将会被自动类型转换。

下面使用JavaScript对本地组件进行封装，

```js
'use strict';

// 导入本地方法的封装
var NativeMethodsMixin = require('NativeMethodsMixin');
// React属性的类型系统
var PropTypes = require('ReactPropTypes');
// 导入React
var React = require('React');
// 视图属性
var ReactIOSViewAttributes = require('ReactIOSViewAttributes');
// 样式
var StyleSheet = require('StyleSheet');
// 使用该方法生成本地组件
var createReactIOSNativeComponentClass = require('createReactIOSNativeComponentClass');
// 使用该方法进行视图属性的合并
var merge = require('merge');
```

紧接着，我们通过``createReactIOSNativeComponentClass``创建RCTSwitch类。

```js
var RCTSwitch = createReactIOSNativeComponentClass({
  // 指定有效的属性
  validAttributes: rkSwitchAttributes,
  // 类名，与RCTSwitchManager相对应
  uiViewClassName: 'RCTSwitch',
});

// 暴露该组件的相关属性，使用merge方法合并了UIView中的所有属性
var rkSwitchAttributes = merge(ReactIOSViewAttributes.UIView, {
  onTintColor: true,
  tintColor: true,
  thumbTintColor: true,
  on: true,
  enabled: true,
});
```
``validAttributes``中的属性对应的值若类型为number, boolean, string类型时，设置为true即可。复杂的数据类型，我们应使用 differ 函数。

最后我们来创建组件类，这里先声明一些数据类型

```js
var SWITCH = 'switch';
type DefaultProps = {
  value: boolean;
  disabled: boolean;
};
type Event = Object;
```

下面是类方法的抽象
```js
var SwitchIOS = React.createClass({
  //导入本地方法
  mixins: [NativeMethodsMixin],
  //设置prop类型，保证组件能够被正确使用
  propTypes: {},
  //定义默认的props
  getDefaultProps(){},
  _onChange(){},
  render(){}
}
```
在类中应该进行``prop``的类型和默认值的声明，当使用不当时，将会出错警告，这也保证程序的健壮性，类型详见 [官方文档](https://facebook.github.io/react/docs/reusable-components.html)。下面是具体的实现：

```js
mixins: [NativeMethodsMixin],
  propTypes: {
    value: PropTypes.bool,
    disabled: PropTypes.bool,
    onValueChange: PropTypes.func,
    onTintColor: PropTypes.string,
    thumbTintColor: PropTypes.string,
    tintColor: PropTypes.string,
  },
  getDefaultProps: function(): DefaultProps {
    return {
      value: false,
      disabled: false,
    };
  },
```

```js
_onChange: function(event: Event) {
    //通过event.nativeEvent传递本地事件给视图组件，这些事件是我们前面定义在obj-c中的
    this.props.onChange && this.props.onChange(event);
    this.props.onValueChange && this.props.onValueChange(event.nativeEvent.value);

    // 保证属性的值确实被改变
    this.refs[SWITCH].setNativeProps({on: this.props.value});
  },

  render: function() {
    return (
      <RCTSwitch
        // 来自ReactIOSViewAttributes.UIView的属性
        ref={SWITCH}
        style={[styles.rkSwitch, this.props.style]}
        // 来自RCTSwitchManager.m中导出的属性
        enabled={!this.props.disabled}
        on={this.props.value}
        onTintColor={this.props.onTintColor}
        thumbTintColor={this.props.thumbTintColor}
        tintColor={this.props.tintColor}
        // RCTSwitchManager.m中导出的方法
        onChange={this._onChange}
      />
    );
  }
```

到此，我们就把RCTSwitch组件封装好了。如果要做成第三方组件，我们还需要把本地代码打包成静态库和xocdeproj文件。