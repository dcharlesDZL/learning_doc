### state-machine 学习
File: **github.com/filecoin-project/storage-fsm/fsm.go**

```go
func (m *Sealing) Plan(events []statemachine.Event, user interface{}) (interface{}, uint64, error) {
	next, err := m.plan(events, user.(*SectorInfo))
	if err != nil || next == nil {
		return nil, uint64(len(events)), err
	}

	return func(ctx statemachine.Context, si SectorInfo) error {
		err := next(ctx, si)
		if err != nil {
			log.Errorf("unhandled sector error (%d): %+v", si.SectorNumber, err)
			return nil
		}

		return nil
	}, uint64(len(events)), nil // TODO: This processed event count is not very correct
}
```
next, err := m.plan(events, user.(*SectorInfo)) 这个方法是根据events来选择接下来要执行的步骤.

**状态机的events从哪里来?是什么?**

```go
func (m *Sealing) plan(events []statemachine.Event, state *SectorInfo) (func(statemachine.Context, SectorInfo) error, error) {
	/////
	// First process all events

	for _, event := range events {
		e, err := json.Marshal(event)
		if err != nil {
			log.Errorf("marshaling event for logging: %+v", err)
			continue
		}

		l := Log{
			Timestamp: uint64(time.Now().Unix()),
			Message:   string(e),
			Kind:      fmt.Sprintf("event;%T", event.User),
		}

		if err, iserr := event.User.(xerrors.Formatter); iserr {
			l.Trace = fmt.Sprintf("%+v", err)
		}

		state.Log = append(state.Log, l)
	}

	p := fsmPlanners[state.State]
	if p == nil {
		return nil, xerrors.Errorf("planner for state %s not found", state.State)
	}

	if err := p(events, state); err != nil {
		return nil, xerrors.Errorf("running planner for state %s failed: %w", state.State, err)
	}

	/////
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

	switch state.State {
	// Happy path
	case Packing:
		return m.handlePacking, nil
	case PreCommit1:
		return m.handlePreCommit1, nil
	case PreCommit2:
		return m.handlePreCommit2, nil
	case PreCommitting:
		return m.handlePreCommitting, nil
	case WaitSeed:
		return m.handleWaitSeed, nil
	case Committing:
		return m.handleCommitting, nil
	case CommitWait:
		return m.handleCommitWait, nil
	case FinalizeSector:
		return m.handleFinalizeSector, nil
	case Proving:
		// TODO: track sector health / expiration
		log.Infof("Proving sector %d", state.SectorNumber)

	// Handled failure modes
	case SealFailed:
		return m.handleSealFailed, nil
	case PreCommitFailed:
		return m.handlePreCommitFailed, nil
	case ComputeProofFailed:
		return m.handleComputeProofFailed, nil
	case CommitFailed:
		return m.handleCommitFailed, nil

		// Faults
	case Faulty:
		return m.handleFaulty, nil
	case FaultReported:
		return m.handleFaultReported, nil

	// Fatal errors
	case UndefinedSectorState:
		log.Error("sector update with undefined state!")
	case FailedUnrecoverable:
		log.Errorf("sector %d failed unrecoverably", state.SectorNumber)
	default:
		log.Errorf("unexpected sector update state: %d", state.State)
	}

	return nil, nil
}
```
m.plan方法有两个参数,分别为发生的events和扇区信息SectorInfo,返回值为处理event的函数(如m.handlerPreCommit1)和error.
函数体主要分为两部分,第一部分主要是处理所有的events,包括log记录,

planOne() 首先判断events是否存在globalMutator
##
##
##
##
##
##
##
##
##
##
##
##




### 只进行Commit2操作
File: ./lotus/cmd/lotus-bench/main.go

**在执行lotus-bench sealing命令时需要加上 --save-commit2-input c2input.json 来保存commit2 input,将c2in的内容写入c2input文件中.**
```sh
# export FIL_PROOFS_PARAMETER_CACHE="/home/ps/filecoin-proof-parameters"
# ./bench-opt sealing --sector-size 8MiB --miner-addr=t01000 --save-commit2-input c2input_8MB_ --skip-unseal
./bench-opt sealing --sector-size 32GiB --save-commit2-input c2input_32GB.json --skip-unseal --skip-commit2
```
在./lotus-bench/main.go中的runSeal函数可以看出来,Phase1Out由sb.SealCommit1得到
```go
c1o, err := sb.SealCommit1(context.TODO(), sid, ticket, seed.Value, pieces, cids)
```
在拿到c1o之后,将c1o保存至Commit2In的结构体中,再写入 c2input.json文件中.
```go
if saveC2inp != "" {
	c2in := Commit2In{
		SectorNum:  int64(i),
		Phase1Out:  c1o,
		SectorSize: uint64(sectorSize),
	}

	b, err := json.Marshal(&c2in)
	if err != nil {
		return nil, nil, err
	}

	if err := ioutil.WriteFile(saveC2inp, b, 0664); err != nil {
		log.Warnf("%+v", err)
	}
}
```
Commit2In结构体定义如下:
```go
type Commit2In struct {
	SectorNum  int64
	Phase1Out  []byte
	SectorSize uint64
}
```
输出的内容为json文件{"SectorNum":1,"Phase1Out":"eyJyZWdpc3RlcmVkX3Byb29mIjoiU3RhY2tlZERyZzUxMk1pQlYxIiwid...==","SectorSize":536870912}
sectorsize 8MB的时候,c2input.json大小200k左右,sectorsize大小为512MB时候,大小为260k左右.




当在运行lotus-bench prove c2input 命令时,程序会读取c2input文件将写入其中的字节流读取出来
```go
paths, done, err := sb.sectors.AcquireSector(ctx, sector, stores.FTSealed|stores.FTCache, 0, true)
```

lotus-bench prove 命令详解
```sh
./bench-new prove c2input_8MB_ 
```
proof: a1f92f6d8537779400b8867803794ca92cd5a0ecab367437a42de52a9ab42eec0d5519cc00b18b16057e19047fdabcc88079f0e9c4e6c74eff727545e1c028459ac682391a5072b949150ce1dcf794aacae1c7f322531be19c558b8bc3acbae119cd1bd34fd6699221ef3ad0c5725ed1ceb7ddc41372991a5134bcf5a61123f2b987e1265c4a43dbe195d649b1838dfb8fbc8cf0e46c61d9285539e7b6df4f2bfc0ede74c7fe725f91458a20a6da00f7773f87ed9b0caaa153e7346cfaca12ff

results (v27) (8388608)
seal: commit phase 2: 12.564761678s (652 KiB/s)


读取parameters.json中匹配c2in.SectorSize的sector信息
```go
if err := paramfetch.GetParams(build.ParametersJson(), c2in.SectorSize); err != nil {
	return xerrors.Errorf("getting params: %w", err)
}
```
获取minerID
```go
maddr, err := address.NewFromString(c.String("miner-addr"))
mid, err := address.IDFromAddress(maddr)
```
通过commit2input中的SectorSize,拿到SealProofType并且生成一个Sealler,相关代码如下:
```go
spt, err := ffiwrapper.SealProofTypeFromSectorSize(abi.SectorSize(c2in.SectorSize))
cfg := &ffiwrapper.Config{
	SealProofType: spt,
}
sb, err := ffiwrapper.New(nil, cfg)
```
最后调用sb.SealCommit2函数生成proof证明
```go
proof, err := sb.SealCommit2(context.TODO(), abi.SectorID{Miner: abi.ActorID(mid), Number: abi.SectorNumber(c2in.SectorNum)}, c2in.Phase1Out)
```
sb.SealCommit2调用ffi.SealCommitPhase2函数
```go
func (sb *Sealer) SealCommit2(ctx context.Context, sector abi.SectorID, phase1Out storage.Commit1Out) (storage.Proof, error) {
	return ffi.SealCommitPhase2(phase1Out, sector.Number, sector.Miner)
}
```
```go
// SealCommitPhase2
func SealCommitPhase2(
	phase1Output []byte,
	sectorNum abi.SectorNumber,
	minerID abi.ActorID,
) ([]byte, error) {
	proverID, err := toProverID(minerID)
	if err != nil {
		return nil, err
	}

	resp := generated.FilSealCommitPhase2(string(phase1Output), uint(len(phase1Output)), uint64(sectorNum), proverID)
	resp.Deref()

	defer generated.FilDestroySealCommitPhase2Response(resp)

	if resp.StatusCode != generated.FCPResponseStatusFCPNoError {
		return nil, errors.New(generated.RawString(resp.ErrorMsg).Copy())
	}

	return []byte(toGoStringCopy(resp.ProofPtr, resp.ProofLen)), nil
}
```

通过minerID拿到proverID	, 再通过调用C库函数generated.FilSealCommitPhase2 传入c1o和sectorNumber和proverID得到resp

```go
resp := generated.FilSealCommitPhase2(string(phase1Output), uint(len(phase1Output)), uint64(sectorNum), proverID)
```



```go
// SealCommitPhase1
func SealCommitPhase1(
	proofType abi.RegisteredProof,
	sealedCID cid.Cid,
	unsealedCID cid.Cid,
	cacheDirPath string,
	sealedSectorPath string,
	sectorNum abi.SectorNumber,
	minerID abi.ActorID,
	ticket abi.SealRandomness,
	seed abi.InteractiveSealRandomness,
	pieces []abi.PieceInfo,
) (phase1Output []byte, err error) {
	sp, err := toFilRegisteredSealProof(proofType)
	if err != nil {
		return nil, err
	}

	proverID, err := toProverID(minerID)
	if err != nil {
		return nil, err
	}

	commR, err := to32ByteCommR(sealedCID)
	if err != nil {
		return nil, err
	}

	commD, err := to32ByteCommD(unsealedCID)
	if err != nil {
		return nil, err
	}

	filPublicPieceInfos, filPublicPieceInfosLen, err := toFilPublicPieceInfos(pieces)
	if err != nil {
		return nil, err
	}

	resp := generated.FilSealCommitPhase1(sp, commR, commD, cacheDirPath, sealedSectorPath, uint64(sectorNum), proverID, to32ByteArray(ticket), to32ByteArray(seed), filPublicPieceInfos, filPublicPieceInfosLen)
	resp.Deref()

	defer generated.FilDestroySealCommitPhase1Response(resp)

	if resp.StatusCode != generated.FCPResponseStatusFCPNoError {
		return nil, errors.New(generated.RawString(resp.ErrorMsg).Copy())
	}

	return []byte(toGoStringCopy(resp.SealCommitPhase1OutputPtr, resp.SealCommitPhase1OutputLen)), nil
}
```