# Increasing the VirtualMachineInstance Density on Nodes

KubeVirt does not yet support classical Memory Overcommit Management or Memory
Ballooning. With other words VirtualMachineInstances can't give back memory
they have allocated. However, a few other things can be tweaked to reduce the
memory footprint and overcommit the per-VMI memory overhead.

## Remove the Graphical Devices

First the safest option to reduce the memory footrint, is disabling VNC access
to the VirtualMachineInstances, by setting
`spec.domain.devices.autottachGraphicsDevice` to `false`. See the video and
graphics device
[documentation](/workloads/virtual-machines/virtualized-hardware-configuration#video-and-graphics-device)
for further details and examples.

This will save a constant ammount of `16MB` per VirtualMachineInstance.

## Overcommit the Guest Overhead

Befor you continue, make sure you make yourself comfortable with the [Out of
Resource
Managment](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)
of Kubernetes.

Every VirtualMachineInstance requests slightly more memory from Kubernetes than
what was requested by the user for the Operating System. The additional memory
is used for the per-VMI overhead consisting of our infrastructure which is
wrapping the actual VirtualMachineInstance process.

In order to increase the VMI density on the node, it is possible to not request
the additional overhead by setting
`spec.domain.resources.overcommitGuestOverhead` to `true`:


```yaml
apiVersion: kubevirt.io/v1alpha2
kind: VirtualMachineInstance
metadata:
  name: testvmi-nocloud
spec:
  terminationGracePeriodSeconds: 30
  domain:
    resources:
    overcommitGuestOverhead: true
      requests:
        memory: 1024M
[...]
```

This will work fine for as long as most of the VirtualMachineInstances will not
request the whole memory. That is especially the case if you have short-lived
VMIs. But if you have long-lived VirtualMachineInstances or do extremely memory
intensive tasks inside the VirtualMachineInstance, your VMIs will use all
memory they are granted sooner or later.

If the node gets under memory pressure, depending on the `kubelet`
configuration the virtual machines may get killed by the OOM handler or by the
`kubelet` itself. It is possible to tweak that behavour based on the
requirements of your VirtualMachineInstances by:

 * Configuring [Soft Eviction Thresholds](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#soft-eviction-thresholds)
 * Configuring [Hard Eviction Thresholds](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#hard-eviction-thresholds)
 * Requesting the right QoS class for VirtualMachineInstances
 * Setting `--system-reserved` and `--kubelet-reserved`
 * Enabling KSM
 * Enabling swap

### Configuring Soft Eviction Thresholds

If configured, VirtualMachineInstances get evicted once the available memory
falls below the threshold specified via `--eviction-soft` and the
VirtualmachineInstance is given the hance to perform a shutdown of the VMI
within a timespan specified via `--eviction-max-pod-grace-period`. The flag
`--eviction-soft-grace-period` specifies for how long a soft eviction condition
must be held before soft evictions are triggered.

If set properly according to the demands of the VMIs, overcommiting should only
lead to soft evictions from time to time for some VMIs. They may even get
re-scheduled to the same node. This is fine because then they will start with
less memory consumption again.

### Configuring Hard Eviction Thresholds

Limits set via `--eviction-hard` will lead to immediate eviction of
VirtualMachineInstances or Pods. This stops VMIs without a grace period and is
comparable with powerloss on a real computer.

If this is hit, VMIs may from time to time simply be killed. The may be
re-scheduled to the same node immediately again, since they start with less
memory consumption again. This can be a simple option, if the memory threshold
is only very seldom hit and the work performed by the VMIs is reproducible or
it can be resumed from some checkpoints.

### Requesting the right QoS Class for VirtualMachineInstances

Different QoS classes get [assigned to Pods and
VirtualMachineInstances](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy)
based on the `requests.memory` and `limits.memory`. KubeVirt right now supports
the QoS classes `Burstable` and `Guaranteed`. `Burstable` VMIs are evicted
before `Guaranteed` VMIs.

This allows creating two classes of VMIs: 

* One type can have equal `requests.memory` and `limits.memory` set and therefore
  gets the `Guaranteed` class assigned. This one will not get evicted and should
  never run into memory issues, but is more demanding.
* One type can have no `limits.memory` or a `limits.memory` which is greater than
  `requests.memory` and therefore gets the `Burstable` class assigned. These VMIs
  will be evicted first.

### Setting `--system-reserved` and `--kubelet-reserved`

It may be important to reserver some memory for other daemons (not DaemonSets)
which are running on the same node (e.g. ssh, dhcp servers, ...). The
reservation can be done with the `--system-reserved` switch. Furthter for the
Kubelet and Docker a special flag called `--kubelet-reserved` exists.

### Enabling KSM

The KSM (Kernel same-page merging) daemon can be started on the node. Depending
on its tuning parameters it can more or less aggressively try to merge
identical pages between applications and VirtualMachineInstances. The more
aggressive it is configured the more CPU it will use itself, so the memory
overcommit advantages comes with a slight CPU performance hit.

Possible tune options are on how often the pages should be scanned for merges
and how much pages it should scan.

### Enabling swap

This will not work for Kubernetes `1.12` or later, since spilling over to swap,
if a memory limit is hit, is explicitly disabled there.

> Note: this will definitely make sure that your VirtualMachines can't crash or
> get evicted from the node but it comes with the cost of pretty unpredictable
> performance once the node runs out of memory and the kubelet may not detect
> that it should evict Pods to increase the performance again.
