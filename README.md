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
- `talos_machine_config`: Machine config for Talos. Should be string.

Example, the variables in the machine config should be handled with care, since there are secrets.
This example also contains a fix for vxlan in vmware for vmxnet3. Check `machine.pods`:

```yaml
talos_machine_config: |
  version: v1alpha1
  debug: false
  persist: true
  machine:
    type: controlplane
    token: {{ talos_machine_token }}
    pods:
      - apiVersion: v1
        kind: Pod
        metadata:
          namespace: kube-system
          name: vxlanfix
          labels:
            pod-security.kubernetes.io/enforce: privileged
        spec:
          containers:
            - name: vxlanfix
              image: sdnvortex/ethtool:latest
              env:
                - name: PATH
                  value: "/bin:/usr/bin:/usr/sbin"
              command:
                - sh
                - "-exc"
                - |
                  ethtool -K eth0 tx-udp_tnl-segmentation off
                  ethtool -K eth0 tx-udp_tnl-csum-segmentation off
                  set +x
                  while true; do sleep 1; done
              securityContext:
                runAsUser: 0
                privileged: true
                capabilities:
                  add:
                    - NET_ADMIN
                  drop:
                    - ALL
              volumeMounts:
                - name: lib-modules
                  mountPath: /lib/modules
                - name: host-proc-sys-net
                  mountPath: /host/proc/sys/net
          hostNetwork: true
          volumes:
            - name: lib-modules
              hostPath:
                path: /lib/modules
                type: Directory
            - name: host-proc-sys-net
              hostPath:
                path: /proc/sys/net
                type: Directory
    ca:
      crt: {{ talos_machine_ca_crt }}
      key: {{ talos_machine_ca_key }}
    certSANs:
      - glb-k8s-common.sbab.se
      - glb.common
    kubelet:
      image: ghcr.io/siderolabs/kubelet:{{ k8s_version }}
      defaultRuntimeSeccompProfileEnabled: true
      disableManifestsDirectory: true
      clusterDNS:
        - 172.17.0.10
    network:
      hostname: {{ inventory_hostname }}
      interfaces:
        - interface: eth0
          addresses:
            - {{ vmware_vm_ip }}/{{ vmware_vm_ip_netmask }}
          routes:
            - network: 0.0.0.0/0
              gateway: {{ vmware_network_gateway }}
          mtu: 1500
          dhcp: false
          vip:
            ip: {{ k8s_vip }}
      nameservers: {{ dns_servers }}
    install:
      disk: /dev/sda
      image: {{ talos_install_image }}
      bootloader: true
      wipe: false
    time:
      servers: {{ ntp_servers}}
    registries: {{ talos_machine_registries }}
    features:
      rbac: true
      stableHostname: true
      apidCheckExtKeyUsage: true
  cluster:
    id: {{ talos_cluster_id }}
    secret: {{ talos_cluster_secret }}
    controlPlane:
      endpoint: https://{{ k8s_vip }}:6443
    clusterName: glb.common
    proxy:
      disabled: true
    network:
      cni:
        name: none
      dnsDomain: glb.common
      podSubnets:
        - 172.16.0.0/16
      serviceSubnets:
        - 172.17.0.0/16
    token: {{ talos_cluster_token }}
    secretboxEncryptionSecret: {{ talos_cluster_secretboxEncryptionSecret }}
    ca:
      crt: {{ talos_cluster_ca_crt }}
      key: {{ talos_cluster_ca_key }}
    aggregatorCA:
      crt: {{ talos_cluster_aggca_crt}}
      key: {{ talos_cluster_aggca_key }}
    serviceAccount:
      key: {{ talos_cluster_serviceAccount_key }}
    apiServer:
      image: registry.k8s.io/kube-apiserver:{{ k8s_version }}
      certSANs:
        - {{ k8s_vip }}
        - glb-k8s-common.sbab.se
      disablePodSecurityPolicy: true
      admissionControl:
        - name: PodSecurity
          configuration:
            apiVersion: pod-security.admission.config.k8s.io/v1alpha1
            defaults:
              audit: restricted
              audit-version: latest
              enforce: baseline
              enforce-version: latest
              warn: restricted
              warn-version: latest
            exemptions:
              namespaces:
                - kube-system
              runtimeClasses: []
              usernames: []
            kind: PodSecurityConfiguration
      auditPolicy:
        apiVersion: audit.k8s.io/v1
        kind: Policy
        rules:
          - level: Metadata
    controllerManager:
      image: registry.k8s.io/kube-controller-manager:{{ k8s_version }}
    scheduler:
      image: registry.k8s.io/kube-scheduler:{{ k8s_version }}
    etcd:
      ca:
        crt: {{ talos_cluster_etcd_ca_crt}}
        key: {{ talos_cluster_etcd_ca_key }}
    extraManifests: {{ talos_cluster_extraManifests }}

```

## VMWare settings
- `vmware_vm_ip`: Static IP for each host. This is used to communicate with your cluster using `talosctl`.

## VMWare DRS
- `vmware_drs_management`: boolean. Set true and make sure you have affinity groups defined in your vmware cluster with name `<location>-vms`
- `vmware_location`: Host variable set if you use vmware_drs_management.
