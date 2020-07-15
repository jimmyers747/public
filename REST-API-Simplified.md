
# RDU301 Simplified REST API
**Objective:** *Remove and Replace Stingray*

---

**NOTE: DOCUMENT IS AUTOMATICALLY GENERATED**

*Do not update this document directly.*  Instead, update `simple_rest_api.py` and run it as a python3 script.

---
## Executive Summary
- This API is a replacement for the `/redfish/v1` API implemented by Stingray.
- All URIs defined below are relative to the root path `/rapi/v1`.
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
    - [/user/admin](#uri---useruser_idcmdset)
        - Change the admin user password
        - Not needed for machine-to-machine communication, more internal from the web service
    - [/sec/cert](#uri---seccert)
        - Security certificate management
        - Handles access keys and certificates
    - [/sec/jwt](#uri---secjwt)
        - Java web token (JWT) key management
        - Handles machine-to-machine token validation keys
    
---
## API Basics
- Loosely modeled after the [Geist API](http://releases.geist.local/documentation/api/latest/geist_api_public.html)
- Synchronous/asynchronous request processing:
  + Some requests, typically those of long duration, are processed asynchronously. The initial HTTP response
    will be `202 Accepted`, and a task ID will be provided to allow monitoring of the task using the
    [/task](#uri---task) resource. Only a few requests are expected to be processed asynchronously.
  + Most resources are always processed either synchronously or asynchronously. However, a few have been enhanced
    to support either processing method. For these cases, the client can choose the processing method using a query
    parameter `async`. It can be set to `yes` or `no` (alternatively `true` or `false`). If this parameter is not
    specified, the processing method defaults in such a way to maintain backward compatibility.
  + If the `query` parameter is used for a resource that only supports one processing method, the query parameter
    must match the supported method or the request will fail.
  + In the resource descriptions provided later in this document, the supported sync/async method(s) is
    provided at the start of the description, if it is not strictly async. In the case of resources that support
    both methods, the default is assumed for any output document, and in examples.
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
            - `code`    : Numeric code (0 = success, 1-254 = error/warning)
            - `msgs`    : Array of text message(s) associated with the status. For error cases, this attribute
                          will typically contain diagnostic information.
            - `task_id` : For asynchronous (e.g. long duration) operations, the task ID to use for monitoring
                          the operation via the [/task](#uri---task) resource. This attribute should always
                          be present in the case of a `202 Accepted` HTTP status code.
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

| **-**               | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |
|---------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------|
| **cntr**            |                                                 | [[X]](#uri---cntrcmdcontrol)                    | [[X]](#uri---cntrcmddelete)                     | [[X]](#uri---cntr)                              |                                                 |
| **cntr.image**      | [[X]](#uri---cntrimagecmdadd)                   |                                                 | [[X]](#uri---cntrimagecmddelete)                | [[X]](#uri---cntrimage)                         |                                                 |
| **cntr.image.id**   |                                                 |                                                 | [[X]](#uri---cntrimageimage\_idcmddelete)       | [[X]](#uri---cntrimageimage\_id)                |                                                 |
| **cntr.service**    | [[X]](#uri---cntrservicecmdadd)                 |                                                 | [[X]](#uri---cntrservicecmddelete)              | [[X]](#uri---cntrservice)                       |                                                 |
| **cntr.service.id** |                                                 | [[X]](#uri---cntrserviceservice\_idcmdcontrol)  | [[X]](#uri---cntrserviceservice\_idcmddelete)   | [[X]](#uri---cntrserviceservice\_id)            |                                                 |
| **diag.log**        |                                                 |                                                 |                                                 | [[X]](#uri---diaglogfilter)                     | [[X]](#uri---diaglogcmdset)                     |
| **diag.net**        |                                                 | [[X]](#uri---diagnetcmdcontrol)                 |                                                 |                                                 |                                                 |
| **diag.power**      |                                                 |                                                 |                                                 | [[X]](#uri---diagpower)                         |                                                 |
| **diag.resource**   |                                                 |                                                 |                                                 | [[X]](#uri---diagresource)                      |                                                 |
| **diag.snapshot**   |                                                 |                                                 |                                                 | [[X]](#uri---diagsnapshot)                      |                                                 |
| **diag.thermal**    |                                                 |                                                 |                                                 | [[X]](#uri---diagthermal)                       |                                                 |
| **-**               | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |
| **eth**             |                                                 |                                                 |                                                 | [[X]](#uri---eth)                               |                                                 |
| **eth.id**          |                                                 |                                                 |                                                 | [[X]](#uri---etheth\_interface\_id)             | [[X]](#uri---etheth\_interface\_idcmdset)       |
| **net**             |                                                 |                                                 |                                                 | [[X]](#uri---net)                               |                                                 |
| **net.https**       |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **net.id**          |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **net.ntp**         |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **net.redis**       |                                                 |                                                 |                                                 | [[X]](#uri---netredis)                          | [[X]](#uri---netrediscmdset)                    |
| **net.ssdp**        |                                                 |                                                 |                                                 | [[X]](#uri---netssdp)                           | [[X]](#uri---netssdpcmdset)                     |
| **net.ssh**         |                                                 |                                                 |                                                 | [[X]](#uri---netnet\_service\_id)               | [[X]](#uri---netnet\_service\_idcmdset)         |
| **sec.cert.srv**    |                                                 |                                                 | [[X]](#uri---seccertsrvcmddelete)               | [[X]](#uri---seccertsrv)                        | [[X]](#uri---seccertsrvcmdset)                  |
| **-**               | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |
| **sec.jwt.cfg**     |                                                 |                                                 |                                                 | [[X]](#uri---secjwtcfg)                         | [[X]](#uri---secjwtcfgcmdset)                   |
| **sec.jwt.id**      |                                                 |                                                 | [[X]](#uri---secjwtkeykey\_idcmddelete)         | [[X]](#uri---secjwtkeykey\_id)                  | [[X]](#uri---secjwtkeykey\_idcmdset)            |
| **sec.jwt.list**    |                                                 |                                                 |                                                 | [[X]](#uri---secjwtkey)                         |                                                 |
| **sw**              |                                                 |                                                 |                                                 | [[X]](#uri---sw)                                |                                                 |
| **sw.rfs**          |                                                 |                                                 |                                                 | [[X]](#uri---swrfs)                             | [[X]](#uri---swrfscmdset)                       |
| **sys**             |                                                 | [[X]](#uri---syscmdcontrol)                     |                                                 | [[X]](#uri---sys)                               |                                                 |
| **task**            |                                                 |                                                 | [[X]](#uri---taskcmddelete)                     | [[X]](#uri---task)                              |                                                 |
| **task.id**         |                                                 | [[X]](#uri---tasktask\_idcmdcontrol)            | [[X]](#uri---tasktask\_idcmddelete)             | [[X]](#uri---tasktask\_id)                      |                                                 |
| **time**            |                                                 |                                                 |                                                 | [[X]](#uri---time)                              | [[X]](#uri---timeutccmdset)                     |
| **user.admin**      |                                                 | [[X]](#uri---useruser\_idcmdcontrol)            |                                                 |                                                 | [[X]](#uri---useruser\_idcmdset)                |
| **vcloud**          |                                                 |                                                 |                                                 | [[X]](#uri---vcloud)                            |                                                 |
| **-**               | **add**                                         | **control**                                     | **delete**                                      | **get**                                         | **set**                                         |


---
## Target URIs
### URI - /
- **Target Id**: root
- **Command**: `get`
- **Output**:
    - **cntr\_control**           : [URI](#uri---cntrcmdcontrol) - `/cntr?cmd=control` *(Global actions on cntr objects.)*
    - **cntr\_image\_delete**     : [URI](#uri---cntrimageimage_idcmddelete) - `/cntr/image/{image_id}?cmd=delete` *(Delete a container image if not in use.)*
    - **cntr\_image\_deploy**     : [URI](#uri---cntrimagecmdadd) - `/cntr/image?cmd=add` *((**async**) Deploy a container image.)*
    - **cntr\_image\_details**    : [URI](#uri---cntrimageimage_id) - `/cntr/image/{image_id}` *(Container image details.)*
    - **cntr\_image\_list**       : [URI](#uri---cntrimage) - `/cntr/image` *(Container image list.)*
    - **cntr\_image\_purge**      : [URI](#uri---cntrimagecmddelete) - `/cntr/image?cmd=delete` *(Purge (collect garbage) container images.)*
    - **cntr\_purge**             : [URI](#uri---cntrcmddelete) - `/cntr?cmd=delete` *(Purge (collect garbage) container services and images.)*
    - **cntr\_service\_control**  : [URI](#uri---cntrserviceservice_idcmdcontrol) - `/cntr/service/{service_id}?cmd=control` *((**default sync, optiontally async**) Control a container service.)*
    - **cntr\_service\_delete**   : [URI](#uri---cntrserviceservice_idcmddelete) - `/cntr/service/{service_id}?cmd=delete` *((**default sync, optiontally async**) Delete a container service.)*
    - **cntr\_service\_deploy**   : [URI](#uri---cntrservicecmdadd) - `/cntr/service?cmd=add` *(Container service deploy.)*
    - **cntr\_service\_details**  : [URI](#uri---cntrserviceservice_id) - `/cntr/service/{service_id}` *(Container service details.)*
    - **cntr\_service\_list**     : [URI](#uri---cntrservice) - `/cntr/service` *(Container summary.)*
    - **cntr\_service\_purge**    : [URI](#uri---cntrservicecmddelete) - `/cntr/service?cmd=delete` *(Purge (collect garbage) container services.)*
    - **cntr\_summary**           : [URI](#uri---cntr) - `/cntr` *(Container summary.)*
    - **diag\_log\_level**        : [URI](#uri---diaglogcmdset) - `/diag/log?cmd=set` *(Set persistent journal logging level.)*
    - **diag\_log\_list**         : [URI](#uri---diaglogfilter) - `/diag/log?{filter}` *(Diagnostic log list.)*
    - **diag\_net\_control**      : [URI](#uri---diagnetcmdcontrol) - `/diag/net?cmd=control` *((**async**) Perform network diagnostics.)*
    - **diag\_power\_details**    : [URI](#uri---diagpower) - `/diag/power` *(Report power diagnostics.)*
    - **diag\_resource\_list**    : [URI](#uri---diagresource) - `/diag/resource` *(List of resources.)*
    - **diag\_snapshot**          : [URI](#uri---diagsnapshot) - `/diag/snapshot` *(Dynamically take a snapshot of diagnostics and metrics.)*
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
    - **sec\_cert\_srv\_delete**  : [URI](#uri---seccertsrvcmddelete) - `/sec/cert/srv?cmd=delete` *(Delete server certificate, reset to factory default.)*
    - **sec\_cert\_srv\_details** : [URI](#uri---seccertsrv) - `/sec/cert/srv` *(Return the content (PEM) of the web server TLS/SSL certificate.)*
    - **sec\_cert\_srv\_update**  : [URI](#uri---seccertsrvcmdset) - `/sec/cert/srv?cmd=set` *(Update the web server TLS/SSL certificate.)*
    - **sec\_jwt\_cfg\_details**  : [URI](#uri---secjwtcfg) - `/sec/jwt/cfg` *(Return JWT subsystem configuration.)*
    - **sec\_jwt\_cfg\_update**   : [URI](#uri---secjwtcfgcmdset) - `/sec/jwt/cfg?cmd=set` *(Update JWT subsystem configuration.)*
    - **sec\_jwt\_delete**        : [URI](#uri---secjwtkeykey_idcmddelete) - `/sec/jwt/key/{key_id}?cmd=delete` *(Delete JWT key. Factory keys cannot be deleted.)*
    - **sec\_jwt\_details**       : [URI](#uri---secjwtkeykey_id) - `/sec/jwt/key/{key_id}` *(Show JWT key details.)*
    - **sec\_jwt\_list**          : [URI](#uri---secjwtkey) - `/sec/jwt/key` *(List installed JWT keys.)*
    - **sec\_jwt\_update**        : [URI](#uri---secjwtkeykey_idcmdset) - `/sec/jwt/key/{key_id}?cmd=set` *(Add/update JWT key.  Note that factory keys cannot be added/updated.)*
    - **sw\_list**                : [URI](#uri---sw) - `/sw` *(List software components.)*
    - **sw\_rfs\_details**        : [URI](#uri---swrfs) - `/sw/rfs` *(Root file system software details.)*
    - **sw\_rfs\_update**         : [URI](#uri---swrfscmdset) - `/sw/rfs?cmd=set` *((**async**) Root file system software update.)*
    - **sys\_details**            : [URI](#uri---sys) - `/sys` *(System details.)*
    - **sys\_shutdown**           : [URI](#uri---syscmdcontrol) - `/sys?cmd=control` *((**async**) Shut the system down.)*
    - **task\_control**           : [URI](#uri---tasktask_idcmdcontrol) - `/task/{task_id}?cmd=control` *(Task cancel/kill.)*
    - **task\_delete**            : [URI](#uri---tasktask_idcmddelete) - `/task/{task_id}?cmd=delete` *(Mark a task for delete.)*
    - **task\_details**           : [URI](#uri---tasktask_id) - `/task/{task_id}`
    - **task\_list**              : [URI](#uri---task) - `/task` *(Task list.)*
    - **task\_purge**             : [URI](#uri---taskcmddelete) - `/task?cmd=delete` *(Purge (collect garbage) tasks.)*
    - **time\_summary**           : [URI](#uri---time) - `/time` *(System time information.)*
    - **time\_update**            : [URI](#uri---timeutccmdset) - `/time/utc?cmd=set` *(Set UTC time.)*
    - **user\_auth**              : [URI](#uri---useruser_idcmdcontrol) - `/user/{user_id}?cmd=control` *(Authenticate `user_id`.  Verify password for user.  Failure will return `401 Unathorized`.)*
    - **user\_password**          : [URI](#uri---useruser_idcmdset) - `/user/{user_id}?cmd=set` *(Changes the password for `user_id`.)*
    - **vcloud\_summary**         : [URI](#uri---vcloud) - `/vcloud` *(This target is TBD.)*
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
        "cntr_control": "/cntr?cmd=control",
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
        "diag_log_level": "/diag/log?cmd=set",
        "diag_log_list": "/diag/log?{filter}",
        "diag_net_control": "/diag/net?cmd=control",
        "diag_power_details": "/diag/power",
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
        "sec_cert_srv_delete": "/sec/cert/srv?cmd=delete",
        "sec_cert_srv_details": "/sec/cert/srv",
        "sec_cert_srv_update": "/sec/cert/srv?cmd=set",
        "sec_jwt_cfg_details": "/sec/jwt/cfg",
        "sec_jwt_cfg_update": "/sec/jwt/cfg?cmd=set",
        "sec_jwt_delete": "/sec/jwt/key/{key_id}?cmd=delete",
        "sec_jwt_details": "/sec/jwt/key/{key_id}",
        "sec_jwt_list": "/sec/jwt/key",
        "sec_jwt_update": "/sec/jwt/key/{key_id}?cmd=set",
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
        "user_auth": "/user/{user_id}?cmd=control",
        "user_password": "/user/{user_id}?cmd=set",
        "vcloud_summary": "/vcloud"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
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
            - **service\_id**    : Service id.
            - **service\_state** : Service state `run`|`stop`|`stop-app`.
#### Description
Container summary.

Service states:

- `run`: Service is running.
- `stop`: Service is stopped either because it was stopped via this API, or
  it has failed. In the case of a service failure, a restart might be attempted
  after some wait time - this is based on the `restart-mode` chosen when the
  service was added.
- `stop-app`: Service is stopped and will not restart due to the presence of a
  `do_not_restart.json` health file. This file will be deleted if a service
  command operation of `start` is done via this API.

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
                "service_id": "3d25301a",
                "service_state": "run"
            }
        ]
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
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
        - **image\_state** : Image state `used`|`orphan`
#### Description
Container image list.

#### Example
```
# Output
{
    "data": [
        {
            "image_id": "796c45706c2b",
            "image_state": "used"
        }
    ],
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /cntr/image/{image\_id}
- **Target Id**: cntr_image_details
- **Command**: `get`
- **Output**:
    - **image\_id**     : Image id.
    - **image\_name**   : Image name.
    - **image\_sha256** : SHA256 sum of the image file.
    - **image\_state**  : Image state `used`|`orphan`
    - **image\_url**    : URL to fetch image
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
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /cntr/image/{image\_id}?cmd=delete
- **Target Id**: cntr_image_delete
- **Command**: `delete`
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
- **Output**:
#### Description
(**async**) Deploy a container image.

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
# Output
{
    "status": {
        "code": 0,
        "msgs": [
            "In progress"
        ],
        "task_id": "1234"
    }
}
```

---

### URI - /cntr/image?cmd=delete
- **Target Id**: cntr_image_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
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
        - **image\_id**      : Image id.
        - **service\_id**    : Service id.
        - **service\_state** : Service state `run`|`stop`|`stop-app`.
#### Description
Container summary.

#### Example
```
# Output
{
    "data": [
        {
            "image_id": "1c2b6f88",
            "service_id": "3d25301a",
            "service_state": "run"
        }
    ],
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /cntr/service/{service\_id}
- **Target Id**: cntr_service_details
- **Command**: `get`
- **Output**:
    - **host\_network**  : Host network enabled.
    - **image\_id**      : Image id
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
    - **service\_id**    : Service id.
    - **service\_state** : Service state `run`|`stop`|`stop-app`.
    - **start\_time**    : Start time.
    - **stop\_time**     : Stop time.
#### Description
Container service details.

#### Example
```
# Output
{
    "data": {
        "host_network": false,
        "image_id": "796c45706c2b",
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
        "service_id": "3d25301a",
        "service_state": "run",
        "start_time": "2020-02-03T01:02:03",
        "stop_time": "2020-02-06T00:01:02"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /cntr/service/{service\_id}?cmd=control
- **Target Id**: cntr_service_control
- **Command**: `control`
- **Input**:
    - **action** : `start`|`stop`
#### Description
(**default sync, optiontally async**) Control a container service.

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
#### Description
(**default sync, optiontally async**) Delete a container service.

Running services will be stopped.

---

### URI - /cntr/service?cmd=add
- **Target Id**: cntr_service_deploy
- **Command**: `add`
- **Input**:
    - **bind\_net\_redis**    : Bind to net-redis service for secure tunnel.
    - **envs**                : 
        - ***Array***:
            - **name**  : Environment var name
            - **value** : Environemnt var value
    - **force\_add**          : Force deployment.
                                Take action if another service with the same `service_class` already exists.
                                If `true`, then remove the previous service and deploy this one.
                                If `false`, then fail this deployment.
    - **host\_network**       : Host network enabled.
    - **image\_id**           : Image id
    - **mounts**              : 
        - ***Array***:
            - **dst\_path**   : Mount destination path.
            - **member\_idx** : See [member\_idx](#point---member_idx)
            - **mount\_type** : Mount type.
            - **src\_path**   : Mount source path.
    - **networks**            : 
        - ***Array***:
            - **dst\_port**   : Destination port.
            - **member\_idx** : See [member\_idx](#point---member_idx)
            - **port\_type**  : Network port type `udp`|`tcp`.
            - **src\_intf**   : Source network interface.
            - **src\_port**   : Source network port.
    - **restart\_mode**       : Action to take on container service abnormal exit `restart`|`reboot`|`ignore`.
    - **restart\_wait**       : Time to wait after abnormal exit before restarting (secs).
    - **service\_exec**       : 
        - **args**    : 
            - ***Array***: Command argument
        - **command** : Start command
    - **service\_id**         : Image id
    - **service\_run\_state** : Service run state `run`|`stop`.
#### Description
Container service deploy.

#### Example
```
# Data Input
{
    "cmd": "add",
    "data": {
        "bind_net_redis": true,
        "envs": [
            {
                "name": "REMOTE_IP",
                "value": "10.11.12.13"
            }
        ],
        "force_add": false,
        "host_network": false,
        "image_id": "796c45706c2b",
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
        "service_exec": {
            "args": [
                "myapp.dll"
            ],
            "command": "dotnet"
        },
        "service_id": "06c2b34d9c68",
        "service_run_state": "run"
    }
}
```

---

### URI - /cntr/service?cmd=delete
- **Target Id**: cntr_service_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
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

### URI - /cntr?cmd=control
- **Target Id**: cntr_control
- **Command**: `control`
- **Input**:
    - **action** : `clean`
#### Description
Global actions on cntr objects.

Performs an operation based on **action**:

- `clean`: Kills and removes **ALL** services and images. Normally used as a tool for recovery.

#### Example
```
# Data Input
{
    "cmd": "control",
    "data": {
        "action": "clean"
    }
}
```

---

### URI - /cntr?cmd=delete
- **Target Id**: cntr_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
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

### URI - /diag/log?cmd=set
- **Target Id**: diag_log_level
- **Command**: `set`
- **Input**:
    - **level** : New logging level (`warning`|`notice`|`info`|`debug`).
#### Description
Set persistent journal logging level.

This target changes the persistent journal logging level.  Valid values for the `level`
attribute are (increasing in verbosity): `warning`, `notice`, `info`, `debug`.  Changing
the logging level will change the verbosity of messages written to the journal.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "level": "info"
    }
}
```

---

### URI - /diag/log?{filter}
- **Target Id**: diag_log_list
- **Command**: `get`
- **Output**:
    - **cursor** : Log cursor at end of provided lines
    - **lines**  : 
        - ***Array***: Log line
#### Description
Diagnostic log list.

Journal log lines are extracted. The filter terms are based on [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html) and
are specified using an HTTP query parameter list. As with [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html), a cursor can be
used to page through logs. 

Supported filter query paramters are:

- `after-cursor=CURSOR` : Start the log after the `CURSOR`.  You obtain a cursor from the `cursor` attribute of a previous log request.
- `dormant=1` : Read logs from the dormant partition (previous installed version).
- `identifier=x` : 
- `lines=N` : Limit number of lines to `N`.
- `no-hostname=1` : Do not include the hostname in the message text.
- `output=FORMAT` : See the `--output=` switch for [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html)
- `output-fields=FIELDS` : See the `--output-fields=` switch for [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html)
- `priority=PRIORITY` : See the `--priority=` switch for [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html)
- `quiet=1` : Do not include boot time information in log text.
- `reverse=1` : Reverse the order of messages (newest first).
- `since=SINCE` : See the `--since=` switch for [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html)
- `since-boot=1` : Start log messages from the most recent boot.
- `unit=UNIT` : To be defined.
- `until=UNTIL` : See the `--until=` switch for [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html)
- `user-unit=UNIT` : To be defined.
- `utc=1` : Timestamps in UTC

#### Example
```
# Output
{
    "data": {
        "cursor": "s=e2510ca6680a471fb5be0b...",
        "lines": [
            "Mar 07 00:42:19 west_rdu NetworkManager[749] Start"
        ]
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
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
    - **task\_data**   : 
        - **exit\_code**      : Exit code from ping
        - **msgs**            : Output (stdout and stderr) from ping
        - **receive\_count**  : Number of packets received
        - **transmit\_count** : Number of packets sent
    - **task\_status** : 
        - **code** : Final status code
        - **msgs** : Final message list
#### Description
(**async**) Perform network diagnostics.

As this is an asynchronous operation, the output shown below will be part of
a `task_details` document.

#### Example
```
# Data Input
{
    "cmd": "control",
    "data": {
        "ping": {
            "ip_addr": "192.168.254.2",
            "ping_count": 2,
            "ping_timeout": 10
        }
    }
}
# Output
{
    "data": {
        "task_data": {
            "exit_code": 0,
            "msgs": [
                "PING 192.168.254.2 (192.168.254.2) 56 data bytes",
                "64 bytes from 192.168.254.2: seq=0 ttl=64 time=1.990 ms",
                "64 bytes from 192.168.254.2: seq=1 ttl=64 time=1.033 ms",
                "",
                "--- 192.168.254.2 ping statistics ---",
                "2 packets transmitted, 2 received, 0% packet loss"
            ],
            "receive_count": 2,
            "transmit_count": 2
        },
        "task_status": {
            "code": 0,
            "msgs": [
                "OK"
            ]
        }
    },
    "status": {
        "code": 0,
        "msgs": [
            "In progress"
        ],
        "task_id": "1234"
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
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /diag/resource
- **Target Id**: diag_resource_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **measurement**    : Measurement info.
        - **resource\_id**   : Resource identifier.
        - **resource\_type** : Resource type.
#### Description
List of resources. For each resource type and ID, a measurement is provided.
The list below shows the supported resource types, followed by a list of supported IDs.
- "cpu" : "minute", "hour", "since\_boot"
- "memory" : "total", "free"
- "filesystem" : "/", "/tmp", "/var/volatile"
For each resource and ID, there is a "measurement" object that contains a "units"
attribute, and one or more attributes containing the measurement value. Examples 
for each resource type are as follows:
- "cpu"
  ```
  "measurement": {
      "units": "percent",
      "value": 2.3
  }
  ```
- "memory"
  ```
  "measurement": {
      "units": "kB",
      "value": 3443776
  }
  ```
- "filesystem"
  ```
  "measurement": {
      "free": 1712211,
      "total": 1721888,
      "units": "kB"
  }
  ```

#### Example
```
# Output
{
    "data": [
        {
            "measurement": "See description above",
            "resource_id": "/tmp",
            "resource_type": "filesystem"
        }
    ],
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /diag/snapshot
- **Target Id**: diag_snapshot
- **Command**: `get`
- **Output**:
    - **lines** : Lines from the snapshot log.
#### Description
Dynamically take a snapshot of diagnostics and metrics.
        
Gathers diagnostic information into a single snapshot log.  This operation
may take up to 10 seconds to complete.   Output is written into the data.lines
array.

#### Example
```
# Output
{
    "data": {
        "lines": [
            "Tue Apr 28 16:50:01 UTC 2020",
            " 16:50:01 up  1:28,  load average: 0.44, 0.31, 0.30",
            "",
            "----------------------------------------",
            "--- cat /etc/gsf/gsf_version.sh",
            "# Root File System Build Info",
            "RFS_VERSION=\"2.1.1.3\"",
            "RFS_COMMIT=\"22cf10a\"",
            "RFS_BUILD_TIME=\"20200428-111810\"",
            "RFS_GIT_BRANCH=\"dev\"",
            "RFS_GIT_REPO_URL=\"git@ghe.int.vertivco.com:SaSS-RDU/rdu-build.git\"",
            "",
            "----------------------------------------",
            "--- df",
            "Filesystem           1K-blocks      Used Available Use% Mounted on",
            "/dev/root             12147376   1000440  10523220   9% /",
            "devtmpfs               1722724         0   1722724   0% /dev",
            "tmpfs                  1723424         0   1723424   0% /dev/shm",
            "tmpfs                  1723424      1176   1722248   0% /run",
            "tmpfs                  1723424         0   1723424   0% /sys/fs/cgroup",
            "tmpfs                  1723424         0   1723424   0% /var/volatile",
            "tmpfs                  1723424        12   1723412   0% /tmp",
            "/dev/sda1                65390     25094     40296  38% /boot",
            "/dev/sda4             12147376   1440800  10082860  13% /mnt/dormant",
            "overlay               12147376   1000440  10523220   9% /var/lib/docker/overlay2/be88bb126e2492ab6dc8427c5cca7625143e4856a8befd4642c5ae80a826bf2a/merged",
            "overlay               12147376   1000440  10523220   9% /var/lib/docker/overlay2/c2c4199f80979a544f7f83512b44e004dfba0b4bc0218de52c6b7ccce7221ff1/merged",
            "overlay               12147376   1000440  10523220   9% /var/lib/docker/overlay2/af23b07c935923a7590d73b790142085eee6d146dd9324a8b7ffc874a1acd1a6/merged",
            "shm                      65536         8     65528   0% /var/lib/docker/containers/e2fa363f01dc3df021f6f90ae6765b6ddfc70ab9d1357071d4ffd25c6be9ddfb/mounts/shm",
            "shm                      65536         8     65528   0% /var/lib/docker/containers/3ce23b7358d5b698f000c0a3f89b01455b3b874cab2d70f7ceb0bc1e45aa84f7/mounts/shm",
            "shm                      65536         8     65528   0% /var/lib/docker/containers/cde4d1cab2ed648c510896acb4af912063a2faec21c5c4920cb73e3f320bd0e2/mounts/shm",
            "",
            "----------------------------------------",
            "--- ps aux",
            "PID   USER     TIME   COMMAND",
            "    1 root       0:01 {systemd} /sbin/init",
            "    2 root       0:00 [kthreadd]",
            "    3 root       0:00 [ksoftirqd/0]",
            "..."
        ]
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
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
        "msgs": [
            "OK"
        ]
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
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /eth/{eth\_interface\_id}
- **Target Id**: eth_interface_details
- **Command**: `get`
- **Output**:
    - **broadcast\_enabled**   : Broadcast is enabled (bool)
    - **carrier**              : Carrier found (bool)
    - **config**               : 
        - **dhcp\_use\_dns** : Use DHCP name servers (bool)
        - **ipv4**           : 
            - **addrs** : 
                - ***Array***:
                    - **address**   : IPv4 address
                    - **broadcast** : Broadcast address
                    - **gateway**   : Gateway address
                    - **mask\_len** : Subnet mask length
                    - **metric**    : Route metric (int)
            - **mode**  : Address mode (`static`|`dhcp`)
        - **ipv6**           : 
            - **addrs** : 
                - ***Array***:
                    - **address**     : IPv6 address
                    - **prefix\_len** : Prefix length
            - **mode**  : Address mode (`static`|`dhcp`)
        - **name\_servers**  : 
            - ***Array***: Name server address (IPv4 and/or IPv6)
    - **current**              : 
        - **ipv4**          : 
            - **addrs**  : 
                - ***Array***:
                    - **address**    : IPv4 address
                    - **broadcast**  : Broadcast address
                    - **mask\_len**  : Subnet mask length
                    - **pref\_lft**  : Preferred lifetime
                    - **valid\_lft** : Valid lifetime
            - **routes** : 
                - ***Array***:
                    - **gateway** : Gateway address
                    - **metric**  : Route metric (int)
        - **ipv6**          : 
            - **addrs** : 
                - ***Array***:
                    - **address**     : IPv6 address
                    - **pref\_lft**   : Preferred lifetime
                    - **prefix\_len** : Prefix length
                    - **valid\_lft**  : Valid lifetime
        - **name\_servers** : 
            - ***Array***: Name server address (IPv4 and/or IPv6)
    - **duplex**               : Duplex (`full`|`half`)
    - **eth\_interface\_id**   : Ethernet interface id.
    - **eth\_interface\_name** : Ethernet interface name.
    - **mac\_address**         : MAC address
    - **mtu\_size**            : Maximum transmission unit (bytes)
    - **multicast\_enabled**   : Multicast is enabled (bool)
    - **oper\_state**          : Operational state (`up`|`down`|`unknown`...)
    - **speed\_mbps**          : Interface speed (Mbps)
#### Description
Ethernet interface details for `eth_interface_id`.

#### Example
```
# Output
{
    "data": {
        "broadcast_enabled": true,
        "carrier": true,
        "config": {
            "dhcp_use_dns": false,
            "ipv4": {
                "addrs": [
                    {
                        "address": "126.1.70.200",
                        "broadcast": "126.255.255.255",
                        "gateway": "126.1.70.1",
                        "mask_len": 24,
                        "metric": 200
                    }
                ],
                "mode": "static"
            },
            "ipv6": {
                "addrs": [
                    {
                        "address": "fe80::2e0:86ff:fe2d:49c6",
                        "prefix_len": 64
                    }
                ],
                "mode": "static"
            },
            "name_servers": [
                "8.8.8.8"
            ]
        },
        "current": {
            "ipv4": {
                "addrs": [
                    {
                        "address": "126.1.70.200",
                        "broadcast": "126.255.255.255",
                        "mask_len": 24,
                        "pref_lft": "forever",
                        "valid_lft": "forever"
                    }
                ],
                "routes": [
                    {
                        "gateway": "126.1.70.1",
                        "metric": 200
                    }
                ]
            },
            "ipv6": {
                "addrs": [
                    {
                        "address": "fe80::2e0:86ff:fe2d:49c6",
                        "pref_lft": "forever",
                        "prefix_len": 64,
                        "valid_lft": "forever"
                    }
                ]
            },
            "name_servers": [
                "8.8.8.8"
            ]
        },
        "duplex": "full",
        "eth_interface_id": "enp2s0",
        "eth_interface_name": "enp2s0(LAN1)",
        "mac_address": "00:e0:86:2d:49:c6",
        "mtu_size": 1500,
        "multicast_enabled": true,
        "oper_state": "up",
        "speed_mbps": 100
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /eth/{eth\_interface\_id}?cmd=set
- **Target Id**: eth_interface_update
- **Command**: `set`
- **Input**:
    - **dhcp\_use\_dns** : Use DHCP name servers (bool)
    - **ipv4**           : 
        - **addrs** : 
            - ***Array***:
                - **address**   : IPv4 address
                - **broadcast** : Broadcast address
                - **gateway**   : Gateway address
                - **mask\_len** : Subnet mask length
        - **mode**  : Address mode (`static`|`dhcp`)
    - **ipv6**           : 
        - **addrs** : 
            - ***Array***:
                - **address**     : IPv6 address
                - **prefix\_len** : Prefix length
        - **mode**  : Address mode (`static`|`dhcp`)
    - **name\_servers**  : 
        - ***Array***: Name server address (IPv4 and/or IPv6)
#### Description
Ethernet interface update for `eth_interface_id`.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "dhcp_use_dns": false,
        "ipv4": {
            "addrs": [
                {
                    "address": "126.1.70.200",
                    "broadcast": "126.255.255.255",
                    "gateway": "126.1.70.1",
                    "mask_len": 24
                }
            ],
            "mode": "static"
        },
        "ipv6": {
            "addrs": [
                {
                    "address": "fe80::2e0:86ff:fe2d:49c6",
                    "prefix_len": 64
                }
            ],
            "mode": "static"
        },
        "name_servers": [
            "8.8.8.8"
        ]
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
        "msgs": [
            "OK"
        ]
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
    - **started**            : Network service is started.
#### Description
Redis secure tunnel service public details.

#### Example
```
# Output
{
    "data": {
        "enabled": true,
        "net_service_id": "redis",
        "net_service_intf": "localhost",
        "port_num": 6380,
        "port_type": "tcp",
        "remote_host": "126.1.70.100",
        "remote_port": 6379,
        "started": true
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
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
    - **started**            : Network service is started.
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
        "net_service_intf": "localhost",
        "port_num": 6380,
        "remote_host": "126.1.70.100",
        "remote_port": 6379,
        "started": true
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
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /net/ssdp?cmd=set
- **Target Id**: net_ssdp_update
- **Command**: `set`
- **Input**:
    - **direct\_notify\_addrs**     : 
        - ***Array***: Direct notify IP addresses.
    - **enabled**                   : Network service is enabled.
    - **net\_service\_intf**        : Network service bind interface.
    - **notify\_ipv6\_scope**       : SSDP notify IPv6 scope `link`|`site`|`organization`
    - **notify\_mcast\_intv\_secs** : SSDP notify multicast interval (secs)
    - **notify\_ttl**               : SSDP notify time-to-live
    - **port\_num**                 : Network port number.
    - **port\_type**                : Network port type `tcp`|`udp`.
#### Description
Update SSDP service configuration.

#### Example
```
# Data Input
{
    "cmd": "set",
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
#### Description
Network service details.

#### Example
```
# Output
{
    "data": {
        "enabled": false,
        "net_service_id": "ssh",
        "net_service_intf": "0.0.0.0",
        "port_num": 22,
        "port_type": "tcp"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /net/{net\_service\_id}?cmd=set
- **Target Id**: net_service_update
- **Command**: `set`
- **Input**:
    - **enabled**            : Network service is enabled.
    - **net\_service\_intf** : Network service bind interface.
    - **port\_num**          : Network port number.
#### Description
Network service details.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "enabled": true,
        "net_service_intf": "0.0.0.0",
        "port_num": 22
    }
}
```

---

### URI - /sec/cert/srv
- **Target Id**: sec_cert_srv_details
- **Command**: `get`
- **Output**:
    - **cert\_pem**   : Server certificate content (PEM).
    - **key\_sha256** : Server private key hash (SHA256).
- **Released In**: 2.1.6

#### Description
Return the content (PEM) of the web server TLS/SSL certificate.

#### Example
```
# Output
{
    "data": {
        "cert_pem": "-----BEGIN CERTIFICATE-----\nMIIGFDCCA/y...",
        "key_sha256": "97a313566ee8206544aefa7e473dc9bd4c91cfcfa8c75f794ec8110fde2598ad"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /sec/cert/srv?cmd=delete
- **Target Id**: sec_cert_srv_delete
- **Command**: `delete`
- **Released In**: 2.1.6

#### Description
Delete server certificate, reset to factory default.
**Restart required**.

---

### URI - /sec/cert/srv?cmd=set
- **Target Id**: sec_cert_srv_update
- **Command**: `set`
- **Input**:
    - **cert\_pem**      : Server certificate content (PEM).
    - **priv\_key\_pem** : Server certificate private key (PEM).
    - **priv\_key\_pw**  : Private key password (if encrypted)
- **Released In**: 2.1.6

#### Description
Update the web server TLS/SSL certificate.
**Restart required**.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "cert_pem": "-----BEGIN CERTIFICATE-----\nMIIGFDCCA/y...",
        "priv_key_pem": "-----BEGIN RSA PRIVATE KEY-----\nMIIJKgIBAAK...",
        "priv_key_pw": "s3o2v-*Tg"
    }
}
```

---

### URI - /sec/jwt/cfg
- **Target Id**: sec_jwt_cfg_details
- **Command**: `get`
- **Output**:
    - **factory\_jwt\_enabled** : Enable/disable factory installed JWT key.
    - **factory\_jwt\_public**  : Enable/disable factory installed JWT public key.
    - **factory\_jwt\_salted**  : Enable/disable factory installed JWT key (salted).
#### Description
Return JWT subsystem configuration.

#### Example
```
# Output
{
    "data": {
        "factory_jwt_enabled": true,
        "factory_jwt_public": true,
        "factory_jwt_salted": true
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /sec/jwt/cfg?cmd=set
- **Target Id**: sec_jwt_cfg_update
- **Command**: `set`
- **Input**:
    - **factory\_jwt\_enabled** : Enable/disable factory installed JWT key.
    - **factory\_jwt\_public**  : Enable/disable factory installed JWT public key.
    - **factory\_jwt\_salted**  : Enable/disable factory installed JWT key (salted).
- **Released In**: 2.1.6

#### Description
Update JWT subsystem configuration.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "factory_jwt_enabled": true,
        "factory_jwt_public": true,
        "factory_jwt_salted": true
    }
}
```

---

### URI - /sec/jwt/key
- **Target Id**: sec_jwt_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **is\_factory**   : JWT key is factory installed.
        - **is\_persisted** : JWT key is persisted.
        - **key\_id**       : JWT key ID.
        - **key\_sha256**   : JWT key content hash (SHA256).
        - **key\_type**     : JWT key type: `hmac`|`rsa`.
- **Released In**: 2.1.6

#### Description
List installed JWT keys.

#### Example
```
# Output
{
    "data": [
        {
            "is_factory": false,
            "is_persisted": false,
            "key_id": "rest-jwt-public-key-1",
            "key_sha256": "97a313566ee8206544aefa7e473dc9bd4c91cfcfa8c75f794ec8110fde2598ad",
            "key_type": "rsa"
        }
    ],
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /sec/jwt/key/{key\_id}
- **Target Id**: sec_jwt_details
- **Command**: `get`
- **Output**:
    - **is\_factory**   : JWT key is factory installed.
    - **is\_persisted** : JWT key is persisted.
    - **key\_id**       : JWT key ID.
    - **key\_type**     : JWT key type: `hmac`|`rsa`.
    - **sha256\_hash**  : JWT key content hash (SHA256).
- **Released In**: 2.1.6

#### Description
Show JWT key details.

#### Example
```
# Output
{
    "data": {
        "is_factory": false,
        "is_persisted": false,
        "key_id": "rest-jwt-public-key-1",
        "key_type": "rsa",
        "sha256_hash": "97a313566ee8206544aefa7e473dc9bd4c91cfcfa8c75f794ec8110fde2598ad"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /sec/jwt/key/{key\_id}?cmd=delete
- **Target Id**: sec_jwt_delete
- **Command**: `delete`
- **Released In**: 2.1.6

#### Description
Delete JWT key. Factory keys cannot be deleted.

---

### URI - /sec/jwt/key/{key\_id}?cmd=set
- **Target Id**: sec_jwt_update
- **Command**: `set`
- **Input**:
    - **is\_persisted** : JWT key is persisted.
    - **key\_content**  : JWT key content, `hmac`: shared secret, `rsa`: public key (PEM).
    - **key\_type**     : JWT key type: `hmac`|`rsa`.
- **Released In**: 2.1.6

#### Description
Add/update JWT key.  Note that factory keys cannot be added/updated.

#### Example
```
# Data Input
{
    "cmd": "set",
    "data": {
        "is_persisted": false,
        "key_content": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqh...",
        "key_type": "rsa"
    }
}
```

---

### URI - /sw
- **Target Id**: sw_list
- **Command**: `get`
- **Output**:
    - ***Array***:
        - **sw\_component\_id** : Software component id.
        - **version**           : Software version.
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
        "msgs": [
            "OK"
        ]
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
        "msgs": [
            "OK"
        ]
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
    - **log\_id** : Log id for any update log output and messages.
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
        "log_id": "abcd1234"
    },
    "status": {
        "code": 0,
        "msgs": [
            "In progress"
        ],
        "task_id": "1234"
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
    - **host\_name**                 : Host name
    - **manufacturer**               : Manufacturer
    - **model\_name**                : Specific model of the unit
    - **model\_type**                : Class or type of the unit
    - **serial\_number**             : Serial number for the unit
    - **sku**                        : Stock keeping unit identifier
    - **system\_id**                 : Unique system/unit identifier
    - **system\_memory\_gib**        : Total system memory in GiB
    - **system\_processor\_count**   : Total number of processors/cores for the system
    - **system\_processor\_summary** : Processor description
    - **uuid**                       : Unit UUID
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
        "host_name": "RDU-BB39946E-5CBB-11D4-9177-00E0862D49C6",
        "manufacturer": "Vertiv",
        "model_name": "RDU301 Computer System",
        "model_type": "Vertiv Gateway",
        "serial_number": "0040471204",
        "sku": "900-357-002-002",
        "system_id": "RDU301-0040471204",
        "system_memory_gib": 4,
        "system_processor_count": 4,
        "system_processor_summary": "Intel(R) Celeron(R) CPU N3160 @ 1.60GHz",
        "uuid": "BB39946E-5CBB-11D4-9177-00E0862D49C6"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /sys?cmd=control
- **Target Id**: sys_shutdown
- **Command**: `control`
- **Input**:
    - **action** : `reboot`|`halt`|`to-dormant`|`factory-reset`
- **Output**:
#### Description
(**async**) Shut the system down.

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
# Output
{
    "status": {
        "code": 0,
        "msgs": [
            "In progress"
        ],
        "task_id": "1234"
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
        - **task\_state** : Task execution state `run`|`exit`
        - **time\_start** : Task start time
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
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /task/{task\_id}
- **Target Id**: task_details
- **Command**: `get`
- **Output**:
    - **task\_id**     : Unique task identifier
    - **task\_state**  : Task execution state `run`|`exit`
    - **task\_status** : 
        - **code** : Task final status code
        - **msgs** : Task final message list
    - **time\_start**  : Task start time
    - **time\_stop**   : Task stop time
#### Example
```
# Output
{
    "data": {
        "task_id": "1122aabb",
        "task_state": "exit",
        "task_status": {
            "code": 0,
            "msgs": [
                "OK"
            ]
        },
        "time_start": "2020-02-03T15:43:00",
        "time_stop": "2020-02-03T15:43:40"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /task/{task\_id}?cmd=control
- **Target Id**: task_control
- **Command**: `control`
- **Output**:
    - **action** : Task action to perform `cancel`|`kill`
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
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /task/{task\_id}?cmd=delete
- **Target Id**: task_delete
- **Command**: `delete`
#### Description
Mark a task for delete.

---

### URI - /task?cmd=delete
- **Target Id**: task_purge
- **Command**: `delete`
- **Input**:
    - **grace\_period** : See [purge\_grace\_period](#point---purge_grace_period)
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
        "msgs": [
            "OK"
        ]
    }
}
```

---

### URI - /time/utc?cmd=set
- **Target Id**: time_update
- **Command**: `set`
- **Input**:
    - **utc** : System UTC time.
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

### URI - /user/{user\_id}?cmd=control
- **Target Id**: user_auth
- **Command**: `control`
- **Input**:
    - **action**   : Action (authenticate)
    - **password** : Password value to confirm
#### Description
Authenticate `user_id`.  Verify password for user.  Failure will return `401 Unathorized`.

#### Example
```
# Data Input
{
    "cmd": "control",
    "data": {
        "action": "authenticate",
        "password": "xxx"
    }
}
```

---

### URI - /user/{user\_id}?cmd=set
- **Target Id**: user_password
- **Command**: `set`
- **Input**:
    - **new\_password** : New password
    - **old\_password** : Optional old password value to confirm
#### Description
Changes the password for `user_id`.  If the **old\_password** field is present, then confirm it prior to
changing to the new password.  Failure to confirm will return `401 Unauthorized`.

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

