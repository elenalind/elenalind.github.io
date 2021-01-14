You know nothing, Jon Snow! OpenStack troubleshooting from a beginner's perspective
===================================================================================

Jon Snow is a prominent character in the Game of Thrones TV series.
He is a skilled swordsman, defined by a strong sense of justice and honor.
Jon Snow proves his leadership skills and strategic thinking in many
situations.

But you might remember Jon meeting the Wildings, a new culture that
he did not fully understand the first time he met them. It took time
until Jon could resonate with them, and he looked like a newb in the
process. This triggered Ygritte to call out, "You know nothing, Jon Snow"
several times. Soon enough, Jon learned how to deal with the Wildings,
he earned their respect and trust, and they followed him as a leader.

I find this analogy to fit people starting to work with OpenStack.
It feels very overwhelming in the beginning, but you can build on
your skills and experience once you get familiar with what project
is responsible for what functionality.

I had the unforeseen-except-perhaps-in-my-dreams chance to get a new
job in 2015, to work with OpenStack. One way I prepared for the new job
was by listening to [OSPod](https://www.podplay.com/en/podcast/27966/ospod),
I've listened to many episodes several times.
I found it interesting to listen to OpenStack's initial cast talking
about the projects they contributed to. Several of these podcast
episodes are quite inspiring. Take [this one](https://www.podplay.com/en/podcast/27966/ospod/episode/1807637/red-hat-flavio-percoco-episode-41-on-ospod)
for example.

OpenStack breaks down to services running on Linux and troubleshooting
OpenStack goes down to troubleshooting these services and checking
their log files. The trick is to know where to look first without
wasting time checking services that are not the cause of the problem
you are investigating. It might sound simple, but it's not, and sometimes
one needs to employ diligent sleuthing to find what the problem is
and fix it.

I started working with OpenStack from its Mitaka release. My first
role was as a support engineer and, I helped telecom operators around
the globe fix problems in their OpenStack deployments. Looking back,
I can draw the line, and I can tell you that once-in-a-blue-moon
rarely I stumbled over bugs in OpenStack. Maybe the reason for that
was that the Mitaka release reached an acceptable maturity level.
Another reason could be the fine-tooth combing of OpenStack services
through all sorts of testing like functional testing, system testing,
and end-to-end testing. Extensive testing prevents going to production
and discovering significant bugs.
Most of the support cases that came in condensed to networking issues:
* Cannot access a VM on a port - networking problem
* Cannot access storage media from VM - networking problem
* VM does not boot anymore - networking problem
* OpenStack controllers froze - networking problem

Most of the time, the networking problems did not result in bugs
filed under the Neutron project. Many cases were resolved
by restarting services, patching network devices, fixing Linux
configuration, properly configuring network devices like routers and
switches.

This article builds on experiences troubleshooting OpenStack.
You can use this approach:
* define what the problem is,
* narrow down what the cause is,
* solve the problem,
* pass on the experience through proper documentation.

Define the problem
------------------

A customer sent in a ticket saying that they cannot connect from
VM A to VM B on port 1521 over TCP, but ping between the two
hosts is working. They mentioned the problem must be in Neutron,
but they are not sure what Neutron logs they should check.

As a first step, I must get a better understanding of the problem.
It's easy to fall for the confirmation bias and ask the customer
for Neutron configuration printouts and logs.
The customer might be sitting in a different time zone, and email
communication can be sparse because of the time difference. Assume I get
the printouts and logs I asked for, and I can't find anything obviously
wrong. Then I need at least one more day to ask for more information.

So to avoid sending the customer scurrying grumpily all over the place
it's good to ask for printouts that confirm the problem they raised.

We cannot access VM B on port 1521? I would assume the service
running on port 1521 on VM B is up and running.

Ask for printouts to confirm it:

```bash
VM_A $ telnet 192.168.102.155 1521
VM_B $ telnet 192.168.102.155 1521
```

where 192.168.102.155 is the IP address on VM B received from the
customer in the problem description.

There might be a problem with the commands above. On most production
servers, `telnet` is not installed, but `cat` should be there.
We can instead ask for:

```
cat < /dev/tcp/<hostname_or_ip>/<port_number>
VM_A $ cat < /dev/tcp/192.168.102.155/1521
VM_B $ cat < /dev/tcp/127.0.0.1/1521
```

In this example, `bash` will attempt to open a TCP connection to the
corresponding socket.

The customer mentioned that ping between the hosts works. We want
to run a command that confirms this fact but that also would check
for things like MTU size, which could be a cause for the problem
they raised:

Ask for a printout of `ping` that tests a bigger packet size,
prohibiting fragmentation, with packets being sent at shorter intervals:

```
VM_A $ ping -s 1472 -M do -i 0.2 -c 50 192.168.102.155
```

What I got back was this:

```
VM_A $ cat < /dev/tcp/193.226.122.155/1521
bash: connect: Connection refused
bash: /dev/tcp/193.226.122.155/1521: Connection refused
```

The service running on port 1521 was not up, which explains the symptoms.
It often happens that the cause of a problem is something simple that
was overlooked, like a service not running,  a service that
keeps restarting, or the VM where the service should be running is down.
In her talk about [troubleshooting Neutron] (https://www.youtube.com/watch?v=aNA8Pvewu2M&t=242),
Rossella Sblendido mentions just that.

It's worth to ask for printouts that confirm the problem description.
It might result in solving a problem in the definition phase.

One might assume there's not much to document when solving the problem
described above. Add the questions you asked together with the customer
replies and printouts to the customer ticket. If someone later finds
your case, it will serve as a good example of asking questions and
defining the problem.

Solve the problem
-----------------

We will now look into a support ticket where a customer says
they rebooted one out of three OpenStack controllers and lost access
to it. In this case, the OpenStack services are running inside three
different VMs and thus, we have three virtual OpenStack controllers.

We should avoid asking questions like "What did you do, what did
you change?" which renders an interlocutor to defense mode. Such
questions might read as accusations when the situation is tensed.

When defining the problem, we could ask:

* "How do you connect to the controller VM?"
* "Please send me the console log, `/var/log/libvirt/qemu/controller-1`"
* "Please send me a printout of `$ script -a VM_log ; sudo virsh console controller-1`"

The console logs show the following:

```
The disk drive for /mnt/backups is not ready yet or not present.
Continue to wait, or Press S to skip mounting or M for manual recovery
```

It seems that a block device is no longer attached to the controller
VM or there is some misconfiguration in `/etc/fstab`.

We need to access the VM, possibly by booting in single mode, and
correct `/etc/fstab` by removing the offending line.
Some months ago, someone added a line to /etc/fstab to mount shared
device and back up files to a remote destination, which does not
exist anymore.

Documentation is again very important. Besides properly documenting
the information exchanged with the customer, you can write a knowledge
object that explains how to connect to a OpenStack controller VM
when the VM won't boot, and it is stuck looking for block devices
that do no exist anymore.

Document the problem
--------------------

The third support case is one where the customer observed slow
access to block devices hosted on the backend storage from some VMs.
Other VMs cannot access their block devices at all, and a third
part of VMs experience no problems whatsoever.

Start with the VMs that cannot access the block devices at all.
How are they spread across computes? Here is an example of how to
filter for the computes where the VMs are running, assuming all VMs
belong to the same `Heat stack`:i list resources filtered on `Nova::Server`,
pipe the output to `xargs` and extract the `hypervisor_hostname` to
get the computes names.
Alternatively, you can loop through a list of the VMs.
Ask for `/var/log/kernel.log`, `dmesg`, `/var/log/syslog` logs from the
computes hosting the VMs that cannot access their block devices.
These logs might show why the devices cannot be accessed.

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

If the VMs cannot access their block devices on the storage
backend, it might be a networking problem.
Look for log entries concerning the storage network interfaces.
The dmesg log show several entries like this one

```
NETDEV WATCHDOG: eth4 (i40e): transmit queue 2 timed out
i40e 0000:83:00.2 eth4: NIC Link is Down
```

The storage interface seems to be down, so try to bring it up.
It worked, and the VM can access the block device again. 

```
ip link set up eth4
```

Apply the same fix for the rest of the computes.
On the computes where we have slow access to storage, reset the
problematic interface.

We fixed the problem reported by the customer, but we do not know yet
what caused it. It is important we continue the investigation otherwise,
the problem might surface again.


If you can't find anything obvious in the logs to help you conclude
what cause the interfaces to go rogue, it might be a good time to turn
to an expert and ask for help.

When you start working with OpenStack, I can highly recommend you find
yourself an expert. Everyone knows who they are: a person
with some years of experience, that can help with almost any technical
problem. It's also the person that everybody wants a piece of. In
most cases, it is the person that writes excellent documentation too.
Ask for help if you get stuck, but prepare your questions thoroughly.
Make sure you can coherently define the problem. Have the logs ready
in case someone needs to check them.
By doing this, you don't waste someone else's time; it's a matter of
showing respect.

The problem happens only on 12 out of 168 computes
Let's look at the network interfaces on these computes.
They look like this:

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

As you can imagine, documentation in this specific case is of crucial
importance. While in the previous cases you might find help using
internet search engines, this is a problem specific to the customer
OpenStack deployment using a particular NIC card.

Besides properly recording this problem with its solution in the
company's internal tools, it would be nice to share the problem
on the internet because it might help someone else.
Write it on your blog, tweet about it, or record a video and upload it
on youtube.

There are many problem-solving strategies and methods. I wish I could
state that I found the three-steps approach that solves any OpenStack
problem, but there is no silver bullet. What can help OpenStack
troubleshooting efficient is to ask the right questions to help you
understand and define the problem, to establish and test a few
hypotheses of a possible cause and narrow down where the problem
is located. Then, once a fix is available, to prioritize properly
documenting the problem with its solution.
