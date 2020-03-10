# RDU301 Simplified REST API - Getting Started
**For API Clients**

## Important Notes
- API doc is [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md)
    - **NOTE**: The API document uses section titles with *query string* parameters like `/cntr/service?cmd=add`.  
      The use of query string parameters is for human-readability only.  The `cmd` parameter should be part of 
      the `POST` JSON document body.  However, if a `cmd` is not provided in the document or as a query parameter,
      then `get` is implied.
    - Also note that API sections like `/cntr/service` are **relative** to the base URL path `/rapi/v1`.  Therefore,
      a full path would be `https://192.168.254.1/rapi/v1/cntr/service`.
- For interim development releases, there is no authentication mechanism
    - JWT authentication will be integrated in the near future


## Mapping from Redfish to Simplified

- Admission
    - Redfish
        - A series of exchanges between client and server to authenticate and exchange data
        - Resulted in starting an `stunnel` between client and server
    - Simplified
        - See `/rapi/v1/net/redis?cmd=set` [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---netrediscmdset)
        - Starts the `stunnel` service
- Set Time
    - Redfish: `PATCH /redfish/v1/Managers/1`, `DateTime` = `YYYY-MM-DD HH:MM:SS`
    - Simplified: See `/rapi/v1/time` [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---timeutccmdset)
- System Info
    - Redfish
        - `GET /redfish/v1/Chassis/RDU301-SERIAL-NUMBER`
        - `GET /redfish/v1/Managers/1/`
        - `GET /redfish/v1/Systems/RDU301-SERIALNUMBER`
    - Simplified: See `/rapi/v1/sys` [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---sys)
- Container Pre-Configuration
    - Redfish: `POST /redfish/v1/Managers/ContainerManager/Actions/Oem/VertivcoRKTManager/VertivcoRKTManager.RKTConfig`
        - `ConfigUrl` : File to download as rdu.config.xml
    - Simplified
        - Convert configuration values to environment variable tag/value pairs
        - See `/rapi/v1/cntr/service?cmd=add` [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---cntrservicecmdadd)
        - Reference the `envs` array
- Containers
    - Redfish
        - Run: `/redfish/v1/Managers/ContainerManager/Actions/Oem/VertivcoRKTManager/VertivcoRKTManager.RKTRun`
            - Includes image name, image url, service file url
            - Requires knowledge of service file parameters
        - List Images: `/redfish/v1/Managers/ContainerManager/Actions/Oem/VertivcoRKTManager/VertivcoRKTManager.RKTListInstalledContainers`
            - Raw output from `rkt image list`
        - List Containers: `/redfish/v1/Managers/ContainerManager/Actions/Oem/VertivcoRKTManager/VertivcoRKTManager.RKTList`
            - Raw output from `rkt list`
        - Stop Container: `/redfish/v1/Managers/ContainerManager/Actions/Oem/VertivcoRKTManager/VertivcoRKTManager.RKTStop`
        - Remove Image or Container: `/redfish/v1/Managers/ContainerManager/Actions/Oem/VertivcoRKTManager/VertivcoRKTManager.RKTRm`
    - See [Container Management API](#container-management-api) below
- Async Tasks
    - Redfish: `GET /redfish/v1/Tasks/Tasks/TASK-ID`
    - Simplified: See `/rapi/v1/task/{task_id}` [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---tasktask_id)
- Firmware Update
    - Redfish: `POST /redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate`
    - Simplified: See `/rapi/v1/sw/rfs?cmd=set` [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---swrfscmdset)
- Reboot
    - Redfish: `POST /redfish/v1/Chassis/RDU301-SERIAL-NUMBER/Actions/Chassis.Reset`
    - Simplified: See `/rapi/v1/sys?cmd=control` [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---syscmdcontrol)


## Container Management API

The API is based on management of images and services (i.e. containers).  The container portion of the API spec starts [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---cntr).
In general, a container (service) can be started by:

- Install an image with `POST` `/rapi/v1/cntr/image` cmd: `add`, specifying an image id.
- Add a service with `POST` `/rapi/v1/cntr/service` cmd: `add`, specifying an image id and a service id.


### Installing an Image
When an image is installed, a unique ID must be specified, which is used throughout the API. Internally, this ID is mapped to a docker "repo:tag", although the "repo:tag" is not visible via the REST API.

Applicable *image* requests that are currently supported are:

- Add an image: `/rapi/v1/cntr/image`, cmd: `add`
- List images: `/rapi/v1/cntr/image`, cmd `get`
- Delete an image: `/rapi/v1/cntr/image/{id}` cmd: `delete`

When a container is installed (and optionally run), a unique ID must be specified, which is used throughout the API. Internally, this is mapped to a docker "name".

### Adding a Service

The systemd service is managed by the RDU301. This means that information that was previously specified via the unit/service file is now provided via the REST API (e.g. mounts,  env vars, start cmd, etc.) using `/cntr/service`, cmd: `add`.

Currently, we don't support a way for the REST user to install a file containing env vars, that is to be run when a container is started. So now, all env vars must be specified via `/cntr/service add` (see the `envs` attribute).

Applicable requests that are currently supported are:

- Add a service and optionally enable/start it: `/rapi/v1/cntr/service`, cmd: `add`
- List services:  `/rapi/v1/cntr/service` cmd: `get`
- Start/stop/enable/disable a service: `/rapi/v1/cntr/service/{id}` cmd: `control`
- Delete a service: `/rapi/v1/cntr/service/{id}` cmd: `delete`

### iCOM-S Redfish Secure Tunnel Service (aka Redfish *Admission*)
The legacy Redfish API provided an *Admission* service that ultimately started an `stunnel` between the RDU301 and iCOM-S.  This
*Admission* protocol has been replaced with the `/net/redis` service described [here](https://github.com/jimmyers747/public/blob/master/REST-API-Simplified.md#uri---netredis).
If this secure tunnel is required to enable Redis communications between iCOM-S containers and the iCOM-S Redis instance, then use
the `/net/redis` (cmd: `set`) service before starting any containers.


## Async Tasks

There is a general mechanism to deal with asynchronous (long duration) operations. Currently this consists of `/rapi/v1/cntr/image` `add` and `/rapi/v1/sw/rfs` `set`. It works like this:

- When you make the request, you will get `202 Accepted` and the output status object will contain a `task_id`.
- You can poll the status of the task using `/rapi/v1/task/{id}` `get`.
- A key attribute is `exit_code` -- if present it indicates the task is complete (you can also monitor `task_state`). Another key attribute is `ext_msg`, an array of JSON objects, the last of which will typically contain the request final result. Intermediate JSON objects might be present with progress info.

Applicable requests that are currently supported are:

- List tasks: `/rapi/v1/task` cmd: `get`
- Get status of a task: `/task/{id}` cmd: `get`

Eventually we will need to clean up old task info, but this isn't supported and shouldn't be critical for now.


## Logging

You can get logging using `/rapi/v1/diag/log` `get` where the query string can be used to specify filters which are mapped to [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html) filters in an obvious way.

The current filter query parameters are:

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

Example: `https://126.1.70.82:443/rapi/v1/diag/log?lines=5&since=yesterday`

As more specific logging is needed (e.g. for container log output), more query parameters can be developed later.


## Supported URIs
Below is a list of URIs that are at least somewhat supported.

    /cntr/image (Container image list.)
    /cntr/image?cmd=add (Deploy a container image.)
    /cntr/image/{image_id}?cmd=delete (Delete a container image if not in use.)

    /cntr/service (Container summary.)
    /cntr/service?cmd=add (Container service deploy.)
    /cntr/service/{service_id}?cmd=delete (Delete a container service.)
    /cntr/service/{service_id}?cmd=control (start/stop/enable/disable a service)

    /diag/log?{filter} (Diagnostic log list.)

    /net (Network service list.)

    /sw/rfs?cmd=set ((async) Root file system software update.)

    /sys (System details.)
    /sys?cmd=control (Reboot,halt, etc)

    /task (Task list.)
    /task/{task_id}

    /time (System time information.)
    /time?cmd=set (Set UTC time.)


## Examples
Get list (empty) of images.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/image
Status: 200
Headers:
  Connection: keep-alive
  Accept: */*
  Accept-Encoding: gzip, deflate
  User-Agent: python-requests/2.9.1
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Connection: keep-alive
  Content-Type: application/json
  Content-Encoding: gzip
  Server: RDU301/2.0.8-U1
  Transfer-Encoding: chunked
  Date: Tue, 10 Mar 2020 18:53:58 GMT
Body:
{
    "data": [],
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

Add an image.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/image?cmd=add
Status: 202
Headers:
  Connection: keep-alive
  Content-Length: 233
  Content-Type: application/json
  User-Agent: python-requests/2.9.1
  Accept: */*
  Accept-Encoding: gzip, deflate
Body:
{
    "cmd": "add",
    "data": {
        "image_id": "rdumanagerservices_id",
        "image_sha256": "3cdbf4239ab7433c24f1241a14ab13f5c45774490594c7da38dacb3ba3f2eb5a",
        "image_size": 302317568,
        "image_url": "http://126.1.70.92:8000/rdumanagerservices.tar"
    }
}
----------------------------- Response ------------------------------
Status: 202
Headers:
  Connection: keep-alive
  Server: RDU301/2.0.8-U1
  Content-Length: 55
  Date: Tue, 10 Mar 2020 18:54:03 GMT
  Content-Type: application/json
Body:
{
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ],
        "task_id": "1"
    }
}
```

Check status (initial).

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/task/1
Status: 200
Headers:
  Connection: keep-alive
  Accept-Encoding: gzip, deflate
  User-Agent: python-requests/2.9.1
  Accept: */*
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Server: RDU301/2.0.8-U1
  Connection: keep-alive
  Date: Tue, 10 Mar 2020 18:54:07 GMT
  Content-Encoding: gzip
  Transfer-Encoding: chunked
  Content-Type: application/json
Body:
{
    "data": {
        "ext_msg": [
            {
                "status": {
                    "code": 0,
                    "msgs": [
                        "In progress"
                    ]
                }
            }
        ],
        "task_id": "1",
        "task_state": "run",
        "time_start": "2020-03-10T18:54:03"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

Check status (completed).

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/task/1
Status: 200
Headers:
  Connection: keep-alive
  User-Agent: python-requests/2.9.1
  Accept: */*
  Accept-Encoding: gzip, deflate
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Date: Tue, 10 Mar 2020 18:55:01 GMT
  Server: RDU301/2.0.8-U1
  Connection: keep-alive
  Content-Type: application/json
  Transfer-Encoding: chunked
  Content-Encoding: gzip
Body:
{
    "data": {
        "exit_code": 0,
        "ext_msg": [
            {
                "status": {
                    "code": 0,
                    "msgs": [
                        "In progress"
                    ]
                }
            },
            {
                "status": {
                    "code": 0,
                    "msgs": [
                        "Complete"
                    ]
                }
            }
        ],
        "task_id": "1",
        "task_state": "exit",
        "time_start": "2020-03-10T18:54:03",
        "time_stop": "2020-03-10T18:55:01"
    },
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

Get new list of images.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/image
Status: 200
Headers:
  User-Agent: python-requests/2.9.1
  Accept: */*
  Connection: keep-alive
  Accept-Encoding: gzip, deflate
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Content-Type: application/json
  Server: RDU301/2.0.8-U1
  Connection: keep-alive
  Content-Encoding: gzip
  Transfer-Encoding: chunked
  Date: Tue, 10 Mar 2020 18:55:05 GMT
Body:
{
    "data": [
        {
            "image_id": "rdumanagerservices_id"
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

Get list (empty) of services.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/service
Status: 200
Headers:
  Accept-Encoding: gzip, deflate
  User-Agent: python-requests/2.9.1
  Connection: keep-alive
  Accept: */*
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Transfer-Encoding: chunked
  Date: Tue, 10 Mar 2020 18:55:13 GMT
  Connection: keep-alive
  Content-Type: application/json
  Server: RDU301/2.0.8-U1
  Content-Encoding: gzip
Body:
{
    "data": [],
    "status": {
        "code": 0,
        "msgs": [
            "OK"
        ]
    }
}
```

Create and start a service.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/service?cmd=add
Status: 200
Headers:
  Content-Length: 986
  Connection: keep-alive
  User-Agent: python-requests/2.9.1
  Content-Type: application/json
  Accept: */*
  Accept-Encoding: gzip, deflate
Body:
{
    "cmd": "add",
    "data": {
        "envs": [
            {
                "name": "RDU_AC_ID",
                "value": "1347830111575602432"
            },
            {
                "name": "RDU_REDIS_HOST",
                "value": "localhost"
            },
            {
                "name": "RDU_REDIS_PORT",
                "value": "6380"
            },
            {
                "name": "RDU_REDIS_REMOTE_HOST",
                "value": "192.168.111.111"
            },
            {
                "name": "RDU_REDIS_REMOTE_PORT",
                "value": "6379"
            },
            {
                "name": "RDU_REDIS_KEY",
                "value": "@7gS3Q<smn/D^AA{"
            },
            {
                "name": "RDU_CERT",
                "value": "/etc/gsf/rdu-manager.pem"
            },
            {
                "name": "RDU_UUID",
                "value": "BB39946E-5CBB-11D4-9177-00E0862D49C6"
            }
        ],
        "host_network": true,
        "image_id": "rdumanagerservices_id",
        "mounts": [
            {
                "dst_path": "/etc/rkt/config",
                "mount_type": "bind",
                "src_path": "/etc/rkt/config"
            },
            {
                "dst_path": "/etc/resolv.conf",
                "mount_type": "bind",
                "src_path": "/etc/resolv.conf"
            }
        ],
        "restart_mode": "restart",
        "restart_wait": 34,
        "service_class": "rdumgr_class",
        "service_enable_state": "enabled",
        "service_exec": {
            "args": [
                "GBPDataAcquisitionServiceCore.dll"
            ],
            "command": "dotnet"
        },
        "service_id": "rdumgr_id",
        "service_run_state": "run"
    }
}
----------------------------- Response ------------------------------
Status: 200
Headers:
  Content-Length: 0
  Date: Tue, 10 Mar 2020 18:55:16 GMT
  Content-Type: application/json
  Connection: keep-alive
  Server: RDU301/2.0.8-U1
Body: None
```

Get updated list of sevices.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/service
Status: 200
Headers:
  Connection: keep-alive
  User-Agent: python-requests/2.9.1
  Accept: */*
  Accept-Encoding: gzip, deflate
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Content-Type: application/json
  Server: RDU301/2.0.8-U1
  Date: Tue, 10 Mar 2020 18:55:19 GMT
  Connection: keep-alive
  Transfer-Encoding: chunked
  Content-Encoding: gzip
Body:
{
    "data": [
        {
            "image_id": "rdumanagerservices_id",
            "service_id": "rdumgr_id",
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

Delete service.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/service/rdumgr_id?cmd=delete
Status: 200
Headers:
  Accept: */*
  Connection: keep-alive
  Accept-Encoding: gzip, deflate
  User-Agent: python-requests/2.9.1
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Content-Type: application/json
  Date: Tue, 10 Mar 2020 18:55:34 GMT
  Content-Length: 0
  Server: RDU301/2.0.8-U1
  Connection: keep-alive
Body: None
```

Delete image.

```
----------------------------- Request ------------------------------
URL: https://126.1.70.82:443/rapi/v1/cntr/image/rdumanagerservices_id?cmd=delete
Status: 200
Headers:
  User-Agent: python-requests/2.9.1
  Connection: keep-alive
  Accept-Encoding: gzip, deflate
  Accept: */*
Body: None
----------------------------- Response ------------------------------
Status: 200
Headers:
  Server: RDU301/2.0.8-U1
  Content-Length: 0
  Date: Tue, 10 Mar 2020 18:55:43 GMT
  Content-Type: application/json
  Connection: keep-alive
Body: None
```
