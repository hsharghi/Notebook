**Run docker resume on system restart**

```bash
docker run -p 6379:6379 -d --restart unless-stopped --name redis redis
```
😄
