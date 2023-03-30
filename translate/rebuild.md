# Self-healing (aka Rebuild)

在DAOS中，如果数据在不同的目标上进行多副本复制，一旦其中一个目标失效，该目标的数据将自动在其他目标上重新构建，因此不会因目标失效而影响数据冗余。在未来的版本中，dao还将支持纠删编码以保护数据;然后可以相应地更新重建过程。
## Rebuild Detection

当目标发生故障时，应及时检测并通知池(Raft) leader，然后leader将目标从池中排除并立即触发重建过程。

### Current status and long-term goal

目前，由于raft leader不能自动排除目标，sysadmin必须手动从池中排除目标，然后触发重建。

将来，leader应该能够及时检测目标故障，然后在没有系统管理员帮助的情况下自动触发重建。

## Rebuild process

The rebuild is divided into 2 phases, scan and pull.

### Scan

最初，leader将通过一个集合RPC将失败通知传播给所有幸存的目标。任何接收到此RPC的目标将开始扫描其对象表，以确定在故障目标上丢失数据冗余的对象。如果是，将它们的id和相关元数据发送到重建目标(重建启动器)。至于如何为错误目标选择重建目标，将在placement/README.md中描述

### Pull

一旦重建启动器从扫描目标获取了对象列表，它将从其他副本获取这些对象的数据，然后在本地写入数据。每个目标将报告其重建状态，重建对象，记录，is_finished?等，给池领导。一旦leader知道所有目标都完成了扫描和重建阶段，它将通知所有目标重建已经完成，并且它们可以释放重建过程中持有的所有资源。s.

<a id="f10.18"></a>
**Rebuild Protocol**
![../../docs/graph/Fig_059.png](../../docs/graph/Fig_059.png "Rebuild Protocol")

上图就是这个过程的一个例子:集群中有5个对象:对象A是3路复制的，对象B、C、D和E是2路复制的。当target-2失败时，作为Raft leader的target-0将失败广播给所有幸存的target，通知它们进入降级模式并进行扫描:

- Target-0 found that object D lost a replica and calculated out target-1 is
the rebuild target for D, so it sent object D's ID and its metadata to
target-1.
- Target-1 found that object A lost a replica and calculated out target-3 is
the rebuild target for A, so it sent object A's ID and its metadata to
target-3.
- Target-4 found objects A and C lost replicas and it calculated out target-3
is the rebuild target for both objects A and C, so it sent IDs for objects A
and C and their metadata to target-3.
- After receiving these object IDs and their metadata, target-1 and target-3
can compute out surviving replicas of these objects, and rebuild these objects
by pulling data from these replicas.

### Multiple pool and targets rebuild

在大规模存储集群中，当从前一个故障进行重建时，可能会发生多个故障。在这种情况下，DAOS既不应该同时处理这些故障，也不应该中断并为以后的故障重置早期的重建进度。否则，为每个故障进行重建所消耗的时间可能会显著增加，并且如果新的故障与正在进行的重建重叠，重建可能永远不会结束。因此，对于多个故障，这些规则适用

- If the rebuild initiator fails during rebuild, then the object shards
being rebuilt on the initiator should be ignored, which will be handled
by next rebuild.
- If rebuild initiator can not fetch the data from other replicas due to
the failure, it will switch to other replicas if available.
- A target in rebuild does not need to re-scan its objects or reset rebuild
progress for the current failure if another failure has occurred.
- When there are multiple failures, if the number of failed targets from
different domains exceeds the fault tolerance level, there could be
unrecoverable errors and applications could suffer from data loss. In this
case, upper layer stack software could see errors while sending I/O to the
object that could have missing data.

The following <a href="#f10.20">figure</a> is an example of this protocol.

<a id="f10.20"></a>
**Multi-failure protocol**
![../../docs/graph/Fig_061.png](../../docs/graph/Fig_061.png "Multi-failure protocol")

- In this example, object A is 2-way replicated, object B, C and D are 3-way
replicated.
- After failure of target-1, target-2 is the initiator of rebuilding object B,
it is pulling data from target-3 and target-4; target-3 is the initiator of
rebuilding object C, it is pulling data from target-0 and target-2.
- Target-3 failed before completing rebuild for target-1, so rebuild of object
C should be abandoned at this point, because target-3 is the initiator of it.
The missing data redundancy of object C will be reconstructed while rebuilding
target-3.
- Because target-3 is also contributor of rebuilding object B, based on the
protocol, the initiator of object B, which is target-2, should switch to
target-4 and continue rebuild of object B.
- Rebuild process of target-1 can complete after finishing rebuild of object B.
By this time, object C still lost a replica. This is quite understandable,
because even if two failures have no overlap, object C will still lose the
replica on target-3.
- In the process of rebuilding target-3, target-4 is the new initiator of
rebuilding object C.

If there are multiple pools being impacted by the failing
target, these pools can be rebuilt concurrently.

## I/O during rebuild

如果在重建期间有并发写，重建协议应该保证新的写永远不会丢失。这些写入要么直接存储在新的对象分片中，要么由重建启动器拉取到新的对象分片中。fetch也应该保证得到正确的数据。为了实现这些，这些协议被应用

1. Fetch will always skip the rebuilding target.
2. Update can complete only if updates of all the object shards have
successfully completed.
- If any of these updates failed, the client will infinitely retry until
it succeeds, or there is a pool map change which shows the target failed.
In the second case, the client will switch to the new pool map, and send
the update to the new destination, which is the rebuild target of this object.
3. There are no synchronization between normal I/O and rebuild process, so during
rebuild process, the data might be written duplicately by rebuild initiator and
normal I/O.

## Rebuild resource throttle

在重建过程中，用户可以设置节流以保证重建不会使用比用户设置更多的资源。用户目前只能设置CPU周期。例如，如果用户将节流设置为50，那么重建最多将使用50%的CPU周期来完成重建工作。CPU周期的默认重建节流是30。

## Rebuild status

As described earlier, each target will report its rebuild status to
the pool leader by IV, then the leader will summurize the status of
all targets, and print out the whole rebuild status by every 2 seconds,
for example these messages.

```
Rebuild [started] (pool 8799e471 ver=41)
Rebuild [scanning] (pool 8799e471 ver=41, toberb_obj=0, rb_obj=0, rec= 0, done 0 status 0 duration=0 secs)
Rebuild [queued] (419d9c11 ver=2)
Rebuild [started] (pool 419d9c11 ver=2)
Rebuild [scanning] (pool 419d9c11 ver=2, toberb_obj=0, rb_obj=0, rec= 0, done 0 status 0 duration=0 secs)
Rebuild [pulling] (pool 8799e471 ver=41, toberb_obj=75, rb_obj=75, rec= 11937, done 0 status 0 duration=10 secs)
Rebuild [completed] (pool 419d9c11 ver=2, toberb_obj=10, rb_obj=10, rec= 1026, done 1 status 0 duration=8 secs)
Rebuild [completed] (pool 8799e471 ver=41, toberb_obj=75, rb_obj=75, rec= 13184, done 1 status 0 duration=14 secs)
```

There are 2 pools being rebuilt (pool 8799e471 and pool 419d9c11,
note: only first 8 letters of the pool uuid are shown here).

```
The 1st line means the rebuild for pool 8799e471 is started, whose pool map version is 41.
The 2nd line means the rebuild for pool 8799e471 is in scanning phase, and no objects & records are being rebuilt yet.
The 3rd line means a rebuild job for pool 419d9c11 is being queued.
The 4th line means the rebuild for pool 419d9c11 is started, whose pool map version is 2.
The 5th line means the rebuild for pool 419d9c11 is in scanning phase, and no objects & records are being rebuilt yet.
The 6th line means the rebuild for pool 8799e471 is in pulling phase, and there are 75 objects to be rebuilt(toberb_obj=75), and all of them are rebuilt(rb_obj=75), but records rebuilt for these objects are not finished yet(done 0) and only 11937 records (rec = 11937) are rebuilt.
The 7th line means the rebuild for pool 419d9c11 is done (done 1), and there are totally 10 objects and 1026 records are rebuilt, which costs about 8 seconds.
The 8th line means the rebuild for pool 8799e471 is done (done 1), and there are totally 75 objects and 13184 records are rebuilt, which costs about 14 seconds.
```

During the rebuild, if the client query the pool status to the pool leader,
which will return its rebuild status to client as well.

```C
struct daos_rebuild_status {
	/** pool map version in rebuilding or last completed rebuild */
	uint32_t		rs_version;
	/** Time (Seconds) for the rebuild */
	uint32_t		rs_seconds;
	/** errno for rebuild failure */
	int32_t			rs_errno;
	/**
	 * rebuild state, DRS_COMPLETED is valid only if @rs_version is non-zero
	 */
	union {
		int32_t			rs_state;
		int32_t			rs_done;
	};

	/* padding of rebuild status */
	int32_t			rs_padding32;

	/* Failure on which rank */
	int32_t			rs_fail_rank;
	/** # total to-be-rebuilt objects, it's non-zero and increase when
	 * rebuilding in progress, when rs_state is DRS_COMPLETED it will
	 * not change anymore and should equal to rs_obj_nr. With both
	 * rs_toberb_obj_nr and rs_obj_nr the user can know the progress
	 * of the rebuilding.
	 */
	uint64_t		rs_toberb_obj_nr;
	/** # rebuilt objects, it's non-zero only if rs_state is completed */
	uint64_t		rs_obj_nr;
	/** # rebuilt records, it's non-zero only if rs_state is completed */
	uint64_t		rs_rec_nr;

	/** rebuild space cost */
	uint64_t		rs_size;
};
```

## Rebuild failure

If the rebuild is failed due to some failures, it will be aborted, and the
related message will be shown on the leader console. For example:

```
Rebuild [aborted] (pool 8799e471 ver=41, toberb_obj=75, rb_obj=75, rec= 11937, done 1 status 0 duration=10 secs)
```

## Rebuilding with Checksums

在重建期间，正在重建的服务器将充当DAOS客户机，因为它将从副本服务器读取数据和校验和，并在将数据用于重建之前验证数据的完整性。如果检测到损坏的数据，那么读取将失败，副本服务器将收到损坏的通知。然后重建将尝试使用不同的副本。

checksum iov参数可用于对象列表和对象获取任务API。这是为了重建提供可以装入校验和的内存。否则，重建将不得不在写入本地VOS实例时重新计算校验和。如果在缓冲区中分配的内存不足，iov_len将设置为所需的容量，并截断填充到缓冲区中的校验和。

下面描述了rebui校验和生命周期的“接触点”


### Rebuild Touch Points

- migrate_fetch_update_(inline|single|bulk) - the rebuild/migrate functions that
  write to vos locally must ensure that the checksum is also written. These must
  use the csum iov param for fetch to get the checksum, then unpack the csums
  into iod_csum.
- obj_enum.c is relied on for enumerating the objects to be rebuilt. Because the
  fetch_update functions will unpack the csums from fetch, it will also unpack
  the csums for enum, so the unpacking process in obj_enum.c will simply copy
  the csum_iov to the io (dc_obj_enum_unpack_io) structure in
  **enum_unpack_recxs()** and then deep copy to the mrone (migrate_one)
  structure in **migrate_one_insert()**.

### Client Task API Touch Points

- **dc_obj_fetch_task_create**: sets csum iov to daos_obj_fetch_t args. These
  args are set to the rw_cb_args.shard_args.api_args and accessed through an
  accessor function (rw_args2csum_iov) in cli_shard.c so that rw_args_store_csum
  can easily access it. This function, called from dc_rw_cb_csum_verify, will
  pack the data checksums received from the server into the iov.
- **dc_obj_list_obj_task_create**: sets csum iov to daos_obj_list_obj_t args.
  args.csum is then copied to obj_enum_args.csum in dc_obj_shard_list(). On enum
  callback (dc_enumerate_cb()) the packed csum buffer is copied from the rpc
  args to obj_enum_args.csum (which points to the same buffer as the caller's)

### Packing/unpacking checksums

当校验和被打包(无论是用于fetch还是object list)时，只包括数据校验和。对于对象列表，只包含内联数据的校验和。在重建过程中，如果数据没有内联，那么重建过程将获取剩余的数据并获得校验和。

- ci_serialize() - "packs" checksums by appending the struct to an iov and then
  appending the checksum info buffer to the iov. This puts the actual checksum
  just after the checksum structure that describes the checksum.
- ci_cast() - "unpacks" the checksum and describing structure. It does this by
  casting an iov's buffer to a dcs_csum_info struct and setting the csum_info's
  checksum pointer to point to the memory just after the structure. It does not
  copy anything, but really just "casts". To get all dcs_csum_infos, a caller
  would cast the iov, copy the csum_info to a destination, then move to the next
  csum_info(ci_move_next_iov) in the iov. Because this process modifies the iov
  structure it is best to use a copy of the iov as a temp structure.
