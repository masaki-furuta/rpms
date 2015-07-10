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
