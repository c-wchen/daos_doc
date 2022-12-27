## dss_module_init_one

 注册RPC处理器

```c

for (i = 0; i < smod->sm_proto_count; i++) {
    // 用于注册客户端和服务端的RPC处理
    rc = daos_rpc_register(smod->sm_proto_fmt[i], smod->sm_cli_count[i],
                           smod->sm_handlers[i], smod->sm_mod_id);
    if (rc) {
        D_ERROR("failed to register RPC for %s: "DF_RC"\n",
                smod->sm_name, DP_RC(rc));
        D_GOTO(err_mod_init, rc);
    }
}
```

daos_rpc_register

		* 将处理列表填充到proto_fmt参数中
  * crt_proto_register(proto_fmt)
    * crt_proto_register_common(cpf)
      * crt_proto_reg_L1(crt_gdata.cg_opc_map, cpf): crt_gdata.cg_opc_map是一个全局容器
        * index = cpf->cpf_base >> 24, 从cpf_base中获取模块ID， 在从map中获取处L2_map
        * crt_proto_reg_L2
          * get_L3_map: 从L2_map通过cpf_ver获取L3_map
            * crt_opc_reg_internal
              * crt_opc_reg(opc, prf_hdlr, prf_co_ops)



crp_pub



cpf_base的计算，对于obj_module'

```c
cpf_base  = DAOS_RPC_OPCODE(0, DAOS_OBJ_MODULE, 0)
    
#define DAOS_RPC_OPCODE(opc, mod_id, rpc_ver)			\
	((opc & OPCODE_MASK) << OPCODE_OFFSET |			\
	 (rpc_ver & RPC_VERSION_MASK) << RPC_VERSION_OFFSET |	\
	 (mod_id & MODID_MASK) << MODID_OFFSET)

// OPCODE_OFFSET = 0， RPC_VERSION_OFFSET = 16， MODID_OFFSET = 24
```



注册DRPC处理器

```c
/* register dRPC handlers */
rc = drpc_hdlr_register_all(smod->sm_drpc_handlers);
```

