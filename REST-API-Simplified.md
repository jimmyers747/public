
# RDU301 Simplified REST API
**Objective:** *Remove and Replace Stingray*

---

**NOTE: DOCUMENT IS AUTOMATICALLY GENERATED**

*Do not update this document directly.*  Instead, update `simple_rest_api.py` and run it as a python3 script.

---
## Executive Summary
- This API is a replacement for the `/redfish/v1` API implemented by Stingray.
- All URIs defined below will be relative to a versioned root path that is yet to be determined but should be something like `/rapi/v2`.
- Initial versions will (obviously) contain a subset of the API as development proceeds.  The **Released In** tag should be populated with a version number as each URI is implemented.
- Implementation priority...
    - [/time](#uri---time)
        - Get/Set system time
        - Allows for testing API infrastructure basic operation
    - [/task](#uri---task)
        - Async task management
        - Needed to support long-running requests
        - Prerequisite for software update and container deployment
    - [/sys](#uri---sys)
        - Top-level system details
        - Reboot
        - Although reboot will not be a necessity for software update, it seems to be an easy thing to implement
    - [/sw](#uri---sw)
        - Software component version information
        - Root filesystem update
    - [/cntr](#uri---cntr)
        - Container management subsystem
        - Will be the most complicated, but the most useful
    - [/diag](#uri---diaglog)
        - Diagnostics (especially logging)
        - Until this subsystem is working we will just use `ssh`
    - [/net](#uri---net)
        - Network service configuration
    - [/eth](#uri---eth)
        - Ethernet interface configuration
        - This will probably be the most troublesome since errors will cause network connectivity to be lost
    - [/user/admin](#uri---usersuser_idcmdset)
        - Change the admin user password
        - Not needed for machine-to-machine communication, more internal from the web service
    - [/sec/cert](#uri---seccert)
        - Security certificate management
        - Should handle TLS certificates and machine-to-machine (JWT) certificates
        - Save this for last since it might be complex
    
---
## API Basics
- Loosely modeled after the [Geist API](http://releases.geist.local/documentation/api/latest/geist_api_public.html)
- Virtually every request is a POST
    - We may eventually translate a GET request into an implied body with nothing but: `"cmd" : "get"`.
    - POST input body
        - `cmd`  : Command verb (`get`, `set`, `add`, `delete`, `control`, ...) interpreted by the URI target
        - `opt`  : Global options for the command (not required)
            - `dbg`  : Debug log level
            - `dev`  : Dev mode token
            - `test` : Test mode token
        - `data` : Optional data map specific to the URI target and command verb.
    - Output JSON body
        - `status` : Status parameters
            - `code` : Numeric code (0 = success, 1-254 = error/warning)
            - `msg`  : Text message translation of the status
            - `log`  : Optional log URL containing command log output
        - `data` : Optional data specific to the URI target, command verb, and status.
- Output file streams
    - API requests that generate a file stream or static reference will return a relative URL that can be used on a subsequent GET
    - A GET on a file stream will return the headers and body associated with that file stream (e.g. like a normal static link)
    - File streams may be transient/temporary and purged at any time based on garbage collection parameters (and may not survive a reboot)

---
## Security
- Authentication/Authorization
    - Username/Password : Using HTTP Basic authentication
    - Session : Using a 'session' cookie
    - JWT : Use the Authentication: Bearer token
    - Token : Local application token for internal use (Authentication header)
    - Authorization context will be mapped to the authentication method.
    - Suggest using RSA256 JWT with payload token retrieved from RDU
        - Public keys can be added/removed/listed via username/password authentication only
        - Token will be supplied in cleartext via the /sec URI target
        - Changed on reboot
        - Failure to authenticate will cause re-retrieval of token
        - Have security check api call that just tests authentication
- Certificates and keys
    - Only authenticated admin users can install certificates and keys
    - Unit ships without any certificates and keys (or maybe a unique key per device)
        - Admin user can 'clear' or 'reset' all certs and keys
        - Without a cert/key, no remote API access is available
        - Must install initial cert/key from WEB, CLI, or USB
    - Cloud-connect unit
        - Ships with Vertiv server certificate
        - Connects to Vertiv on startup to retrieve configuration
        - Other certificates can be installed from Vertiv


---
## Actions

| **-**                | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |
|----------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------|
| **cntr**             |                                                 |                                                 | [[X]](#uri---cntrcmddelete)                     | [[X]](#uri---cntr)                              |                                                 |
| **cntr.image**       | [[X]](#uri---cntrimagecmdadd)                   |                                                 | [[X]](#uri---cntrimagecmddelete)                | [[X]](#uri---cntrimage)                         |                                                 |
| **cntr.image.id**    |                                                 |                                                 | [[X]](#uri---cntrimageimage\_idcmddelete)       | [[X]](#uri---cntrimageimage\_id)                |                                                 |
| **cntr.service**     | [[X]](#uri---cntrservicecmdadd)                 |                                                 | [[X]](#uri---cntrservicecmddelete)              | [[X]](#uri---cntrservice)                       |                                                 |
| **cntr.service.id**  |                                                 | [[X]](#uri---cntrserviceservice\_idcmdcontrol)  | [[X]](#uri---cntrserviceservice\_idcmddelete)   | [[X]](#uri---cntrserviceservice\_id)            |                                                 |
| **diag.log**         |                                                 |                                                 | [[X]](#uri---diaglogcmddelete)                  | [[X]](#uri---diaglog)                           |                                                 |
| **diag.log.id**      |                                                 |                                                 | [[X]](#uri---diagloglog\_idcmddelete)           | [[X]](#uri---diagloglog\_id)                    |                                                 |
| **diag.net.id**      |                                                 | [[X]](#uri---diagnetcmdcontrol)                 |                                                 |                                                 |                                                 |
| **diag.power**       |                                                 |                                                 |                                                 | [[X]](#uri---diagpower)                         |                                                 |
| **diag.resource**    |                                                 |                                                 |                                                 | [[X]](#uri---diagresource)                      |                                                 |
| **diag.resource.id** |                                                 |                                                 |                                                 | [[X]](#uri---diagresourceresource\_id)          |                                                 |
| **-**                | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |
| **diag.snapshot**    |                                                 |                                                 |                                                 | [[X]](#uri---diagsnapshot)                      |                                                 |
| **diag.thermal**     |                                                 |                                                 |                                                 | [[X]](#uri---diagthermal)                       |                                                 |
| **eth**              |                                                 |                                                 |                                                 | [[X]](#uri---eth)                               |                                                 |
| **eth.id**           |                                                 |                                                 |                                                 | [[X]](#uri---etheth\_interface\_id)             | [[X]](#uri---etheth\_interface\_idcmdset)       |
| **net**              |                                                 |                                                 |                                                 | [[X]](#uri---net)                               |                                                 |
| **net.https**        |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **net.id**           |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **net.ntp**          |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **net.redis**        |                                                 |                                                 |                                                 | [[X]](#uri---netredis)                          | [[X]](#uri---netrediscmdset)                    |
| **net.ssdp**         |                                                 |                                                 |                                                 | [[X]](#uri---netssdp)                           | [[X]](#uri---netssdpcmdset)                     |
| **-**                | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |
| **net.ssh**          |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **sec.cert**         |                                                 |                                                 |                                                 | [[X]](#uri---seccert)                           |                                                 |
| **sec.cert.id**      |                                                 |                                                 | [[X]](#uri---seccertcert\_idcmddelete)          | [[X]](#uri---seccertcert\_id)                   | [[X]](#uri---seccertcert\_idcmdset)             |
| **sw**               |                                                 |                                                 |                                                 | [[X]](#uri---sw)                                |                                                 |
| **sw.rfs**           |                                                 |                                                 |                                                 | [[X]](#uri---swrfs)                             | [[X]](#uri---swrfscmdset)                       |
| **sys**              |                                                 | [[X]](#uri---syscmdcontrol)                     |                                                 | [[X]](#uri---sys)                               |                                                 |
| **task**             |                                                 |                                                 | [[X]](#uri---taskcmddelete)                     | [[X]](#uri---task)                              |                                                 |
| **task.id**          |                                                 | [[X]](#uri---tasktask\_idcmdcontrol)            | [[X]](#uri---tasktask\_idcmddelete)             | [[X]](#uri---tasktask\_id)                      |                                                 |
| **time**             |                                                 |                                                 |                                                 | [[X]](#uri---time)                              | [[X]](#uri---timeutccmdset)                     |
| **user.admin**       |                                                 |                                                 |                                                 |                                                 | [[X]](#uri---usersuser\_idcmdset)               |
| **vcloud**           |                                                 |                                                 |                                                 | [[X]](#uri---vcloud)                            |                                                 |
| **-**                | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |


---
## Target URIs
### URI - /
- **Target Id**: root
- **Command**: `get`
- **Output**:
    - **cntr\_image\_delete**     : [URI](#uri---cntrimageimage_idcmddelete) - `/cntr/image/{image_id}?cmd=delete` *(Delete a container image if not in use.)*
    - **cntr\_image\_deploy**     : [URI](#uri---cntrimagecmdadd) - `/cntr/image?cmd=add` *(Deploy a container image.)*
    - **cntr\_image\_details**    : [URI](#uri---cntrimageimage_id) - `/cntr/image/{image_id}` *(Container image details.)*
    - **cntr\_image\_list**       : [URI](#uri---cntrimage) - `/cntr/image` *(Container image list.)*
    - **cntr\_image\_purge**      : [URI](#uri---cntrimagecmddelete) - `/cntr/image?cmd=delete` *(Purge (collect garbage) container images.)*
    - **cntr\_purge**             : [URI](#uri---cntrcmddelete) - `/cntr?cmd=delete` *(Purge (collect garbage) container services and images.)*
    - **cntr\_service\_control**  : [URI](#uri---cntrserviceservice_idcmdcontrol) - `/cntr/service/{service_id}?cmd=control` *(Control a container service.)*
    - **cntr\_service\_delete**   : [URI](#uri---cntrserviceservice_idcmddelete) - `/cntr/service/{service_id}?cmd=delete` *(Delete a container service.)*
    - **cntr\_service\_deploy**   : [URI](#uri---cntrservicecmdadd) - `/cntr/service?cmd=add` *(Container service deploy.)*
    - **cntr\_service\_details**  : [URI](#uri---cntrserviceservice_id) - `/cntr/service/{service_id}` *(Container service details.)*
    - **cntr\_service\_list**     : [URI](#uri---cntrservice) - `/cntr/service` *(Container summary.)*
    - **cntr\_service\_purge**    : [URI](#uri---cntrservicecmddelete) - `/cntr/service?cmd=delete` *(Purge (collect garbage) container services.)*
    - **cntr\_summary**           : [URI](#uri---cntr) - `/cntr` *(Container summary.)*
    - **diag\_log\_delete**       : [URI](#uri---diagloglog_idcmddelete) - `/diag/log/{log_id}?cmd=delete` *(Delete log `log_id`.)*
    - **diag\_log\_details**      : [URI](#uri---diagloglog_id) - `/diag/log/{log_id}` *(Output log details for `log_id`.)*
    - **diag\_log\_list**         : [URI](#uri---diaglog) - `/diag/log` *(Diagnostic log list.)*
    - **diag\_log\_purge**        : [URI](#uri---diaglogcmddelete) - `/diag/log?cmd=delete` *(Purge (collect garbage) diagnostic logs.)*
    - **diag\_net\_control**      : [URI](#uri---diagnetcmdcontrol) - `/diag/net?cmd=control` *((**async**) Perform network diagnostics.)*
    - **diag\_power\_details**    : [URI](#uri---diagpower) - `/diag/power` *(Report power diagnostics.)*
    - **diag\_resource\_details** : [URI](#uri---diagresourceresource_id) - `/diag/resource/{resource_id}` *(Details diagnostics for system resources.)*
    - **diag\_resource\_list**    : [URI](#uri---diagresource) - `/diag/resource` *(Resource diagnostic list.)*
    - **diag\_snapshot**          : [URI](#uri---diagsnapshot) - `/diag/snapshot` *((**async**) Dynamically take a snapshot of diagnostics and metrics.)*
    - **diag\_thermal\_details**  : [URI](#uri---diagthermal) - `/diag/thermal` *(Report thermal diagnostics.)*
    - **eth\_interface\_details** : [URI](#uri---etheth_interface_id) - `/eth/{eth_interface_id}` *(Ethernet interface details for `eth_interface_id`.)*
    - **eth\_interface\_list**    : [URI](#uri---eth) - `/eth` *(Ethernet interface list.)*
    - **eth\_interface\_update**  : [URI](#uri---etheth_interface_idcmdset) - `/eth/{eth_interface_id}?cmd=set` *(Ethernet interface update for `eth_interface_id`.)*
    - **net\_redis\_details**     : [URI](#uri---netredis) - `/net/redis` *(Redis secure tunnel service public details.)*
    - **net\_redis\_update**      : [URI](#uri---netrediscmdset) - `/net/redis?cmd=set` *(Configure secure communications between RDU301 Redis and client.)*
    - **net\_service\_details**   : [URI](#uri---netnet_service_id) - `/net/{net_service_id}` *(Network service details.)*
    - **net\_service\_update**    : [URI](#uri---netnet_service_idcmdset) - `/net/{net_service_id}?cmd=set` *(Network service details.)*
    - **net\_ssdp\_details**      : [URI](#uri---netssdp) - `/net/ssdp` *(SSDP service public details.)*
    - **net\_ssdp\_update**       : [URI](#uri---netssdpcmdset) - `/net/ssdp?cmd=set` *(Update SSDP service configuration.)*
    - **net\_summary**            : [URI](#uri---net) - `/net` *(Network service list.)*
    - **sec\_cert\_delete**       : [URI](#uri---seccertcert_idcmddelete) - `/sec/cert/{cert_id}?cmd=delete` *(TBD)*
    - **sec\_cert\_details**      : [URI](#uri---seccertcert_id) - `/sec/cert/{cert_id}` *(TBD)*
    - **sec\_cert\_list**         : [URI](#uri---seccert) - `/sec/cert` *(TBD)*
    - **sec\_cert\_update**       : [URI](#uri---seccertcert_idcmdset) - `/sec/cert/{cert_id}?cmd=set` *(TBD)*
    - **sw\_list**                : [URI](#uri---sw) - `/sw` *(List software components.)*
    - **sw\_rfs\_details**        : [URI](#uri---swrfs) - `/sw/rfs` *(Root file system software details.)*
    - **sw\_rfs\_update**         : [URI](#uri---swrfscmdset) - `/sw/rfs?cmd=set` *((**async**) Root file system software update.)*
    - **sys\_details**            : [URI](#uri---sys) - `/sys` *(System details.)*
    - **sys\_shutdown**           : [URI](#uri---syscmdcontrol) - `/sys?cmd=control` *(Shut the system down.)*
    - **task\_control**           : [URI](#uri---tasktask_idcmdcontrol) - `/task/{task_id}?cmd=control` *(Task cancel/kill.)*
    - **task\_delete**            : [URI](#uri---tasktask_idcmddelete) - `/task/{task_id}?cmd=delete` *(Mark a task for delete.)*
    - **task\_details**           : [URI](#uri---tasktask_id) - `/task/{task_id}`
    - **task\_list**              : [URI](#uri---task) - `/task` *(Task list.)*
    - **task\_purge**             : [URI](#uri---taskcmddelete) - `/task?cmd=delete` *(Purge (collect garbage) tasks.)*
    - **time\_summary**           : [URI](#uri---time) - `/time` *(System time information.)*
    - **time\_update**            : [URI](#uri---timeutccmdset) - `/time/utc?cmd=set` *(Set UTC time.)*
    - **user\_password**          : [URI](#uri---usersuser_idcmdset) - `/users/{user_id}?cmd=set` *(Changes the password for `user_id`.)*
    - **vcloud\_summary**         : [URI](#uri---vcloud) - `/vcloud` *(This target is TBD.)*
- **Released In**: None

#### Description
The root API entry point provides a *table of contents* for the rest of the API.  Retrieving this
target will provide a list of identifiers to their relative location within the API.  A location
can have substitutions in the form `{substitution_id}` where the braces `{}` are literal and the
text within the braces may be logical substitution tags or query string literals that logically map 
to body content keys.

#### Example
```
# Output
{
    "data": {
        "cntr_image_delete": "/cntr/image/{image_id}?cmd=delete",
        "cntr_image_deploy": "/cntr/image?cmd=add",
        "cntr_image_details": "/cntr/image/{image_id}",
        "cntr_image_list": "/cntr/image",
        "cntr_image_purge": "/cntr/image?cmd=delete",
        "cntr_purge": "/cntr?cmd=delete",
        "cntr_service_control": "/cntr/service/{service_id}?cmd=control",
        "cntr_service_delete": "/cntr/service/{service_id}?cmd=delete",
        "cntr_service_deploy": "/cntr/service?cmd=add",
        "cntr_service_details": "/cntr/service/{service_id}",
        "cntr_service_list": "/cntr/service",
        "cntr_service_purge": "/cntr/service?cmd=delete",
        "cntr_summary": "/cntr",
        "diag_log_delete": "/diag/log/{log_id}?cmd=delete",
        "diag_log_details": "/diag/log/{log_id}",
        "diag_log_list": "/diag/log",
        "diag_log_purge": "/diag/log?cmd=delete",
        "diag_net_control": "/diag/net?cmd=control",
        "diag_power_details": "/diag/power",
        "diag_resource_details": "/diag/resource/{resource_id}",
        "diag_resource_list": "/diag/resource",
        "diag_snapshot": "/diag/snapshot",
        "diag_thermal_details": "/diag/thermal",
        "eth_interface_details": "/eth/{eth_interface_id}",
        "eth_interface_list": "/eth",
        "eth_interface_update": "/eth/{eth_interface_id}?cmd=set",
        "net_redis_details": "/net/redis",
        "net_redis_update": "/net/redis?cmd=set",
        "net_service_details": "/net/{net_service_id}",
        "net_service_update": "/net/{net_service_id}?cmd=set",
        "net_ssdp_details": "/net/ssdp",
        "net_ssdp_update": "/net/ssdp?cmd=set",
        "net_summary": "/net",
        "sec_cert_delete": "/sec/cert/{cert_id}?cmd=delete",
        "sec_cert_details": "/sec/cert/{cert_id}",
        "sec_cert_list": "/sec/cert",
        "sec_cert_update": "/sec/cert/{cert_id}?cmd=set",
        "sw_list": "/sw",
        "sw_rfs_details": "/sw/rfs",
        "sw_rfs_update": "/sw/rfs?cmd=set",
        "sys_details": "/sys",
        "sys_shutdown": "/sys?cmd=control",
        "task_control": "/task/{task_id}?cmd=control",
        "task_delete": "/task/{task_id}?cmd=delete",
        "task_details": "/task/{task_id}",
        "task_list": "/task",
        "task_purge": "/task?cmd=delete",
        "time_summary": "/time",
        "time_update": "/time/utc?cmd=set",
        "user_password": "/users/{user_id}?cmd=set",
        "vcloud_summary": "/vcloud"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /cntr
- **Target Id**: cntr_summary
- **Command**: `get`
- **Output**:
    - **images**   : 
        - ***Array***:
            - **image\_id**    : Image id.
            - **image\_name**  : Image name.
            - **image\_state** : Image state `used`|`orphan`
    - **services** : 
        - ***Array***:
            - **service\_class** : Service class.
            - **service\_id**    : Service id.
            - **service\_name**  : Service name.
            - **service\_state** : Service state `run`|`stop`|`enabled`|`disabled`.
- **Released In**: None

#### Description
Container summary.

#### Example
```
# Output
{
    "data": {
        "images": [
            {
                "image_id": "796c45706c2b",
                "image_name": "rdumanagerservices:202002031",
                "image_state": "used"
            }
        ],
        "services": [
            {
                "service_class": "gbpdataacquisition",
                "service_id": "3d25301a",
                "service_name": "gbpdataacquisitiond02adb1f35d343a4acaf7b6861dc2714",
                "service_state": "run"
            }
        ]
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /cntr/image
- **Target Id**: cntr_image_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **image\_id**    : Image id.
        - **image\_name**  : Image name.
        - **image\_state** : Image state `used`|`orphan`
- **Released In**: None

#### Description
Container image list.

#### Example
```
# Output
{
    "data": [
        {
            "image_id": "796c45706c2b",
            "image_name": "rdumanagerservices:202002031",
            "image_state": "used"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /cntr/image/{image\_id}
- **Target Id**: cntr_image_details
- **Command**: `add`
- **Output**:
    - **image\_id**     : Image id.
    - **image\_name**   : Image name.
    - **image\_sha256** : SHA256 sum of the image file.
    - **image\_state**  : Image state `used`|`orphan`
    - **image\_url**    : URL to fetch image
- **Released In**: None

#### Description
Container image details.

#### Example
```
# Output
{
    "data": {
        "image_id": "796c45706c2b",
        "image_name": "rdumanagerservices:202002031",
        "image_sha256": "a9b730edf4a80c4677a10d23625a7c4ce851fb01b2f5981a8d784c77879c4e46",
        "image_state": "used",
        "image_url": "http://192.168.254.88/cimg/rdumanagerservices.202002031.aci"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /cntr/image/{image\_id}?cmd=delete
- **Target Id**: cntr_image_delete
- **Command**: `delete`
- **Released In**: None

#### Description
Delete a container image if not in use.

---

### URI - /cntr/image?cmd=add
- **Target Id**: cntr_image_deploy
- **Command**: `add`
- **Input**:
    - **image\_id**     : Unique image id.
    - **image\_sha256** : SHA256 sum of the image file.
    - **image\_size**   : Size of image file in bytes.
    - **image\_url**    : URL to fetch image
- **Released In**: None

#### Description
Deploy a container image.

#### Example
```
# Data Input
{
    "cmd": "add",
    "data": {
        "image_id": "796c45706c2b",
        "image_sha256": "a9b730edf4a80c4677a10d23625a7c4ce851fb01b2f5981a8d784c77879c4e46",
        "image_size": 236233844,
        "image_url": "http://192.168.254.88/cimg/rdumanagerservices.202002031.aci"
    }
}
```

---

### URI - /cntr/image?cmd=delete
- **Target Id**: cntr_image_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
- **Released In**: None

#### Description
Purge (collect garbage) container images.

Orphaned images will be purged according to the `grace_period`.

#### Example
```
# Data Input
{
    "cmd": "delete",
    "data": {
        "grace_period": "1d"
    }
}
```

---

### URI - /cntr/service
- **Target Id**: cntr_service_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **service\_class** : Service class.
        - **service\_id**    : Service id.
        - **service\_name**  : Service name.
        - **service\_state** : Service state `run`|`stop`|`enabled`|`disabled`.
- **Released In**: None

#### Description
Container summary.

#### Example
```
# Output
{
    "data": [
        {
            "service_class": "gbpdataacquisition",
            "service_id": "3d25301a",
            "service_name": "gbpdataacquisitiond02adb1f35d343a4acaf7b6861dc2714",
            "service_state": "run"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /cntr/service/{service\_id}
- **Target Id**: cntr_service_details
- **Command**: `get`
- **Output**:
    - **host\_network**  : Host network enabled.
    - **images**         : 
        - ***Array***: Image id
    - **mounts**         : 
        - ***Array***:
            - **dst\_path**   : Mount destination path.
            - **member\_idx** : See [member\_idx](#point---member_idx)
            - **mount\_type** : Mount type.
            - **src\_path**   : Mount source path.
    - **networks**       : 
        - ***Array***:
            - **dst\_port**   : Destination port.
            - **member\_idx** : See [member\_idx](#point---member_idx)
            - **port\_type**  : Network port type `udp`|`tcp`.
            - **src\_intf**   : Source network interface.
            - **src\_port**   : Source network port.
    - **service\_class** : Service class.
    - **service\_id**    : Service id.
    - **service\_name**  : Service name.
    - **service\_state** : Service state `run`|`stop`|`enabled`|`disabled`.
    - **start\_time**    : Start time.
    - **stop\_time**     : Stop time.
- **Released In**: None

#### Description
Container service details.

#### Example
```
# Output
{
    "data": {
        "host_network": false,
        "images": [
            "796c45706c2b"
        ],
        "mounts": [
            {
                "dst_path": "/etc/rkt/config",
                "member_idx": 1,
                "mount_type": "bind",
                "src_path": "/etc/rkt/config"
            }
        ],
        "networks": [
            {
                "dst_port": 6380,
                "member_idx": 1,
                "port_type": "udp",
                "src_intf": "0.0.0.0",
                "src_port": 6379
            }
        ],
        "service_class": "gbpdataacquisition",
        "service_id": "3d25301a",
        "service_name": "gbpdataacquisitiond02adb1f35d343a4acaf7b6861dc2714",
        "service_state": "run",
        "start_time": "2020-02-03T01:02:03",
        "stop_time": "2020-02-06T00:01:02"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /cntr/service/{service\_id}?cmd=control
- **Target Id**: cntr_service_control
- **Command**: `control`
- **Input**:
    - **action** : `start`|`stop`|`enable`|`disable`
- **Released In**: None

#### Description
Control a container service.

#### Example
```
# Data Input
{
    "cmd": "control",
    "data": {
        "action": "stop"
    }
}
```

---

### URI - /cntr/service/{service\_id}?cmd=delete
- **Target Id**: cntr_service_delete
- **Command**: `delete`
- **Released In**: None

#### Description
Delete a container service.

Running services will be stopped.  Enabled services will be disabled.

---

### URI - /cntr/service?cmd=add
- **Target Id**: cntr_service_deploy
- **Command**: `add`
- **Output**:
    - **force\_add**     : Force deployment.
                                Take action if another service with the same `service_class` already exists.
                                If `true`, then remove the previous service and deploy this one.
                                If `false`, then fail this deployment.
    - **host\_network**  : Host network enabled.
    - **image**          : Image id
    - **mounts**         : 
        - ***Array***:
            - **dst\_path**   : Mount destination path.
            - **member\_idx** : See [member\_idx](#point---member_idx)
            - **mount\_type** : Mount type.
            - **src\_path**   : Mount source path.
    - **networks**       : 
        - ***Array***:
            - **dst\_port**   : Destination port.
            - **member\_idx** : See [member\_idx](#point---member_idx)
            - **port\_type**  : Network port type `udp`|`tcp`.
            - **src\_intf**   : Source network interface.
            - **src\_port**   : Source network port.
    - **restart\_mode**  : Action to take on container service abnormal exit `restart`|`reboot`|`ignore`.
    - **restart\_wait**  : Time to wait after abnormal exit before restarting (secs).
    - **service\_class** : Service class.
    - **service\_name**  : Service name.
    - **service\_state** : Service state `run`|`stop`|`enabled`|`disabled`.
- **Released In**: None

#### Description
Container service deploy.

#### Example
```
# Output
{
    "data": {
        "force_add": false,
        "host_network": false,
        "image": "796c45706c2b",
        "mounts": [
            {
                "dst_path": "/etc/rkt/config",
                "member_idx": 1,
                "mount_type": "bind",
                "src_path": "/etc/rkt/config"
            }
        ],
        "networks": [
            {
                "dst_port": 6380,
                "member_idx": 1,
                "port_type": "udp",
                "src_intf": "0.0.0.0",
                "src_port": 6379
            }
        ],
        "restart_mode": "restart",
        "restart_wait": 30,
        "service_class": "gbpdataacquisition",
        "service_name": "gbpdataacquisitiond02adb1f35d343a4acaf7b6861dc2714",
        "service_state": "run"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /cntr/service?cmd=delete
- **Target Id**: cntr_service_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
- **Released In**: None

#### Description
Purge (collect garbage) container services.

Container service resources that are no longer needed will be purged according to the `grace_period`.

#### Example
```
# Data Input
{
    "cmd": "delete",
    "data": {
        "grace_period": "30m"
    }
}
```

---

### URI - /cntr?cmd=delete
- **Target Id**: cntr_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
- **Released In**: None

#### Description
Purge (collect garbage) container services and images.

Services and images will be purged according to the `grace_period`.

#### Example
```
# Data Input
{
    "cmd": "delete",
    "data": {
        "grace_period": "1d"
    }
}
```

---

### URI - /diag/log
- **Target Id**: diag_log_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **create\_time** : Log creation time.
        - **log\_id**      : Log identifier
        - **log\_size**    : Last known size of the log in bytes.
        - **log\_state**   : Log usage state: `open`|`closed`|`orphan`
        - **log\_url**     : URL to get log content
        - **mod\_time**    : Last known modification time.
        - **task\_id**     : Optional `task_id` that produced the log.
- **Released In**: None

#### Description
Diagnostic log list.

#### Example
```
# Output
{
    "data": [
        {
            "create_time": "2020-02-05T10:21:49",
            "log_id": "abcd1234",
            "log_size": 54665,
            "log_state": "open",
            "log_url": "https://192.168.254.88/log/abcd1234",
            "mod_time": "2020-02-05T10:24:22",
            "task_id": "abcd1234"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /diag/log/{log\_id}
- **Target Id**: diag_log_details
- **Command**: `get`
- **Output**:
    - **create\_time** : Log creation time.
    - **log\_id**      : Log identifier
    - **log\_size**    : Last known size of the log in bytes.
    - **log\_state**   : Log usage state: `open`|`closed`|`orphan`
    - **log\_url**     : URL to get log content
    - **mod\_time**    : Last known modification time.
    - **task\_id**     : Optional `task_id` that produced the log.
- **Released In**: None

#### Description
Output log details for `log_id`.

#### Example
```
# Output
{
    "data": {
        "create_time": "2020-02-05T10:21:49",
        "log_id": "abcd1234",
        "log_size": 54665,
        "log_state": "open",
        "log_url": "https://192.168.254.88/log/abcd1234.log",
        "mod_time": "2020-02-05T10:24:22",
        "task_id": "abcd1234"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /diag/log/{log\_id}?cmd=delete
- **Target Id**: diag_log_delete
- **Command**: `delete`
- **Released In**: None

#### Description
Delete log `log_id`.

---

### URI - /diag/log?cmd=delete
- **Target Id**: diag_log_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
- **Released In**: None

#### Description
Purge (collect garbage) diagnostic logs.

Orphaned logs will be purged according to the `grace_period`.

#### Example
```
# Data Input
{
    "cmd": "delete",
    "data": {
        "grace_period": "30m"
    }
}
```

---

### URI - /diag/net?cmd=control
- **Target Id**: diag_net_control
- **Command**: `control`
- **Input**:
    - **ping** : 
        - **ip\_addr**      : IP address to ping
        - **ping\_count**   : Number of packets to send.
        - **ping\_timeout** : Ping timeout (secs).
- **Output**:
    - **log\_id**  : Log id for the network diagnostics.
    - **task\_id** : Async task id
- **Released In**: None

#### Description
(**async**) Perform network diagnostics.

#### Example
```
# Data Input
{
    "cmd": "control",
    "data": {
        "ping": {
            "ip_addr": "192.168.254.2",
            "ping_count": 5,
            "ping_timeout": 10
        }
    }
}
# Output
{
    "data": {
        "log_id": "abcd1234",
        "task_id": "3344eeff"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /diag/power
- **Target Id**: diag_power_details
- **Command**: `get`
- **Output**:
    - **pwr\_control**  : 
        - ***Array***:
            - **consumed\_watts**      : Power consumed (watts).
            - **consumed\_watts\_avg** : Power consumed average (watts)
            - **consumed\_watts\_max** : Power consumed max (watts)
            - **consumed\_watts\_min** : Power consumed min (watts)
            - **limit\_watts**         : Power limit (watts).
            - **member\_idx**          : See [member\_idx](#point---member_idx)
            - **pwr\_control\_name**   : Name of power controller.
    - **pwr\_supplies** : 
        - ***Array***:
            - **input\_voltage**       : Line input voltage.
            - **input\_voltage\_type** : Line input voltage type.
            - **member\_idx**          : See [member\_idx](#point---member_idx)
            - **pwr\_supply\_name**    : Name of power supply.
            - **pwr\_supply\_type**    : Type of power supply.
- **Released In**: None

#### Description
Report power diagnostics.

#### Example
```
# Output
{
    "data": {
        "pwr_control": [
            {
                "consumed_watts": 1.72,
                "consumed_watts_avg": 1.71,
                "consumed_watts_max": 2.25,
                "consumed_watts_min": 1.49,
                "limit_watts": 0,
                "member_idx": 1,
                "pwr_control_name": "RDU301 Power"
            }
        ],
        "pwr_supplies": [
            {
                "input_voltage": 12,
                "input_voltage_type": "AC120V",
                "member_idx": 1,
                "pwr_supply_name": "RDU301 Power Supply",
                "pwr_supply_type": "AC"
            }
        ]
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /diag/resource
- **Target Id**: diag_resource_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **resource\_class** : Resource class.
        - **resource\_id**    : Resource identifier.
        - **resource\_name**  : Resource name.
- **Released In**: None

#### Description
Resource diagnostic list.

#### Example
```
# Output
{
    "data": [
        {
            "resource_class": "filesystem",
            "resource_id": "storage",
            "resource_name": "root filesystem"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /diag/resource/{resource\_id}
- **Target Id**: diag_resource_details
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **ext**             : 
            - **val** : Resource details based on class.
        - **resource\_class** : Resource class.
        - **resource\_id**    : Resource identifier.
        - **resource\_name**  : Resource name.
- **Released In**: None

#### Description
Details diagnostics for system resources.

Possible resource classes:
- `cpu`
- `filesystem`
- `memory`

#### Example
```
# Output
{
    "data": [
        {
            "ext": {
                "val": ""
            },
            "resource_class": "filesystem",
            "resource_id": "storage",
            "resource_name": "root filesystem"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /diag/snapshot
- **Target Id**: diag_snapshot
- **Command**: `get`
- **Output**:
    - **log\_id**  : Log id of the snapshot tgz archive that is created.
    - **task\_id** : Async task id
- **Released In**: None

#### Description
(**async**) Dynamically take a snapshot of diagnostics and metrics.
        
Starts a task to gather diagnostic information into a single snapshot log.  The end result of
the task will be a tgz archive file suitable for engineering analysis.

When the snapshot task completes, the diagnostic log file at `log_id` will be populated.

#### Example
```
# Output
{
    "data": {
        "log_id": "abcd1234",
        "task_id": "3344eeff"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /diag/thermal
- **Target Id**: diag_thermal_details
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **max\_reading\_range**        : Maximum reading range (celsius).
        - **member\_idx**                : See [member\_idx](#point---member_idx)
        - **min\_reading\_range**        : Maximum reading range (celsius).
        - **physical\_context**          : Sensor context.
        - **reading\_celsius**           : Sensor reading (celsius).
        - **sensor\_name**               : Sensor name.
        - **sensor\_number**             : Sensor number.
        - **upper\_threshold\_critical** : Upper limit (critical).
        - **upper\_threshold\_fatal**    : Upper limit (fatal).
- **Released In**: None

#### Description
Report thermal diagnostics.

#### Example
```
# Output
{
    "data": [
        {
            "max_reading_range": 67,
            "member_idx": 1,
            "min_reading_range": 42,
            "physical_context": "CPU",
            "reading_celsius": 54,
            "sensor_name": "CPU1 Temperature",
            "sensor_number": 1,
            "upper_threshold_critical": 90,
            "upper_threshold_fatal": 95
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /eth
- **Target Id**: eth_interface_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **eth\_interface\_id**   : Ethernet interface id.
        - **eth\_interface\_name** : Ethernet interface name.
- **Released In**: None

#### Description
Ethernet interface list.

#### Example
```
# Output
{
    "data": [
        {
            "eth_interface_id": "enp2s0",
            "eth_interface_name": "enp2s0(LAN1)"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /eth/{eth\_interface\_id}
- **Target Id**: eth_interface_details
- **Command**: `get`
- **Output**:
    - **auto\_neg**            : Auto-negotiate (bool)
    - **enabled**              : Enabled (bool)
    - **eth\_interface\_id**   : Ethernet interface id.
    - **eth\_interface\_name** : Ethernet interface name.
    - **full\_duplex**         : Full duplex (bool)
    - **host\_name**           : Host name.
    - **ipv4\_addresses**      : 
        - ***Array***:
            - **address**           : IPv4 address
            - **address\_origin**   : Address origin (`static`|`dhcp`)
            - **gateway**           : Gateway address
            - **member\_idx**       : See [member\_idx](#point---member_idx)
            - **subnet\_mask\_len** : Subnet mask length
    - **ipv6\_addresses**      : 
        - ***Array***:
            - **address**         : IPv6 address
            - **address\_origin** : Address origin (`static`|`dhcpv6`)
            - **address\_state**  : Address state
            - **member\_idx**     : See [member\_idx](#point---member_idx)
            - **prefix\_len**     : Prefix length
    - **mac\_address**         : MAC address
    - **mtu\_size**            : Maximum transmission unit (bytes)
    - **name\_servers**        : 
        - ***Array***: Name server address (IPv4 and/or IPv6)
    - **speed\_mbps**          : Interface speed (Mbps)
- **Released In**: None

#### Description
Ethernet interface details for `eth_interface_id`.

#### Example
```
# Output
{
    "data": {
        "auto_neg": true,
        "enabled": true,
        "eth_interface_id": "enp2s0",
        "eth_interface_name": "enp2s0(LAN1)",
        "full_duplex": true,
        "host_name": "RDU-BB39946E-5CBB-11D4-9177-00E0862D49C6",
        "ipv4_addresses": [
            {
                "address": "126.1.70.200",
                "address_origin": "static",
                "gateway": "126.1.70.1",
                "member_idx": 1,
                "subnet_mask_len": 24
            }
        ],
        "ipv6_addresses": [
            {
                "address": "fe80::2e0:86ff:fe2d:49c6",
                "address_origin": "dhcpv6",
                "address_state": "preferred",
                "member_idx": 1,
                "prefix_len": 64
            }
        ],
        "mac_address": "00:e0:86:2d:49:c6",
        "mtu_size": 1500,
        "name_servers": [
            "8.8.8.8"
        ],
        "speed_mbps": 100
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /eth/{eth\_interface\_id}?cmd=set
- **Target Id**: eth_interface_update
- **Command**: `set`
- **Output**:
    - **enabled**         : Enabled (bool)
    - **host\_name**      : Host name.
    - **ipv4\_addresses** : 
        - ***Array***:
            - **address**           : IPv4 address
            - **address\_origin**   : Address origin (`static`|`dhcp`)
            - **gateway**           : Gateway address
            - **member\_idx**       : See [member\_idx](#point---member_idx)
            - **subnet\_mask\_len** : Subnet mask length
    - **ipv6\_addresses** : 
        - ***Array***:
            - **address**         : IPv6 address
            - **address\_origin** : Address origin (`static`|`dhcpv6`)
            - **address\_state**  : Address state
            - **member\_idx**     : See [member\_idx](#point---member_idx)
            - **prefix\_len**     : Prefix length
    - **name\_servers**   : 
        - ***Array***: Name server address (IPv4 and/or IPv6)
- **Released In**: None

#### Description
Ethernet interface update for `eth_interface_id`.

#### Example
```
# Output
{
    "data": {
        "enabled": true,
        "host_name": "RDU-BB39946E-5CBB-11D4-9177-00E0862D49C6",
        "ipv4_addresses": [
            {
                "address": "126.1.70.200",
                "address_origin": "static",
                "gateway": "126.1.70.1",
                "member_idx": 1,
                "subnet_mask_len": 24
            }
        ],
        "ipv6_addresses": [
            {
                "address": "fe80::2e0:86ff:fe2d:49c6",
                "address_origin": "dhcpv6",
                "address_state": "preferred",
                "member_idx": 1,
                "prefix_len": 64
            }
        ],
        "name_servers": [
            "8.8.8.8"
        ]
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /net
- **Target Id**: net_summary
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **enabled**            : Network service is enabled.
        - **net\_service\_id**   : Network service id.
        - **net\_service\_intf** : Network service bind interface.
        - **port\_num**          : Network port number.
        - **port\_type**         : Network port type `tcp`|`udp`.
- **Released In**: None

#### Description
Network service list.

The RDU301 currently supports the following network services:

| id     | port     | notes              |
|--------|----------|--------------------|
| http   | tcp:80   | disabled           |
| https  | tcp:443  |                    |
| ntp    | udp:123  |                    |
| redis  | tcp:6380 | stunnel connection |
| snmp   | udp:161  |                    |
| ssdp   | udp:1900 |                    |
| ssh    | tcp:22   |                    |

#### Example
```
# Output
{
    "data": [
        {
            "enabled": false,
            "net_service_id": "http",
            "net_service_intf": "0.0.0.0",
            "port_num": 80,
            "port_type": "tcp"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /net/redis
- **Target Id**: net_redis_details
- **Command**: `get`
- **Output**:
    - **enabled**            : Network service is enabled.
    - **net\_service\_id**   : Network service id.
    - **net\_service\_intf** : Network service bind interface.
    - **port\_num**          : Network port number.
    - **port\_type**         : Network port type `tcp`|`udp`.
    - **remote\_host**       : Redis remote host
    - **remote\_port**       : Redis remote port
- **Released In**: None

#### Description
Redis secure tunnel service public details.

#### Example
```
# Output
{
    "data": {
        "enabled": true,
        "net_service_id": "redis",
        "net_service_intf": "0.0.0.0",
        "port_num": 6380,
        "port_type": "tcp",
        "remote_host": "126.1.70.100",
        "remote_port": 6379
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /net/redis?cmd=set
- **Target Id**: net_redis_update
- **Command**: `set`
- **Input**:
    - **certificate**        : Secure tunnel certificate content to replace /etc/gsf/rdu-manager.pem
    - **enabled**            : Network service is enabled.
    - **key**                : Redis key text
    - **net\_service\_intf** : Network service bind interface.
    - **port\_num**          : Network port number.
    - **remote\_host**       : Redis remote host
    - **remote\_port**       : Redis remote port
- **Released In**: None

#### Description
Configure secure communications between RDU301 Redis and client.

Formally known as ***admission***, this target allows the client to configure a secure Redis tunnel between the RDU301
and the client.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "certificate": "-----BEGIN CERTIFICATE-----\nMIID1zCCAr+gAwI...",
        "enabled": true,
        "key": "@7gS3Q<smn/D^AA{",
        "net_service_intf": "0.0.0.0",
        "port_num": 6380,
        "remote_host": "126.1.70.100",
        "remote_port": 6379
    }
}
```

---

### URI - /net/ssdp
- **Target Id**: net_ssdp_details
- **Command**: `get`
- **Output**:
    - **direct\_notify\_addrs**     : 
        - ***Array***: Direct notify IP addresses.
    - **enabled**                   : Network service is enabled.
    - **net\_service\_id**          : Network service id.
    - **net\_service\_intf**        : Network service bind interface.
    - **notify\_ipv6\_scope**       : SSDP notify IPv6 scope `link`|`site`|`organization`
    - **notify\_mcast\_intv\_secs** : SSDP notify multicast interval (secs)
    - **notify\_ttl**               : SSDP notify time-to-live
    - **port\_num**                 : Network port number.
    - **port\_type**                : Network port type `tcp`|`udp`.
- **Released In**: None

#### Description
SSDP service public details.

#### Example
```
# Output
{
    "data": {
        "direct_notify_addrs": [
            "126.1.70.100"
        ],
        "enabled": true,
        "net_service_id": "ssdp",
        "net_service_intf": "0.0.0.0",
        "notify_ipv6_scope": "site",
        "notify_mcast_intv_secs": 900,
        "notify_ttl": 2,
        "port_num": 1900,
        "port_type": "udp"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /net/ssdp?cmd=set
- **Target Id**: net_ssdp_update
- **Command**: `set`
- **Output**:
    - **direct\_notify\_addrs**     : 
        - ***Array***: Direct notify IP addresses.
    - **enabled**                   : Network service is enabled.
    - **net\_service\_intf**        : Network service bind interface.
    - **notify\_ipv6\_scope**       : SSDP notify IPv6 scope `link`|`site`|`organization`
    - **notify\_mcast\_intv\_secs** : SSDP notify multicast interval (secs)
    - **notify\_ttl**               : SSDP notify time-to-live
    - **port\_num**                 : Network port number.
    - **port\_type**                : Network port type `tcp`|`udp`.
- **Released In**: None

#### Description
Update SSDP service configuration.

#### Example
```
# Output
{
    "data": {
        "direct_notify_addrs": [
            "126.1.70.100"
        ],
        "enabled": true,
        "net_service_intf": "0.0.0.0",
        "notify_ipv6_scope": "site",
        "notify_mcast_intv_secs": 900,
        "notify_ttl": 2,
        "port_num": 1900,
        "port_type": "udp"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /net/{net\_service\_id}
- **Target Id**: net_service_details
- **Command**: `get`
- **Output**:
    - **enabled**            : Network service is enabled.
    - **net\_service\_id**   : Network service id.
    - **net\_service\_intf** : Network service bind interface.
    - **port\_num**          : Network port number.
    - **port\_type**         : Network port type `tcp`|`udp`.
- **Released In**: None

#### Description
Network service details.

#### Example
```
# Output
{
    "data": {
        "enabled": false,
        "net_service_id": "http",
        "net_service_intf": "0.0.0.0",
        "port_num": 80,
        "port_type": "tcp"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /net/{net\_service\_id}?cmd=set
- **Target Id**: net_service_update
- **Command**: `get`
- **Output**:
    - **enabled**            : Network service is enabled.
    - **net\_service\_intf** : Network service bind interface.
    - **port\_num**          : Network port number.
- **Released In**: None

#### Description
Network service details.

#### Example
```
# Output
{
    "data": {
        "enabled": false,
        "net_service_intf": "0.0.0.0",
        "port_num": 80
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /sec/cert
- **Target Id**: sec_cert_list
- **Command**: `get`
- **Released In**: None

#### Description
TBD

---

### URI - /sec/cert/{cert\_id}
- **Target Id**: sec_cert_details
- **Command**: `get`
- **Released In**: None

#### Description
TBD

---

### URI - /sec/cert/{cert\_id}?cmd=delete
- **Target Id**: sec_cert_delete
- **Command**: `delete`
- **Released In**: None

#### Description
TBD

---

### URI - /sec/cert/{cert\_id}?cmd=set
- **Target Id**: sec_cert_update
- **Command**: `set`
- **Released In**: None

#### Description
TBD

---

### URI - /sw
- **Target Id**: sw_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **sw\_component\_id** : Software component id.
        - **version**           : Software version.
- **Released In**: None

#### Description
List software components.

#### Example
```
# Output
{
    "data": [
        {
            "sw_component_id": "rfs",
            "version": "2.0.8"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /sw/rfs
- **Target Id**: sw_rfs_details
- **Command**: `get`
- **Output**:
    - **build\_commit**     : Software build commit.
    - **build\_time**       : Software build timestamp.
    - **sw\_component\_id** : Software component id.
    - **version**           : Software version.
- **Released In**: None

#### Description
Root file system software details.

#### Example
```
# Output
{
    "data": {
        "build_commit": "abcdef1",
        "build_time": "2020-02-03T00:00:00",
        "sw_component_id": "rfs",
        "version": "2.0.8"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /sw/rfs?cmd=set
- **Target Id**: sw_rfs_update
- **Command**: `set`
- **Input**:
    - **image\_url**     : Image URL to download new RFS software (`.rp3`).
    - **update\_mode**   : Update mode hint: `normal`|`network-copy-only`|`factory-defaults`
    - **update\_reboot** : Reboot on success: `reboot`|`no-reboot`
- **Output**:
    - **log\_id**  : Log id for any update log output and messages.
    - **task\_id** : Async task id
- **Released In**: None

#### Description
(**async**) Root file system software update.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "image_url": "http://192.168.254.251/depot/rdu/rdu3-2.0.8.rp3",
        "update_mode": "normal",
        "update_reboot": "reboot"
    }
}
# Output
{
    "data": {
        "log_id": "abcd1234",
        "task_id": "3344eeff"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /sys
- **Target Id**: sys_details
- **Command**: `get`
- **Output**:
    - **api\_version**               : Fixed version string for the REST API
    - **asset\_tag**                 : Asset tag string stored in the BIOS.
    - **boot\_source\_override**     : 
        - **enabled** : Boot source override enabled
        - **mode**    : Boot source override mode
        - **target**  : Boot source override target
    - **firmware\_version**          : Firmware version
    - **manufacturer**               : Manufacturer
    - **model\_name**                : Specific model of the unit
    - **model\_type**                : Class or type of the unit
    - **serial\_number**             : Serial number for the unit
    - **sku**                        : Stock keeping unit identifier
    - **system\_id**                 : Unique system/unit identifier
    - **system\_memory\_gib**        : Total system memory in GiB
    - **system\_processor\_count**   : Total number of processors/cores for the system
    - **system\_processor\_summary** : Processor description
- **Released In**: None

#### Description
System details.

#### Example
```
# Output
{
    "data": {
        "api_version": "2.0",
        "asset_tag": "",
        "boot_source_override": {
            "enabled": "Continuous",
            "mode": "UEFI",
            "target": "Hdd"
        },
        "firmware_version": "2.0.8",
        "manufacturer": "Vertiv",
        "model_name": "RDU301 Computer System",
        "model_type": "Vertiv Gateway",
        "serial_number": "0040471204",
        "sku": "900-357-002-002",
        "system_id": "RDU301-0040471204",
        "system_memory_gib": 4,
        "system_processor_count": 4,
        "system_processor_summary": "Intel(R) Celeron(R) CPU N3160 @ 1.60GHz"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /sys?cmd=control
- **Target Id**: sys_shutdown
- **Command**: `control`
- **Input**:
    - **action** : `reboot`|`halt`|`to-dormant`|`factory-reset`
- **Released In**: None

#### Description
Shut the system down.

Performs a shutdown based on **action**:

- `reboot`: Reboot to current (active) partition.
- `halt`: Halt the system. **REQUIRES POWER CYCLING TO RESTART**
- `to-dormant`: Reboot to *dormant* partition.
- `factory-reset`: Reset to factory defaults.  **ALL CONFIGURATION IS LOST**

Default is `reboot`.

#### Example
```
# Data Input
{
    "cmd": "control",
    "data": {
        "action": "reboot"
    }
}
```

---

### URI - /task
- **Target Id**: task_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **task\_id**    : Unique task identifier
        - **task\_state** : Task execution state
        - **time\_start** : Task start time
- **Released In**: None

#### Description
Task list.

#### Example
```
# Output
{
    "data": [
        {
            "task_id": "1122aabb",
            "task_state": "run",
            "time_start": "2020-02-03T15:43:00"
        }
    ],
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /task/{task\_id}
- **Target Id**: task_details
- **Command**: `get`
- **Output**:
    - **ext\_data**    : 
        - **ext\_val** : Values in `ext_data` are optional and dependent on the creating task.
    - **ext\_msg**     : 
        - ***Array***: Task messages
    - **status\_code** : Exit status code
    - **status\_msg**  : Exit status message
    - **task\_id**     : Unique task identifier
    - **task\_state**  : Task execution state
    - **time\_start**  : Task start time
    - **time\_stop**   : Task start time
- **Released In**: None

#### Example
```
# Output
{
    "data": {
        "ext_data": {
            "ext_val": ""
        },
        "ext_msg": [
            "Installation successful."
        ],
        "status_code": 0,
        "status_msg": "OK",
        "task_id": "1122aabb",
        "task_state": "run",
        "time_start": "2020-02-03T15:43:00",
        "time_stop": "2020-02-03T15:43:40"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /task/{task\_id}?cmd=control
- **Target Id**: task_control
- **Command**: `control`
- **Output**:
    - **action** : Task action to perform `cancel`|`kill`
- **Released In**: None

#### Description
Task cancel/kill.

#### Example
```
# Output
{
    "data": {
        "action": "cancel"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /task/{task\_id}?cmd=delete
- **Target Id**: task_delete
- **Command**: `delete`
- **Released In**: None

#### Description
Mark a task for delete.

---

### URI - /task?cmd=delete
- **Target Id**: task_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
- **Released In**: None

#### Description
Purge (collect garbage) tasks.

Completed tasks will be purged according to the `grace_period`.

#### Example
```
# Data Input
{
    "cmd": "delete",
    "data": {
        "grace_period": "30m"
    }
}
```

---

### URI - /time
- **Target Id**: time_summary
- **Command**: `get`
- **Output**:
    - **uptime** : Uptime in seconds.
    - **utc**    : System UTC time.
- **Released In**: None

#### Description
System time information.

#### Example
```
# Output
{
    "data": {
        "uptime": 1234.567,
        "utc": "2020-02-03T04:05:06"
    },
    "status": {
        "code": 0,
        "msg": "OK"
    }
}
```

---

### URI - /time/utc?cmd=set
- **Target Id**: time_update
- **Command**: `set`
- **Input**:
    - **utc** : System UTC time.
- **Released In**: None

#### Description
Set UTC time.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "utc": "2020-02-03T04:05:06"
    }
}
```

---

### URI - /users/{user\_id}?cmd=set
- **Target Id**: user_password
- **Command**: `set`
- **Input**:
    - **new\_password** : New password
    - **old\_password** : Optional old password value to confirm
- **Released In**: None

#### Description
Changes the password for `user_id`.  If the **old\_password** field is present, then confirm it prior to
changing to the new password.  Failure to confirm is a failure of the request.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "new_password": "yyy",
        "old_password": "xxx"
    }
}
```

---

### URI - /vcloud
- **Target Id**: vcloud_summary
- **Command**: `get`
- **Released In**: None

#### Description
This target is TBD.

---



## Common Points
### Point - member\_idx
Member index.  The array index of this member within the parent collection.

---

### Point - purge\_grace\_period
Purge grace period.  Defines a time after which objects can be purged.

Grace period can take the form `0d0h0m0s` where the `0` can be replaced by an integer, and each letter denotes:

- `d` : days
- `h` : hours
- `m` : minutes
- `s` : seconds

Each field is optional.  If no letter is provided, seconds are assumed.

---

