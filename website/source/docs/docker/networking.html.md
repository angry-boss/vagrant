---
layout: "docs"
page_title: "Networking - Docker Provider"
sidebar_current: "providers-docker-networking"
description: |-
  The Vagrant Docker provider supports using the private network using the
  `docker network` commands.
---

# Networking

Vagrant uses the `docker network` command under the hood to create and manage
networks for containers. Vagrant will do its best to create and manage networks
for any containers configured inside the Vagrantfile. Each docker network is grouped
by the subnet used for a requested ip address.

For each newly unique network, Vagrant will run the `docker network create` subcommand
with the provided options from the network config inside your Vagrantfile. If multiple
networks share the same subnet, Vagrant will reuse that existing network for multiple
containers. Once these networks have been created, Vagrant will attach these
networks to the requested containers using the `docker network connect` for each
network.

Vagrant names the networks inside docker as `vagrant_network` or `vagrant_network_<subnet here>`
where `<subnet_here>` is the subnet for the network if defined by the user. An
example of these networks is shown later in this page. If no subnet is requested
for the network, Vagrant will connect the `vagrant_network` to the container.

When destroying containers through Vagrant, Vagrant will clean up the network if
there are no more containers using the network.

## Docker Network Options

Most of the options work similar to other Vagrant providers. Defining either an
ip or using `type: 'dhcp'` will give you a network on your container.

```ruby
docker.vm.network :private_network, type: "dhcp"
docker.vm.network :private_network, ip: "172.20.128.2"
```

If you want to set something specific with a new network you can use scoped options
which align with the command line flags for the [docker network create](https://docs.docker.com/engine/reference/commandline/network_create/)
command. If there are any specific options you want to enable from the `docker network create`
command, you can specify them like this:

```ruby
docker.vm.network :private_network, type: "dhcp", docker_network__internal: true
```

This will enable the `internal` option for the network when created with `docker network create`.

Where `option` corresponds to the given flag that will be provided to the `docker network create`
command. Similarly, if there is a value you wish to enable when connecting a container
to a given network, you can use the following value in your network config:

```ruby
docker_connect__option: "value"
```

When the docker provider creates a new network a netmask is required. If the netmask
is not provided, Vagrant will default to a `/24` for IPv4 and `/64` for IPv6. To provide
a different mask, set it using the `netmask` option:

```ruby
docker.vm.network :private_network, ip: "172.20.128.2", netmask: 16
```

For networks which set the type to "dhcp", it is also possible to specify a specific
subnet for the network connection. This allows containers to connect to networks other
than the default `vagrant_network` network. The docker provider supports specifying
the desired subnet in two ways. The first is by using the `ip` and `netmask` options:

```ruby
docker.vm.network :private_network, type: "dhcp", ip: "172.20.128.0", netmask: 24
```

The second is by using the `subnet` option:

```ruby
docker.vm.network :private_network, type: "dhcp", subnet: "172.20.128.0/24"
```

### Public Networks

The Vagrant docker provider also supports defining public networks. The easiest way
to define a public network is by setting the `type` option to "dhcp":

```ruby
docker.vm.network :public_network, type: "dhcp"
```

A bridge interface is required when setting up a public network. When no bridge
device name is provided, Vagrant will prompt for the appropriate device to use. This
can also be set using the `bridge` option:

```ruby
docker.vm.network :public_network, type: "dhcp", bridge: "eth0"
```

The `bridge` option also supports a list of interfaces which can be used for
setting up the network. Vagrant will inspect the defined interfaces and use
the first active interface when setting up the network:

```ruby
docker.vm.network :public_network, type: "dhcp", bridge: ["eth0", "wlan0"]
```

The available IP range for the bridge interface must be known when setting up
the docker network. Even though a DHCP service may be available on the public
network, docker will manage IP addresses provided to containers. This means
that the subnet provided when defining the available IP range for the network
should not be included within the subnet managed by the DHCP service. Vagrant
will prompt for the available IP range information, however, it can also be
provided in the Vagrantfile using the `docker_network__ip_range` option:

```ruby
docker.vm.network :public_network, type: "dhcp", bridge: "eth0", docker_network__ip_range: "192.168.1.252/30"
```

Finally, the gateway for the interface is required during setup. The docker
provider will default the gateway address to the first address available for
the subnet of the bridge device. Vagrant will prompt for confirmation to use
the default address. The address can also be manually set in the Vagrantfile
using the `docker_network__gateway` option:

```ruby
docker.vm.network :public_network, type: "dhcp", bridge: "eth0", docker_network__gateway: "192.168.1.2"
```

More examples are shared below which demonstrate creating a few common network
interfaces.

## Docker Network Example

The following Vagrantfile will generate these networks for a container:

1. A IPv4 IP address assigned by DHCP
2. A IPv4 IP address 172.20.128.2 on a network with subnet 172.20.0.0/16
3. A IPv6 IP address assigned by DHCP on subnet 2a02:6b8:b010:9020:1::/80

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "docker"  do |docker|
    docker.vm.network :private_network, type: "dhcp", docker_network__internal: true
    docker.vm.network :private_network,
        ip: "172.20.128.2", netmask: "16"
    docker.vm.network :private_network, type: "dhcp", subnet: "2a02:6b8:b010:9020:1::/80"
    docker.vm.provider "docker" do |d|
      d.build_dir = "docker_build_dir"
    end
  end
end
```

You can test that your container has the proper configured networks by looking
at the result of running `ip addr`, for example:

```
brian@localghost:vagrant-sandbox % docker ps                                                             ??[???][master]
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS              PORTS                                              NAMES
370f4e5d2217        196a06ef12f5                           "tail -f /dev/null"      5 seconds ago       Up 3 seconds        80/tcp, 443/tcp                                    vagrant-sandbox_docker-1_1551810440
brian@localghost:vagrant-sandbox % docker exec 370f4e5d2217 ip addr                                      ??[???][master]
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
24: eth0@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
27: eth1@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.19.0.2/16 brd 172.19.255.255 scope global eth1
       valid_lft forever preferred_lft forever
30: eth2@if31: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:14:80:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.128.2/16 brd 172.20.255.255 scope global eth2
       valid_lft forever preferred_lft forever
33: eth3@if34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:15:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.21.0.2/16 brd 172.21.255.255 scope global eth3
       valid_lft forever preferred_lft forever
    inet6 2a02:6b8:b010:9020:1::2/80 scope global nodad
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe15:2/64 scope link
       valid_lft forever preferred_lft forever
```

You can also connect your containers to a docker network that was created outside
of Vagrant:

```
$ docker network create my-custom-network --subnet=172.20.0.0/16
```

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "docker"  do |docker|
    docker.vm.network :private_network, type: "dhcp" name: "my-custom-network"
    docker.vm.provider "docker" do |d|
      d.build_dir = "docker_build_dir"
    end
  end
end
```

Vagrant will not delete or modify these outside networks when deleting the container, however.

## Useful Debugging Tips

The `docker network` command provides some helpful insights to what might be going
on with the networks Vagrant creates. For example, if you want to know what networks
you currently have running on your machine, you can run the `docker network ls` command:

```
brian@localghost:vagrant-sandbox % docker network ls                                                     ??[???][master]
NETWORK ID          NAME                                        DRIVER              SCOPE
a2bfc26bd876        bridge                                      bridge              local
2a2845e77550        host                                        host                local
f36682aeba68        none                                        null                local
00d4986c7dc2        vagrant_network                             bridge              local
d02420ff4c39        vagrant_network_2a02:6b8:b010:9020:1::/80   bridge              local
799ae9dbaf98        vagrant_network_172.20.0.0/16               bridge              local
```

You can also inspect any network for more information:

```
brian@localghost:vagrant-sandbox % docker network inspect vagrant_network                                ??[???][master]
[
    {
        "Name": "vagrant_network",
        "Id": "00d4986c7dc2ed7bf1961989ae1cfe98504c711f9de2f547e5dfffe2bb819fc2",
        "Created": "2019-03-05T10:27:21.558824922-08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "370f4e5d2217e698b16376583fbf051dd34018e5fd18958b604017def92fea63": {
                "Name": "vagrant-sandbox_docker-1_1551810440",
                "EndpointID": "166b7ca8960a9f20a150bb75a68d07e27e674781ed9f916e9aa58c8bc2539a61",
                "MacAddress": "02:42:ac:13:00:02",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

## Caveats

For now, Vagrant only looks at the subnet when figuring out if it should create
a new network for a guest container. If you bring up a container with a network,
and then change or add some new options (but leave the subnet the same), it will
not apply those changes or create a new network.

Because the `--link` flag for the `docker network connect` command is considered
legacy, Vagrant does not support that option when creating containers and connecting
networks.

## More Information

For more information on how docker manages its networks, please refer to their
documentation:

- https://docs.docker.com/network/
- https://docs.docker.com/engine/reference/commandline/network/
