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

* [Map items with a transform function](#map-items-with-a-transform-function)

drop this in your project:

```swift
extension Future where T: Sequence {
    func mapItems<U>(_ transform: @escaping (T.Element) -> U) -> Future<[U]> {
        return map { items in items.map(transform) }
    }
}
```

Then you can do this:

```swift
final class CategoryController {
    /// Returns a list of all categories
    func index(_ req: Request) throws -> Future<[Category]> {
        guard let host = req.http.headers.firstValue(name: .host) else {
            throw Abort(.badRequest)
        }

        return Category.query(on: req).all()
            .mapItems { Category(id: $0 ...) }
    }
}
```
