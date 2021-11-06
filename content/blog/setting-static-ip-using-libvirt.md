+++
title = "Setting Static IP Using Libvirt"
date = 2019-12-27T18:32:51+02:00
draft = false
+++
As part of the work on my pet project, [catapult](https://github.com/PUMATeam/catapult) (on which I will expand in another post), I was looking for a way to set a static IP address for a VM managed by libvirt. This VM serves me as the testing host which I use for development. Having the same IP always provides a couple of benefits:

	1. Using the same API parameters whenever I test
	2. I can easily script the deployment of the test host for other to use
	3. Will be super useful when I finally start writing the system tests

Googling around I came across the solution of simply adding the following element to the network definition:

```xml
<host mac='52:54:00:8a:7b:9d' name='fc_host' ip='192.168.122.45'/>
```

However, this requires recreating the network. Since I wanted the starting of my test VM to be scripted, editing the network configuration using `virsh` seemed quite tedious. So instead, I resorted to using the Python SDK, and came up with the following code (my Python skill is quite lacking, so it is not very good, but it does the job):

```python
#!/bin/python
import libvirt
import sys

from xml.etree import ElementTree as ET

def should_update(section, mac, ip):
    for node in section.findall(".//host"):
        if node.attrib["mac"] == mac and node.attrib["ip"] == ip:
            return False

    return True

def find_dhcp(root):
    dhcp_section = root.find("./ip/dhcp")
    return dhcp_section

def update_network(dom_name, root, mac, ip, dhcp_section):
    host = ET.Element("host", mac=mac, name=dom_name, ip=ip)
    dhcp_section.append(host)

    return ET.tostring(root).decode()
conn = libvirt.open("qemu:///system")

dom_name = sys.argv[1]
try:
    dom = conn.lookupByName(dom_name)
except:
    sys.exit(-1)

ip_address = sys.argv[3]
mac_address = sys.argv[4]

network = conn.networkLookupByName(sys.argv[2])
network_xml = network.XMLDesc(0)
xml_root = ET.fromstring(network_xml)
dhcp_section = find_dhcp(xml_root)

if not should_update(dhcp_section, mac_address, ip_address):
    print("IP address is already configured")
    sys.exit(0)

updated_network = update_network(
        dom_name,
        xml_root,
        mac_address,
        ip_address,
        dhcp_section)

if network.isActive():
    network.destroy()

network = conn.networkDefineXML(updated_network)
network.setAutostart(True)
network.create()
```

So this code doesn't do much other than check the DHCP configuration of the network,  then injects the XML element to the network definition

```xml
<host mac='52:54:00:8a:7b:9d' name='fc_host' ip='192.168.122.45'/>
```

Then it destroys and starts the network if needed.

This is not to pleasant, but it mostly works fine. However, I revisited the issue again, for reasons I cannot recall. And found an <a href="https://www.redhat.com/archives/libvirt-users/2014-March/msg00110.html" target="_blank" rel="noopener">old reply</a> in the libvirt mailing list, where the net-update command was pointed out:

```bash
virsh net-update default add-last ip-dhcp-host \
"<host mac='52:54:00:8a:7b:9d' name='fc_host' ip='192.168.122.45'/>" \
--live --config
```
Which I find very convenient, since I can simply do what I wanted without restarting the network and I can throw away the Python script, and my test VM definition script will now look like this:

```bash
#!/bin/bash
echo "Defining VM..."

VM_NAME=${1:-"fchost"}
LIBVIRT_NETWORK=${2:-"default"}
VM_IP=${3:-"192.168.122.45"}

if virsh list --all | grep -q "${VM_NAME}"; then
    echo "${VM_NAME} is already installed... "
else
    dom=$(virt-install --import --name "${VM_NAME}" \
        --memory 1024 --vcpus 1 --cpu host \
        --disk os.img,bus=virtio \
        --os-type=linux \
        --graphics spice \
        --noautoconsole \
        --network=default,model=virtio \
        --connect qemu:///system \
        --print-xml)
    echo $dom | virsh define /dev/stdin
fi

fc_host_status=$(virsh list | grep fc_host | tr -s \"[:blank:]\" | cut -d ' ' -f4)
if [  "${fc_hoststatus}" == 'running' ]; then
    echo "${VM_NAME} is already running"
    exit 0
fi

mac_address=$(virsh dumpxml "${VM_NAME}" | grep "mac address" | awk -F\' '{ print $2}')
echo "Setting IP address to ${VM_IP} for MAC address ${MAC_ADDRESS}"

xml_entry="<host mac=\"${mac_address}\" name=\"${VM_NAME}\" ip=\"${VM_IP}\"/>"

# TODO: check both IP and MAC, since this wouldn't work if we use a new VM after previously defining one
if virsh net-dumpxml "${LIBVIRT_NETWORK}" | grep -q "${VM_NAME}"; then
    echo "IP address is already configured"
else
    virsh net-update ${LIBVIRT_NETWORK} add-last ip-dhcp-host "${xml_entry}" --live --config
fi

echo "starting ${VM_NAME}..."
virsh start "${VM_NAME}"
```

If I still wanted to use python to do this, it would be much simpler:

```python
import libvirt
import sys

conn = libvirt.open('qemu:///system')
dom_name = sys.argv[1]

try:
    dom = conn.lookupByName(dom_name)
except:
    sys.exit(-1)

ip_address = sys.argv[3]
mac_address = sys.argv[4]

network = conn.networkLookupByName(sys.argv[2])
flags = (libvirt.VIR_NETWORK_UPDATE_AFFECT_LIVE |
            libvirt.VIR_NETWORK_UPDATE_AFFECT_CONFIG)
entry = "<host mac='{}' name='{}' ip='{}'/>".format(
        mac_address, dom_name, ip_address)
network.update(
        libvirt.VIR_NETWORK_UPDATE_COMMAND_ADD_LAST,
        libvirt.VIR_NETWORK_SECTION_IP_DHCP_HOST,
        0,
        entry,
        flags)
```
_
