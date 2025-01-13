# Running Talos Linux on HomeAssistant Yellow

## Assumptions
- Talos is installed on an NVMe (etcd is not very eMMC/SD card friendly)
- IoT/Access network is not on the default network interface but needs to be configured via [multus]()
- HomeAssistant ZHA stack (though the same setup works with Zigbee2Mqtt as well)

## Talconfig
```yaml
---
clusterName: helios
talosVersion: v1.9.1
kubernetesVersion: v1.32.0
endpoint: https://100.64.0.7:6443
domain: cluster.local
allowSchedulingOnMasters: true
additionalApiServerCertSans:
  - helios.kube-cp.m4tbit.de
clusterPodNets:
  - 10.244.0.0/16
clusterSvcNets:
  - 10.96.0.0/12
nodes:
  - ipAddress: 100.64.0.7
    hostname: cm4-8c-8g-4000g-yellow0
    controlPlane: true
    machineSpec: { mode: metal, arch: arm64 }
    schematic:
      overlay:
        image: siderolabs/sbc-raspberrypi
        name: rpi_generic
        options:
            configTxtAppend: |
              # Disable SD Card
              dtparam=sd=off

              # CM4 USB
              dtoverlay=dwc2,dr_mode=host

              # Silabls Radio
              dtoverlay=miniuart-bt
              dtoverlay=uart4,ctsrts

              # Debug console
              dtoverlay=uart5

              # User-defined LED
              dtoverlay=gpio-led,gpio=44,label=usr,trigger=heartbeat
              dtoverlay=gpio-led,gpio=42,label=act,trigger=activity

              # RTC
              dtparam=i2c_arm=on
              dtparam=i2c_vc=on
              dtoverlay=i2c-rtc,pcf85063a,i2c_csi_dsi

              # 2GHz CPU (cooling of HA yellow is sufficient here)
              over_voltage=6
              arm_freq=2000
      customization:
        extraKernelArgs:
        - 'console=tty0'
        - 'console=ttyAMA5,115200'  # USB Console
        - 'net.ifnames=0'
        - 'sysctl.kernel.kexec_load_disabled=1'
        - 'talos.dashboard.disabled=1'
    installDisk: /dev/nvme0n1
    networkInterfaces:
      - interface: eth0
        dhcp: false
        addresses: [ 100.64.0.7/24 ]
        routes:
        - { network: '0.0.0.0/0', gateway: '100.64.0.1' }
        vlans:
          # IoT network is on vlan100
          - vlanId: 100
            dhcp: false
    nameservers:
      - 100.64.0.1
    patches:
    - |-
      - op: add
        path: /machine/sysctls
        value:
          # disable IPv6 SLAAC on interface consumed by multus/CNI as bride parent
          net/ipv6/conf/eth0.100/autoconf: 0
          net/ipv6/conf/eth0.100/accept_ra: 0
```

## Homeassistant Setup with Multus

#### NetworkAttachmentDefinition
```yaml
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: eth0-vlan-100
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "macvlan",
          "capabilities": { "ips": true },
          "master": "eth0.100",
          "mode": "bridge",
          "ipam": {
            "type": "static"
          }
        }, {
          "capabilities": { "mac": true },
          "type": "tuning"
        }
      ]
    }
```

#### HomeAssistant StatefulSet
```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: homeassistant
spec:
  selector:
    matchLabels:
      name: homeassistant
  serviceName: "homeassistant"
  replicas: 1
  template:
    metadata:
      labels:
        name: homeassistant
      annotations:
        # Make sure to change the mac address/subnet. This mounts a 2nd network interface within the pod
        k8s.v1.cni.cncf.io/networks: |-
          [
            { "name": "eth0-vlan-100",
              "ips": [ "192.168.1.5/24" ],
              "mac": "XX:XX:XX:XX:XX:XX"
            }
          ]
    spec:
      containers:
        - name: homeassistant
          image: homeassistant/home-assistant:2025.1.0
          ports:
            - containerPort: 8123
              name: http
          volumeMounts:
            - name: config
              mountPath: /config
            # /dev/ttyAMA4 is the Silabs ZigBee radio
            - name: ttyama4
              mountPath: /dev/ttyAMA4
          securityContext:
            privileged: true
      volumes:
        - name: ttyama4
          hostPath:
            path: /dev/ttyAMA4
            type: CharDevice
  volumeClaimTemplates:
    - metadata:
        name: config
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 64Gi
```
