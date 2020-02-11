# Common Issues

## Conflicting Programs

Certain programs such as `dnsmasq`, `dockerd` and `lxd` interfere with Gravity operation and need be uninstalled.

## IPv4 Forwarding

IPv4 forwarding on servers is required for internal Kubernetes load balancing and must be turned on.

```bsh
sysctl -w net.ipv4.ip_forward=1
echo net.ipv4.ip_forward=1 >> /etc/sysconf.d/10-ipv4-forwarding-on.conf
```

## Network Bridge Driver

Network bridge driver should be loaded, which could be ensured with `modprobe br_netfilter` or `modprobe bridge` command depending on the distribution.
Distributions based on kernels prior to 3.18 might have this compiled into the kernel.
In CentOS 7.2, the module is called `bridge` instead of `br_netfilter`.

Additionally, it should be configured to pass IPv4 traffic to `iptables` chains:

```bsh
sysctl -w net.bridge.bridge-nf-call-iptables=1
echo net.bridge.bridge-nf-call-iptables=1 >> /etc/sysconf.d/10-bridge-nf-call-iptables.conf
```

For more information see the relevant [System Requirements](/requirements/#br_netfilter-module)
section.

## Overlay Driver

If the application uses overlay or overlay2 Docker storage driver, the
`overlay` kernel module should be loaded. See the relevant
[System Requirements](/requirements/#overlay-module) section for details.

## D_TYPE Support in Filesystem

If the application uses overlay or overlay2 Docker storage driver, the backing
filesystem needs to support `d_type` for overlay to work properly.
For example, XFS does not support `d_type` if it has been formatted with the
`-n ftype=0` option.

!!! warning
    Starting with v1.13.0, Docker deprecates support for file systems without `d_type`
    support for overlay/overlay2 storage drivers.

Running on file systems without `d_type` support causes Docker to skip the attempt
to use the overlay or overlay2 driver. Existing installs will continue to run,
but produce an error. This is to allow users to migrate their data.
In a future version, this will be a fatal error, which will prevent Docker from
starting.

## System Time Sync

Kubernetes master nodes are sensitive to system time differences between nodes they're running on. It is recommended to have the system time synchronized against an NTP time source via tools like `ntpd`.

##  Firewalld

If `firewalld` is enabled in the system, Kubernetes services may not be able to communicate under default configuration.

```bsh
firewall-cmd --zone=trusted --add-source=10.244.0.0/16 --permanent # pod subnet
firewall-cmd --zone=trusted --add-source=10.100.0.0/16 --permanent # service subnet
firewall-cmd --zone=trusted --add-interface=eth0 --permanent       # enable eth0 in trusted zone so nodes can communicate
firewall-cmd --zone=trusted --add-masquerade --permanent           # masquerading so packets can be routed back
firewall-cmd --reload
systemctl restart firewalld
```

Note that pod and service subnet addresses may be [customized at install time](/installation/).

## Azure Hyper-V Clock Sync

Azure is adjusting VM clock from host, which may result in time drift between nodes.
Currently a recommended workaround is installing NTP client such as `chrony`, and disabling Hyper-V time sync:

```
function get_vmbus_attr {
  local dev_path=$1
  local attr=$2

  cat $dev_path/$attr | head -n1
}

function get_timesync_bus_name {
  local timesync_bus_id='{9527e630-d0ae-497b-adce-e80ab0175caf}'
  local vmbus_sys_path='/sys/bus/vmbus/devices'

  for device in $vmbus_sys_path/*; do
    local id=$(get_vmbus_attr $device "id")
    local class_id=$(get_vmbus_attr $device "class_id")
    if [ "$class_id" == "$timesync_bus_id" ]; then
      echo vmbus_$id; exit 0
    fi
  done
}

timesync_bus_name=$(get_timesync_bus_name)
if [ ! -z "$timesync_bus_name" ]; then
  # disable Hyper-V host time sync
  echo $timesync_bus_name > /sys/bus/vmbus/drivers/hv_util/unbind
fi
```

## VMWare ESXi VXLAN Port Conflict

On virtual machines powered by VMWare ESXi (part of VMWare vSphere suite) of
some versions port `8472` may be used for VXLAN that encapsulates all VM-to-VM
traffic which conflicts with the default VXLAN port used by Gravity and affects
the cluster's overlay network.

See vSphere [port and protocol requirements](https://docs.vmware.com/en/VMware-NSX-Data-Center-for-vSphere/6.4/com.vmware.nsx.install.doc/GUID-E7C4E61C-1F36-457C-ACC5-EAF955C46E8B.html)
for more information.

To support such systems Gravity provides the ability to override the default
VXLAN port at install time via a command-line flag:

```shell
sudo ./gravity install --vxlan-port=9473
```

## Kubernetes Pods Stuck in Terminating State

Some linux distributions have included the kernel setting `fs.may_detach_mounts` with a default value of 0. This can cause conflicts with the docker daemon, where docker is then unable to unmount mount points within the container. Kubernetes will show pods as stuck in the terminating state if docker is unable to clean up one of the underlying containers.

If the installed kernel exposes the option `fs.may_detach_mounts` we recommend always setting this value to 1, or you may experience issues terminating pods in your cluster.

```shell
sysctl -w fs.may_detach_mounts=1
echo "fs.may_detach_mounts = 1" >> /etc/sysctl.d/10-may_detach_mounts.conf
```

## Running Privileged Containers

By default privileged containers are not allowed in Gravity clusters. In some
cases privileged containers may be required though, specifically for applications
that wish to utilize custom network plugins or dynamic volume provisioners.

To allow privileged containers, set the following field in your cluster image
manifest:

```yaml
systemOptions:
  allowPrivileged: true
```

See [Securing a Cluster](/cluster/#securing-a-cluster) for more details.

## Customizing Helm Values

When using Helm charts, it is possible to customize Helm values at
build/install/upgrade time by providing `--values` and `--set` flags to
the respective `tele build`, `gravity install` and `gravity upgrade`
commands.

See [Helm Integration](/pack/#helm-integration) for more details.

## Cluster Status

Gravity provides the `gravity status` command to view [Cluster Status](/cluster/#cluster-status).
This tool can help identify issues with the Cluster.