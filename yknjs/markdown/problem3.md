The customer observed
* Slow access to block devices on some VMs
* No access to block devices on some VMs
<!-- Note -->
Let's look at a third problem, the Recurring type.

A customer sent a ticket. They observed slow access to block devices
hosted on the backend storage from some VMs. On other VMs they cannot
access the block devices hosted on the backend storage at all.


```
openstack stack resource list --nested-depth 10  $STACK_UUID \
  --filter type=OS::Nova::Server \
  -c physical_resource_id -f value | \
  xargs -n1 openstack server show \
  -c hypervisor_hostname
```
```
for i in  $(cat list) ; do openstack server show \
-c hypervisor_hostname$; done
```
```
/var/log/dmesg
/var/log/kernel.log
/var/log/syslog
```
<!-- Note -->
Step 1. Define the problem
Start with the VMs that cannot access the block devices at all.
How are they spread across computes? Here is an example on how to
filter for the computes where the VMs are running, assuming all VMs
belong to the same Heat stack: list resources filtered on Nova::Server,
pipe the output to xargs and extract the hypervisor_hostname to
get the computes names.
Alternatively, you can loop through a list of the VMs.
Ask for /var/log/kernel.log, dmesg, /var/log/syslog logs from the
computes hosting the VMs that cannot access their block devices.
These logs might show why the devices cannot be accessed.


```
NETDEV WATCHDOG: eth4 (i40e): transmit queue 2 timed out
i40e 0000:83:00.2 eth4: NIC Link is Down
```
<!-- Note -->
Step 2. Establish a few hypotheses of a possible cause
Here's an idea: If the VM cannot access the block devices on the storage
backend, it might be a networking problem.
Look for log entries concerning the storage network interfaces.
The dmesg log show several entries like this one


```
ip link set up eth4
```
<!-- Note -->
Step 3. Test these hypotheses
The storage interface seems to be down, so try to bring it up.
It worked and the VM can access the block device again. 


```
ip link set down eth4
ip link set up eth4
```
<!-- Note -->
Step 4. Apply the solution
Apply the same fix for the rest of the computes.
On the computes where we have slow access to storage, reset the
problematic interface


Document the solution
<!-- Note -->
Step 5. Document the solution.
Write a wiki article where you detail the symptoms of the problem,
the logs with the relevant entries and the fix.
Write short sentences and add command outputs and log snippets.
I personally like a lot the RedHat Knowledgebase at https://access.redhat.com/
, well, when they don't hide the solutions. 
(https://access.redhat.com/solutions/2070703)
We came all the way to step 5, however,the problem is not solved.


Problem solved?
<!-- Note -->
A few days later, the customer sent a new ticket saying they
saw the same issues again. Some VMs face slow access to the block devices,
and some cannot access the devices at all.
We never found the cause for this problem, but merely a workaround.
If I try to deactivate and activate back the problematic network
interfaces, it will be a matter of time until the problem shows up again.


Find an expert
<!-- Note -->
When you start working with OpenStack, I can highly recommend you find
yourself an expert. Everyone knows who they are: a person
with some years of experience, that can help with almost any technical
problem. It's also the person that everybody wants a piece of. In
most cases, it is the person that writes excellent documentation too.
Ask for help if you get stuck, but prepare your questions thoroughly.
Make sure you can coherently define the problem. Have the logs ready
in case someone needs to check them.
By doing this you don't waste someone else's time; it's a matter of
showing respect.


The problem happens only on 12 out of 168 computes
<!-- Note -->
To further investigate the problem described earlier, you might need
access to the customer production platform.
You can of course, keep asking questions, but it will take a longer
time to come to a solution.
So we go back to Step 1: Define the problem
The problem seems to be limited to some of computes only.
It happens on 12 computes out of 168.
Let's look at the network interfaces on these computes.
They look like this:


Check the network interfaces
```
root@compute-40-3 # lshw  -C network -businfo
Bus info          Device          Class          Description
============================================================
pci@0000:01:00.0  eth0            network        I350 Gigabit Network Connection
pci@0000:01:00.1  eth1            network        I350 Gigabit Network Connection
pci@0000:83:00.0  eth2            network        Ethernet Controller X710 for 10GbE SFP+
pci@0000:83:00.1                  network        Ethernet Controller X710 for 10GbE SFP+
pci@0000:83:00.2  eth4            network        Ethernet Controller X710 for 10GbE SFP+
pci@0000:83:00.3                  network        Ethernet Controller X710 for 10GbE SFP+
```
<!-- Note -->
So we have two interfaces for the control network, eth0 and eth1.
We have two interfaces for the storage network, eth2 and eth4.
And we have two interfaces for the traffic network.
The storage network interfaces use the kernel driver; you can see their
device name in this output.
The traffic network interfaces, the ones that don't have a device name.
use the DPDK driver, ( the technology to offload TCP packet processing
from the operating system kernel to processes running in user space,
you want this for higher performance.)
All 4 of them share the same bus, look at the PCI addresses.


<!-- .slide: data-background-image="https://www.servethehome.com/wp-content/uploads/2020/04/3rd-Party-Intel-X710-DA4-NIC-Ports-LP-Bracket.jpg" data-background-color="white" data-background-size="contain" -->
<!-- Note -->
The network card looks like this:
(from here https://www.servethehome.com/3rd-party-intel-x710-da4-quad-10gbe-nic-review/ ).
So we have network interfaces controlled by the kernel driver and
network interfaces controlled by the DPDK driver, on the same
physical card.
Step 2. Establish a few hypotheses of a possible cause
The problem showed up on the computes having these NIC installed.
We assume it is related to these NICs.


```
$ iperf -c 192.168.122.111 
------------------------------------------------------------
Client connecting to 192.168.122.111, TCP port 5001
TCP window size:  586 KByte (default)
------------------------------------------------------------
[  3] local 192.168.122.100 port 59454 connected with 192.168.122.111 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec  8.43 GBytes  9.4 Gbits/sec
```
<!-- Note -->
Step 3. Test these hypotheses
We ran iperf on the NICs with ports controlled by multiple drivers
and observed:
* packet drops on the ports controlled by the kernel driver, and lower
  TCP throughput.
* complete traffic loss due to "tx hang" on the ports controlled by
  the kernel driver.


The manufacturer provided a fix:
* [dpdk-dev,v4,4/4] net/i40e: fix interrupt conflict when using multi-driver
* [dpdk-dev,v4,3/4] net/i40e: fix multiple driver support issue
<!-- Note -->
Step 4. Apply the solution
We raised a bug report towards the manufacturer of the network card.
Later, the manufacturer provided a patch that solved the problem.
https://dpdk.org/dev/patchwork/patch/34947/
https://dpdk.org/dev/patchwork/patch/34948/


Document the solution
<!-- Note -->
Time to go back to the wiki page and update it. Specify that you can
restart the interface, as workaround. To solve the problem, you
need to load a newer driver provided by the NIC manufacturer.

And the nicest thing you can do is to share this problem with the
world. Write it on your blog, tweet about it, make a video and put
on youtube.
