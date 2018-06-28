# What is FlexNetworking?

FlexNetworking is a modern Rx and Codable-optimized networking library that is built specifically for apps that make calls to one or more APIs. We, the developers, have been using it in production since the very first version for networking-heavy client apps and constantly evolving it to fit what we do better. So far, it has been used in three social networking apps and two team-based content creation apps.

Let's keep this short and check out some usage first.

# Usage

```swift
// asynchronous usage with a defined API endpoint
AppNetworking.runRequestAsync(path: "/conversations/\(user.id)/messages", method: "POST", body: ["text": "hello"]) { result in 
    switch result {
    case .success(let response):
        if response.status == 200, let json = response.asJSON {
            // use response.asJSON or response.asString, or response.rawData if you want to decode it manually
            let id = json["id"].string
        } else {
            // probably an internal server error, bad request error or permission/auth error -- handle appropriately
        }
    case .failure(let error):
        // `error`, when coalesced to a string, will contain enough information to diagnose the exact error in the majority of cases.
        // there is a special case of error, RequestError.noInternet(_) which you can catch if you are specifically interested in capturing cases where 
    }
}

// synchronous usage
let response = try AppNetworking.runRequest(path: "/users/\(user.id)/profile-picture", method: "GET", body: nil)
if response.status == 200, let data = response.rawData {
    profileImageView.image = UIImage(data: data) // this is a really bad example, please use SDWebImage or something for this
} else {
    print("Bad response status getting user's profile picture; details in \(response)")
}

// rx + codable usage. this is where flex really shines
struct SearchFilters: Codable {
    var query: String? = nil
    var minimumScore: Int = 0
}

let userDefinedFilters = SearchFilters(query: "programming", minimumScore: 1)

FlexNetworking.rx.requestCodable(path: "/users/\(user.id)", method: "GET", codableBody: userDefinedFilters)
    .subscribe(onSuccess: { (posts: [Post]) in 
        // automatically called on main queue, but you can of course change this by calling observeOn(_:) with another scheduler
        self.posts = posts
        self.collectionView.reloadData()
    }, onError: (error) in {
        Log.e("Error getting posts with search filters:", error)
        // error will either be a detailed request error, a detailed decoding error containing the response that failed to decode, or an error from your custom hooks
        // SEE: section labeled "Benefit: Detailed Errors" below
    }).addTo(viewModel.disposeBag)

```

Essentially, you choose which way you want the result delivered (async, synchronous, or as an rx Single) and be on your merry way. These are the parameters that you can pass to the request methods:
- `urlSession`: a `URLSession` to run the request on.
- `path`: a `String` containing the URL of the page you want to request.
- `method`: a `String` containing any HTTP method (GET, POST, PUT, PATCH, DELETE etc.)
- `body`: for a querystring (GET) request, this should be a `Dictionary` instance. for any other request, this can be a `Dictionary`, `RawBody(data:contentType:)` (from the `FlexNetworking` module), or `JSON` (if `FlexNetworking/SwiftyJSON subpod is installed`).
- `headers`: a string-to-string dictionary of request headers.

You can either call everything like `FlexNetworking().runRequest` (previous static methods are deprecated and have been removed), or create an instance like:

```
let APINetworking = FlexNetworking(...)
```

and call things with `APINetworking.runRequest(...)`. The latter option is much better in practice because it allows you to keep several instances with their own configurations and hooks...more on that later. For now, I have to tell you about why I made yet another networking library.

# Why??

Unlike other libraries which are either inflexible or not swifty enough (by our standards - looking at you Alamofire with your all-arg-optional objective-C style callbacks where you aren't guaranteed to have data OR an error and no one has figured out a stylistically clean way to deal with this, except the core swift team, and it's called enums! 🙄), Flex allows you to run any type of network request, any way you want it, in the Swiftiest way possible. 

Later, we found ourselves using a LOT of `Codable` and a LOT of `Rx` and decided to build them into this library as first class citizens, so we basically can't even fathom using any other library for our Rx-based and Codable-based apps now.

Finally, the error handling is amazing and a lot better than other libs we've used in the past. We have specifically built the thrown errors and the `Response` structs so that it is easy to diagnose what exactly went wrong by just logging the error or the response, because we were pissed about opaque errors that didn't provide enough information to diagnose based on logs, leading to bugs being shelved and left unresolved ("no repro case"). Hours of troubleshooting can be avoided with 5 minutes of work on the implementation (that's literally how long adding the requestParameters to the response struct took me, and it has already saved me hours and tons of frustration).

Logging an error will give you its specific category (including "noInternet" instead of `code == -1020` and "cancelledByUser" instead of `code == -999`) so you can handle certain ones specifically (e.g. queue an operation to retry if the internet connection cuts out) or just log them so troubleshooting is easier when issues eventually do crop up. 

Logging a response will tell you its status, body, and all original parameters that formed the URL request that was actually executed.

## More Detail on Detailed Errors

The basic process of making a request throws a closed set of errors (UNLESS you throw your own errors in pre-and-post- request hooks. more on that later)

These are members of the enum `RequestError` and fall into six categories:
- `.noInternet(Error)`
- `.miscURLSessionError(Error) // wraps an error directly from URLSession, except if the error carries the specific error code denoting no internet`
- `.invalidURL(message: String) // message will include the invalid path that could not be used as a URL for a request`
- `.emptyResponseError(Response) // includes all of the initial request parameters`
- `.cancelledByCaller // thrown when the request is cancelled by the user.`
- `.unknownError(message: String) // thrown in the rare case when Apple's URLSession returns a nil response and nil error in the handler. this should not happen in normal operation`

`FlexNetworking+RxSwift` expands upon this by also introducing a `DecodingError` struct, which is used in `.rx.requestCodable`. If an error is encountered while decoding the output type you were expecting from the request, an instance of `DecodingError` will be cascaded down the observer chain. **Printing this instance will give you context about the request parameters, the response (including status), and the exact error that the decoder ran into while attempting to decode the output type.**

# Configuration

Flex also lets you specify pre- and post- request hooks on a per-instance basis. 

## Pre-request hooks

**Pre-request hooks** let you globally modify the request parameters (URL session, request URL, HTTP method, request body, and request headers) before a request is made. Common use cases include:
- prepending an API endpoint to all requests made by an instance of `FlexNetworking` specifically made for API calls
- passing token headers to all requests made by an instance of `FlexNetworking` specifically made for API calls
- logging request parameters

Pre-request hooks simply conform to the `PreRequestHook` protocol and define a function that maps one value of `RequestParameters` to another. The predefined conforming class is `BlockPreRequestHook` which takes a custom block, allowing you to implement your own logic. Pre-request hooks are run in the order in which they are passed to `FlexNetworking.init`. If you want to apply more sophisticated pre-request logic, you may define your own type that conforms to the `PreRequestHook` protocol. There will be one instance of your hook, and it will live through the lifetime of the `FlexNetworking` instance it is hooked to, so you can use this to your advantage if you need to maintain additional state between requests. 

## Post-request hooks

**Post-request hooks** let you implement post-request logic that handles when a request is successfully executed but returns a recoverable error that you would like to automatically recover from. Common use cases include:
- initiating token refresh automatically when a request fails due to an expired token and retrying the original request that surfaced the fact that the token expired (semi-transactional)
- exponential backoff in the event of rate limiting
- logging notable responses

Post-request hooks simply conform to the `PostRequestHook` protocol and define a function that maps the latest `Response` and the initial `RequestParameters` (from the pre-request hooks) to one of three actions: 
- `continue` - continue to the next item in the chain, passing the response through to the next hook. *(you might use this to apply some side effects and be on your merry way with the same response)*
- `makeNewRequest(RequestParameters)` - make a new request and run the next hook on the response from this request. *(if the request fails to complete, no more hooks will run.)*
- `completed` - skip the rest of the chain, passing the latest response all the way through to the original caller.

The predefined conforming class is `BlockPostRequestHook`, which, as before, takes a custom block. Post-request hooks are run in the order in which they are passed to `FlexNetworking.init`, with the caveat that part of the chain will be skipped if it ever encounters the `.completed` command. 

If you want to apply more sophisticated post-request logic than a simple block allows, you may define your own type that conforms to the `PostRequestHook` protocol. Bear in mind that there will be one instance of your hook, and it will live through the lifetime of the `FlexNetworking` instance it is hooked to, so you can use this to your advantage if you need to maintain additional state between requests. 

**Note that requests made from post-request hooks do not have pre-request hooks run on them.**

**NB:** Both the Rx and standard bindings use concurrent dispatch queues for scheduling the hooks so if you sleep within a hook, you will only block the (non-main) thread on which the request is being made. If you develop more sophisticated hooks, **please make sure that their `execute` methods are thread-safe**, and please follow good concurrency practices: synchronize access where necessary, but keep synchronized operations to a minimum to avoid bottlenecks.

**Tip:** you may throw at any point in pre- and post-request hooks, which will halt the request then and there and bubble the error all the way back up to the caller.

## Why request hooks?? Why don't we just make custom methods that call FlexNetworking in the body?

These request hooks, especially the post-request hooks, are designed such that they are a series of well-defined operations on clearly structured input data producing clearly structured output data, making them potentially agnostic to the actual implementation you use make the request. 

In simpler terms, whether you use the normal throwing synchronous request method, the asynchronous request method, or the Rx bindings (really, Rx is the crucial part), all your pre- and post-request hooks will run without having to change any of the logic. They are simply things that return a changed version of immutable data. They may keep their own state which, again, is 100% agnostic of the internal methods you are using to make HTTP requests.

You can still make custom methods that call FlexNetworking in the body, I don't mind - but using pre- and post- request hooks means that, not only can you mix Rx and non-Rx without any method overriding, but we can also keep adding new bindings (like the Rx one) and new method signatures in the future and you will be able to use them out of the box forever, as long as we don't change the definition of "request parameters". *(And even if we do, you will only have one place to change it per hook.)*

## Examples of hooks doing useful stuff

Here are examples of pre- and post-request hooks that implement some of the use cases above (**prepending an endpoint, passing token headers, and initiating token refresh automatically**).

```swift
// with flex, you can specify encoders and decoders for Codable requests.
// if we are using a JSON Spring backend, for example, we may want to pass Date objects as millisecond timestamps.
let defaultEncoder: JSONEncoder = {
    let encoder = JSONEncoder()
    encoder.dateEncodingStrategy = .millisecondsSince1970
    return encoder
}()

let defaultDecoder: JSONDecoder = {
    let decoder = JSONDecoder()
    decoder.dateDecodingStrategy = .millisecondsSince1970
    return decoder
}()

let AppNetworking = FlexNetworking(
    preRequestHooks: [
        BlockPreRequestHook { (requestParameters) in
            // mutate path to allow us to use relative API paths and send authorization header with every request when present
            let (urlSession, path, method, body, headers) = requestParameters

            var additionalHeaders = headers

            if ActiveUser.isLoggedIn {
                additionalHeaders["Authorization"] = "Bearer \(ActiveUser.token)"
            }

            return (urlSession, Constants.apiEndpoint.appending(path), method, body, additionalHeaders)
        }
    ],
    postRequestHooks: [
        BlockPostRequestHook { (response, originalRequestParameters) -> PostRequestHookResult in
            let (urlSession, path, _, _, _) = originalRequestParameters

            if response.status == 401, let refreshToken = ActiveUser.refreshToken {
                // do token refresh if we got a 401
                let loginRequestParameters: RequestParameters = (
                    urlSession: urlSession,
                    path: Constants.apiEndpoint.appending("/token-refresh"),
                    method: "POST",
                    body: ["refreshToken": refreshToken],
                    headers: [:]
                )

                return .makeNewRequest(loginRequestParameters)
            } else {
                // or abort the post-request chain if we got a successful response - no need to refresh the token
                return .completed
            }
        },
        BlockPostRequestHook { (tokenRefreshResponse, originalRequestParameters) -> PostRequestHookResult in
            guard let rawData = loginResponse.rawData else {
                throw SimpleError(message: "no data in token refresh login response")
            }

            do {
                let tokenRefreshDTO = try defaultDecoder.decode(TokenRefreshDTO.self, from: rawData)

                let token = tokenRefreshDTO.token
                ActiveUser.token = token

                var headers = originalRequestParameters.headers
                headers["Authorization"] = "Bearer \(token)"

                // copy the original request from before the token refresh, but add the new token to the headers
                var retryRequestParameters = originalRequestParameters
                retryRequestParameters.headers = headers

                return .makeNewRequest(retryRequestParameters)
            } catch let e {
                Log.e(e)
                
                // TODO kick the user back to a login screen

                throw e // this will raise a request error at the call site.
            }
        }
    ],
    defaultEncoder: defaultEncoder,
    defaultDecoder: defaultDecoder
)
```

# TODO

- The Rx integration has really good handling of request cancellation, but the non-Rx version does not. We should explore ways to address that.
- Eventually address authentication in a more comprehensive way, and possibly bundle common hooks that are useful for common auth schemes
- Codable request/response integration for non-Rx
- MULTIPART!!!
- **Whatever you think is important: please leave an issue!**