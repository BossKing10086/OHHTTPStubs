OHHTTPStubs
===========

A class to stub network requests easily: test your apps with **fake network data** (stubbed from file) and **custom response times**.

`OHHTTPStubs` is very useful to write **Unit Tests** and return fake network data from your fixtures, or to **simulate slow networks** in order to check your application behavior in bad network conditions.
It works with `NSURLConnection`, `AFNetworking`, or any networking framework you chose to use.

[![Build Status](https://travis-ci.org/AliSoftware/OHHTTPStubs.png?branch=master)](https://travis-ci.org/AliSoftware/OHHTTPStubs)

----

* [How it works](#how-it-works)
* [Documentation](#documentation)
* [Usage examples](#usage-examples)
 * [Stub all requests with some given NSData](#stub-all-requests-with-some-given-nsdata)
 * [Stub only requests to your WebService](#stub-only-requests-to-your-webservice)
 * [Set request and response time](#set-request-and-response-time)
 * [Simulate a down network](#simulate-a-down-network)
* [Advanced Usage](#advanced-usage)
 * [Use macros to build fixture paths](#use-macros-to-build-fixture-paths)
 * [Using download speed instead of responseTime](#using-download-speed-instead-of-responsetime)
 * [Stack multiple request handlers](#stack-multiple-request-handlers)
* [Installing in your projects](#installing-in-your-projects)
* [About OHHTTPStubs Unit Tests](#about-ohhttpstubs-unit-tests)
* [Change Log](#change-log)
* [License and Credits](#license-and-credits)

----

## How it works

`OHHTTPStubs` is aimed to be very simple to use. It uses block to intercept outgoing requests and allow you to return your own data instead.
Simply call `stubRequestsPassingTest:withStubResponse:` to install your define stub responses and when to stub them.

Then for every request sent, whatever the framework used (`NSURLConnection`,
[`AFNetworking`](https://github.com/AFNetworking/AFNetworking/), …):

* The block passed as first argument of `stubRequestsPassingTest:withStubResponse:` will be called to check if we need to stub this request.
* If this block returned anything that evaluates to YES, the block passed as second argument will be called to let you return an `OHHTTPStubsResponse` object, describing the fake response to return.

_(In practice, it uses the URL Loading System of Cocoa and a custom `NSURLProtocol` to intercept the requests and stub them)_


## Documentation

`OHHTTPStubs` headers are fully documented using Appledoc-like / Headerdoc-like comments in the header files.
When you [install it using CocoaPods](#installing-in-your-projects), you will get a docset for free installed in your Xcode Organizer.

You should find your happiness and (almost) all you have ever wondered in there, or directly in the header comments.

Don't hesitate to take a look into `OHHTTPStubsResponse.h`, `OHHTTPStubsResponse+JSON.h` and `OHHTTPStubsResponse.HTTPMessage.h` to see all the commodity constructors, constants and macros available.


## Usage examples


### Stub all requests with some given NSData

With the code below, every network request (because you returned YES in the first block) will return a stubbed response containing the data `"Hello World!"`:

    [OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
        return YES; // Stub ALL requests without any condition
    } withStubResponse:^OHHTTPStubsResponse*(NSURLRequest *request) {
        // Stub all those requests with "Hello World!" string
        NSData* stubData = [@"Hello World!" dataUsingEncoding:NSUTF8StringEncoding];
        return [OHHTTPStubsResponse responseWithData:stubData statusCode:200 headers:nil];
    }];
     

### Stub only requests to your WebService

This is typically used in your Unit Tests to stub specific requests targeted to a given host or WebService for example.

With the code below, only requests to the `mywebservice.com` host will be stubbed. Requests to any other host will hit the real world:

    [OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
        return [request.URL.host isEqualToString:@"mywebservice.com"];
    } withStubResponse:^OHHTTPStubsResponse*(NSURLRequest *request) {
        // Stub it with our "wsresponse.json" stub file
        return [OHHTTPStubsResponse responseWithFileAtPath:OHPathForFileInBundle(@"wsresponse.json",nil)
                statusCode:200 headers:@{"Content-Type":@"text/json"}];
    }];

This example also demonstrate how to **easily return the content of a given file in your application bundle**.
This is useful if you have all your fixtures (stubbed responses for your Unit Tests) in your Xcode project linked with your Unit Test target.

> Note: You may even put all your fixtures in a custom bundle (let's call it Fixtures.bundle) and then use the helper macros to get it with `OHPathForFileInBundle(@"wsresponse.json",OHResourceBundle(@"Fixtures"))`.

### Set request and response time

The `OHHTTPStubsResponse` you return can also contain timing information, like if `OHHTTPStubs` should simulate a latency by delaying the response or sending the data in small chunks during a given duration instead of all at a time.
_This is useful to simulate slow networks, for example, and to check that your user interface does not freeze in such occasions and that you though about displaying some activity indicators while waiting for your network responses, etc._

You may simply set the `requestTime` and `responseTime` of your `OHHTTPStubsReponse` to indicate this timing information, but an alternate, easiest way is to use `requestTime:response:` method that simply set them both and return the `OHHTTPStubsResponse` itself, so those calls can be chained, like this:

    [OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
        return [request.URL.host isEqualToString:@"mywebservice.com"];
    } withStubResponse:^OHHTTPStubsResponse*(NSURLRequest *request) {
        return [[OHHTTPStubsResponse responseWithJSONObject:someDict statusCode:200 headers:nil]
                requestTime:1.0 responseTime:3.0];
    }];

`OHHTTPStubs` will wait `requestTime` before sending the `NSHTTPURLResponse` (and before sending any chunk of data), and then start sending chunks of the stub data regularly during `responseTime` so that after the given `responseTime` all the stub data is sent.
This simulates regular reception of some data dispatched during the `responseTime` like if the server took time sending the content of a large response.

At the end, you will only have the full content of your stub data after `requestTime+responseTime`, time after which the `completion` block or `connectionDidFinishLoading:` delegate method will be called.

> Note that you can specify a network speed instead of a `responseTime` by using a negative value. [See below](#using-download-speed-instead-of-responsetime).

This code also show how you can create a `OHHTTPStubsReponse with a JSON object. `responseWithJSONObject:` will serialize the JSON object (`NSDictionary` in our example) and add the `"Content-Type: text/json"` header if not present already.

### Simulate a down network

You may also return a network error for your stub. For example, you may use this to simulate an absence of network connection:

    [OHHTTPStubsResponse responseWithError:[NSError errorWithDomain:NSURLErrorDomain code:kCFURLErrorNotConnectedToInternet userInfo:nil]];


## Advanced Usage

### Use macros to build fixture paths

`OHHTTPStubsResponse.h` include useful macros to build a path to your fixtures, like `OHPathForFileInBundle`, `ohPathForFileInDocumentsDir` and `OHResourceBundle`. You are encouraged to use them to build your path more easily.

_Especially, they use `[NSBundle bundleForClass:self.class]` to reference your app bundle (and not `[NSBundle mainBundle]` as one may think), so that they still work with OCUnit and XCTestKit when unit-testing your app in the Simulator._

### Using download speed instead of responseTime

When building the `OHHTTPStubsResponse` object, you can specify a response time (in seconds) so that the sending of the fake response will be postponed. This allows you to simulate a slow network for example.

If you specify a negative value for the responseTime parameter, instead of being interpreted as a time in seconds, it will be interpreted as a download speed in KBytes/s. In that case, the response time will be computed using the size of the response's data to simulate the indicated download speed.

The `OHHTTPStubsResponse` header defines some constants for standard download speeds:
* `OHHTTPStubsDownloadSpeedGPRS`   :    56 kbps (7 KB/s)
* `OHHTTPStubsDownloadSpeedEDGE`   :   128 kbps (16 KB/s)
* `OHHTTPStubsDownloadSpeed3G`     :  3200 kbps (400 KB/s)
* `OHHTTPStubsDownloadSpeed3GPlus` :  7200 kbps (900 KB/s)
* `OHHTTPStubsDownloadSpeedWifi`   : 12000 kbps (1500 KB/s)


### Stack multiple request handlers

You can call `stubRequestsPassingTest:withStubResponse:` multiple times. It will just add the response handlers in an internal list of handlers.

_This may be useful to install different stubs in various places in your code, or to separate different stubbing conditions more easily. See the `OHHTTPStubsDemo` project for a typical example._

When a network request is performed by the system, the response handlers are called in the reverse order that they have been added, the last added handler having priority over the first added ones.
The first handler that returns YES for the first parameter of `stubRequestsPassingTest:withStubResponse:` is then used to reply to the request.

* You can remove the latest added handler with the `removeLastRequestHandler` method, and all handlers with the `removeAllRequestHandlers` method.
* You can also remove any given handler with the `removeRequestHandler:` method. This method takes as a parameter the object returned by `stubRequestsPassingTest:withStubResponse:`.
_Note: this returned object is already retained by `OHHTTPStubs` while the stub is installed, so there is no need to keep a `__strong` reference to it._

----

## Installing in your projects

[CocoaPods](http://cocoapods.org/) is the easiest way to add third-party libraries like `OHHTTPStubs` in your projects. Simply add `pod 'OHHTTPStubs'` to your Podfile and you are done.

_Note: `OHHTTPStubs` uses APIs that were introduced in iOS5+, so it needs a deployment target of iOS5 minimum._

> **Warning: Be careful anyway to include `OHHTTPStubs` only in your test targets, or only use it in `#if DEBUG` portions, so that its code is not included in your release for the AppStore !**

In case you don't want to use CocoaPods (but you should!!!), the `OHHTTPStubs` project is provided as a Xcode project that generates a static library, so simply add its xcodeproj to your workspace and link your app against the `libOHHTTPStubs.a` library. See [here](https://github.com/AliSoftware/OHHTTPStubs/wiki/Detailed-Integration-Instruction) for detailed instructions.

## About `OHHTTPStubs` Unit Tests

If you want to be able to run `OHHTTPStubs`' Unit Tests, be sure you cloned the [`AFNetworking`](https://github.com/AFNetworking/AFNetworking/) submodule (by using the `--recursive` option when cloning your repo, or using `git submodule init` and `git submodule update`) as it is used by some of `OHHTTPStubs` unit tests.

Every contribution to add more unit tests is welcome.

## Change Log

The changelog is available [here in the dedicated wiki page](https://github.com/AliSoftware/OHHTTPStubs/wiki/ChangeLog).


## License and Credits

This project and library has been created by Olivier Halligon (@AliSoftware) and is under the MIT License.

It has been inspired by [this article from InfiniteLoop.dk](http://www.infinite-loop.dk/blog/2011/09/using-nsurlprotocol-for-injecting-test-data/).

I would also like to thank to @kcharwood for its contribution, and everyone who contributed to this project on GitHub.

