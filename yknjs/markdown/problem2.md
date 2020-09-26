The Openstack controller VM does not come up after a reboot
<!-- Note -->
The second problem:
A customer sent a ticket. They rebooted the first out of three
OpenStack controllers and lost access to it. In this case, the
controller was a VM, not a bare-metal controller.
Time to ask some questions.


Don't ask:

"What did you do, what did you change?"
<!-- Note -->
Don't ask, "What did you do, what did you change?". From my personal
experience, the interlocutor switches to defense mode. Your question
reads, "what did you change to break the controller, you incompetent
 person!".


Step 1: Ask

* "How do you connect to the VM?"

* Ask for the console log, `/var/log/libvirt/qemu/controller-1`

* `$ script -a VM_log ; sudo virsh console controller-1`
<!-- Note -->
Ask instead:
*  How do you  access the controller?
  (I sometimes try to stay away from Yes/No question, that's the easy
  way out, so I don't ask "You mean you cannot SSH?")
* If you cannot ssh to the controller, ask for the console log.
* Or you can ask the customer to connect to the VM using
  virsh console and send you the output.


Step 2: Establish a hypothesis

The logs show:
```
The disk drive for /mnt/backups is not ready yet or not present.
Continue to wait, or Press S to skip mounting or M for manual recovery
```
<!-- Note -->
Step 2. Establish a few hypotheses of a possible cause
The console log showed this, the VM hangs on boot.
It seems that some disk drive is no longer attached to the controller
VM or there is some misconfiguration in /etc/fstab.


Step 3,4: Test the hypothesis and Apply the solution

Fix `/etc/fstab`
<!-- Note -->
Step 3. Test these hypotheses :
Boot in single mode, and check out the/etc/fstab file.
Step 4. Apply the solution:
Remove the offending line, if the customer agrees, and reboot the VM.
If not, fix the misconfiguration.
Some months ago, someone added a line to /etc/fstab to mount shared
device and back up files to a remote destination, which does not
exist anymore.


Document the solution
<!-- Note -->
Write the solution in short sentences in the ticket.
Add command outputs and log snippets.
Create a wiki page too. Some else might run into this
problem. It's helpful to have a well-written solution on how to
recover a controller or a VM that won't boot when something is wrong
in /etc/fstab.
