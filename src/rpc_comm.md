## 初始化HG_Init_opt

获取info_string

rc = crt_get_info_string(primary, provider, &info_string, idx): 其中provider来自于enum crt_provider_t

* crt_provider_name_get: provider_name来自于struct crt_na_dict crt_na_dict[] ，ofi+sockets
* crt_provider_ctx0_port_get： 获取端口，默认31316
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



