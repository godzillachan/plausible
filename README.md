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

**PLAUSIBLY-DENIABLE ENVIRONMENT**: we are going to set up a *plausibly-deniable
environment* sitting on top of your FDE outer shell. This provides the true
layer of encryption and obfuscation and increases the difficulty for an adversary
to locate and piece together this environment.

This environment is assembled from sensitive data hosted off-site, and
from *backing pages* stored on the *outer shell*'s local filesystem.

## IMPLEMENTATION
The PDE consists of a series of *backing pages* on the local filesystem of the *outer
shell*.  These *backing pages* are assembled to form a single *multiple-device* (md) on
demand.

```
<diagram showing backing pages represented as squares, denote
one backing page as special via dotted outline>
+------+ +------+ +------+ +------+ +------+
| safe | |      | |      | |      | |      |
+------+ +------+ +------+ +------+ +------+
```

We denote one backing page as *safe-zone*.  This *safe-zone* must be large enough
to hold an actual filesystem with innocuous content, e.g., a fully unpacked Linux
kernel source tree for an embedded platform such as a Raspberry Pi.  Because we put
*safe-zone* content at the beginning of the device, we can show this safe-zone in
good faith when demanded.

**NOTE**: any write operation you make to the safe-zone risks corrupting the true
content of your PDE depending on the choice of filesystem you use.  For a higher
level of predictability, a log-structured filesystem such as F2FS could be used to
store the safe-zone content.

The true plausibly-deniable zone (PDZ) exists behind the safe-zone to store our
true hidden content.

```
<diagram showing backing pages with safe-zone content and pdz pages behind it>
+------+ +------+ +------+ +------+ +------+
| safe | | pdz1 | | pdz2 | | pdz3 | | pdz4 |
+------+ +------+ +------+ +------+ +------+
```

The PDZ is implemented via LUKS (Linux Unified Key Setup) cryptographic system
where we detach the *LUKS header*, and employ *payload offset* so that we can
reach into the PDZ when we wish to access the content.

```
<diagram showing the above assembly diagram, with additional block depicting LUKS
header, and a payload-offset arrow pointing to the beginning of the PDZ>
+------+ +------+ +------+ +------+ +------+
| safe | | pdz1 | | pdz2 | | pdz3 | | pdz4 |
+------+ +------+ +------+ +------+ +------+
--------------> payload offset
```

## FREEDOM-CONSOLE
We ship a Python-based console to help create the environment described above.
The console consists of the following major command groups:
* pages
* md
* keys
* headers
* pde

The **pages** command is used to create backing pages on your outer-shell.  By default
we create as many pages as will fit in the remaining space of your outer-shell.
However you can use parameters to the *pages* command to tune this behavior.

```
> pages create -h
usage: pages create [-h] [-m [MAXIMUM]] [-d [DATA_PAGESIZE]] [-n] [--root ROOT]

optional arguments:
  -h, --help            show this help message and exit
  -m [MAXIMUM], --maximum [MAXIMUM]
                        number of data pages to allocate, default is to allocate as many pages as would fit in available space
  -d [DATA_PAGESIZE], --data_pagesize [DATA_PAGESIZE]
                        size of each page to allocate, defaults to 1GiB.
  -n, --simulated       perform a trial run with no changes made
  --root ROOT           root directory of the backing pages
```

Once you have created the *backing pages*, you can assemble them into a
*multiple-device* (md) to be used by LUKS layer.  The **md** command is used
to manage it.
```
> md -h
usage: md [-h] {start,stop,status,populate-safezone} ...

Operate on multiple-disk device that holds the plausibly-deniable environment.

optional arguments:
  -h, --help            show this help message and exit

Subcommands:
  {start,stop,status,populate-safezone}
    start               start multiple-disk device
    stop                stop multiple-disk device
    status              status of the multiple-disk device
    populate-safezone   populate multiple-disk device with innocuous content
```

Once you have a working *multiple-device* (md), you can proceed to create
cryptographic keys and LUKS headers.  This is done with the help of **keys** and
**headers** commands.

```
> keys -h
usage: keys [-h] {create,remove,list} ...

Operate on cryptographic keys that are used to lock and unlock content of the plausibly-deniable environment.

optional arguments:
  -h, --help            show this help message and exit

Keys Subcommands:
  {create,remove,list}
    create              create random cryptographic keys
    remove              remove cryptographic keys
    list                list cryptographic keys
```

and:
```
> headers -h
usage: headers [-h] {create,remove,list} ...

Operate on LUKS headers that are used to access the plausibly-deniable content.

optional arguments:
  -h, --help            show this help message and exit

LUKS Header Subcommands:
  {create,remove,list}
    create              create random headers
    remove              remove headers
    list                list headers and associated information
```

**NOTE**: it is especially important to keep the output of the **headers**
command in a safe place.  It contains the UUID name of the LUKS headers
along with the UUID of the binary cryptographic keys, along with the
offset into the key material, as well as the payload offset into the
PDZ.

After creating the headers, you need to designate one set of *header*, *key*,
and *key-offset* as the true tuple that will unlock your PDZ content.

You will then *bless* your PDZ via the **pde** *start* sub-command.

```
> pde start -h
usage: pde start [-h] [--bless] --header HEADER --key KEY --offset OFFSET

optional arguments:
  -h, --help       show this help message and exit
  --bless          create a fresh filesystem on the plausibly-deniable environment

required arguments:
  --header HEADER  header uuid
  --key KEY        key uuid
  --offset OFFSET  offset into cryptographic key
```

By *blessing* our PDZ, we are opening the *plausibly-deniable zone* with the tuple
(header, key, key-offset) as well as creating a filesystem on it.  Since only
you know this tuple, it will be extremely difficult for an adversary to determine
the correct tuple via brute-force attacks.

#### SPECIAL NOTES
