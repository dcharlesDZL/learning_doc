创建sector manager
```go
smgr, err := sectorstorage.New(ctx, lr, stores.NewIndex(), &ffiwrapper.Config{
				SealProofType: spt,
            }, sectorstorage.SealerConfig{true, true, true, true}, nil, sa)
```
初始化miner时,创建sector manager,此时根据sector size 配置spt(seal proof type),以及sealer config默认配置全部服务(p1 p2 c2 unseal),若想实现调度单个服务或是某几个服务,或许应该在这里进行修改.
sectorstorage.New详解:
```go
func New(ctx context.Context, ls stores.LocalStorage, si stores.SectorIndex, cfg *ffiwrapper.Config, sc SealerConfig, urls URLs, sa StorageAuth) (*Manager, error) {
	lstor, err := stores.NewLocal(ctx, ls, si, urls)
	if err != nil {
		return nil, err
	}

	prover, err := ffiwrapper.New(&readonlyProvider{stor: lstor, index: si}, cfg)
	if err != nil {
		return nil, xerrors.Errorf("creating prover instance: %w", err)
	}

	stor := stores.NewRemote(lstor, si, http.Header(sa))

	m := &Manager{
		scfg: cfg,

		ls:         ls,
		storage:    stor,
		localStore: lstor,
		remoteHnd:  &stores.FetchHandler{Local: lstor},
		index:      si,

		sched: newScheduler(cfg.SealProofType),

		Prover: prover,
	}

	go m.sched.runSched()

	localTasks := []sealtasks.TaskType{
		sealtasks.TTAddPiece, sealtasks.TTCommit1, sealtasks.TTFinalize, sealtasks.TTFetch, sealtasks.TTReadUnsealed,
	}
	if sc.AllowPreCommit1 {
		localTasks = append(localTasks, sealtasks.TTPreCommit1)
	}
	if sc.AllowPreCommit2 {
		localTasks = append(localTasks, sealtasks.TTPreCommit2)
	}
	if sc.AllowCommit {
		localTasks = append(localTasks, sealtasks.TTCommit2)
	}
	if sc.AllowUnseal {
		localTasks = append(localTasks, sealtasks.TTUnseal)
	}

	err = m.AddWorker(ctx, NewLocalWorker(WorkerConfig{
		SealProof: cfg.SealProofType,
		TaskTypes: localTasks,
	}, stor, lstor, si))
	if err != nil {
		return nil, xerrors.Errorf("adding local worker: %w", err)
	}

	return m, nil
}
```
我们可以看到在创建一个manager时,首先根据spt创建一个Manager实例,并且起一个协程运行调度器,
```go
	m := &Manager{
		scfg: cfg,

		ls:         ls,
		storage:    stor,
		localStore: lstor,
		remoteHnd:  &stores.FetchHandler{Local: lstor},
		index:      si,

		sched: newScheduler(cfg.SealProofType),

		Prover: prover,
	}
    go m.sched.runSched()
```
在添加完tasktype之后manager调用m.AddWorker向调度器的newWorkers通道中传入一个workerHandle结构体指针.
```go
m.sched.newWorkers <- &workerHandle{
	w:         w,
	info:      info,
	preparing: &activeResources{},
	active:    &activeResources{},
}
```
此时调度器接收到sh.newWorkers通道传来的数据时候,调用sh.schedNewWorker(w)
```go
case w := <-sh.newWorkers:
			sh.schedNewWorker(w)
```
如果配置了sealer config 中的 p1 p2 c2 unseal 服务,则依次添加到localtask中,同样的,若想实现调度其中的一个或是某几个服务,在sectorstorage.New的初始化中相应的项置false.
此时,调度器根据前面的配置添加一个worker,函数返回manger.
这时,manager允许调度之前所配置的任务类型(默认fetch p1 p2 c1 c2 unseal,全部接受调度).
go m.sched.runSched 详解:
此时调度器循环并阻塞来接收调度请求,分别有newWorker workerClosing schedule workerFree closing五种调度请求.
- 当有newWorker调度请求时,调度器执行调度一个新worker.
- 当有workerClosing调度请求时,调度器根据此时丢掉一个worker.
- 当有调度请求时,将请求添加至调度队列中.即当有调度器需要调度p1 p2 c1 c2等任务时,此时调用 m.sched.Schedule 函数,将workerRequest的结构体指针传入sh.schedule的通道中,此时调用sh.maybeSchedRequest(req)函数,将调度请求入队.调用maybeSchedRequest时,首先查找可以接收请求的workerID,查询添加到scheduler的worker list(即sh.workers),将可接受调度的worker添加到acceptable中.如果存在可调度的worker,那么对其进行比较,最终分配acceptable[0]的worker进行调度,调用sh.assignWorker函数
	```go
		if len(acceptable) > 0 {
		{
			var serr error

			sort.SliceStable(acceptable, func(i, j int) bool {
				rpcCtx, cancel := context.WithTimeout(req.ctx, selectorTimeout)
				defer cancel()
				r, err := req.sel.Cmp(rpcCtx, req.taskType, sh.workers[acceptable[i]], sh.workers[acceptable[j]])

				if err != nil {
					serr = multierror.Append(serr, err)
				}
				return r
			})

			if serr != nil {
				return false, xerrors.Errorf("error(s) selecting best worker: %w", serr)
			}
		}

		return true, sh.assignWorker(acceptable[0], sh.workers[acceptable[0]], req)
	}
	```
- 当有释放worker调度请求时, 调用sh.onWorkerFreed(wid)函数,此时若worker已经调度过,则调度队列移除该worker,队列长度减1.
- 当有关闭调度器请求时,调用sh.schedClose()函数,此时关闭调度器.

```go
func (sh *scheduler) runSched() {
	go sh.runWorkerWatcher()

	for {
		select {
		case w := <-sh.newWorkers:
			sh.schedNewWorker(w)
		case wid := <-sh.workerClosing:
			sh.schedDropWorker(wid)
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
		case wid := <-sh.workerFree:
			sh.onWorkerFreed(wid)
		case <-sh.closing:
			sh.schedClose()
			return
		}
	}
}
```