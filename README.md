# ![AppSpector](https://github.com/appspector/ios-sdk/raw/master/appspector-logo.png)

With AppSpector you can remotely debug your app running in the same room or on another continent. 
You can measure app performance, view database content, logs, network requests and many more in realtime. 
This is the instrument that you've been looking for. Don't limit yourself only to simple logs. 
Debugging don't have to be painful!

[![GitHub release](https://img.shields.io/github/release/appspector/ios-sdk.svg)](https://github.com/appspector/ios-sdk)

## Installation
Each app you want to use with AppSpector SDK you have to register on the web (https://app.appspector.com).
After adding the application navigate to app settings and copy API key.

#### Manually
<!-- integration-manual-start -->
To manually link just download AppSpectorSDK.zip, extract it and drop AppSpectorSDK.framework to your XCode project.
Then navigate to your project settings and under 'General' tab add AppSpectorSDK framework to 'Embedded Binaries' section.

If you plan either to submit builds with AppSpector SDK to the Apple TestFlight for testing or archive them for AdHoc distribution you'll have to perform one more step: create a new “Run Script Phase” in your app’s target’s “Build Phases” and paste the following script:

```
file="AppSpectorSDK.framework/AppSpectorSDK"
archs="$(lipo -info "${file}" | rev | cut -d ':' -f1 | rev)"
stripped=""
for arch in $archs; do
    if ! [[ "${VALID_ARCHS}" == *"$arch"* ]]; then
        lipo -remove "$arch" -output "$file" "$file" || exit 1
        stripped="$stripped $arch"
    fi
done
echo "AppSPector: stripped archs: $stripped"

```

This script is required as a workaround for this [Apple AppStore bug](http://www.openradar.me/radar?id=6409498411401216)


<!-- integration-manual-end -->

#### CocoaPods
<!-- integration-pods-start -->
To use cocoapods add this line to your podfile and run `pod install`:

```
pod 'AppSpectorSDK'
```
<!-- integration-pods-end -->

#### Carthage
<!-- integration-carthage-start -->
- Install [Carthage](https://github.com/Carthage/Carthage#installing-carthage)
- Add github "appspector/ios-sdk" to your Cartfile
- Run `carthage update`
- Drag AppSpectorSDK.framework from the appropriate platform directory in Carthage/Build/ to the “Linked Frameworks and Libraries” section of your Xcode project’s “General” settings
<!-- integration-carthage-end -->

[Join our slack to discuss setup process and features](https://slack.appspector.com)

## Configure
AppSpector uses modules called monitors to track different app activities and gather stats.
We provide a bunch of monitors out of the box which could be used together or in any combinations.
To start AppSpector you need to build instance of `AppSpectorConfig` and provide your API key.
You can start exact monitors with:

```configWithAPIKey:(NSString *)apiKey monitorIDs:(NSSet <NSString *> *)monitorIDs``` 

Or start all available with:

```configWithAPIKey:(NSString *)apiKey```

Available monitors:

```
AS_SCREENSHOT_MONITOR
AS_SQLITE_MONITOR
AS_HTTP_MONITOR
AS_COREDATA_MONITOR
AS_PERFORMANCE_MONITOR
AS_LOG_MONITOR
AS_LOCATION_MONITOR
```



#### Objective-C
<!-- integration-objc-example-start -->
Starting only selected monitors:
```objective-c
NSSet *monitorIDs = [NSSet setWithObjects:AS_HTTP_MONITOR, AS_LOG_MONITOR, nil];
AppSpectorConfig *config = [AppSpectorConfig configWithAPIKey:@"API_KEY" monitorIDs:monitorIDs];
[AppSpector runWithConfig:config];
```

Or all at once:
```
AppSpectorConfig *config = [AppSpectorConfig configWithAPIKey:@"API_KEY"];
[AppSpector runWithConfig:config];
```
<!-- integration-objc-example-end -->

#### Swift
<!-- integration-swift-example-start -->
Starting only selected monitors:
```swift
let config = AppSpectorConfig(apiKey: "API_KEY", monitorIDs: [AS_HTTP_MONITOR, AS_LOG_MONITOR])
AppSpector.run(with: config)
```
or to start all monitors:
```
let config = AppSpectorConfig(apiKey: "API_KEY")
AppSpector.run(with: config)
```
<!-- integration-swift-example-end -->

## Start/Stop SDK
AppSpector start is two step process.
When you link with AppSpector framework it starts to collect data immediately after load. When you call `startWithConfig` method - AppSpector opens a connection to the backend and from that point you can see your session on the frontend.

You can manually control AppSpector state by calling `start` and `stop` methods.
`stop` tells AppSpector to disable all data collection and close current session.
`start` starts it again using config you provided at load. This will be a new session, all activity between `stop` and `start` calls will not be tracked.

#### Objective-C
<!-- start-stop-objc-example-start -->
```objective-c
[AppSpector stop];
[AppSpector start];
```
<!-- start-stop-objc-example-end -->

#### Swift
<!-- start-stop-swift-example-start -->
```swift
AppSpector.stop()
AppSpector.start()
```
<!-- start-stop-swift-example-end -->

## Features
AppSpector provides 8 monitors that tracks different activities inside your app:

#### Screenshot monitor
Simply captures screenshot from the device.

#### SQLite monitor
Provides browser for sqlite databases found in your app. Allows to track all queries, shows DB scheme and data in DB. You can issue custom SQL query on any DB and see results in browser immediately.

#### HTTP monitor
Shows all HTTP traffic in your app. You can examine any request, see request/response headers and body.
We provide XML and JSON highliting for request/responses with formatting and folding options so even huge responses are easy to look through.

#### CoreData monitor
Browser for CoreData stores in your app. Shows model scheme just like Xcode editor, allows to navigate data, follow relations, switching contexts and running custom fetch requests against any model / context.

#### Performance monitor
Displays real-time graphs of the CPU / Memory/ Network / Disk / Battery usage.

#### Logs monitor
Displays all logs generated by your app. We provide integration with popular logging framework [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack), all your logs written with loggers from it will be displayed with respect to their logging levels.

#### Location monitor
Most of the apps are location-aware. Testing it requires changing locations yourself. In this case, location mocking is a real time saver. Just point to the location on the map and your app will change its geodata right away.

#### Environment monitor
Gathers all of the environment variables and arguments in one place, info.plist, cli arguments and much more.


## Filtering your data
Sometimes you may want to adjust or completely skip some pieces of data AppSpector gather. We have a special feature called Sanitizing for this, for now it's available only for HTTP and logs monitors, more coming.
For these two monitors you can provide a filter which allows to modify or block events before AppSpector sends them to the backend. Filter is a callback you assign to a `AppSpectorConfig` property `httpSanitizer` for HTTP monitor or `logSanitizer` for logs monitor. Filter callback gets event as its argument and should return it.

Some examples. Let's say we want to skip our auth token from requests headers:
```
[config.httpSanitizer setFilter:^ASHTTPEvent *(ASHTTPEvent *event) {
    if ([event.request.allHTTPHeaderFields.allKeys containsObject:@"YOUR-AUTH-HEADER"]) {
        [event.request setValue:@"redacted" forHTTPHeaderField:@"YOUR-AUTH-HEADER"];
    }

    return event;
}];
```

Or we want to raise log level to `warning` for all messages containing word 'token':
```
[config.logSanitizer setFilter:^ASLogMonitorEvent *(ASLogMonitorEvent *event) {
    if ([event.message rangeOfString:@"token"].location != NSNotFound) {
        event.level = ASLogEventLevelWarn;
    }

    return event;
}];
```


## Feedback
Let us know what do you think or what would you like to be improved: [info@appspector.com](mailto:info@appspector.com).
