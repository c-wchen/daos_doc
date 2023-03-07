## 初始化HG_Init_opt

获取info_string

rc = crt_get_info_string(primary, provider, &info_string, idx): 其中provider来自于enum crt_provider_t

* crt_provider_name_get: provider_name来自于struct crt_na_dict crt_na_dict[] ，ofi+sockets
* crt_provider_ctx0_port_get： 获取端口
* crt_provider_domain_get：
* crt_provider_ip_str_get

## 注册HG_Register

```c
rc = crt_hg_reg(hg_class, CRT_HG_RPCID,
                (crt_proc_cb_t)crt_proc_in_common,
                (crt_proc_cb_t)crt_proc_out_common,
                (crt_hg_rpc_cb_t)crt_rpc_handler_common);

```

crt_hg_reg: 调用Mecury的注册接口HG_Register

* crt_rpc_handler_common(hg_handle_t hg_hdl): lookup对应的处理函数

  * opc_info = crt_opc_lookup(crt_gdata.cg_opc_map, opc, CRT_UNLOCK)：获取opc_info

  * crt_rpc_common_hdlr
    * crt_ctx->cc_rpc_cb：调用处理函数

## 消息转发

crt_req_send: 发送一个请求

* crt_req_send_internal
  * crt_req_send_immediately
    * crt_hg_req_create
      * HG_Create
    * crt_hg_req_send
      * HG_Forward

## CaRT消息序列化

```c
/* common for update/fetch */
#define DAOS_ISEQ_OBJ_RW	/* input fields */		 \
	((struct dtx_id)	(orw_dti)		CRT_RAW) \
	((daos_unit_oid_t)	(orw_oid)		CRT_RAW) \
	((uuid_t)		(orw_pool_uuid)		CRT_VAR) \
	((uuid_t)		(orw_co_hdl)		CRT_VAR) \
	((uuid_t)		(orw_co_uuid)		CRT_VAR) \
	((uint64_t)		(orw_epoch)		CRT_VAR) \
	((uint64_t)		(orw_epoch_first)	CRT_VAR) \
	((uint64_t)		(orw_api_flags)		CRT_VAR) \
	((uint64_t)		(orw_dkey_hash)		CRT_VAR) \
	((uint32_t)		(orw_map_ver)		CRT_VAR) \
	((uint32_t)		(orw_nr)		CRT_VAR) \

// 根据定义的DAOS_ISEQ_OBJ_RW生成对应的消息结构体和消息编解码函数
CRT_RPC_DECLARE(obj_rw,		DAOS_ISEQ_OBJ_RW, DAOS_OSEQ_OBJ_RW)
```

### 生成消息结构体

CRT_RPC_DECLARE ->

CRT_GEN_STRUCT(rpc_name##_in_packed, fields_in)

```c
struct obj_rw_in {
    struct dtx_id orw_dti;
    daos_unit_oid_t orw_oid;
    ....
}
```

如果类型时CRT_ARRAY，会生成类似数组结构

```c
struct {						
    uint64_t		 ca_count;		
    CRT_GEN_GET_TYPE(seq)	*ca_arrays;		
},
```



### 生成消息处理函数

CRT_RPC_DEFINE(obj_rw, DAOS_ISEQ_OBJ_RW, DAOS_OSEQ_OBJ_RW)->

CRT_RPC_DEFINE->

CRT_GEN_PROC_FUNC

```c
static int
crt_proc_obj_rw(crt_proc_t proc, struct type_name *ptr) {
    // 迭代处理obj_rw_in结构体中的字段
    BOOST_PP_REPEAT(BOOST_PP_SEQ_SIZE(seq), CRT_GEN_PROC_FIELDS, seq)
}

// 参考
rc = daos_sgls_copy_data_out(rw_args->rwaa_sgls,
                             orw->orw_nr,
                             orwo->orw_sgls.ca_arrays,
                             orwo->orw_sgls.ca_count);
```

### 序列化的4种格式

\#define CRT_VAR   (0)

基本变量类型处理，在处理函数中调用crt_proc_uint16_t、crt_proc_uint32_t、crt_proc_uint64_t等进行处理

\#define CRT_PTR   (1)

生成结构体会在结构体时会在变量前面添加```*```

\#define CRT_ARRAY  (2)

会根据ca_count处理每个数组元素，如果数组中的元素时其他数组类型时，也需要写一个crt_proc_xxx的处理函数

\#define CRT_RAW   (3)

直接内存拷贝





