---
title: "Native tests for iOS expo modules"
date: 2024-01-09T19:00:00+01:00
authors: ["Mikołaj Wilczek"]
---

## Problem

You want to test the native code you’ve written for your React Native app using expo modules.

Custom native functions are great candidates for unit testing (if you still haven’t started writing them) as often they are separated from the view layer.

### Simple approach

The simplest solution is to use the [React Native Testing Library](https://callstack.github.io/react-native-testing-library/docs/getting-started) or other tools to test it directly from the React Native.

But this approach introduces some additional issues you need to consider:

- You test your native code with the whole communication bridge. I believe this is a good aspect, but introduces another points of failure.
- You need to create another bridging functions to thoroughly test your native code. Usually, it is a good practice to replace the chain of native calls with the one to reduce bridge overload, but in that case, you will either make a lot of bridge calls or introduce hard to test function.
- Sometimes it is a good idea to use the native testing stack to test. For iOS - you can utilise performance tests or snapshot testing.

### Alternative approach

First, we want to create a prebuilt [expo app](https://docs.expo.dev/get-started/installation/) with [the expo module](https://docs.expo.dev/modules/get-started/):

```bash
npx create-expo-app --template
npx expo prebuild
npx create-expo-module@latest --local
```

Additionaly, we need to open Xcode project (yourmodule.xcodeproj -> pay attention to open xcodeproj not xcworkspace). And add test target by: File -> New -> Target and then select `Unit Testing Bundle`.
Note changes made for `.xcodeproj` and `Podfile` will be removed after using `npx expo prebuild --clean`. Consider creating own template or post execution script to add them.

Then, we want to look at the `podspec` file. The most important change is adding `s.test_spec`, which points to the test direction as per https://guides.cocoapods.org/using/test-specs.html. Pay attention that test files and source files should not overlap, so it is best to keep them in separate folders. `.podspec` is the library configuration file for Cocoapods. In short, iOS has a dependency manager (and project generator) called Cocoapods, written in ruby. Apple already offers SwiftPM as a replacement, but due to other architecture, Cocoapods is still a popular option.

To add tests to your expo module, add this before last `end` (assuming you have tests files in `Test` folder):

```ruby
  s.test_spec '<Module Name>Tests' do |test_spec|
    test_spec.source_files = 'Tests/*.{h,m,swift}'
  end
```

So your file should like this (I've added Quick/Nimble dependencies for testing). It is good to use [Quick/Nimble](https://github.com/Quick/Quick) as it make tests more similar to JavaScript test frameworks:

```ruby
Pod::Spec.new do |s|
  s.name           = '<Module Name>’
  s.version        = '1.0.0'
  s.summary        = 'A sample project summary'
  s.description    = 'A sample project description'
  s.author         = ''
  s.homepage       = 'https://docs.expo.dev/modules/'
  s.platform       = :ios, '13.0'
  s.source         = { git: '' }
  s.static_framework = true

  s.dependency 'ExpoModulesCore'

  # Swift/Objective-C compatibility
  s.pod_target_xcconfig = {
    'DEFINES_MODULE' => 'YES',
    'SWIFT_COMPILATION_MODE' => 'wholemodule'
  }

  s.source_files = "*.{h,m,mm,swift,hpp,cpp}"

  s.test_spec '<Module Name>Tests' do |test_spec|
    test_spec.source_files = 'Tests/*.{h,m,swift}'
    test_spec.dependency 'Quick'
    test_spec.dependency 'Nimble'
  end
end
```

After we declare tests in our module, we want to link them to the main project to be available in the Xcode project.
To link tests, we need to open Podfile and replace:

```bash
use_expo_modules!
```

With:

```bash
pod 'ExpoModulesTestCore', :path => "../node_modules/expo-modules-test-core/ios"
use_expo_modules!({ includeTests: true })
```

We need `ExpoModulesTestCore` as it is declared as a test dependency in standard expo modules. We can install it by:

```bash
npx expo install expo-modules-test-core
```

This will import tests during Cocoapods installation (using `pod install`) and autocreate lib test targets to test. Then you can access tests by using the left pane in Xcode.

Happy testing!
