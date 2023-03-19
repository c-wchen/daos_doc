## 判断请求往那个XSTREAM发送

XSTREAM 0

* 管理面
* POOL请求(创建/连接等)
* RDB
* 容器请求
* 重构
* IV状态同步
* 广播

XSTREAM 1

* 心跳请求

target XSTREAM

* IO请求
* 指定请求是发给target

```c
static inline int
daos_rpc_tag(int req_type, int tgt_idx)
{
	switch (req_type) {
	/* for normal IO request, send to the main service thread/context */
	case DAOS_REQ_IO:
	case DAOS_REQ_TGT:
		return DAOS_IO_CTX_ID(tgt_idx);
	case DAOS_REQ_SWIM:
		return 1;
	/* target tag 0 is to handle below requests */
	case DAOS_REQ_MGMT:   // 管理面
	case DAOS_REQ_POOL:  // POOL相关请求
	case DAOS_REQ_RDB:   // RDB请求
	case DAOS_REQ_CONT:    // container相关请求
	case DAOS_REQ_REBUILD:  // 重构
	case DAOS_REQ_IV:   // IV状态同步
	case DAOS_REQ_BCAST:  // 广播
		return 0;
	default:
		D_ASSERTF(0, "bad req_type %d.\n", req_type);
		return -DER_INVAL;
	};
}
```

