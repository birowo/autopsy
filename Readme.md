# Autopsy

dissect dead node service core dumps with mdb via a smart os vm

## alpha - setup

File size of vm resources is too large for github
or npm. We need to a) properly host the assets
and b) write asset downloading into the setup script

* `git clone` this repo
* download [assets.zip][] - extract within repo
* `npm link` 


## Why?

mdb is an awesome debugger that comes with smart os.

There's an extension for it that allows you to 
postmortem node core files

However, a tremendous amount of people run node
on linux (not least because of docker and what not). 

But it turns out that mdb can analyse linux core files,
we just have to give it the core file and the node binary
that was running when the core file was generated.

The problem is, actually getting you linux core file
into an environment that is running a version of mdb 
that this can work with is... painful.

So, autopsy abstracts that pain away, and installs an `autopsy`
executable on linux that essentially acts as a proxy to the 
mdb client within the smartos vm.

You can also run autopsy on OS X, but you'll need the linux
node binary and core file to pass to it. 


## Setup

### Preqrequisites

Autopsy depends on virtual box, on ubuntu/debian we can do

```sh
sudo apt-get install virtualbox
```

There's also currently a hard dependency on `expect`

```sh
sudo apt-get install expect
```

On OS X - well we can work it out ;)

### Install

```
sudo npm install -g autopsy
```

This will install autopsy on the system and immediately
setup a smartos vm in virtual box. The install will take
a long time. 


Once finished the following executables will be available

* autopsy-start - starts the vm
* autopsy-stop - stops the vm
* autopsy-status - gets vm status
* autopsy-setup - runs setup (only needful for troubleshooting)
* autopsy - provides the CLI proxy to mdb in the vm

## autopsy

The autopsy command takes the following args

```sh
autopsy [node-binary] core-file
```

On OS X the node binary is not optional, on linux
if not supplied the current installed node binary
will be used. 

When this command is run the following occurs

1. copy the node binary and core file into the vm (in a purpose built smartos zone)
2. ssh into the vm, login to the relevant smartos zone
3. run mdb with the copied core dump and node binary files
4. inject `::load v8` to get the v8 related debugging commands
5. pipe host machine process.stdin to the mdb interactive environment and pipe the mdb stdout to the host machine stdout

For using mdb see the [mdb reference docs][]


## Generating a core file

In production, if we run our node processes with `--abort-on-uncaught-exception` we will always get a core dump when a process crashes (that is,
as long as our linux environment is set up correctly)

You can also manually generate a core file using `process.abort()`.

Finally a core file can also be obtained by attaching `gdb` to a running processing and executing `generate-core`. 

## Setting up Linux to generate core files

If you're using an ubuntu server (and probably debian etc. etc.) you may have apport installed - this intercepts core files so we need to get rid of it

```
sudo apt-get purge apport
```

Next you need to make sure that linux is configured to allocate
space for the core file, like so

```
ulimit -c unlimited
```

Put this in a start up script and what not. 

## Why's it so big?

We'll get it smaller (hopefully), this is a first pass
and we're focusing on functionality. But the size it's 
because we're like.. running an entire virtual machine. 

## Example

The `example` folder has a `core` and `node` file that we're
generated by the `die.js` file 

You can try out autopsy with these two files (on OS X and 
Linux), from the same folder as this readme do

```
autopsy example/node example/core 
```

Once the mdb console appears you can try 

```
> ::jsstack
```

For starters, and then if you want to get fancy 

```
> ::findjsobjects -p myproperty
137289672551
> 137289672551::jsprint
```

## Future

* reduce size of everything
* maybe work with saved machine state to decrease boot time
* can we get rid of the dep on virtual box I wonder?
* work out if this is a problem "mdb: warning: librtld_db failed to initialize; shared library information will not be available" - may need a patch in the vm
* get ssh keys into the vm instead of passing a pw with expect, granted the pw being plain text in this case doesn't matter, but we might be able to get rid of the dependency on expect if we get rid of the pw. 
* vm shouldn't need 5gb of ram, sort that out. 
* extend this into another project that runs node processes in smart os zones (just like docker containers) - but with the added benefit of native smart os core dumps which can be analysed with mdb


## Other

* there is totally a reason for the method in terms of vm setup
* yes - we need to use an iso (and not the usb image)
* yes - the vmdk file needs to be separate from the smartos.ova file

[mdb reference docs]: https://github.com/joyent/mdb_v8/blob/master/docs/usage.md#node-specific-mdb-command-reference
[assets.zip]: https://drive.google.com/file/d/0B7fVI2pg3JazU1RwZFhTN3hwV0E/view?usp=sharing

