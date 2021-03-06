# Heat
(not an acronym)


# Heat
enables you to deploy
## complete
virtual environments


# HOT
Heat Orchestration Template

100% YAML


What can we
# do
with Heat?


### `OS::Nova::Server`
Configures Nova guests


```yaml
resources:
  mybox:
    type: "OS::Nova::Server"
    properties:
      name: mybox
      image: "Ubuntu 16.04 Xenial Xerus"
      flavor: "1C-1GB"
      key_name: mykey
```


Now we could just
# create
this stack


```bash
openstack stack create -t stack.yml mystack
```


But as it is,

it's not very
# flexible


Let's add some
## parameters


```yaml
parameters:
  flavor:
    type: string
    description: Flavor to use for servers
    default: "1C-1GB"
  image:
    type: string
    description: Image name or ID
    default: "Ubuntu 16.04 Xenial Xerus"
  key_name:
    type: string
    description: Keypair to inject into newly created servers
```


And some
## intrinsic functions


```yaml
resources:
  mybox:
    type: "OS::Nova::Server"
    properties:
      name: mybox
      image: { get_param: image }
      flavor: { get_param: flavor }
      key_name: { get_param: key_name }
```


And now we can
# set
these parameters


```bash
openstack stack create -t stack.yml \
  --parameter key_name=mykey \
  --parameter image=cirros-0.3.3-x86_64 \
  mystack
```


How about we add some
## network connectivity
Wouldn't that be nice?


### `OS::Neutron::Net`
### `OS::Neutron::Subnet`
Defines Neutron networks


```yaml
  mynet:
    type: "OS::Neutron::Net"
    properties:
      name: management-net

  mysub_net:
    type: "OS::Neutron::Subnet"
    properties:
      name: management-sub-net
      network: { get_resource: mynet }
      cidr: 192.168.122.0/24
      gateway_ip: 192.168.101.1
      enable_dhcp: true
      allocation_pools: 
        - start: "192.168.101.2"
          end: "192.168.101.50"
```


## `get_resource`
Cross-reference between resources

Automatic dependency


### `OS::Neutron::Router`
### `OS::Neutron::RouterGateway`
### `OS::Neutron::RouterInterface`
Configures Neutron routers


```yaml
parameters:
  public_net:
    type: string
    description: Public network ID or name

resources:
  router:
    type: OS::Neutron::Router
  router_gateway:
    type: OS::Neutron::RouterGateway
    properties:
      router: { get_resource: router }
      network: { get_param: public_net }
  router_interface:
    type: OS::Neutron::RouterInterface
    properties:
      router: { get_resource: router }
      subnet: { get_resource: mysub_net }
```


### `OS::Neutron::Port`
Configures Neutron ports


```yaml
  mybox_management_port:
    type: "OS::Neutron::Port"
    properties:
      network: { get_resource: mynet }
```

```yaml
  mybox:
    type: "OS::Nova::Server"
    properties:
      name: deploy
      image: { get_param: image }
      flavor: { get_param: flavor }
      key_name: { get_param: key_name }
      networks:
        - port: { get_resource: mybox_management_port }
```


### `OS::Neutron::SecurityGroup`
Configures Neutron security groups


```yaml
  mysecurity_group:
    type: OS::Neutron::SecurityGroup
    properties:
      description: Neutron security group rules
      name: mysecurity_group
      rules:
      - remote_ip_prefix: 0.0.0.0/0
        protocol: tcp
        port_range_min: 22
        port_range_max: 22
      - remote_ip_prefix: 0.0.0.0/0
        protocol: icmp
        direction: ingress
```

```yaml
  mybox_management_port:
    type: "OS::Neutron::Port"
    properties:
      network: { get_resource: mynet }
      security_groups:
        - { get_resource: mysecurity_group }
```


### `OS::Neutron::FloatingIP`
Allocates floating IP addresses


```yaml
  myfloating_ip:
    type: "OS::Neutron::FloatingIP"
    properties:
      floating_network: { get_param: public_net }
      port: { get_resource: mybox_management_port }
```


### `outputs`
Return stack values or attributes


```yaml
outputs:
  public_ip:
    description: Floating IP address in public network
    value: { get_attr: [ myfloating_ip, floating_ip_address ] }
```


```bash
openstack stack output show \
  mystack public_ip
```
