# Sequences

Sequence is a monotonic increasing list of numbers.

## Motivation

- [Sequences #3215](https://github.com/voedger/voedger/issues/3215)

## Requirements

QName for sequence view is defined in `appdef/sys`:

- `sys.Sequences`

QNames for sequences are defined in `appdef/sys`:

- `sys.PLogOffsetSeq`
- `sys.WLogOffsetSeq`
- `sys.RecIDSeq`
- `sys.CRecIDSeq`

To access sequences `ISequencer` interface is used.
`ISequencer` defined in `sequences` package.
`ISequencer` implemented by `Sequencer` struct. `Sequencer` struct is defined in `sequences/sequencer` package.

## Interface `ISequence`

### `sequence.New(str istructs.IAppStructs, pid istructs.PartitionID) ISequencer`

- Actor: `AppPartition`
- When: `AppPartition` is deployed
- Flow:
  - Create a new `Sequencer` instance
  - Start `recovery()`

### `ISequencer.StartEvent(wait time.Duration, ws istructs.WSID) (plogOffset, wlogOffset istructs.Offset)`

- Actor: CP
- When: CP needs to process a request
- Flow:
  - Switch on `status`:
    - case `Recover`:
      - Wait for `status` is `Ready` during duration. If wait fails, return (`istructs.NullOffset`, `istructs.NullOffset`), else fall through to `Ready`
    - case `Ready`:
      - Set status to `Eventing`
      - Strore WSID to `ws` field
      - Increment `plogOffset` and `wlogOffset[ws]` counters
      - Return (`plogOffset, wlogOffset[ws]`)
    - else panic

### `ISequencer.NextRecID() istructs.RecordID`

- Actor: CP
- When: CP needs the next ODoc/ORecord or WDoc/WRecord ID
- Flow:
  - Check `status` is `Eventing`. If not, panic
  - Increment `recID` counter
  - Return `recID`

### `ISequencer.NextCRecID() istructs.RecordID`

- Actor: CP
- When: CP needs the next CDoc/CRecord ID
- Flow:
  - Check `status` is `Eventing`. If not, panic
  - Increment `cRecID[ws]` counter
  - Return `cRecID[ws]`

### `ISequencer.FinishEvent()`

- Actor: CP
- When: CP stores PLogEvent record
- Flow:
  - Check `status` is `Eventing`. If not, panic
  - Start `flush()`
  - Set `status` to `Ready`

### `ISequencer.CancelEvent()`

- Actor: CP
- When: CP fail with PLogEvent record
- Flow:
  - Check status is `Eventing`. If not, panic
  - Start `recovery()`

## Implementation `Sequencer`

### `Sequencer.pid istructs.PartitionID`

- Initialized from `sequence.New()`

### `Sequencer.vr istructs.ViewRecords`

- Initialized from `sequence.New()`

### `Sequencer.status uint8`

- `Recover`
  - assigned from `recovery()`
- `Ready`
  - assigned from `recovery()` after success finished
  - assigned from `FinishEvent()` after `flush()`
- `Eventing`
  - assigned from `StartEvent()`

### `Sequencer.plogOffset istructs.Offset`

- Assigned from `recovery()`
- Incremented by 1 in `StartEvent()`

### `Sequencer.ws istructs.WSID`

- Assigned from `StartEvent()`

### `Sequencer.wlogOffset map[istructs.WSID]istructs.Offset`

- Assigned from `recovery()`
- Incremented by 1 in `StartEvent()`

### `Sequencer.recID map[istructs.WSID]istructs.RecordID`

- Assigned from `recovery()`
- Incremented by 1 in `NextRecID()`

### `Sequencer.cRecID map[istructs.WSID]istructs.RecordID`

- Assigned from `recovery()`
- Incremented by 1 in `NextCRecID()`

### `Sequencer.recovery()`

- When:
  - `sequence.New()`
  - `Sequencer.CancelEvent()`
- Flow:
  - Set status `Recover`
  - starts go routine, which:
    - Read system view `sys.Sequences` with PK `pid`, for each record:
      - switch `Seq` field value:
        - case `sys.PLogOffsetSeq`:
          - Set `plogOffset` from `Value` field
        - case `sys.WLogOffsetSeq`:
          - Set `wlogOffset[ws]` for `ws` from `WSID` field from `Value` field
        - case `sys.RecIDSeq`:
          - Set `recID[ws]` for `ws` from `WSID` field from `Value` field
        - case `sys.CRecIDSeq`:
          - Set `cRecID[ws]` for `ws` from `WSID` field from `Value` field
    - Read 
    - Set status `Ready`


### Sequencer.flush()


### Structs

- IDBatch
    - PLogOffset
    - `map[{WSID, QName}, istructs.RecordID]`

### NextPLogOffset(PartitionID, duration) Offset

- Actor: CP
- When: CP needs to process a request
- Flow:
    - `Status[PartitionID] == Clean && !IsRecoveryRunning(PartitionID)`):
        - Set `Status[PartitionID]` to `InProcess`
        - Return NextPLogOffset
    - `IsRecoveryRunning(PartitionID)`: 
        - Wait for duration
        - If wait fails, return 0
        - Repeat Flow
    - `Status[PartitionID] == InProcess`: panic

### NextInSequence(PartitionID, WSID, QName) ID

- Actor: CP
- When: CP needs the next number in a sequence.
- Flow:
    - `Status[PartitionID] == InProcess`: 
        - Generate the next ID
    - panic

### Flush(PartitionID) IDBatch

- Actor: CP
- When: After CP saves the PLog record successfully
- Flow
    - `Status[PartitionID] == InProcess`:
        - Include all generated IDs into IDBatch and send it to the flusher routine for FlushBatch
        - `Status[PartitionID] = Clean`
    - panic

### Invalidate(PartitionID) IDBatch

- Actor: CP
- When: After CP fails to save the PLog record
- Flow:
    - `Status[PartitionID] == Dirty`:
        - ???
        - ???
    - panic
    - Flow: `startRecovery(PartitionID)`

### startRecovery(PartitionID)

???
- When: Partition is deployed or invalidated
- Only one routine per PartitionID is allowed
- Read recovery PLog position (recoveryOffset)
- Run projector starting from recoveryOffset
- Start rproutine

### recovery(PartitionID)

Flow:
  - ??? Should repeat

## Components

- pkg/isequencer
    - ISequencer
- pkg/isequencer/sequencer
    - provide: Factory(PartitionID) cleanup()
        - Start recovery
        - cleanup should stop flush/recovery and wait
- appparts.AppPartitions
    - IAppPqrtition.Sequencer() ISequencer
    - On deploy starts recovery
- QNames
    - appdef/sys

