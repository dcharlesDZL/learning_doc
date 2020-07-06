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


- Miner初始化时添加一个scheduler.
- Miner初始化时添加一个worker,通过调用AddWorker函数添加一个worker.
```go
go runSched(){
    ...
}
```
- 运行sched
循环等待调度器中通道的元素
1. case w := <- sh.newWorkers:
2. case wid := <- sh.workerClosing:
3. case req := <- sh.schedule:
4. case wid := <- sh.workerFree:
5. case <- sh.closing:


其中,当通道中有元素sh.schedule 传入时进行workers的调度,sh.schedule 元素有sh.sched.Scheduler 函数产生,
sh.sched.Scheduler 由P1,P2,C1,C2各服务调用.

当有调用workers请求时,如果当时无可处理请求的worker可供调度,则将request push到sh.schedQueue队列中.
如果此时有多个workers(大于1个)可供调度,则进行切片排序.
首先按照workers能够处理诸如P1,P2,C1,C2服务类型数量的多少进行排序选择,优先选择可接受TaskType少的worker,即越"专一"的worker会被优先选择,当worker与worker之间**处理任务类型数量**相同时,则比较可用资源比例.
调度器偏好于已使用资源少的worker,其调用的函数为
```go
func (a *activateResources) utilization () {
    ...
}
```
返回cpu, memMin, memMax中占用最高的数值,即为此worker的最大占用率.worker之间比较选择最大占用率小的worker,即最终按照可用资源由大到小进行worker排序.
并最终选择可用资源最大的worker来处理当前服务.