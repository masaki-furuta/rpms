* Refence:

https://lists.gnu.org/archive/html/qemu-devel/2015-04/msg00619.html
https://github.com/cminyard/qemu/tree/stable-2.3-ipmi .
https://gist.github.com/bot11/

/usr/share/man/man1/ipmi_sim.1.gz
/usr/share/man/man5/ipmi_lan.5.gz
/usr/share/man/man5/ipmi_sim_cmd.5.gz
/usr/share/man/man8/ipmilan.8.gz

* Testing

[root@fedora22 ~]# ipmitool -I lanplus -H localhost -U admin power status
Password: (password) 
Chassis Power is on

[root@fedora22 ~]# ipmitool -I lanplus -H localhost -U admin power cycle
Password: 
Chassis Power Control: Cycle
