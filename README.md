# Cavy Native Reporter

[![npm version](https://badge.fury.io/js/cavy-native-reporter.svg)](https://badge.fury.io/js/cavy-native-reporter)

A reporter for [Cavy], a React Native testing framework, that reports test
results to native Android or iOS test runners.

By default Cavy reports a completed test run to [cavy-cli][cli].
Cavy Native Reporter provides an alternative reporter for Cavy which fires
a Native Module callback when tests are finished. You can then wire this
in to a native test runner such as XCTest (examples below).

You may want to do this if you already have some application tests that are
native, e.g. if you already use XCTest to test parts of your app. This could
be because not all of your app is React Native, or if you app makes heavy
use of native code. You may also want to use it if you have an existing
CI pipeline set up for running a native test framework, and don't want to
adapt it for Cavy & cavy-cli.

You probably don't need this if your app is purely a React Native app
and you have no existing native tests. You can probably just use cavy-cli
instead.

## Installation
To get started using Cavy with native reporting, install using `yarn`:

    yarn add cavy cavy-native-reporter --dev

or `npm`:

    npm i --save-dev cavy cavy-native-reporter

Then link the package:

    react-native link cavy-native-reporter

## Usage

Check out [the sample app](https://github.com/pixielabs/cavy-native-reporter/tree/master/sampleApp)
for a full example of how you may want to integrate cavy-native-reporter into
your native testing setup.

#### 1. Set up Cavy

Follow the instructions in the [Cavy] README to set up a testing entry point
for your app and write your tests.

#### 2. Import and use cavy-native-reporter

Import the `CavyNativeReporter` from cavy-native-reporter. The module
`CavyNativeReporter` has a method `reporter`, that takes the test report
and calls a callback function that you can define in your native tester code.
See examples of how you might do this in [Reporting to Native Tests](#reporting-to-native-tests) below.

```js
import React, { Component } from 'react';
import { AppRegistry } from 'react-native';
import App from './app';
import { Tester, TestHookStore } from 'cavy';
import IntegrationSpecs from './specs/IntegrationSpecs';

// Extra import for cavy-native-reporter:
import CavyNativeReporter from 'cavy-native-reporter';

const testHookStore = new TestHookStore();

class TestableApp extends Component {
  render() {
    return (
      <Tester specs={IntegrationSpecs}
        store={testHookStore}
        // Extra prop for cavy-native reporter:
        reporter={CavyNativeReporter.reporter}>
        <App/>
      </Tester>
    );
  }
}

AppRegistry.registerComponent('App', () => TestableApp);
```

By default, Cavy sends a test report to [cavy-cli][cli]. Using
cavy-native-reporter overrides this functionality.

## Reporting to Native Tests

### iOS XCTest (Objective C)
##### `onFinishWithBlock`
Call `onFinishWithBlock` from within your native code and pass in a block
taking the single argument, `report`. Your block will be called as soon as the
test report is available from Cavy.

The `report` argument passed to the block is an `NSDictionary` with the
following structure:

```objc
{
  duration = "0.2";
  errorCount = 0;
  results = (
    {
      message = "Test suite description: test number 1";
      passed = 1;
    },
    {
      message = "Test suite description: test number 2";
      passed = 0;
    }
  );
}
```

If you need to, you can iterate over the test results and log more detailed
messages.

#### Example
To set up your own XCTestCase that makes use of `cavy-native-reporter`:
1. Open your project's `.xcodeproj` (or `.xcworkspace`) in Xcode.
2. In the Project navigator view, navigate to the folder containing your XCTest
test cases.
3. Create a new test case (select New File -> Unit Test Case Class).
4. Import `<CavyNativeReporter/CavyNativeReporter.h>`
5. Write your test!

Taking the sample app as an example, we have an XCTestCase `BridgeTest` which
waits for Cavy tests to run and fails if any test returns an error:

```objc
#import <XCTest/XCTest.h>
#import <CavyNativeReporter/CavyNativeReporter.h>

@interface BridgeTest : XCTestCase

@end

@implementation BridgeTest

- (void)testBridge {
  // Make a new expectation.
  XCTestExpectation *expectation = [[XCTestExpectation alloc] initWithDescription: @"Cavy tests passed"];

  [CavyNativeReporter onFinishWithBlock: ^void(NSDictionary* report) {
    // Pull the error count from the report object.
    long errorCount = [report[@"errorCount"] integerValue];
    // Fail if there are errors.
    if (errorCount > 0) {
      XCTFail(@"Cavy tests had one or more errors");
    }
    // Fulfill the expectation.
    [expectation fulfill];
  }];

  // Wait for expectation to fulfill.
  [self waitForExpectations:@[expectation] timeout:100];
}

@end
```

### iOS XCTest (Swift)
You could achieve the same thing in Swift with the following steps:

1. Follow the steps above for creating a new test case, but choose Swift when
prompted to choose a language.
2. Make sure that a Bridging Header file has also been created (Xcode will
usually prompt you to create one if this is your first Swift file in the
project).
3. Import `<CavyNativeReporter/CavyNativeReporter.h>` in your Bridging Header.
4. Write your test!

The following Swift code is equivalent to the Objective-C example above (note
that the method `onFinishWithBlock` is renamed when you reference it in Swift):

```swift
import XCTest

class BridgeTest: XCTestCase {
  func testBridge() {
    let expectation = XCTestExpectation(description: "Cavy tests passed");

    CavyNativeReporter.onFinish { report in
      NSLog("%@", report);
      let errorCount = report["errorCount"] as! Int;
      if (errorCount > 0) {
        XCTFail("Cavy tests had one or more errors");
      }
      expectation.fulfill();
    }

    wait(for: [expectation], timeout: 5);
  }
}
```

#### Useful links for iOS Testing
- [Writing XCTest classes](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/04-writing_tests.html)
- [Running tests](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/05-running_tests.html#//apple_ref/doc/uid/TP40014132-CH5-SW1)
- [Overview of testing with XCTest](https://www.objc.io/issues/15-testing/xctest/)

### Android
##### `waitForReport`
Call the `waitForReport` method from within your native code to wait for a test
report to be available from Cavy.

##### `cavyReport`
Call the static member variable `cavyReport` from within your native code to
access the test report from Cavy.

#### Example
To set up your own JUnit test that makes use of `cavy-native-reporter`:

1. Open your project's `android` folder in Android Studio.
2. Create a file for your instrumented Android tests at
`module-name/src/androidTest/java/`. Switching to Project view in Android
studio should help with this. [Follow this link for more detailed instructions on setting up Instumented Android tests](https://developer.android.com/studio/test#test_types_and_location) i.e. tests that run on an Android device
or emulator.
3. Add the following dependencies to `android/app/build.gradle` under
`dependencies` (don't touch the `build.gradle` in the app folder itself!):

```java
dependencies: {
  androidTestImplementation 'junit:junit:4.12'
  androidTestImplementation 'androidx.test:runner:1.1.0'
  androidTestImplementation 'androidx.test:rules:1.1.0'
  ...
}
```
4. Add the following to `android/app/build.gradle` under `defaultConfig`:

```java
defaultConfig {
  testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
  ...
}
```
5. Write your test!


Taking the sample app as an example, we have an JUnit test `BridgeTest` which
waits for Cavy tests to run and fails if any test returns an error:

```java
package com.sampleapp.bridgetest;

import androidx.test.rule.ActivityTestRule;

import org.junit.Rule;
import org.junit.Test;
import static org.junit.Assert.*;

import com.cavynativereporter.RNCavyNativeReporterModule;
// This should be the identifier for your own app's main activity.
import com.sampleapp.MainActivity;

public class BridgeTest {
  // This rule launches the main activity before each test annotated with @Test.
  @Rule
  public ActivityTestRule<MainActivity> activityRule = new ActivityTestRule(MainActivity.class);

  @Test
  public void testBridge() throws Exception {
    // Wait 60 seconds to receive a test report from Cavy.
    RNCavyNativeReporterModule.waitForReport(60);
    // Pull the error count from the report object.
    double errorCount = RNCavyNativeReporterModule.cavyReport.getDouble("errorCount");
    // Note: Third argument is the `delta` allowed between the actual and
    // expected double value.
    assertEquals(0.0, errorCount, 0.0);
  }
}

```

#### Useful links for Android Testing
- [Writing instrumented unit tests](https://developer.android.com/training/testing/unit-testing/instrumented-unit-tests)
- [Using JUnit4 Rules](https://developer.android.com/training/testing/junit-rules)
- [Using the JUnit Runner](https://developer.android.com/training/testing/junit-runner)

## Thank you!
Cavy Native Reporter was inspired by work done by [Nozbe](https://nozbe.com) on
[WatermelonDB](https://github.com/Nozbe/WatermelonDB), a high-performance
database framework for React that uses Cavy for running native integration
tests.

Thank you to [Radek](https://github.com/radex) in particular who really helped
get it off the ground :heart:.

## Contributing
Before contributing, please read the [code of conduct](CODE_OF_CONDUCT.md).
- Check out the latest master to make sure the feature hasn't been implemented
  or the bug hasn't been fixed yet.
- Check out the issue tracker to make sure someone already hasn't requested it
  and/or contributed it.
- Fork the project.
- Start a feature/bugfix branch.
- Commit and push until you are happy with your contribution.
- Please try not to mess with the package.json, version, or history. If you
  want to have your own version, or is otherwise necessary, that is fine, but
  please isolate to its own commit so we can cherry-pick around it.

[cavy]: https://github.com/pixielabs/cavy
[cli]: https://github.com/pixielabs/cavy-cli
