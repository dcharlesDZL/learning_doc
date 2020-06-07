name：lotus-seal-worker
<!-- version：BuildVersion + CurrentCommit -->
flags：
        --workerrepo
        --storagerepo
        --enable-gpu-proving
command： run
flags：
        --address           string (required)
        --no-local-storage  boolean  ？？？
        --precommit1        boolean
        --precommit2        boolean
        --commit            boolean

lotus-seal-worker  --enable-gpu-proving --workerrepo path --storagerepo path run --address ADDRESS --precommit1

1获取存储矿工的API
2请求并获取context
3比较存储矿工版本与当前build版本是否匹配
4获取参与矿工的address
5获取sectorsize
        如果有commit，获取参数
6添加sealtask precommit1 precommit2 commit2 至少有一个task，否则报错。
7打开storagerepo
8新建FS实例
9不存在FS，worker 初始化 FS
10获取repo.Lock    ？？？lock是什么
11newLocal
12封装一个sector 大小为ssize
13新建remote实例
14新建worker实例 newLocalWorker     sealproof、tasktype
15router、rpcserver
16注册worker服务
17等待请求
18建立socket，连接worker rpc
19最终起一个tcp服务监听address




lotus-storage-miner sectors list 获取sector状态
pledge sector
        获取unpadded sector size
        sectorsize ->prooftype
        next sector -> new sector
        pledge sector -> pieces 数量
        new sector



//lotus-storage-miner/info.go
lotus-storage-miner info
返回：
Miner: maddr
Sector Size: ssize
Byte Power: 调用types.BigDiv(a,b)
Actual Power: 调用types.BigDiv(a,b)
Commited：secCount 调用api.StateMinerSectorCount(ctx, maddr, types.EmptyTSK)
Proving：if nfaults == 0 ，secCount 
else Proving: %s (%s Faulty, %.2f%%)\n",
PreCommit：
Locked：
Available：
Market (Escrow): 
Market (Locked):
Sectors: 调用sectorInfo(ctx, nodeApi)
        Total: len(sectors)

        st, err := napi.SectorsStatus(ctx, s)
        buckets[sealing.SectorState(st.State)]++
        sorted = append(sorted, stateMeta{i: i, state: state})
        _, _ = color.New(stateOrder[s.state].col).Printf("\t%s: %d\n", s.state, s.i)

//lotus-storage-miner init
get sector-size&fasPrice&symlink to imported sectors&check proof parameters&get full nodeapi
get repoPath
call storageMinerInit
if --actor address //get miner address
        if --genesis-miner
                //create a new sector manager
                smgr, err := sectorstorage.New(ctx, lr, stores.NewIndex(), &ffiwrapper.Config{SealProofType: spt,}, sectorstorage.SealerConfig{true, true, true}, nil, sa)
                        //create a manager instance
                        m := &manager{...}
                        //goroutine run schedule
                        go m.sched.runSched()
                        //append sealtask, default PreCommit1 PreCommit2 Commit1 Commit2
                        //add worker
                        err = m.AddWorker(ctx, NewLocalWorker(WorkerConfig{SealProof: cfg.SealProofType,TaskTypes: localTasks,}, stor, lstor, si))
                //create storagewpp 
                epp, err := storage.NewWinningPoStProver(api, smgr, ffiwrapper.ProofVerifier, dtypes.MinerID(mid))
                //create a miner
                m := miner.NewMiner(api, epp, beacon, a){}
else 
//create a storage miner
a, err := createStorageMiner(ctx, api, peerid, gasPrice, cctx)
mds.Put(datastore.NewKey("miner-address"), addr.Bytes())


### PreCommitting
func (m *Sealing) handlePreCommitting(ctx statemachine.Context, sector SectorInfo)

//sealing plan
Plan(events, user) --> 记录events，获取user state -->
fsmPlanners 不同状态回调不同的function
        state: Packing --> m.handlePacking -->Precommit1

        state: PreCommit1 --> m.handleCommit1 -->Precommit2, sealfailed, packingfailed
        state: PreCommit2 --> m.handlePreCommit2 -->PreCommitting, sealfailed, PrecommitedFailed
        state: PreCommitting --> m.handlePreCommitting  --> PreCommited, sealPrecommitedFailed, chianPrecommitFailed
        state: WaitSeed  --> m.handleWaitSeed   -->SeedReady, ChainPrecommitedFailed
        state: Committing  -->planCommitting --> m.handleCommitting  -->CommitWait 
                                             --> seedReady -->Committing
                                             -->SectorComputeProofFailed --> ComputeProofFailed
                                             -->SectorSealPreCommitFailed|SectorCommitFailed --> CommitFailed
        state: CommitWait  --> m.handleCommitWait  --> Proving -->FinalizeSector
                                                   --> CommitFailed
        state: FinalizeSector  --> m.handleFinalizeSector  -->Proving
        state: Proving  --> FaultReport



var fsmPlanners = map[SectorState]func(events []statemachine.Event, state *SectorInfo) error{
	UndefinedSectorState: planOne(on(SectorStart{}, Packing)),
	Packing:              planOne(on(SectorPacked{}, PreCommit1)),
	PreCommit1: planOne(
		on(SectorPreCommit1{}, PreCommit2),
		on(SectorSealPreCommitFailed{}, SealFailed),
		on(SectorPackingFailed{}, PackingFailed),
	),
	PreCommit2: planOne(
		on(SectorPreCommit2{}, PreCommitting),
		on(SectorSealPreCommitFailed{}, SealFailed),
		on(SectorPackingFailed{}, PackingFailed),
	),
	PreCommitting: planOne(
		on(SectorSealPreCommitFailed{}, SealFailed),
		on(SectorPreCommitted{}, WaitSeed),
		on(SectorChainPreCommitFailed{}, PreCommitFailed),
	),
	WaitSeed: planOne(
		on(SectorSeedReady{}, Committing),
		on(SectorChainPreCommitFailed{}, PreCommitFailed),
	),
	Committing: planCommitting,
	CommitWait: planOne(
		on(SectorProving{}, FinalizeSector),
		on(SectorCommitFailed{}, CommitFailed),
	),

	FinalizeSector: planOne(
		on(SectorFinalized{}, Proving),
	),

	Proving: planOne(
		on(SectorFaultReported{}, FaultReported),
		on(SectorFaulty{}, Faulty),
	),

	SealFailed: planOne(
		on(SectorRetrySeal{}, PreCommit1),
	),
	PreCommitFailed: planOne(
		on(SectorRetryPreCommit{}, PreCommitting),
		on(SectorRetryWaitSeed{}, WaitSeed),
		on(SectorSealPreCommitFailed{}, SealFailed),
	),
	ComputeProofFailed: planOne(
		on(SectorRetryComputeProof{}, Committing),
	),
	CommitFailed: planOne(
		on(SectorSealPreCommitFailed{}, SealFailed),
		on(SectorRetryWaitSeed{}, WaitSeed),
		on(SectorRetryComputeProof{}, Committing),
		on(SectorRetryInvalidProof{}, Committing),
	),

	Faulty: planOne(
		on(SectorFaultReported{}, FaultReported),
	),
	FaultedFinal: final,
}
// Now decide what to do next

	/*

		*   Empty
		|   |
		|   v
		*<- Packing <- incoming
		|   |
		|   v
		*<- PreCommit1 <--> SealFailed
		|   |                 ^^^
		|   v                 |||
		*<- PreCommit2 -------/||
		|   |                  ||
		|   v          /-------/|
		*   PreCommitting <-----+---> PreCommitFailed
		|   |                   |     ^
		|   v                   |     |
		*<- WaitSeed -----------+-----/
		|   |||  ^              |
		|   |||  \--------*-----/
		|   |||           |
		|   vvv      v----+----> ComputeProofFailed
		*<- Committing    |
		|   |        ^--> CommitFailed
		|   v             ^
		*<- CommitWait ---/
		|   |
		|   v
		*<- Proving
		|
		v
		FailedUnrecoverable

		UndefinedSectorState <- ¯\_(ツ)_/¯
		    |                     ^
		    *---------------------/

	*/
        