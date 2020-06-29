### lotus-seal-worker 命令学习
```sh
lotus-seal-worker run --address 192.168.6.162:2345 --commit --no-local-storage 
```
启动一个worker进程时,首先会阻塞获取storage miner API.
比较worker 与miner版本是否一致.
获取miner actor address
获取actor需要处理的sector size.
如果参数中包含"--commit" ,则获取filecoin参数.
```go
if cctx.Bool("commit") {
    if err := paramfetch.GetParams(ctx, build.ParametersJSON(), uint64(ssize)); err != nil {
        return xerrors.Errorf("get params: %w", err)
    }
}
```
向worker进程添加tasktypes,默认添加fetch commit1 finalize.
```go
var taskTypes []sealtasks.TaskType
taskTypes = append(taskTypes, sealtasks.TTFetch, sealtasks.TTCommit1, sealtasks.TTFinalize)
if cctx.Bool("commit") {
    taskTypes = append(taskTypes, sealtasks.TTCommit2)
}
```
如果带有precommit1 precommit2 commit,则分别添加precommit1 precommit2 commit2任务类型.
获取FS Repo,如果repo不存在,则进行初始化repo操作.
- 连接到master
```go
localStore, err := stores.NewLocal(ctx, lr, nodeApi, []string{"http://" + cctx.String("address") + "/remote"})
```
- 根据sectorsize,获取一个storage miner address
```go
spt, err := ffiwrapper.SealProofTypeFromSectorSize(ssize)
sminfo, err := lcli.GetAPIInfo(cctx, repo.StorageMiner)
remote := stores.NewRemote(localStore, nodeApi, sminfo.AuthHeader())
```
- 创建/暴露一个worker
```go
workerApi := &worker{
    LocalWorker: sectorstorage.NewLocalWorker(sectorstorage.WorkerConfig{
        SealProof: spt,
        TaskTypes: taskTypes,
    }, remote, localStore, nodeApi),
}
```
- 同时起一个go协程,注册一个worker,连接到miner节点,监听miner 的请求
```go
go func() {
    if err := nodeApi.WorkerConnect(ctx, "ws://"+cctx.String("address")+"/rpc/v0"); err != nil {
        log.Errorf("Registering worker failed: %+v", err)
        cancel()
        return
    }
}()
```
```go
func (sm *StorageMinerAPI) WorkerConnect(ctx context.Context, url string) error {
	w, err := connectRemoteWorker(ctx, sm, url)
	if err != nil {
		return xerrors.Errorf("connecting remote storage failed: %w", err)
	}

	log.Infof("Connected to a remote worker at %s", url)

	return sm.StorageMgr.AddWorker(ctx, w)
}
```
```go
func connectRemoteWorker(ctx context.Context, fa api.Common, url string) (*remoteWorker, error) {
	token, err := fa.AuthNew(ctx, []auth.Permission{"admin"})
	if err != nil {
		return nil, xerrors.Errorf("creating auth token for remote connection: %w", err)
	}

	headers := http.Header{}
	headers.Add("Authorization", "Bearer "+string(token))

	wapi, closer, err := client.NewWorkerRPC(url, headers)
	if err != nil {
		return nil, xerrors.Errorf("creating jsonrpc client: %w", err)
	}

	return &remoteWorker{wapi, closer}, nil
}
```

```go
func NewWorkerRPC(addr string, requestHeader http.Header) (api.WorkerAPI, jsonrpc.ClientCloser, error) {
	var res apistruct.WorkerStruct
	closer, err := jsonrpc.NewMergeClient(addr, "Filecoin",
		[]interface{}{
			&res.Internal,
		},
		requestHeader,
	)

	return &res, closer, err
}
```


