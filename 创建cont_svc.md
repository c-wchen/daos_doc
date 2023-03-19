## CONT_CREATE

### 1. 客户端操作

> 理论上连接上pool之后, pool leader已经知道, 后续直接用缓存就行了,但是也可能leader任期已经过了,导致需要重新选择

dc_cont_create

​	dc_pool_choose_svc_rank

​		rsvc_client_choose

​	  cont_req_create(daos_task2ctx(task), &ep, CONT_CREATE, &rpc): 创建cont svc, 最终只会发送给pool svc

### 2. 服务务端操作

**ds_cont_op_handler**

​	cont_svc_lookup_leader: 判断当前是否是leader,并将pool leader设置到cont_hint中,用于告诉客户端pool leader是谁

​		cont_op_with_svc

​			cont_create // 参考下面cont_create

​	ds_rsvc_set_hint

​	cont_svc_put_leader

**cont_create**

​	daos_prop_dup(&cont_prop_default, false, false): 填充默认属性

​	cont_create_prop_prepare:  填充客户端传过来的属性

​	rdb_tx_create_kvs: 创建存储容器的RDB KVS

​	cont_prop_write: 将属性写入到KVS, 其中key通过RDB_STRING_KEY进行定义

**在cont_create中创建4个KVS**

* svc->cs_conts

* ds_cont_prop_snapshots

* ds_cont_attr_user

* ds_cont_prop_handles

## container属性

参考: enum daos_cont_props

```c
DAOS_PROP_CO_LABEL    // 容器名  
DAOS_PROP_CO_REDUN_FAC  // 冗余级别
DAOS_PROP_CO_ACL    // 权限级别, 同linux权限级别一致／ower:users:groups
DAOS_PROP_CO_EC_CELL_SZ  // EC的fen
...
```



