---
id: backward-compatibility-fabric-components
title: Fabric Components as Native Components
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import constants from '@site/core/TabsConstants';
import BetaTS from './\_markdown_beta_ts_support.mdx';
import NewArchitectureWarning from '../\_markdown-new-architecture-warning.mdx';

<NewArchitectureWarning/>

:::info
The creation of a backward compatible Fabric Component requires the knowledge of how to create a Fabric Component. To recall these concepts, have a look at this [guide](pillars-fabric-components).

Fabric Components only work when the New Architecture is properly setup. If you already have a library that you want to migrate to the New Architecture, have a look at the [migration guide](../new-architecture-intro) as well.
:::

Creating a backward compatible Fabric Component lets your users continue leverage your library, independently from the architecture they use. The creation of such a component requires a few steps:

1. Configure the library so that dependencies are prepared set up properly for both the Old and the New Architecture.
1. Update the codebase so that the New Architecture types are not compiled when not available.
1. Uniform the JavaScript API so that your user code won't need changes.

<BetaTS />

While the last step is the same for all the platforms, the first two steps are different for iOS and Android.

## Configure the Fabric Component Dependencies

### <a name="dependencies-ios" />iOS

The Apple platform installs Fabric Components using [Cocoapods](https://cocoapods.org) as dependency manager.

Every Fabric Component defines a `podspec` that looks like this:

```ruby
require "json"

package = JSON.parse(File.read(File.join(__dir__, "package.json")))

folly_version = '2021.07.22.00'
folly_compiler_flags = '-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1 -Wno-comma -Wno-shorten-64-to-32'

Pod::Spec.new do |s|
  # Default fields for a valid podspec
  s.name            = "<FC Name>"
  s.version         = package["version"]
  s.summary         = package["description"]
  s.description     = package["description"]
  s.homepage        = package["homepage"]
  s.license         = package["license"]
  s.platforms       = { :ios => "11.0" }
  s.author          = package["author"]
  s.source          = { :git => package["repository"], :tag => "#{s.version}" }

  s.source_files    = "ios/**/*.{h,m,mm,swift}"
  # React Native Core dependency
  s.dependency "React-Core"

  # The following lines are required by the New Architecture.
  s.compiler_flags = folly_compiler_flags + " -DRCT_NEW_ARCH_ENABLED=1"
  s.pod_target_xcconfig    = {
      "HEADER_SEARCH_PATHS" => "\"$(PODS_ROOT)/boost\"",
      "OTHER_CPLUSPLUSFLAGS" => "-DFOLLY_NO_CONFIG -DFOLLY_MOBILE=1 -DFOLLY_USE_LIBCPP=1",
      "CLANG_CXX_LANGUAGE_STANDARD" => "c++17"
  }

  s.dependency "React-RCTFabric"
  s.dependency "React-Codegen"
  s.dependency "RCT-Folly", folly_version
  s.dependency "RCTRequired"
  s.dependency "RCTTypeSafety"
  s.dependency "ReactCommon/turbomodule/core"
end
```

The **goal** is to avoid installing the dependencies when the app is prepared for the Old Architecture.

When we want to install the dependencies, we use the following commands depending on the architecture:

```sh
# For the Old Architecture, we use:
pod install

# For the New Architecture, we use:
RCT_NEW_ARCH_ENABLED=1 pod install
```

Therefore, we can leverage this environment variable in the `podspec` to exclude the settings and the dependencies that are related to the New Architecture:

```diff
+ if ENV['RCT_NEW_ARCH_ENABLED'] == '1' then
    # The following lines are required by the New Architecture.
    s.compiler_flags = folly_compiler_flags + " -DRCT_NEW_ARCH_ENABLED=1"
    # ... other dependencies ...
    s.dependency "ReactCommon/turbomodule/core"
+ end
end
```

This `if` guard prevents the dependencies from being installed when the environment variable is not set.

### Android

To create a module that can work with both architectures, you need to configure Gradle to choose which files need to be compiled depending on the chosen architecture. This can be achieved by using **different source sets** in the Gradle configuration.

:::note
Please note that this is currently the suggested approach. While it might lead to some code duplication, it will ensure the maximum compatibility with both architectures. You will see how to reduce the duplication in the next section.
:::

To configure the Fabric Component so that it picks the proper sourceset, you have to update the `build.gradle` file in the following way:

```diff title="build.gradle"
+// Add this function in case you don't have it already
+ def isNewArchitectureEnabled() {
+    return project.hasProperty("newArchEnabled") && project.newArchEnabled == "true"
+}
// ... other parts of the build file
defaultConfig {
        minSdkVersion safeExtGet('minSdkVersion', 21)
        targetSdkVersion safeExtGet('targetSdkVersion', 31)
+        buildConfigField("boolean", "IS_NEW_ARCHITECTURE_ENABLED", isNewArchitectureEnabled().toString())
+    }
+
+    sourceSets {
+        main {
+            if (isNewArchitectureEnabled()) {
+                java.srcDirs += ['src/newarch']
+            } else {
+                java.srcDirs += ['src/oldarch']
+            }
+        }
    }
}
```

This changes do three main things:

1. The first lines define a function that returns whether the New Architecture is enabled or not.
2. The `buildConfigField` line defines a build configuration boolean field called `IS_NEW_ARCHITECTURE_ENABLED`, and initialize it using the function declared in the first step. This allows you to check at runtime if a user has specified the `newArchEnabled` property or not.
3. The last lines leverage the function declared in step one to decide which source sets we need to build, depending on the choosen architecture.

## Update the codebase

### iOS

The second step is to instruct Xcode to avoid compiling all the lines using the New Architecture types and files when we are building an app with the Old Architecture.

A Fabric Component requires an header file and an implementation file to add the actual `View` to the module.

For example, the `RNMyComponentView.h` header file could look like this:

```objective-c
#import <React/RCTViewComponentView.h>
#import <UIKit/UIKit.h>

#ifndef NativeComponentExampleComponentView_h
#define NativeComponentExampleComponentView_h

NS_ASSUME_NONNULL_BEGIN

@interface RNMyComponentView : RCTViewComponentView
@end

NS_ASSUME_NONNULL_END

#endif /* NativeComponentExampleComponentView_h */
```

The implementation `RNMyComponentView.mm` file, instead, could look like this:

```objective-c
#import "RNMyComponentView.h"

// <react/renderer imports>

#import "RCTFabricComponentsPlugins.h"

using namespace facebook::react;

@interface RNMyComponentView () <RCTMyComponentViewViewProtocol>

@end

@implementation RNMyComponentView {
    UIView * _view;
}

+ (ComponentDescriptorProvider)componentDescriptorProvider
{
    // ... return the descriptor ...
}

- (instancetype)initWithFrame:(CGRect)frame
{
  // ... initialize the object ...
}

- (void)updateProps:(Props::Shared const &)props oldProps:(Props::Shared const &)oldProps
{
  // ... set up the props ...

  [super updateProps:props oldProps:oldProps];
}

Class<RCTComponentViewProtocol> MyComponentViewCls(void)
{
  return RNMyComponentView.class;
}

@end
```

To make sure that Xcode skips these files, we can wrap **both** of them in some `#ifdef RCT_NEW_ARCH_ENABLED` compilation pragma. For example, the header file could change as follows:

```diff
+ #ifdef RCT_NEW_ARCH_ENABLED
#import <React/RCTViewComponentView.h>
#import <UIKit/UIKit.h>

// ... rest of the header file ...

#endif /* NativeComponentExampleComponentView_h */
+ #endif
```

The same two lines should be added in the implementation file, as first and last lines.

The above snippet uses the same `RCT_NEW_ARCH_ENABLED` flag used in the previous [section](#dependencies-ios). When this flag is not set, Xcode skips the lines within the `#ifdef` during compilation and it does not include them into the compiled binary. The compiled binary will have a the `RNMyComponentView.o` object but it will be an empty object.

### Android

As we can't use conditional compilation blocks on Android, we will define two different source sets. This will allow to create a backward compatible TurboModule with the proper source that is loaded and compiled depending on the used architecture.

Therefore, you have to:

1. Create a Native Component in the `src/oldarch` path. See [this guide](../native-components-android) to learn how to create a Native Component.
2. Create a Fabric Component in the `src/newarch` path. See [this guide](pillars-fabric-components) to learn how to create a Fabric Component.

and then instruct Gradle to decide which implementation to pick.

Some files can be shared between a Native and a Fabric Component: these should be created or moved into a folder that is loaded by both the architectures. These files are:

- the `<MyComponentView>.java` that instantiate and configure the Android View for both the components.
- the `<MyComponentView>ManagerImpl.java` file where which contains the logic of the ViewManager that can be shared between the Native and the Fabric Component.
- the `<MyComponentView>Package.java` file used to load the component.

The final folder structure looks like this:

```sh
my-component
├── android
│   ├── build.gradle
│   └── src
│       ├── main
│       │   ├── AndroidManifest.xml
│       │   └── java
│       │       └── com
│       │           └── MyComponent
│       │               ├── MyComponentView.java
│       │               ├── MyComponentViewManagerImpl.java
│       │               └── MyComponentViewPackage.java
│       ├── newarch
│       │   └── java
│       │       └── com
│       │           └── MyComponentViewManager.java
│       └── oldarch
│           └── java
│               └── com
│                   └── MyComponentViewManager.java
├── ios
├── js
└── package.json
```

The code that should go in the `MyComponentViewManagerImpl.java` and that can be shared between the Native Component and the Fabric Component is, for example:

```java title="example of MyComponentViewManager.java"
package com.MyComponent;
import androidx.annotation.Nullable;
import com.facebook.react.uimanager.ThemedReactContext;

public class MyComponentViewManagerImpl {

    public static final String NAME = "MyComponent";

    public static MyComponentView createViewInstance(ThemedReactContext context) {
        return new MyComponentView(context);
    }

    public static void setFoo(MyComponentView view, String param) {
        // implement the logic of the foo function using the view and the param passed.
    }
}
```

Then, the Native Component and the Fabric Component can be updated using the function declared in the shared manager.

For example, for a Native Component:

```java title="Native Component using the ViewManagerImpl"
public class MyComponentViewManager extends SimpleViewManager<MyComponentView> {

    ReactApplicationContext mCallerContext;

    public MyComponentViewManager(ReactApplicationContext reactContext) {
        mCallerContext = reactContext;
    }

    @Override
    public String getName() {
        // static NAME property from the shared implementation
        return MyComponentViewManagerImpl.NAME;
    }

    @Override
    public MyComponentView createViewInstance(ThemedReactContext context) {
        // static createViewInstance function from the shared implementation
        return MyComponentViewManagerImpl.createViewInstance(context);
    }

    @ReactProp(name = "foo")
    public void setFoo(MyComponentView view, String param) {
        // static custom function from the shared implementation
        MyComponentViewManagerImpl.setFoo(view, param);
    }

}
```

And, for a Fabric Component:

```java title="Fabric Component using the ViewManagerImpl"
// Use the static NAME property from the shared implementation
@ReactModule(name = MyComponentViewManagerImpl.NAME)
public class MyComponentViewManager extends SimpleViewManager<MyComponentView>
        implements MyComponentViewManagerInterface<MyComponentView> {

    private final ViewManagerDelegate<MyComponentView> mDelegate;

    public MyComponentViewManager(ReactApplicationContext context) {
        mDelegate = new MyComponentViewManagerDelegate<>(this);
    }

    @Nullable
    @Override
    protected ViewManagerDelegate<MyComponentView> getDelegate() {
        return mDelegate;
    }

    @NonNull
    @Override
    public String getName() {
        // static NAME property from the shared implementation
        return MyComponentViewManagerImpl.NAME;
    }

    @NonNull
    @Override
    protected MyComponentView createViewInstance(@NonNull ThemedReactContext context) {
        // static createViewInstance function from the shared implementation
        return MyComponentViewManagerImpl.createViewInstance(context);
    }

    @Override
    @ReactProp(name = "foo")
    public void setFoo(MyComponentView view, @Nullable String param) {
        // static custom function from the shared implementation
        MyComponentViewManagerImpl.setFoo(view, param]);
    }
}
```

For a step-by-step example on how to achieve this, have a look at [this repo](https://github.com/react-native-community/RNNewArchitectureLibraries/tree/feat/back-fabric-comp).

## Unify the JavaScript specs

<BetaTS />

The last step makes sure that the JavaScript behaves transparently to chosen architecture.

For a Fabric Component, the source of truth is the `<YourModule>NativeComponent.js` (or `.ts`) spec file. The app accesses the spec file like this:

```ts
import MyComponent from 'your-component/src/index';
```

The **goal** is to conditionally `export` from the `index` file the proper object, given the architecture chosen by the user. We can achieve this with a code that looks like this:

<Tabs groupId="fabric-component-backward-compatibility"
      defaultValue={constants.defaultFabricComponentSpecLanguage}
      values={constants.fabricComponentSpecLanguages}>
<TabItem value="Flow">

```ts
// @flow
import { requireNativeComponent } from 'react-native';

const isFabricEnabled = global.nativeFabricUIManager != null;

const myComponent = isFabricEnabled
  ? require('./MyComponentNativeComponent').default
  : requireNativeComponent('MyComponent');

export default myComponent;
```

</TabItem>
<TabItem value="TypeScript">

```ts
import requireNativeComponent from 'react-native/Libraries/ReactNative/requireNativeComponent';

const isFabricEnabled = global.nativeFabricUIManager != null;

const myComponent = isFabricEnabled
  ? require('./MyComponentNativeComponent').default
  : requireNativeComponent('MyComponent');

export default myComponent;
```

</TabItem>
</Tabs>

Whether you are using Flow or TypeScript for your specs, we understand which architecture is running by checking if the `global.nativeFabricUIManager` object has been set or not.

:::caution
Please note that the New Architecture is still experimental. The `global.nativeFabricUIManager` API might change in the future for a function that encapsulate this check.
:::

- If that object is `null`, the app has not enabled the Fabric feature. It's running on the Old Architecture, and the fallback is to use the default Native Components implementation ([iOS](../native-components-ios) or [Android](../native-components-android)).
- If that object is set, the app is running with Fabric enabled and it should use the `<MyComponent>NativeComponent` spec to access the Fabric Component.
