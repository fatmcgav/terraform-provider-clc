# terraform-provider-clc

Terraform provider for CenturyLinkCloud.


## Installation

1. Download the plugin from the [releases tab][3]
2. Put it somewhere in your path or note the full path to the binary.
3. Create or modify your `~/.terraformrc` file. You'll need at least this:

```
providers {
    clc = "~/bin/terraform-provider-clc"
}
```


Alternatively,

`go install github.com/CenturyLinkCloud/terraform-provider-clc/bin/terraform-provider-clc`



[3]:https://github.com/CenturyLinkCloud/terraform-provider-clc/releases



## Example

sample main.tf

    resource "clc_group" "web" {
      location_id = "WA1"
      name = "TERRA"
      parent = "Default Group"
    }
    
    resource "clc_server" "srv" {
      name_template = "trusty"
      source_server_id = "UBUNTU-14-64-TEMPLATE"
      group_id = "${clc_group.web.id}"
      cpu = 2
      memory_mb = 2048
      password = "Green123$"
    }    


More [examples][4]

[4]:https://github.com/CenturyLinkCloud/terraform-provider-clc/tree/master/examples



## Troubleshooting

in addition to terraform's [debug env vars](https://terraform.io/docs/configuration/environment-variables.html), invoking terraform with `DEBUG=1` will dump a `plugin.log` file in the local directory with additional debugging output specific to the clc plugin.




## Usage



### Provider Configuration

#### `clc`

```
provider "clc" {
  username = ""
  password = ""
  account = ""
}
```

* `username` & `password` - credentials used to log in on https://control.ctl.io
* `account` - Account alias

These may also be provided as ENV vars if not found in your terraform configurations.

`CLC_USERNAME`, `CLC_PASSWORD`, `CLC_ACCOUNT`


### Resource Configuration

#### `clc_group`

Creates a new group or resolves to an existing group.

```
resource "clc_group" "group01" {
  location_id = "CA1"
  name = "cluster01"
  parent = "somegroup"
}
```

value                             | Type     | Forces New | Value Type | Description
--------------------------------- | -------- | ---------- | ---------- | -----------
`name`                            | Required | no         | string     | Name of the group.
`parent`                          | Required | no         | string     | Name of the parent group this group will fall under.
`location_id`                     | Required | no         | string     | Datacenter name.
`description`                     | Optional | no         | string     | Description.
`custom_fields`                   | Optional | no         | list<map>  | Custom metadata fields. { id , value } (requires pre-registered account-wide custom fields)

### `clc_server`

Creates a new server instance.

```
resource "clc_server" "web01" {
    name_template = "SRV" # eventual name will be generated by the platform
    source_server_id = "UBUNTU-14-64-TEMPLATE"
    group_id = "${clc_group.NAME.id}"
    cpu = 2
    memory_mb = 2048
    password = "$UP3R$3CUR3!"
}
```



value                             | Type     | Forces New | Value Type | Description
--------------------------------- | -------- | ---------- | ---------- | -----------
`name_tempate`                    | Required | no         | string     | Name of the server. Will be permuted by platform
`source_server_id`                | Required | yes        | string     | VM image to use
`group_id`                        | Required | no         | string     | ID of the group this server will belong to
`cpu`                             | Required | no         | int        | Number of virtual CPUs
`memory_mb`                       | Required | no         | int        | Provisioned memory in MB
`type`                            | Required | no         | string     | Type of build: { standard, hyperscale, bareMetal }
`password`                        | Required | no         | string     | Root password
`description`                     | Optional | no         | string     | Description of server
`power_state`                     | Optional | no         | string     | Default: `started`. Set/Change power state: { started | on | stopped | off | paused | reboot | reset | shutdown }
`private_ip_address`              | Optional | no         | string     | Generated if not provided
`network_id`                      | Optional | no         | string     | ID of the network to place server in
`additional_disks`                | Optional | no         | list<map>  | Specify additional disks. { path, size_gb, type } (see docs)
`custom_fields`                   | Optional | no         | list<map>  | Custom metadata fields. { id , value } (requires pre-registered account-wide custom fields)


full API options documented: [https://www.ctl.io/api-docs/v2/#servers-create-server]


#### `clc_public_ip`

Creates a new public IP and attaches to existing server instance on a specified ip address.

```

resource "clc_public_ip" "mgmt" {
  server_id = "${clc_server.<SOME_NAMED_RESOURCE>.id}"
  internal_ip_address = "${clc_server.<SOME_NAMED_RESOURCE>.private_ip_address}"
  source_restrictions
     { cidr = "108.19.92.15/32" }
  ports
    {
      protocol = "TCP"
      port = 22
    }
  ports
    {
      protocol = "TCP"
      port = 80
    }
  ports
    {
      protocol = "TCP"
      port = 8080
      port_to = 9000
    }
}
```

value                             | Type     | Forces New | Value Type | Description
--------------------------------- | -------- | ---------- | ---------- | -----------
`server_id`                       | Required | yes        | string     | Name of the server to attach IP on.
`internal_ip_address`             | Optional | no         | string     | Internal address of the NIC to attach to. If not provided, a new internal IP will be provisioned.
`ports`                           | Required | no         | list<map>  | List of allowed ports to open on provisioned IP.
`soruce_restrictions`             | Optional | no         | list<map>  | List of CIDR blocks to deny inbound traffic on.


#### `clc_load_balancer`

Creates a loadbalancer object suitable for http(s) traffic.

```

resource "clc_load_balancer" "api" {
  data_center = "${clc_group.<SOME_NAMED_RESOURCE>.location_id}"
  name = "api"
  description = "load balanced web traffic"
  status = "enabled"
}

output "lb_ip" {
  value = "clc_load_balancer.api.ip_address"
}
```

full API options documented: [https://www.ctl.io/api-docs/v2/#shared-load-balancers-create-shared-load-balancer]


#### `clc_load_balancer_pool`

Creates a loadbalancer pool and associates a pool of servers with a load balancer.

```

resource "clc_load_balancer_pool" "http" {
  data_center = "${clc_group.<SOME_NAMED_RESOURCE>.location_id}"
  load_balancer = "${clc_load_balancer.<SOME_NAMED_RESOURCE>.id"
  port = 80
  nodes
    {
      status = "enabled"
      ipAddress = "${clc_server.node01.private_ip_address}"
      privatePort = 3000
    }
  nodes
    {
      ...
    }
}

```

full API options documented: [https://www.ctl.io/api-docs/v2/#shared-load-balancers-create-load-balancer-pool]



## Building/Developing

  `godep get .`

  `go test .`

  `TF_ACC=1 go test -v -timeout=60m .` run acceptance tests. (requires ENV vars)
  
  `DEBUG=1 TF_LOG=trace TF_LOG_PATH=./plugin.log TF_ACC=1 go test  -v -timeout=60m -run LoadBalancer` narrow tests to regexp w/ api tracing

  `go build -o path/to/desired/terraform-provider-clc bin/terraform-provider-clc/main.go`
