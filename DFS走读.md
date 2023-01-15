打开文件接口

```c
int
dfs_open(dfs_t *dfs, dfs_obj_t *parent, const char *name, mode_t mode,
	 int flags, daos_oclass_id_t cid, daos_size_t chunk_size,
	 const char *value, dfs_obj_t **_obj)
```

->dfs_open_stat

```c
strncpy(obj->name, name, len + 1); // 将文件名复制到obj->name
```

->open_file

->insert_entry

```c
d_iov_t		sg_iovs[INODE_AKEYS]; // 将文件的属性（创建时间、修改时间、对象ID）等插入到该结构体中，该结构体表现为sgl->iov

d_iov_set(&dkey, (void *)name, len); // 将文件名作为作为dkey


```

->daos_obj_update

```c
dc_obj_update_task_create // 创建任务
    rc = dc_task_create(dc_obj_list_dkey, tse, ev, task); //将dc_obj_list_dkey进行事件调度
dc_task_schedule
```

->dc_obj_list_dkey

-> obj_list_common

```c
obj_list_common(task, DAOS_OBJ_DKEY_RPC_ENUMERATE, args);
```



写文件接口

```c
int
dfs_write(dfs_t *dfs, dfs_obj_t *obj, d_sg_list_t *sgl, daos_off_t off,
	  daos_event_t *ev)
```

* dfs: 挂载信息
* 文件对象信息
* 数据SGL
* 下发数据偏移

