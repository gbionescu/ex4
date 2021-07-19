# TPM measurement under Qemu with OVMF+swtpm.

I had two attempts at starting this up:

1. Running the VM in an alpine container - unsuccessful.
2. Running the VM on an opensuse machine - successful.

## 1. Running the VM in an alpine container - unsuccessful.

Steps to reproduce:

1. Start an alpine container: `docker run -it --rm alpine /bin/sh`.
2. Fetch `swtpm` according to instructions here: https://github.com/stefanberger/swtpm/wiki
    2.1. First, build `libtpms`: as shown here: https://github.com/stefanberger/libtpms/wiki
    2.2. Build `swtpm`.
    2.3. (Optional) openssl might have to be rebuilt in order to successfully execute step `2.2` since the `AES_set_encrypt_key` symbol is not built-in. 
See the package definition here: https://git.alpinelinux.org/aports/tree/main/openssl
Using the upstream package definition I managed to rebuild a package that contains `AES_set_encrypt_key` without any changes.

Instructions on how to build any alpine package can be found here: https://wiki.alpinelinux.org/wiki/Creating_an_Alpine_package

3. Launch a swtpm instance - in this case the path is fixed to `/vm`:

```bash
export path_to_vm=/vm

mkdir $path_to_vm
mkdir ${path_to_vm}/mytpm0
swtpm socket --tpmstate dir=${path_to_vm}/mytpm0 --ctrl type=unixio,path=${path_to_vm}/mytpm0/swtpm-sock --log level=20
```

4. Install qemu and ovmf.

5. Create a VM with TPM built-in the kernel - an Ubuntu Server VM should work.

6. Launch qemu with a command like the following:
```bash
qemu-system-x86_64 -M pc -enable-kvm -m 1024m -nographic \
    -smp 4,sockets=4,cores=1,threads=1 \
    -drive file=/usr/share/OVMF/OVMF_CODE.fd,if=pflash,format=raw,unit=0,readonly=on \
    -drive file=/var/lib/libvirt/qemu/nvram/generic_VARS.fd,if=pflash,format=raw,unit=1 \
    -drive file=./images/generic-2.qcow2,format=qcow2,if=none,id=drive-virtio-disk0 \
    -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x8,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 \
    -boot menu=on \
    -nographic
```

Note: Change the paths according to the paths on your system.

7. Log in to the VM and run:
```bash
sudo strings /sys/kernel/security/tpm0/binary_bios_measurements
```

Notes: 
- using the alpine container I was not getting any output from `/sys/kernel/security/tpm0/binary_bios_measurements`.
- `/dev/tpm0` exists and `dmesg | grep -i tpm` was showing successful initialization of the TPM device.
- after trying this several times, I started suspecting that the `swtpm` instance that I was using was not emulating the TPM right,
which could be caused by the fact that I was running everything in the Alpine container.
- Read the documentation more carefully and attempted the second strategy.


## 2. Running the VM on an opensuse machine - successful.

1. Install opensuse on a VM or a machine. See the instructions here: https://doc.opensuse.org/documentation/leap/startup/html/book-startup/art-opensuse-installquick.html

2. On the opensuse machine, install `swtpm` and `libtpm` as instructed here: https://software.opensuse.org//download.html?project=security&package=swtpm

3. Create a VM with TPM built-in the kernel - I used Ubuntu Server 21.04.

4. Create an XML file with the following content:
```XML
<domain type='qemu'>
  <name>linux2020</name>
  <uuid>2eba5dc4-a455-4d7a-bb0d-9ce4afc5ffd4</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://libosinfo.org/linux/2020"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit='KiB'>4012032</memory>
  <currentMemory unit='KiB'>4012032</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-6.0'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/qemu/ovmf-x86_64-code.bin</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/linux2020_VARS.fd</nvram>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <vmport state='off'/>
  </features>
  <cpu mode='custom' match='exact' check='none'>
    <model fallback='forbid'>qemu64</model>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/images/generic.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
    </disk>
    <controller type='usb' index='0' model='ich9-ehci1'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1d' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
      <master startport='0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1d' function='0x0' multifunction='on'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci2'>
      <master startport='2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1d' function='0x1'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci3'>
      <master startport='4'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1d' function='0x2'/>
    </controller>
    <controller type='sata' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1f' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pcie-root'/>
    <controller type='pci' index='1' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='1' port='0x10'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0' multifunction='on'/>
    </controller>
    <controller type='pci' index='2' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='2' port='0x11'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x1'/>
    </controller>
    <controller type='pci' index='3' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='3' port='0x12'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x2'/>
    </controller>
    <controller type='pci' index='4' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='4' port='0x13'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x3'/>
    </controller>
    <controller type='pci' index='5' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='5' port='0x14'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x4'/>
    </controller>
    <controller type='virtio-serial' index='0'>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </controller>
    <interface type='network'>
      <mac address='52:54:00:c0:af:44'/>
      <source network='default2'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
    </interface>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
    </channel>
    <channel type='spicevmc'>
      <target type='virtio' name='com.redhat.spice.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='2'/>
    </channel>
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <tpm model='tpm-tis'>
      <backend type='emulator' version='2.0'/>
    </tpm>
    <graphics type='spice' autoport='yes'>
      <listen type='address'/>
      <image compression='off'/>
    </graphics>
    <sound model='ich9'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1b' function='0x0'/>
    </sound>
    <audio id='1' type='spice'/>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
    </video>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='2'/>
    </redirdev>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='3'/>
    </redirdev>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
    </memballoon>
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </rng>
  </devices>
</domain>
```

Notes:

- Make sure that you change the paths to the local paths on your system.
- Make sure that the TPM module exists:

```xml
    <tpm model='tpm-tis'>
      <backend type='emulator' version='2.0'/>
    </tpm>
```

5. Run `virsh create step4.xml`.

6. Switch to the VM console using either `virsh console` or `virt-manager`.

7. Log in to the VM and run: 

```bash
sudo strings /sys/kernel/security/tpm0/binary_bios_measurements
```

If `strings` is not installed, `cat` works too.

8. Video of a VM booting with TPM measurement enabled here: https://www.youtube.com/watch?v=qxpdt01T0x0. 

Timestamp of the `binary_bios_measurements` output here: https://youtu.be/qxpdt01T0x0?t=123
