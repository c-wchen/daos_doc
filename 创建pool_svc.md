

## 选择rank创建pool svc

daos_server远程调用DRPC_METHOD_MGMT_POOL_CREATE

ds_mgmt_drpc_pool_create->ds_mgmt_create_pool->ds_mgmt_pool_svc_create

​	ds_pool_svc_dist_create

​		select_svc_ranks: 根据pool的冗余因子svc_rf创建pool svc的rank

​		ds_rsvc_dist_start

​			bcast_create(RSVC_START...ranks): 想所有的rank的xstream 0广播

​       (d_rank_list_t *)ranks, &rpc);

​		rsvc_client_choose: 如果创建失败, 根据服务端返回的pool leader或者下一个rank进行建连操作

​	pool_req_create(info->dmi_ctx, &ep, POOL_CREATE, &rpc);

**select_svc_ranks**

* 冗余因子默认是2(DAOS_PROP_PO_SVC_REDUN_FAC_DEFAULT),具体需要看管里面传过来的numsvcreps参数
* 创建pool svc的个数 = 冗余因子 * 2 + 1, 当然
* 当前选择就是选择所有target的前面几个,后续可能需要根据故障域进行选择

## rank确认是否是pool leader

> 注意: 
>
> * 同一个rank可能承载不同pool服务
> * pool leader只能在承载RDB的rank上产生,也就是pool svc

pool_svc_lookup_leader: 其中类型rsvc_hint参数表示命中的leader, 用于告诉客户端pool leader是谁

​	ds_pool_failed_lookup: 查找该pool已经失败,如果失败直接返回DER_NO_SERVICE

​	ds_rsvc_lookup_leader

​		ds_rsvc_lookup: 先从rsvc_hash缓存中获取,如果没有获取到通过rdb路径/mnt/daos/pool_uuid/rdb-pool判断是否已经启动rdb,如果rdb启动,返回成功找到

**rdb路径**

/mnt/daos/pool_uuid/rdb-pool

## 启动rsvc/pool svc

调用RPC/RSVC_START创建rsvc服务

ds_rsvc_start_handler

​	ds_rsvc_start

​		start

​			rdb_start

​				rdb_raft_start

​		d_hash_rec_insert(&rsvc_hash..)：添加rdb entry缓存 

## pool_module模块加载

pool_module加载时扫描/mnt/daos/路径下目录生成pool uuid, 如果pool或者rsvc没启动,重新启动

**sm_setup->setup**

​	ds_pool_start_all

​		pool_start_all

​			ds_mgmt_tgt_pool_iterate(start_one, NULL /* arg */);

​				common_pool_iterate(dss_storage_path, cb, arg) : dss_storage_path = "mnt/daos/"

​					opendir->readdir->uuid_parse: 根据目录名解析pool uuid, 然后调用start_one回调

**start_one**

​	ds_pool_start

​	pool_svc_rdb_path->ds_rsvc_start:  如果rdb也存在, 启动rsvc

> 注意:
>
> 每个pool都会启动不同的rsvc rdb, 多个pool可能会在同一个节点创建各自pool的rdb; 所以在这步启动的rdb是其他pool在这个节点的rdb, 不是自己的.

​		



​	