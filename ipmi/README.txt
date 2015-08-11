-----------------------------------------------------------------------

I. Check IPMI with qemu-kvm


* Refence:

https://lists.gnu.org/archive/html/qemu-devel/2015-04/msg00619.html
https://github.com/cminyard/qemu/tree/stable-2.3-ipmi .
https://gist.github.com/bot11/

/usr/share/man/man1/ipmi_sim.1.gz
/usr/share/man/man5/ipmi_lan.5.gz
/usr/share/man/man5/ipmi_sim_cmd.5.gz
/usr/share/man/man8/ipmilan.8.gz

* Testing

1) Run similator

[root@fedora22 ~]# ipmi_sim 
IPMI Simulator version 1.0.13
# This is an example simulation setup for ipmi_sim.  It creates a single
# management controller as a BMC.  That will have the standard watchdog
# sensor and we add a temperature sensor.

# The BMC is the MC at address 20
mc_setbmc 0x20

# Now add the BMC
mc_add 0x20 0 no-device-sdrs 0x23 9 8 0x9f 0x1291 0xf02 persist_sdr
sel_enable 0x20 1000 0x0a

# Add a temperature sensor and its SDR.  Note that sensor 0 is already
# taken as the watchdog sensor.
sensor_add 0x20 0 1 0x01 0x01
#main_sdr_add 0x20 \
#            00 00 51 01 31 \
#            20 00 01 03 01 67 88 01 01 c0 0f c0 7f 38 38 00 \
#            01 00 00 01 00 00 00 00 00 03 60 b0 00 b0 00 a0 \
#            90 70 00 00 00 00 00 00 00 00 c6 'D 'J 't 'e 'm \
#            'p
# Start with the value set to 0x60
sensor_set_value 0x20 0 1 0x60 0
# Set just the upper thresholds with the values 0x70, 0x90, and 0xa0
sensor_set_threshold 0x20 0 1 settable 111000 0xa0 0x90 0x70 00 00 00
# Enable all upper threshold events events
sensor_set_event_support 0x20 0 1 enable scanning per-state \
        000111111000000 000111111000000 \
        000111111000000 000111111000000

# Turn on the BMC
mc_enable 0x20
> 

2) connect BMC from remote.

[root@fedora22 ~]# ipmitool -I lanplus -H localhost -U admin power status
Password: (password) 
Chassis Power is on

[root@fedora22 ~]# ipmitool -I lanplus -H localhost -U admin power cycle
Password: 
Chassis Power Control: Cycle

-----------------------------------------------------------------------

II. Install pacemaker and setup fence_ipmilan with IPMI fencing.


0) Prepare Fedora 22.

  Go https://getfedora.org and download x86_64 and install it!

1) Setup libvirt, qemu-kvm

~~~
yum -y install libvirt qemu-kvm virt-manager virt-install
~~~

2) Download , review and modify before running script to create bridge device for PMU

This script will setup 3 bridges for 3 NICs ,which is connected to L3 switch respectively (eth0: vlan2, eth1: vlan3, eth2: vlan4).

You need to check SIP and DNS, DOM to fit your network, in my case, eth0 has been set , so skipped.

  https://github.com/masaki-furuta/rpms/blob/master/ipmi/root-script/create_br012.w541

Once finished, you'll see something similar configuration on your host.

~~~
[root@w541 root-script]# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.54ee7551b10f       no              enp0s25
br1             8000.1683e515bca8       no              enp0s20u1
br2             8000.1e50754a2295       no              enp0s20u3
~~~

3) Install IPMI related tools

Please check and run:

  https://github.com/masaki-furuta/rpms/blob/master/ipmi/scripts/setup_qemu-ipmi

4) Install qemu ipmi branch

Download 2.3.0-5 or 6.

2.3.0-5

  https://github.com/masaki-furuta/rpms/tree/master/ipmi/qemu-2.3.0-5

2.3.0-6

  https://github.com/masaki-furuta/rpms/tree/master/ipmi/qemu-2.3.0-5

Then run https://github.com/masaki-furuta/rpms/blob/master/ipmi/install_rpms to install all RPMs.

5) Create 3 VMs with IPMI wrapper script.

Download , review and modify before running script.

  https://github.com/masaki-furuta/rpms/blob/master/ipmi/root-script/create_pcs3.w541

This script will create *.emu and lan.conf.* files for IPMI wrapper scripts.

  i) /etc/ipmi/rh7.1-pcs1.emu ,/etc/ipmi/rh7.1-pcs2.emu ,/etc/ipmi/rh7.1-pcs3.emu
 ii) /etc/ipmi/lan.conf.rh7.1-pcs1 ,/etc/ipmi/lan.conf.rh7.1-pcs2 ,/etc/ipmi/lan.conf.rh7.1-pcs3 

And run qemu with following options ($NUM is VM#, $CL_NAME is name of VM):

  In /etc/ipmi/lan.conf.*:

~~~
...
  startcmd "qemu-system-x86_64 -machine accel=kvm -vnc :${NUM} -boot menu=on -hda /var/lib/libvirt/images/${CL_NAME}${NUM}.img -cdrom /var/lib/libvirt/images/rh
el-server-7.1-x86_64-dvd.iso -m 2048 -net nic,model=virtio,macaddr=52:54:00:00:00:0${NUM} -net tap,ifname=tap${NUM}_0,script=no -net nic,model=virtio,macaddr=52
:54:00:00:00:1${NUM},vlan=3 -net tap,ifname=tap${NUM}_1,script=no,downscript=no,vlan=3 -net nic,model=virtio,macaddr=52:54:00:00:00:2${NUM},vlan=4 -net tap,ifna
me=tap${NUM}_2,script=no,downscript=no,vlan=4"
...
~~~

Also creates TAP device to each Bridge for 3 NICs on each VM.
(br0 is for VID2, br1 is for VID3, br2 is for VID4).

~~~
[root@w541 root-script]# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.54ee7551b10f       no              enp0s25
                                                        tap1_0
                                                        tap2_0
                                                        tap3_0
br1             8000.1683e515bca8       no              enp0s20u1
                                                        tap1_1
                                                        tap2_1
                                                        tap3_1
br2             8000.1e50754a2295       no              enp0s20u3
                                                        tap1_2
                                                        tap2_2
                                                        tap3_2
~~~
Then run IPMI simulator

~~~
...
 ... xterm -e ipmi_sim -c /etc/ipmi/lan.conf.${CL_NAME}${NUM}
...
~~~

Also checked IPMI functionarity 

~~~
...
 ... ipmitool -I lanplus -H 10.0.2.21${NUM} -U admin -P password power status
...
~~~

And Finally connecting to VM via VNC.


~~~
...
...  vncviewer :${NUM} &
...
~~~

5) Install pacemaker on each node.


5-1

# yum install -y pcs fence-agents-all

5-2

# firewall-cmd --permanent --add-service=high-availability
# firewall-cmd --add-service=high-availability

5-3

# passwd hacluster

5-4

# systemctl start pcsd.service
# systemctl enable pcsd.service

5-5

# pcs cluster auth node1 node2 node3 -u hacluster

5-6

# pcs cluster setup --name my_cluster node1 node2 node3

5-7

# pcs cluster start --all
# pcs cluster enable --all

5-8

# pcs status
Cluster name: my_cluster
Last updated: Tue Aug 11 03:36:07 2015
Last change: Tue Aug 11 02:47:58 2015
Stack: corosync
Current DC: node3 (3) - partition with quorum
Version: 1.1.12-a14efad
3 Nodes configured


Online: [ node1 node2 node3 ]

Full list of resources:

PCSD Status:
  node1: Online
  node2: Online
  node3: Online

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled


5-9

# fence_ipmilan -a 10.0.2.211 -P -l admin -p password -o status -vv
Executing: /usr/bin/ipmitool -I lanplus -H 10.0.2.211 -U admin -P password -p 623 -L ADMINISTRATOR chassis power status

0 Chassis Power is on
 
# fence_ipmilan -a 10.0.2.212 -P -l admin -p password -o status -vv
# fence_ipmilan -a 10.0.2.213 -P -l admin -p password -o status -vv



# pcs stonith create stonith-ipmilan-10.0.2.211 fence_ipmilan pcmk_host_list="node1" ipaddr="10.0.2.211" action="reboot" login="admin" passwd="password" lanplus="1" delay=15 op monitor interval=60s  
# pcs stonith create stonith-ipmilan-10.0.2.212 fence_ipmilan pcmk_host_list="node2" ipaddr="10.0.2.212" action="reboot" login="admin" passwd="password" lanplus="1" delay=15 op monitor interval=60s  
# pcs stonith create stonith-ipmilan-10.0.2.213 fence_ipmilan pcmk_host_list="node3" ipaddr="10.0.2.213" action="reboot" login="admin" passwd="password" lanplus="1" delay=15 op monitor interval=60s  

5-10

# pcs stonith show
 stonith-ipmilan-10.0.2.211     (stonith:fence_ipmilan):        Started 
 stonith-ipmilan-10.0.2.212     (stonith:fence_ipmilan):        Started 
 stonith-ipmilan-10.0.2.213     (stonith:fence_ipmilan):        Started 

[root@node1 ~]# pcs stonith fence node2
Node: node2 fenced


EOF









