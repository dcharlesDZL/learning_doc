### lotus-storage-miner 命令学习
首先在 https://faucet.testnet.filecoin.io/miner.html 创建storage miner,输入创建的bls wallet,bls钱包由下面命令创建.
```sh
./lotus wallet new bls
```
```sh
./lotus-storage-miner init --actor=t0118166 --owner=t3sjtzv6snp2ecc2mb2skuhgbb7b4ry7eizri7kgztpkfeg7bqt7y2sfpbfvkbssj4nhrl2bsbow7oqzrf57cq --genesis-miner
```
<!-- ```sh
./lotus-storage-miner init --sector-size 2KiB --genesis-miner --no-local-storage --gas-price 100 --create-worker-key
``` -->
执行lotus-storage-miner init命令时,
首先获得sector size,其次获得filecoin参数.
接下来连接全节点(full node),查看storage repo 是否存在,默认~/.lotusstorage,并检查是否已经初始化.
检查本地与远程版本是否一致.
如果指定了预先密封的扇区(pre-sealed-sectors),miner则会获得其对应的repo路径.
接下来执行storageMinerInit函数,初始化一个miner,
```go
if err := storageMinerInit(ctx, cctx, api, r, ssize, gasPrice); err != nil {
    log.Errorf("Failed to initialize lotus-storage-miner: %+v", err)
    path, err := homedir.Expand(repoPath)
    if err != nil {
        return err
    }
    log.Infof("Cleaning up %s after attempt...", path)
    if err := os.RemoveAll(path); err != nil {
        log.Errorf("Failed to clean up failed storage repo: %s", err)
    }
    return xerrors.Errorf("Storage-miner init failed")
}
```
当设置--actor参数和--genesis-miner时,初始化创世矿工时会创建一个sector storage manager,负责调度
miner ,同时会获取seal proof type 和minerID 以及Storage auth.
最终,通过full node API, epp(storage winning PoST Prover),miner address 创建一个miner
```go
func storageMinerInit(ctx context.Context, cctx *cli.Context, api lapi.FullNode, r repo.Repo, ssize abi.SectorSize, gasPrice types.BigInt) error {
	...
	if cctx.Bool("genesis-miner") {
		...
		smgr, err := sectorstorage.New(ctx, lr, stores.NewIndex(), &ffiwrapper.Config{
				SealProofType: spt,
			}, sectorstorage.SealerConfig{true, true, true, true}, nil, sa)
			if err != nil {
				return err
			}
			epp, err := storage.NewWinningPoStProver(api, smgr, ffiwrapper.ProofVerifier, dtypes.MinerID(mid))
			if err != nil {
				return err
			}
			m := miner.NewMiner(api, epp, a)
			...
			}
	...
}
```
其中又调用 createStorageMiner 此时,返回一个miner address.
```go
a, err := createStorageMiner(ctx, api, peerid, gasPrice, cctx)
```
```go
func createStorageMiner(ctx context.Context, api lapi.FullNode, peerid peer.ID, gasPrice types.BigInt, cctx *cli.Context) (address.Address, error) {
	log.Info("Creating StorageMarket.CreateStorageMiner message")

	var err error
	var owner address.Address
	if cctx.String("owner") != "" {
		owner, err = address.NewFromString(cctx.String("owner"))
	} else {
		owner, err = api.WalletDefaultAddress(ctx)
	}
	if err != nil {
		return address.Undef, err
	}

	ssize, err := units.RAMInBytes(cctx.String("sector-size"))
	if err != nil {
		return address.Undef, fmt.Errorf("failed to parse sector size: %w", err)
	}

	worker := owner
	if cctx.String("worker") != "" {
		worker, err = address.NewFromString(cctx.String("worker"))
	} else if cctx.Bool("create-worker-key") { // TODO: Do we need to force this if owner is Secpk?
		worker, err = api.WalletNew(ctx, crypto2.SigTypeBLS)
	}
	// TODO: Transfer some initial funds to worker
	if err != nil {
		return address.Undef, err
	}

	collateral, err := api.StatePledgeCollateral(ctx, types.EmptyTSK)
	if err != nil {
		return address.Undef, err
	}

	spt, err := ffiwrapper.SealProofTypeFromSectorSize(abi.SectorSize(ssize))
	if err != nil {
		return address.Undef, err
	}

	params, err := actors.SerializeParams(&power.CreateMinerParams{
		Owner:         owner,
		Worker:        worker,
		SealProofType: spt,
		Peer:          abi.PeerID(peerid),
	})
	if err != nil {
		return address.Undef, err
	}

	createStorageMinerMsg := &types.Message{
		To:    builtin.StoragePowerActorAddr,
		From:  owner,
		Value: types.BigAdd(collateral, types.BigDiv(collateral, types.NewInt(100))),

		Method: builtin.MethodsPower.CreateMiner,
		Params: params,

		GasLimit: 10000000,
		GasPrice: gasPrice,
	}

	signed, err := api.MpoolPushMessage(ctx, createStorageMinerMsg)
	if err != nil {
		return address.Undef, err
	}

	log.Infof("Pushed StorageMarket.CreateStorageMiner, %s to Mpool", signed.Cid())
	log.Infof("Waiting for confirmation")

	mw, err := api.StateWaitMsg(ctx, signed.Cid(), build.MessageConfidence)
	if err != nil {
		return address.Undef, err
	}

	if mw.Receipt.ExitCode != 0 {
		return address.Undef, xerrors.Errorf("create storage miner failed: exit code %d", mw.Receipt.ExitCode)
	}

	var retval power.CreateMinerReturn
	if err := retval.UnmarshalCBOR(bytes.NewReader(mw.Receipt.Return)); err != nil {
		return address.Undef, err
	}

	log.Infof("New storage miners address is: %s (%s)", retval.IDAddress, retval.RobustAddress)
	return retval.IDAddress, nil
}
```
结果生成Created new storage miner: t0xxxx



```sh
./lotus-storage-miner run --api 2345 --nosync --enable-gpu-proving
```
首先获取full node API,同lotus daemon 版本进行比较,并检查storage repo路径.
miner API连接远程地址,获取repo的endpoint



