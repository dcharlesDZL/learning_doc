```go
case req := <-sh.schedule:
    scheduled, err := sh.maybeSchedRequest(req)
    if err != nil {
        req.respond(err)
        continue
    }
    if scheduled {
        continue
    }

    heap.Push(sh.schedQueue, req)
```
