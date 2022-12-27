# Mercury RPC Layer

The RPC layer is used for sending and receiving RPCs. RPC arguments are generally small. For handling larger arguments, this layer is complemented with a bulk interface that is covered in the next section.

> RPC层用于发送和接收RPC请求。RPC参数通常很小。处理大参数，该层还补充了一个bulk接口，下一节将介绍。

## HG Interface

The HG interface directly builds on top of the NA interface. Sending an RPC generally results in two NA operations being posted: an expected receive for the response and an unexpected send for the RPC request. The entire interface is therefore non-blocking with a goal of asynchronously executing RPCs on a given target. In order to achieve that, the HG interface provides the following primitives: *target address lookup*, *RPC registration*, *RPC execution*, *progress* and *cancelation*.

### Initialization

To initialize the HG RPC interface, two options are available, either by using the default `HG_Init()` function and specifying an init info string as described in the NA plugin [section](https://mercury-hpc.github.io/user/na/#available-plugins):

```
hg_class_t *
HG_Init(const char *info_string, hg_bool_t listen);
```

Or by using the `HG_Init_opt()` function, which allows for passing extra options.

| Option              | Description                                                  |
| :------------------ | :----------------------------------------------------------- |
| `na_init_info`      | See NA initialization [options](https://mercury-hpc.github.io/user/na/#na-interface). |
| `na_class`          | Enables initialization from an existing NA class.            |
| `request_post_init` | Controls the initial number of requests that are posted on context creation. Default value is: 256 |
| `request_post_incr` | Controls the number of requests that are incrementally posted when the initial number of requests is exhausted. Default value is: 256 |
| `auto_sm`           | Controls whether shared-memory should be automatically used when origin and target share the same node. |
| `sm_info_string`    | Overrides default init string for NA SM plugin.              |
| `no_bulk_eager`     | Prevents small bulk data from being automatically embedded along with the RPC request. |
| `no_loopback`       | Disables internal loopback interface that enables forwarding of RPC to self addresses. |

```
struct hg_init_info {
    struct na_init_info na_init_info;
    na_class_t *na_class;
    hg_uint32_t request_post_init;
    hg_uint32_t request_post_incr;
    hg_bool_t auto_sm;
    const char *sm_info_string;
    hg_bool_t no_bulk_eager;
    hg_bool_t no_loopback;

struct hg_init_info {
    struct na_init_info na_init_info;   /* NA Init Info */
    na_class_t *na_class;               /* NA class */
    hg_bool_t auto_sm;                  /* Use NA SM plugin with local addrs */
    hg_bool_t stats;                    /* (Debug) Print stats at exit */
};

hg_class_t *
HG_Init_opt(const char *na_info_string, hg_bool_t na_listen,
            const struct hg_init_info *hg_init_info);
```

Similar to the NA layer, the `HG_Init()` call results in the creation of a new `hg_class_t` object. The `hg_class_t` object can later be released after a call to:

```
hg_return_t
HG_Finalize(hg_class_t *hg_class);
```

Once the interface has been initialized, a context of execution must be created, which (similar to the NA layer) internally associates a specific queue to the operations that will complete:

```
hg_context_t *
HG_Context_create(hg_class_t *hg_class);
```

It can then be destroyed using:

```
hg_return_t
HG_Context_destroy(hg_context_t *context);
```

### Registration

Before an RPC can be sent, the HG class needs a way of identifying it so that a callback corresponding to that RPC can be executed on the target. Additionally, the functions to serialize and deserialize the function arguments associated to that RPC must be provided. This is done through the `HG_Register_name()` function. Note that this step can be slightly simplified by using the `MERCURY_REGISTER` macro. Registration must be done on both the origin and the target with the same `func_name` identifier. Alternatively `HG_Register()` can be used to pass a user-defined unique identifier and avoid internal hashing of the provided function name.

```
typedef hg_return_t (*hg_proc_cb_t)(hg_proc_t proc, void *data);
typedef hg_return_t (*hg_rpc_cb_t)(hg_handle_t handle);

hg_id_t
HG_Register_name(hg_class_t *hg_class, const char *func_name,
                 hg_proc_cb_t in_proc_cb, hg_proc_cb_t out_proc_cb,
                 hg_rpc_cb_t rpc_cb);
```

It is also possible (but not necessary) to deregister an existing RPC ID, in the case where an RPC should no longer be received by using:

```
hg_return_t
HG_Deregister(hg_class_t *hg_class, hg_id_t id);
```

In the case where an RPC does not require a response, one can indicate that no response is expected (and therefore avoid waiting for a message to be sent back) by using the following call (on an already registered RPC):

```
hg_return_t
HG_Registered_disable_response(hg_class_t *hg_class, hg_id_t id, hg_bool_t disable);
```

### Target Address Lookup

As mentioned in the [overview](https://mercury-hpc.github.io/user/overview/) section, there is no real distinction between client and server since it may be desirable for a client to also act as a server for other processes. Therefore, the interface only uses the distinction of *origin* and *target*.

Similar to NA's API, the first step consists of retrieving the target's address, this can be done by calling on the target:

```
hg_return_t
HG_Addr_self(hg_class_t *hg_class, hg_addr_t *addr);
```

And then convert it to string using:

```
hg_return_t
HG_Addr_to_string(hg_class_t *hg_class, char *buf, hg_size_t *buf_size, hg_addr_t addr);
```

The string can then be exchanged back with the origin through out-of-band mechanisms (e.g., using a file, etc), which can then look the target up using the function:

```
hg_return_t
HG_Addr_lookup(hg_class_t *hg_class, const char *name, hg_addr_t *addr);
```

All addresses must then be freed using:

```
hg_return_t
HG_Addr_free(hg_class_t *hg_class, hg_addr_t addr);
```

### RPC Execution

Executing an RPC is generally composed of two parts, one on the origin, which will send the RPC request and receive a response, one on the target, which will receive the request, execute it and send a response back.

#### Origin

Using the RPC ID defined after a call to `HG_Register()`, one can use the `HG_Create()` call to define a new `hg_handle_t` object that can be used (and later re-used without reallocating resources) to set/get input/output arguments.

```
hg_return_t
HG_Create(hg_context_t *context, hg_addr_t addr, hg_id_t id, hg_handle_t *handle);
```

This handle can be destroyed with `HG_Destroy()`—and a reference count prevents resources from being released while the handle is still in use.

```
hg_return_t
HG_Destroy(hg_handle_t handle);
```

The second step is to pack the input arguments within a structure, for which a serialization function is provided with the `HG_Register()` call. The `HG_Forward()` function can then be used to send that structure (which describes the input arguments). This function is non-blocking. When it completes, the associated callback can be executed by calling `HG_Trigger()`.

```
typedef hg_return_t (*hg_cb_t)(const struct hg_cb_info *callback_info);

hg_return_t
HG_Forward(hg_handle_t handle, hg_cb_t callback, void *arg, void *in_struct);
```

When `HG_Forward()` completes (i.e., when the user callback can be triggered), the RPC has been remotely executed and a response with the output results has been sent back. This output can then be retrieved (usually within the callback) with the following function:

```
hg_return_t
HG_Get_output(hg_handle_t handle, void *out_struct);
```

Retrieving the output may result in the creation of memory objects, which must then be released by calling:

```
hg_return_t
HG_Free_output(hg_handle_t handle, void *out_struct);
```

To be safe and if necessary, one must make a copy of the results before calling `HG_Free_output()`. Note that in the case of an RPC with no response, completion occurs after the RPC has been successfully sent (i.e., there is no output to retrieve).

#### Target

On the target, the RPC callback function passed to the `HG_Register()` call must be defined.

```
typedef hg_return_t (*hg_rpc_cb_t)(hg_handle_t handle);
```

Whenever a new RPC is received, that callback will be invoked. The input arguments can be retrieved with:

```
hg_return_t
HG_Get_input(hg_handle_t handle, void *in_struct);
```

Retrieving the input may result in the creation of memory objects, which must then be released by calling:

```
hg_return_t
HG_Free_input(hg_handle_t handle, void *in_struct);
```

When the input has been retrieved, the arguments contained in the input structure can be passed to the actual function call. When the execution is done, an output structure can be filled with the return value and/or the output arguments of the function. It can then be sent back using:

```
typedef hg_return_t (*hg_cb_t)(const struct hg_cb_info *callback_info);

hg_return_t
HG_Respond(hg_handle_t handle, hg_cb_t callback, void *arg, void *out_struct);
```

This call is also non-blocking. When it completes, the associated callback is placed onto a completion queue. It can then be triggered after a call to `HG_Trigger()`. Note that in the case of an RPC with no response, calling `HG_Respond()` will return an error.

### Progress and Cancelation

Mercury uses a callback model. Callbacks are passed to non-blocking functions and are pushed to the context's completion queue when the operation completes. Explicit progress is made by calling `HG_Progress()`. `HG_Progress()` returns when an operation completes, is in the completion queue or `timeout` is reached.

```
hg_return_t
HG_Progress(hg_context_t *context, unsigned int timeout);
```

When an operation completes, calling `HG_Trigger()` allows the callback execution to be separately controlled from the main progress loop.

Warning

Operations may complete with an `HG_SUCCESS` return code or with an error return code if failure occured after the operation was submitted. The `ret` field from the `hg_cb_info` struct should therefore always be checked for potential errors.

```
hg_return_t
HG_Trigger(hg_context_t *context, unsigned int timeout,
           unsigned int max_count, unsigned int *actual_count);
```

In some cases, one may want to call `HG_Progress()` then `HG_Trigger()` or have them execute in parallel by using separate threads.

When it is desirable to cancel an HG operation, one can call `HG_Cancel()` on a HG handle to cancel an on-going `HG_Forward()` or `HG_Respond()` operation. Please refer to this [page](https://mercury-hpc.github.io/user/cancel/) for additional details regarding cancellation of operations and handling of timeouts.

```
hg_return_t
HG_Cancel(hg_handle_t handle);
```

Cancelation is always asynchronous. When/if the operation is successfully canceled, it will be pushed to the completion queue and the callback `ret` value will set with an `HG_CANCELED` error return code.