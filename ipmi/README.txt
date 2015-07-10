see https://lists.gnu.org/archive/html/qemu-devel/2015-04/msg00619.html and https://github.com/cminyard/qemu/tree/stable-2.3-ipmi .

refence:

/usr/share/man/man1/ipmi_sim.1.gz
/usr/share/man/man5/ipmi_lan.5.gz
/usr/share/man/man5/ipmi_sim_cmd.5.gz
/usr/share/man/man8/ipmilan.8.gz

root@fedora22 ~]# ipmitool -I lanplus -H localhost -U admin power status
Password: 
Chassis Power is on
