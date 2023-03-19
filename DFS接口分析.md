## dfs_init

**daos_init**

* 初始化日志路径和打印级别
* 通过环境变量获取daos_agent的通信地址（dc_agent_sockpath）
* 生成表示客户端jobid(uname-pid)
* 通过环境变量获取CaRT配置，主要用来识别CaRT使用什么协议和获取所有rank uri
* CaRT通信上下文初始化crt_context_create、调度器初始化
* 初始化placement模块，创建pl_htable哈希表
* 注册管理模块接口
* pool/container初始化（rsvc的client leader不是很清楚？），并且注册模块接口
* 注册object模块接口，oclass->EC属性的对应关系建立

## dfs_connect

dfs_hdl_lookup(pool, DFS_H_POOL, NULL)： 从本地获取pool handle,如果没找到走daos_pool_connect进行获取句柄

cont_hdl = dfs_hdl_lookup(cont, DFS_H_CONT, pool): 从本地获取cont handle，如果没找到走daos_cont_open获取句柄

**daos_pool_connect**

dc_pool_connect

​	init_pool

​		dc_mgmt_sys_attach: 从mgmt获取pool下的rank list

​		rsvc_client_init

​	dc_pool_connect_internal

​		dc_pool_choose_svc_rank: 选择pool leader

​			dc_mgmt_pool_find： 获取pool下的所有rank

​			rsvc_client_choose:  根据rank list依次往后选择即可

​		远程RPC调用POOL_CONNECT，建立pool连接并获取pool_buf用来建立pool map

​		pool_connect_cp: 处理返回结果，设置当前已经知道pool leader是谁、更新pool map等

​			pool_rsvc_client_complete_rpc： 如果当前rank 处理失败，选择下一个rank进行建立连接（tse_task_reinit）

> NOTE: POOL_CONNECT往XStream 0进行发送，代码如下
>
> tgt_ep->ep_tag = daos_rpc_tag(DAOS_REQ_POOL, tgt_ep->ep_tag);

**daos_cont_open**

dc_cont_open

​	dc_cont_open_internal

​		dc_pool_choose_svc_rank: 选leader, 连接pool成功之后直接使用已经知道的leader

​		cont_req_create(daos_task2ctx(task), &ep, cont_op, &rpc)： 远程调用CONT_OPEN_BYLABEL/CONT_OPEN打开容器

​		cont_open_complete

​			如果pool leader过期了的话，需要重新选择下一个作为pool leader

​			daos_props_2cont_props

打开容器获取到的属性信息

```c
struct cont_props {
	uint32_t	 dcp_chunksize;
	uint32_t	 dcp_dedup_size;
	uint64_t	 dcp_alloced_oid;
	/**
	 * Use more bits for compression type since compression level is
	 * encoded in there.
	 */
	uint32_t	 dcp_compress_type;
	uint16_t	 dcp_csum_type;
	uint16_t	 dcp_encrypt_type;
	uint32_t	 dcp_redun_lvl;
	uint32_t	 dcp_redun_fac;
	uint32_t	 dcp_ec_cell_sz;
	uint32_t	 dcp_ec_pda;
	uint32_t	 dcp_rp_pda;
	uint32_t	 dcp_global_version;
	uint32_t	 dcp_obj_version;
	uint32_t	 dcp_csum_enabled:1,
			 dcp_srv_verify:1,
			 dcp_dedup_enabled:1,
			 dcp_dedup_verify:1,
			 dcp_compress_enabled:1,
			 dcp_encrypt_enabled:1;
};
```

## dfs_mount

open_dir(dfs, NULL, amode | S_IFDIR, flags, &root_dir, 1, &dfs->root): 打开/创建根目录对象

open_sb(coh, false, omode, dfs->super_oid, &dfs->attr, &dfs->super_oh, &dfs->layout_v): 创建超级块

超级块的9个AKEY值

````c
/** A-key name of SB magic */
#define MAGIC_NAME	"DFS_MAGIC"
/** A-key name of SB version */
#define SB_VER_NAME	"DFS_SB_VERSION"
/** A-key name of DFS Layout Version */
#define LAYOUT_VER_NAME	"DFS_LAYOUT_VERSION"
/** A-key name of Default chunk size */
#define CS_NAME		"DFS_CHUNK_SIZE"
/** A-key name of Default Object Class for objects */
#define OC_NAME		"DFS_OBJ_CLASS"
/** A-key name of Default Object Class for directories */
#define DIR_OC_NAME	"DFS_DIR_OBJ_CLASS"
/** A-key name of Default Object Class for files */
#define FILE_OC_NAME	"DFS_FILE_OBJ_CLASS"
/** Consistency mode of the DFS container */
#define CONT_MODE_NAME	"DFS_MODE"
/** A-key name of the object class hints */
#define CONT_HINT_NAME	"DFS_HINTS"
````

> 使用DFS写对象之前,就已经创建了2个对象: 超级块对象(00)和根对象(01)

## dfs_open

dfs_open_stat: 打开文件时parent对象可以直接为空, 默认根对象保存对象元数据

​	open_file

​    	daos_tx_open: 该项是更具容器创建属性判断是否需要启动dtx

​		fetch_entry: 先提取对象判断是否存在,如果存在直接返回提取对象的句柄

​		oid_gen(dfs, cid, true, &file->oid)

​		daos_array_open_with_attr

​			dc_obj_open

​				obj_layout_create: 创建obj->shard->target映射关系

​					obj_pl_place: 调用placement算法

​		insert_entry: 根据文件名作为DK插入parent目录对象(待确认目录对象是怎么布局的)

​	open_dir

​		create_dir

文件元数据, 以文件名作为DK的方式,将数据插入父对象(创建时间,修改时间,权限模式,用户组/用户等信息)

> 注意该文件名区分相对路径还是绝对路径,只表示一个唯一标识符,当然在dfuse只有一个根对象保存所有对象信息,所以文件名可能时相对路径

```c
struct dfs_entry {
	/** mode (permissions + entry type) */
	mode_t			mode;
	/* Length of value string, not including NULL byte */
	daos_size_t		value_len;
	/** Object ID if not a symbolic link */
	daos_obj_id_t		oid;
	/* Time of last access (sec) */
	uint64_t		atime;  // 最晚访问实践
	/* Time of last access (nsec) */
	uint64_t		atime_nano;
	/* Time of last modification (sec) */
	uint64_t		mtime; // 修改实践
	/* Time of last modification (nsec) */
	uint64_t		mtime_nano;
	/* for regular file, the time of last modification of the object */
	uint64_t		obj_hlc;
	/* Time of last status change (sec) */
	uint64_t		ctime;  // 创建实践
	/* Time of last status change (nsec) */
	uint64_t		ctime_nano;
	/** chunk size of file or default for all files in a dir */
	daos_size_t		chunk_size;
	/** oclass of file or all files in a dir */
	daos_oclass_id_t	oclass;
	/** uid - not enforced at this level. */
	uid_t			uid;
	/** gid - not enforced at this level. */
	gid_t			gid;
	/** Sym Link value */
	char			*value;
};

```

打开对象信息掩码

| DAOS信息掩码 | 返回的常量  | 含义                             |
| ------------ | :---------- | :------------------------------- |
| S_IFMT       | `_S_IFMT`   | 文件类型掩码                     |
| S_IFDIR      | `_S_IFDIR`  | 目录                             |
| S_IFCHR      | `_S_IFCHR`  | 特殊字符（指示设备，如果已设置） |
| S_IFREG      | `_S_IFREG`  | 常规                             |
| S_IREAD      | `_S_IREAD`  | 读取权限，所有者                 |
| S_IWRITE     | `_S_IWRITE` | 写入权限，所有者                 |
| S_IEXEC      | `_S_IEXEC`  | 执行/搜索权限，所有者            |

## dfs_write

详情查看dfs_write

## dfs_read

详情查看dfs_read

## dfs_umount

  daos_obj_close(dfs->root.oh, NULL): 关闭根节点对象

  daos_obj_close(dfs->super_oh, NULL): 关闭超级块对象

## dfs_disconnect

dfs_hdl_release

​	ch_rec_free

​		rec_free

​			daos_pool_disconnect

​			daos_cont_close

## daos_fini

释放资源,并将module_initialized置为0

## 非关键性接口

**local2global/global2local**

以后缀global2local,local2global结尾的接口,用来多个进程之间共享句柄(可以通过消息通道,文件等方式进行共享glob参数)

**dfs_remove**

remove_entry

​	daos_obj_punch

​		obj_punch_common

​			obj_shards_2_fwtgts

​			obj_req_fanout

