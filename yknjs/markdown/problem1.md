VM A cannot connect to VM B on port 1521 over TCP
<!-- Note -->
The first problem:
* A customer sent a ticket. They cannot connect to port 1521
on VM B, from VM A, but ping between the two hosts is OK. They also
say the problem must lie with the OpenStack Neutron, and they need help
to figure out what the problem is.


Step 1: Define the problem

Collect information

Do not fall for the confirmation bias
<!-- Note -->
Step 1. Define the problem.
Do not fall for the confirmation bias( an unintentional bias caused by
the collection and use of data in a way that favors a preconceived
notion.). This means do not jump at asking for Neutron logs because
someone said the problem must be in OpenStack Neutron.
Ask for logs and commands output to confirm and define the problem.


Ask for command output:

```
VM_A $ telnet 193.226.122.155 1521
VM_B $ telnet 193.226.122.155 1521
```
```
cat < /dev/tcp/<hostname_or_ip>/<port_number>

VM_A $ cat < /dev/tcp/193.226.122.155/1521

VM_B $ cat < /dev/tcp/127.0.0.1/1521
```
<!-- Note -->
Ask for telnet to port 1521 from VM A and VM B.
If the service is not running, we would not
want to miss that.
Use hostname instead of IP, if needed.
Many times telnet or curl are not installed.
To test if a port is open you can also use cat and /dev/tcp
like in the last example.


Ping works?
```
VM_A $ ping 193.226.122.155 -c 50
```
```
VM_A $ ping -s 1472 -M do -i 0.2 -c 50 193.226.122.155
PING alice.example.com (193.226.122.155) 1472(1500) bytes of data.
1480 bytes from alice.example.com (193.226.122.155): icmp_seq=1 ttl=64 time=1.07 ms
1480 bytes from alice.example.com (193.226.122.155): icmp_seq=2 ttl=64 time=0.492 ms
1480 bytes from alice.example.com (193.226.122.155): icmp_seq=3 ttl=64 time=0.436 ms
1480 bytes from alice.example.com (193.226.122.155): icmp_seq=4 ttl=64 time=0.393 ms
1480 bytes from alice.example.com (193.226.122.155): icmp_seq=5 ttl=64 time=0.430 ms
1480 bytes from alice.example.com (193.226.122.155): icmp_seq=6 ttl=64 time=0.427 ms
```
```
VM_A $ ping -s 1500 -M do 193.226.122.155
PING alice.example.com (193.226.122.155) 1500(1528) bytes of data.
ping: local error: message too long, mtu=1500
ping: local error: message too long, mtu=1500
ping: local error: message too long, mtu=1500
```
<!-- Note -->
Does ping work all the time? Ask for output of ping command.
Try also ping with bigger packet size, prohibiting fragmentation,
and a shorter time interval between packets.
That's the ping that helps you find out a path's MTU.


I received this:
```
VM_B $ cat < /dev/tcp/127.0.0.1/1521
bash: connect: Connection refused
bash: /dev/tcp/127.0.0.1/1521: Connection refused
```
```
VM_A $ cat < /dev/tcp/193.226.122.155/1521
bash: connect: Connection refused
bash: /dev/tcp/193.226.122.155/1521: Connection refused
```
<!-- Note -->
This is what I got back. Do you see what the problem is?
The service is not running.


[Rossella Sblendido](https://www.youtube.com/watch?v=aNA8Pvewu2M&t=242)
<!-- .slide: data-background-image="images/Rossella.jpg" data-background-color="white" data-background-size="contain" -->
<!-- Note -->
As Rossella Sblendido says in her talk on troubleshooting Neutron,
(https://www.youtube.com/watch?v=aNA8Pvewu2M&t=242)
when someone says I cannot ping a VM, check if the VM is up.
Totally check out her talk.
Surprisingly the service running on 1521 port was not up.
Start the service, and the problem is solved.
This was an example of solving a problem from step 1, where we try
to define the problem.


Document the solution
<!-- Note -->
However, don't skip the "Document the solution" step.
You might think that there is not much to document here. Don't close
the ticket with "Problem Solved" before making sure the communication
with the customer is recorded in the ticket, the questions
you asked and the replies you received are there. Someone bumping
into this ticket might learn about asking questions in a clear manner.


Lesson learned:

* Write short and clear questions, ponder on language barriers.
* Consider time zone difference.

<!-- Note -->
The lesson learned is: Try to understand what the problem is by asking
short and clear questions.
Note that if there is a time zone difference between you and the
customer, it might take a day until you get a reply. If the questions
are not formulated in a clear matter, you will lose an additional day
to clarify what you asked for in the first place.


Don't ask "Send me the Neutron security group rules"

You could ask, send me output of this command:
```
openstack stack resource list --nested-depth 10 stack_UUID \
 --filter type=OS::Neutron::SecurityGroup \
 -c physical_resource_id -f value \
 | xargs openstack security group show
```
<!-- Note -->
Don't ask "Send me the security group" which might have
been the problem in the case above.
Ask instead, send me the output of this command.
Customers use Heat templates to deploy many resources in one go.
With this command you need the stack ID only, then you can filter
on security groups and gracefully pipe the output to xargs magic
to get the security group rules:


```
openstack stack resource list --nested-depth 10 stack_UUID --filter type=OS::Neutron::SecurityGroup  -c physical_resource_id -f value | xargs openstack security group show
+-----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field           | Value                                                                                                                                                                                                                                              |
+-----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| created_at      | 2020-09-11T08:33:26Z                                                                                                                                                                                                                               |
| description     | Neutron security group rules                                                                                                                                                                                                                       |
| id              | ec0cd819-648b-47d4-93df-7737aba754d7                                                                                                                                                                                                               |
| location        | cloud='somewhere-on-planet-earth', project.domain_id=, project.domain_name='Domain', project.id='559ab787c1a745a2a023a9a324274317', project.name='some-project', region_name='aPlaceOnEarth', zone=                                                     |
| name            | stack_UUID-server_security_group-ftf7alybvujz                                                                                                                                                             |
| project_id      | 559ab787c1a745a2a023a9a324274317                                                                                                                                                                                                                   |
| revision_number | 4                                                                                                                                                                                                                                                  |
| rules           | created_at='2020-09-11T08:33:26Z', direction='ingress', ethertype='IPv4', id='8226a5a2-d98a-41ae-ba77-f30a8a1e42e8', remote_group_id='ec0cd819-648b-47d4-93df-7737aba754d7', updated_at='2020-09-11T08:33:26Z'                                     |
|                 | created_at='2020-09-11T08:33:26Z', direction='ingress', ethertype='IPv4', id='abeff64e-4f2e-4805-82ca-575638c19036', protocol='icmp', remote_ip_prefix='0.0.0.0/0', updated_at='2020-09-11T08:33:26Z'                                              |
|                 | created_at='2020-09-11T08:33:26Z', direction='egress', ethertype='IPv6', id='b312a377-f695-4c9d-bc57-a66cafaa66c3', updated_at='2020-09-11T08:33:26Z'                                                                                              |
|                 | created_at='2020-09-11T08:33:26Z', direction='ingress', ethertype='IPv4', id='b480ec02-fd91-43f8-86ab-76f7ef68d669', port_range_max='1521', port_range_min='1521', protocol='tcp', remote_ip_prefix='0.0.0.0/0', updated_at='2020-09-11T08:33:26Z' |
|                 | created_at='2020-09-11T08:33:26Z', direction='egress', ethertype='IPv4', id='e6672bae-9808-42c4-aa6d-88bfd8a5dcfe', updated_at='2020-09-11T08:33:26Z'                                                                                              |
| stateful        | None                                                                                                                                                                                                                                               |
| tags            | []                                                                                                                                                                                                                                                 |
| updated_at      | 2020-09-11T08:33:26Z                                                                                                                                                                                                                               |
+-----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
```
direction='ingress', ethertype='IPv4', id='b48...', port_range_max='1521', port_range_min='1521', protocol='tcp', remote_ip_prefix='0.0.0.0/0'
```
<!-- Note -->
And you will get back this.
