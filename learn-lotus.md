### miner 初始化
lotus-storage-miner init --actor t01000 --owner OWNERKEY --sector-size 1GiB --genesis-miner

1. 获取sector大小
```go
sectorSizeInt, err := units.RAMInBytes(cctx.String("sector-size"))
ssize := abi.SectorSize(sectorSizeInt)
```
2. 获取参数
```go
if err := paramfetch.GetParams(build.ParametersJson(), uint64(ssize)); err != nil {
			return xerrors.Errorf("fetching proof parameters: %w", err)
		}
```
3. 获取nodeAPI
```go
api, closer, err := lcli.GetFullNodeAPI(cctx)
```
4. 检查全节点同步状态（如果指明--nosync，则不检查）
```go
if !cctx.Bool("genesis-miner") && !cctx.Bool("nosync") {
    if err := lcli.SyncWait(ctx, api); err != nil {
        return xerrors.Errorf("sync wait: %w", err)
    }
}
```
5. 获取repopath
```go
repoPath := cctx.String(FlagStorageRepo)
```
6. 检查全节点版本
```go
v, err := api.Version(ctx)
```
7. 创建repo，指定矿工存储目录
8. 初始化repo在指定目录
```go
if err := r.Init(repo.StorageMiner); err != nil {
    return err
}
```
9. 获取storage miner repo排它锁（exclusive lock）
```go
lr, err := r.Lock(repo.StorageMiner)
```
10. 如果未设置--no-local-storage，创建localStorage
```go
if !cctx.Bool("no-local-storage") {
    b, err := json.MarshalIndent(&stores.LocalStorageMeta{
        ID:       stores.ID(uuid.New().String()),
        Weight:   10,
        CanSeal:  true,
        CanStore: true,
    }, "", "  ")
    if err != nil {
        return xerrors.Errorf("marshaling storage config: %w", err)
    }

    if err := ioutil.WriteFile(filepath.Join(lr.Path(), "sectorstore.json"), b, 0644); err != nil {
        return xerrors.Errorf("persisting storage metadata (%s): %w", filepath.Join(lr.Path(), "sectorstore.json"), err)
    }

    localPaths = append(localPaths, stores.LocalPath{
        Path: lr.Path(),
    })
}
```
11. 根据storageConfig设置storage
```go
if err := lr.SetStorage(func(sc *stores.StorageConfig) {
    sc.StoragePaths = append(sc.StoragePaths, localPaths...)
}); err != nil {
    ...
}
```
12. 初始化storageMiner
获取peerID metadata.如果设定了genesis-miner,则获得SealProofType、MinerID、StorageAuth、SectorStorageManager。
```go
if err := storageMinerInit(ctx, cctx, api, r, ssize, gasPrice); err != nil {
    ...
}
```
```go
func storageMinerInit(ctx context.Context, cctx *cli.Context, api lapi.FullNode, r repo.Repo, ssize abi.SectorSize, gasPrice types.BigInt) error {
	lr, err := r.Lock(repo.StorageMiner)
	if err != nil {
		return err
	}
	defer lr.Close()

	log.Info("Initializing libp2p identity")

	p2pSk, err := makeHostKey(lr)
	if err != nil {
		return xerrors.Errorf("make host key: %w", err)
	}

	peerid, err := peer.IDFromPrivateKey(p2pSk)
	if err != nil {
		return xerrors.Errorf("peer ID from private key: %w", err)
	}

	mds, err := lr.Datastore("/metadata")
	if err != nil {
		return err
	}

	var addr address.Address
	if act := cctx.String("actor"); act != "" {
		a, err := address.NewFromString(act)
		if err != nil {
			return xerrors.Errorf("failed parsing actor flag value (%q): %w", act, err)
		}

		if cctx.Bool("genesis-miner") {
			if err := mds.Put(datastore.NewKey("miner-address"), a.Bytes()); err != nil {
				return err
			}

			spt, err := ffiwrapper.SealProofTypeFromSectorSize(ssize)
			if err != nil {
				return err
			}

			mid, err := address.IDFromAddress(a)
			if err != nil {
				return xerrors.Errorf("getting id address: %w", err)
			}

			sa, err := modules.StorageAuth(ctx, api)
			if err != nil {
				return err
			}

			smgr, err := sectorstorage.New(ctx, lr, stores.NewIndex(), &ffiwrapper.Config{
				SealProofType: spt,
			}, sectorstorage.SealerConfig{true, true, true}, nil, sa)
			if err != nil {
				return err
			}
			epp, err := storage.NewWinningPoStProver(api, smgr, ffiwrapper.ProofVerifier, dtypes.MinerID(mid))
			if err != nil {
				return err
			}

			gen, err := api.ChainGetGenesis(ctx)
			if err != nil {
				return err
			}

			beacon, err := drand.NewDrandBeacon(gen.Blocks()[0].Timestamp, build.BlockDelay)
			if err != nil {
				return err
			}

			m := miner.NewMiner(api, epp, beacon, a)
			{
				if err := m.Start(ctx); err != nil {
					return xerrors.Errorf("failed to start up genesis miner: %w", err)
				}

				cerr := configureStorageMiner(ctx, api, a, peerid, gasPrice)

				if err := m.Stop(ctx); err != nil {
					log.Error("failed to shut down storage miner: ", err)
				}

				if cerr != nil {
					return xerrors.Errorf("failed to configure storage miner: %w", err)
				}
			}

			if pssb := cctx.String("pre-sealed-metadata"); pssb != "" {
				pssb, err := homedir.Expand(pssb)
				if err != nil {
					return err
				}

				log.Infof("Importing pre-sealed sector metadata for %s", a)

				if err := migratePreSealMeta(ctx, api, pssb, a, mds); err != nil {
					return xerrors.Errorf("migrating presealed sector metadata: %w", err)
				}
			}

			return nil
		}

		if pssb := cctx.String("pre-sealed-metadata"); pssb != "" {
			pssb, err := homedir.Expand(pssb)
			if err != nil {
				return err
			}

			log.Infof("Importing pre-sealed sector metadata for %s", a)

			if err := migratePreSealMeta(ctx, api, pssb, a, mds); err != nil {
				return xerrors.Errorf("migrating presealed sector metadata: %w", err)
			}
		}

		if err := configureStorageMiner(ctx, api, a, peerid, gasPrice); err != nil {
			return xerrors.Errorf("failed to configure storage miner: %w", err)
		}

		addr = a
	} else {
		a, err := createStorageMiner(ctx, api, peerid, gasPrice, cctx)
		if err != nil {
			return xerrors.Errorf("creating miner failed: %w", err)
		}

		addr = a
	}

	log.Infof("Created new storage miner: %s", addr)
	if err := mds.Put(datastore.NewKey("miner-address"), addr.Bytes()); err != nil {
		return err
	}

	return nil
}
```
下面的New方法创建了SectorStorageManager时，同时起了一个goroutine来运行调度器scheduler，调用runSched方法进行workers的调度，scheduler不断查询worker的状态。调度器的任务分为newWorker、workerClosing、schedule、workerFree、closing，分别对应sh.schedNewWorker(w)、sh.schedDropWorker(wid)、scheduled, err := sh.maybeSchedRequest(req)、sh.onWorkerFreed(wid)、sh.schedClose()操作，调度器操作时会进行上锁，保证同一时刻只能有一个线程进行处理。runSched方法中同时起了一个goroutine观察worker状态。
```go
func New(ctx context.Context, ls stores.LocalStorage, si stores.SectorIndex, cfg *ffiwrapper.Config, sc SealerConfig, urls URLs, sa StorageAuth) (*Manager, error) {
	lstor, err := stores.NewLocal(ctx, ls, si, urls)
	if err != nil {
		return nil, err
	}

	prover, err := ffiwrapper.New(&readonlyProvider{stor: lstor}, cfg)
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
		sealtasks.TTAddPiece, sealtasks.TTCommit1, sealtasks.TTFinalize, sealtasks.TTFetch,
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
```go
go m.sched.runSched()
```
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

13. 成功创建Storage Miner



### lotus-storage-miner run --api 2345 --enable-gpu-proving 

1. 获取全节点API
```go
nodeApi, ncloser, err := lcli.GetFullNodeAPI(cctx)
```
2. 比较编译版本和运行版本是否一致
3. 检查全节点同步状态
3. 获取存储仓库路径
```go
		storageRepoPath := cctx.String(FlagStorageRepo)
        r, err := repo.NewFS(storageRepoPath)
```
4. 创建miner 节点
```go
		var minerapi api.StorageMiner
		stop, err := node.New(ctx,
			node.StorageMiner(&minerapi),
			node.Online(),
			node.Repo(r),

			node.ApplyIf(func(s *node.Settings) bool { return cctx.IsSet("api") },
				node.Override(new(dtypes.APIEndpoint), func() (dtypes.APIEndpoint, error) {
					return multiaddr.NewMultiaddr("/ip4/127.0.0.1/tcp/" + cctx.String("api"))
				})),
			node.Override(new(api.FullNode), nodeApi),
        )
```
5. 获取本仓库的endpoint的API，启动全节点
```go
endpoint, err := r.APIEndpoint()
remoteAddrs, err := nodeApi.NetAddrsListen(ctx)
```
6. miner节点连接网络
```go
if err := minerapi.NetConnect(ctx, remoteAddrs); err != nil {
    return err
}
```
7. 声明一个listener
```go
lst, err := manet.Listen(endpoint)
```
8. 路由注册要匹配的路由并指派一个处理程序handler
9. 起一个rpc服务器并注册miner API
```go
rpcServer := jsonrpc.NewServer()
rpcServer.Register("Filecoin", apistruct.PermissionedStorMinerAPI(minerapi))
```


### lotus-storage-miner find
#### 在存储系统寻找sector
1. 获取storage miner API
2. 获取活动的矿工及ID
```go
ma, err := nodeApi.ActorAddress(ctx)
mid, err := address.IDFromAddress(ma)
```
3. 由minerID和sector数量获取sectorID
```go
sid := abi.SectorID{
			Miner:  abi.ActorID(mid),
			Number: abi.SectorNumber(snum),
        }
```
4. 分别找到Unsealed、Sealed、Cache的Sector索引
```go
u, err := nodeApi.StorageFindSector(ctx, sid, stores.FTUnsealed, false)
s, err := nodeApi.StorageFindSector(ctx, sid, stores.FTSealed, false)
c, err := nodeApi.StorageFindSector(ctx, sid, stores.FTCache, false)
```
5. 打印出本地或远程sector信息


### lotus-storage-miner pledge
1. miner对sector进行填充随机数据
```go
return nodeApi.PledgeSector(ctx)
```
* 获取unpaddedsize
获取seal proof type

2. 
3. 

**====================**
Lotus Full Node:

Lotus Storage Miner:

Lotus Seal Worker:

lotus-seal-worker是一个额外的进程，其可以帮助 lotus-storage-miner 减轻处理事务的压力。你可以将worker进程与miner进程运行在同一台电脑上，运行lotussealworker run 命令，seal worker将从矿工仓库自动获取到认证token。可以运行lotusstorageminer info命令检查远程worker是否正确得连接到你的miner。
也可以使用完全独立的计算机进行密封工作（sealing task），需要对~/.lotusstorage/config.toml中的LOTUS_STORAGE_PATH进行配置。
远程worker机器配置好相关环境变量和配置之后，运行lotussealworker run命令，此时可以在miner机器上查看worker的状态，在miner机器上运行lotus-storage-miner workers list命令。

###　lotus-storage-miner workers list
1. miner 进程调用接口访问worker状态
```go
stats, err := nodeApi.WorkerStats(ctx)
```


### lotus-seal-worker run --enable-gpu-proving --address ADDRESS --precommit1 --precommit2 --commit
--address 远程矿工地址
--enable-gpu-proving  开启gpu加速
--precommit1 --precommit2 --commit 开启precommit1 precommit2 commit处理功能
1. 获取miner节点API
```go
nodeApi, closer, err := lcli.GetStorageMinerAPI(cctx)
```
2. 比较编译版本和运行版本是否一致
3. 当开启commit时,获取ipfs网络参数
4. 向taskType变量添加任务类型
4. 获取storage repo
5. 创建file system的repo实例
6. 如果repo不存在,初始化一个worker repo
7. 如果启动时没有添加--no-local-storage flag 生成localpath,创建sectorstore.json
8. **获得repo的排它锁/独占锁**,当前worker对这个仓库有了读写权.
9. 打开本地存储,连接master节点,即指定的端点
10. 设置远程扇区存储
11. 获取miner的API信息
12. 本地存储与远程miner建立连接
13. 创建或者暴露一个本地worker
14. 路由注册要匹配的路由并指派一个处理程序handler
15. 起一个rpc服务器并注册worker API
16. 等待处理rpc服务端请求
17. 同时,起一个goroutine,监听miner的请求
```go
nodeApi.WorkerConnect(ctx, "ws://"+cctx.String("address")+"/rpc/v0")
```
../github.com/filecoin-project/sector-storage/manager.go
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
func (m *Manager) AddWorker(ctx context.Context, w Worker) error {
	info, err := w.Info(ctx)
	if err != nil {
		return xerrors.Errorf("getting worker info: %w", err)
	}

	m.sched.newWorkers <- &workerHandle{
		w:         w,
		info:      info,
		preparing: &activeResources{},
		active:    &activeResources{},
	}
	return nil
}
```

当网络上有数据存储请求过来时,miner节点进行开始进行抵押扇区存储操作PledgeSector,此时起一个goroutine执行操作,当有下一个存储请求过来时,并发执行
### 存储过程
1. 存储客户端获取网络矿工,指定矿工,并支付gas
```sh
./lotus client deal <Data CID> <miner> <price> <duration>
```
```go
func (a *API) ClientStartDeal(ctx context.Context, params *api.StartDealParams) (*cid.Cid, error) {
	exist, err := a.WalletHas(ctx, params.Wallet)
	if err != nil {
		return nil, xerrors.Errorf("failed getting addr from wallet: %w", params.Wallet)
	}
	if !exist {
		return nil, xerrors.Errorf("provided address doesn't exist in wallet")
	}

	mi, err := a.StateMinerInfo(ctx, params.Miner, types.EmptyTSK)
	if err != nil {
		return nil, xerrors.Errorf("failed getting peer ID: %w", err)
	}

	md, err := a.StateMinerProvingDeadline(ctx, params.Miner, types.EmptyTSK)
	if err != nil {
		return nil, xerrors.Errorf("failed getting peer ID: %w", err)
	}

	rt, err := ffiwrapper.SealProofTypeFromSectorSize(mi.SectorSize)
	if err != nil {
		return nil, xerrors.Errorf("bad sector size: %w", err)
	}

	if uint64(params.Data.PieceSize.Padded()) > uint64(mi.SectorSize) {
		return nil, xerrors.New("data doesn't fit in a sector")
	}

	providerInfo := utils.NewStorageProviderInfo(params.Miner, mi.Worker, mi.SectorSize, mi.PeerId)

	dealStart := params.DealStartEpoch
	if dealStart <= 0 { // unset, or explicitly 'epoch undefined'
		ts, err := a.ChainHead(ctx)
		if err != nil {
			return nil, xerrors.Errorf("failed getting chain height: %w", err)
		}

		dealStart = ts.Height() + dealStartBuffer
	}

	result, err := a.SMDealClient.ProposeStorageDeal(
		ctx,
		params.Wallet,
		&providerInfo,
		params.Data,
		dealStart,
		calcDealExpiration(params.MinBlocksDuration, md, dealStart),
		params.EpochPrice,
		big.Zero(),
		rt,
	)

	if err != nil {
		return nil, xerrors.Errorf("failed to start deal: %w", err)
	}

	return &result.ProposalCid, nil
}
```
2. 获取矿工信息
3. 获取矿工证明deadline
4. 根据扇区大小获取证明类型
5. 根据以上信息,拿到存储提供商信息
6. 获取链上交易起始高度
7. 提交存储请求
8. 返回值:提案cid

workers与miner之间的通讯是通过rpc实现,miner初始化时,获取PreSeal元数据,生成多个sector info,拿到每个sector的deal schedule.
miner拿到多个sectors时,分别对每一个sector进行AddPiece SealPrecommit1 SealPrecommit2 SealCommit1 SealCommit2 Verify操作
```go
	for _, sector := range meta.Sectors {
		sectorKey := datastore.NewKey(sealing.SectorStorePrefix).ChildString(fmt.Sprint(sector.SectorID))

		dealID, err := findMarketDealID(ctx, api, sector.Deal)
		if err != nil {
			return xerrors.Errorf("finding storage deal for pre-sealed sector %d: %w", sector.SectorID, err)
		}
		commD := sector.CommD
		commR := sector.CommR

		info := &sealing.SectorInfo{
			State:        sealing.Proving,
			SectorNumber: sector.SectorID,
			Pieces: []sealing.Piece{
				{
					Piece: abi.PieceInfo{
						Size:     abi.PaddedPieceSize(meta.SectorSize),
						PieceCID: commD,
					},
					DealInfo: &sealing.DealInfo{
						DealID: dealID,
						DealSchedule: sealing.DealSchedule{
							StartEpoch: sector.Deal.StartEpoch,
							EndEpoch:   sector.Deal.EndEpoch,
						},
					},
				},
			},
			CommD:            &commD,
			CommR:            &commR,
			Proof:            nil,
			TicketValue:      abi.SealRandomness{},
			TicketEpoch:      0,
			PreCommitMessage: nil,
			SeedValue:        abi.InteractiveSealRandomness{},
			SeedEpoch:        0,
			CommitMessage:    nil,
		}

		b, err := cborutil.Dump(info)
		if err != nil {
			return err
		}

		if err := mds.Put(sectorKey, b); err != nil {
			return err
		}

		if sector.SectorID > maxSectorID {
			maxSectorID = sector.SectorID
        }
    }
```