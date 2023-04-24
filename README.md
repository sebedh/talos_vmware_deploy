# talos_vmware_deploy

Deploy Talos k8s in VMWare VSphere 7

# Requirements
- VMWare Vsphere 7
- vpshere-automation-sdk-python (https://github.com/vmware/vsphere-automation-sdk-python)
- pyvim
- pyvmomi
- talosctl

Installing vsphere-automation-sdk-python

```
pip install -U lib/*/*.whl
pip install -U <SDK_DIRECTORY_PATH>
```

# Required Variables
So far only static IP is supported

## Talos
`talos_machine_config`: Machine config for Talos. Should be string.

## VMWare settings
`vmware_vm_ip`: Static IP for each host. This is used to communicate with your cluster using `talosctl`.

## VMWare DRS
`vmware_drs_management`: boolean. Set true and make sure you have affinity groups defined in your vmware cluster with name <location>-vms
`vmware_location`: Host variable set if you use vmware_drs_management.
