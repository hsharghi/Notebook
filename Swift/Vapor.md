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

* [Check if resource in url exists](#check-if-resource-in-url-exists)

```swift
        // if inside a request closure
        let client = try req.client()
        let fileExists = client.send(.HEAD, to: "http://domain.com/image.png").map { (response) -> Bool in
            return response.http.status.code == 200
        }
        
        let fileExists = HTTPClient.connect(hostname: "domain.com", on: req).flatMap { (client) -> Future<Bool> in
        let request = HTTPRequest(method: .HEAD, url: "/image.png")
        return client.send(request).map({ (response) in
           return response.status.code == 200
        })
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

deleting all Redis keys with given prefix

```swift
extension Future where T == Void {
    func deleteInRedis(prefix: String, on container: Container) -> Future<Void> {
        return flatMap {
            container.withPooledConnection(to: .redis) { (redis: RedisClient) in
                redis
                    .command("KEYS", [
                        .init(stringLiteral: "\(prefix)*")
                    ])
                    .flatMap { (result: RedisData) in
                        guard let keys = result.array, !keys.isEmpty else {
                            return container.future()
                        }

                        return redis
                            .command("DEL", keys)
                            .transform(to: ())
                    }
            }
        }
    }
}
```

group by multiple models

```swift
.decode(ModelB.self).groupBy(\ModelB.column).decode(Original Model.self)
```

add unique flag in migration

```
extension User: Migration {
    static func prepare(on conn: MySQLDatabase.Connection) -> Future<Void> {
        return Database.create(self, on: conn) { builder in
            try addProperties(to: builder)
            builder.unique(on: \.email)
        }
    }
}
```

Enable query logging 
`databases.enableLogging(on: .sqlite)`
```swift
    let mySQL = MySQLDatabase(config: config)

    var database = DatabasesConfig()
    database.enableLogging(on: .mysql)
    database.add(database: mySQL, as: .mysql)
    services.register(database)
```


Codable Struct with custom Decoder/Encode

```swift
import Foundation

struct Message {
    var data: [String: [String: String]]
}

extension Message: Codable {
    init(from decoder: Decoder) throws {
        let data = try [String: [String: String]].init(from: decoder)
        self.init(data: data)
    }
    
    func encode(to encoder: Encoder) throws {
        return try data.encode(to: encoder)
    }
}

let message = Message(data: ["Alphabet": ["A": "a", "B": "b"], "Number": ["One": "1", "Two": "2"]])
let data = try JSONEncoder().encode(message)
print(String(data: data, encoding: .utf8))

let decoded = try JSONDecoder().decode(Message.self, from: data)
print(decoded)
```


* [Use alsoDecode to prevent N+1 problem](#use-alsodecode-to-prevent-N-1-problem)

```swift
func tabledata(req: Request) throws -> Future<[IdeaOverview]> {
    return Idea.query(on: req)
        .join(\User.id, to: \Idea.creatorUserId)
        .alsoDecode(User.self)
        .all()
        .map { (rows: [(Idea, User)] -> [IdeaOverview] in
            rows.map { IdeaOverview(idea: $0.0, user: $0.1, businessValue: nil) }
        }
}
```


Get request body as String

```swift
request.http.body.consumeData(on: req)
.map { String(data: $0, encoding: .utf8) }
.map { bodyString in
    //do what you need to do 
}
```


map and flatMap for nil optional

```swift
//
//  Future+NilMap.swift
//  Created on 3/10/19
//

import Async

public protocol OptionalConstrainable {
    associatedtype Element
    var asOptional: Optional<Element> { get }
}

extension Optional: OptionalConstrainable {
    public var asOptional: Optional<Wrapped> { return self }
}

extension Future where Expectation: OptionalConstrainable {
    /// Calls the supplied closure if the chained `Future`'s optional return value resolves to a nil.
    ///
    /// The closure gives you a chance to return something non-nil.
    ///
    /// The callback expects a non-`Future` return. See `nilFlatMap` for a Future return.
    public func nilMap(_ callback: @escaping () throws -> Expectation.Element) -> Future<Expectation.Element> {
        let promise = eventLoop.newPromise(Expectation.Element.self)
        
        addAwaiter { result in
            switch result {
            case .error(let error):
                promise.fail(error: error)
            case .success(let e):
                if let result = e.asOptional {
                    promise.succeed(result: result)
                } else {
                    do {
                        try promise.succeed(result: callback())
                    } catch {
                        promise.fail(error: error)
                    }
                }
                
            }
        }
        
        return promise.futureResult
    }
    
    /// Calls the supplied closure if the chained `Future`'s optional return value resolves to a nil.
    ///
    /// The closure gives you a chance to return something non-nil.
    ///
    /// The callback expects a `Future` return. See `nilMap` for a non-`Future` return.
    public func nilFlatMap(_ callback: @escaping () throws -> Future<Expectation.Element>) -> Future<Expectation.Element> {
        let promise = eventLoop.newPromise(Expectation.Element.self)
        
        addAwaiter { result in
            switch result {
            case .error(let error):
                promise.fail(error: error)
            case .success(let e):
                if let result = e.asOptional {
                    promise.succeed(result: result)
                } else {
                    do {
                        let mapped = try callback()
                        mapped.cascade(promise: promise)
                    } catch {
                        promise.fail(error: error)
                    }
                }
                
            }
        }
        
        return promise.futureResult
    }
}
```

Custom keys for DB, json response and ...


```swift
final class MyModel: ... {
    // ...
    let dogBreed: String

    enum CodingKeys: String, CodingKeys {
        case dogBreed = "dog_breed"
    }
}
```


Use .wait() on queries with this Request extension

```swift
func allMatchesList(_ request: Request) throws -> Future<[FootballMatchForm]> {
    return request.dispatch { request -> [FootballMatchForm] in
        let matches = FootballMatch.query(on: request).range(..<3).all().wait()
 
        let matchIDs = matches.map { $0.requireID() }
        let allTeams = Team.query(on: request).filter(\Team.matchID ~~ matchIDs).all().wait()
        let teamsByMatch = [FootballMatch.ID: [Team]](grouping: allTeams, by: { $0.teamID })
            
        let teamIDs = allTeams.map { $0.requireID() }
        let allPlayers = Player.query(on: request).filter(\Player.teamID ~~ teamIDs).all().wait()
        let playersByTeam = [Team.ID: [Player]](grouping: allPlayers, by: { $0.teamID })
            
        let forms = matches.map { match -> FootballMatchForm in
            let teams = teamsByMatch[match.requireID()] ?? []
            let players = teams.flatMap { playersByTeam[$0.requireID()] ?? [] }

            return FootballMatchForm(
                match: match,
                teams: teams,
                players: players)
        }

        return forms
    }
}

extension Request {
    func dispatch<T>(handler: @escaping (Request) throws -> T) -> Future<T> {
        let promise = eventLoop.newPromise(T.self)

        DispatchQueue.global().async {
            do {
                let result = try handler(self)
                promise.succeed(result: result)
            } catch {
                promise.fail(error: error)
            }
        }

        return promise.futureResult
    }
}
```

