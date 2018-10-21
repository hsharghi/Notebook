# Vapor framework notes and snippets


## [Increase body size of request]

```
services.register(NIOServerConfig.default(hostname: "0.0.0.0", port: 8080, maxBodySize: 20_000_000))
```
