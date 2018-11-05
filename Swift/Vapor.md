# Vapor framework notes and snippets

* [Increase body size of request](#increase-body-size-of-request)

```swift
services.register(NIOServerConfig.default(hostname: "0.0.0.0", port: 8080, maxBodySize: 20_000_000))
```

* [Get path to Public directory](#increase-body-size-of-request)

```swift
/// i.e. /Users/Name/Documents/Project/Public/
let directory = DirectoryConfig.detect().workDir.finished(with: "/") + "Public".finished(with: "/")

```
