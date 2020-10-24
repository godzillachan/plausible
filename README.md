# DEMOCRACY-BUILDER
This project provides a framework to quickly set up an environment that
can be used to provide privacy and plausible deniability by setting up an
encrypted volume alongside your normal operating environment.

## COMPONENTS
Your environment will consist of the following components: *outer shell*, and
a *plausibly-deniable interior*.

**OUTER SHELL**: the outer shell is any reasonably recent Linux distribution
installed using FDE (Full-Disk Encryption) support.  This environment provides
the first layer of defense against an adversary by requiring a passphrase in
order to access the on-disk content of the operating system.  You would set up
this outer shell however you like.  It can be your favorite distribution
installed on bare metal, or it can be a virtual machine running alongside your
production environment.

This outer shell *must not* contain any sensitive material, e.g., your
private SSH keypair to GitHub repository, your PGP key, your chat log, etc.

This is where you would install common tools, e.g., your compilers, various
language interpreters, or web browsers.

*You should consider the outer shell a throw-away environment and be prepared
to reveal the passphrase when demanded.*

Additionally, special care must be given if your outer shell is a virtual
machine.  See the [special notes](#special-notes) for more details.

**PLAUSIBLY-DENIABLE INTERIOR**: we are going to set up a *plausibly-deniable
interior* environment sitting on top of your FDE outer shell. This provides
the true layer of encryption and obfuscation and increases the difficulty for
an adversary to locate and piece together this environment.

This interior is assembled from sensitive data hosted off-site, and
from *physical volumes* (PV) stored on the *outer shell*'s local filesystem.

## IMPLEMENTATION
The PDI consists of physical volumes on the local filesystem of the *outer
shell*.  These physical volumes are assembled to form a single *logical volume*
(LV) on demand.

<diagram showing five chunks of data represented as squares, denote
one chunk as special via dotted outline>

We denote one physical volume as *safe-zone*.  This volume must be large enough
to hold an actual filesystem with innocuous content, e.g., a fully unpacked Linux
filesystem for an embedded platform such as a Raspberry Pi.

These PV's are assembled to form a logical volume as soon as *loopback devices*
are created.  Furthermore, because we put safe-zone content on the LV, we can 
show this safe-zone in good faith when demanded.

**NOTE**: any write operation you make to the safe-zone risks corrupting the true
content of your PDI depending on the choice of filesystem you use.  For a higher
level of predictability, a log-structured filesystem such as F2FS could be used to
store the safe-zone content. 

Once the PDI LV is created, we will have a plausible-deniability zone (PDZ) behind
the safe-zone to store our true hidden content.

<diagram showing the assembled logical volume consisting of the safe-zone content
and blank area behind it>

The PDZ is implemented via LUKS (Linux Unified Key Setup) cryptographic system
where we detach the *LUKS header*, and employ *payload offset* so that we can
reach into the PDZ when we wish to access the content.

<diagram showing the above assembly diagram, with additional block depicting LUKS
header, and a payload-offset arrow pointing to the beginning of the PDZ>

#### SPECIAL NOTES
