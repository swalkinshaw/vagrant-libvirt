# Vagrant Libvirt Provider

This is a [Vagrant](http://www.vagrantup.com) plugin that adds an
[Libvirt](http://libvirt.org) provider to Vagrant, allowing Vagrant to
control and provision machines via Libvirt toolkit.

**Note:** Actual version is still a development one. Feedback is
welcome and can help a lot :-)

## Features

* Control local Libvirt hypervisors.
* Vagrant `up`, `destroy`, `suspend`, `resume`, `halt`, `ssh`, `reload`, `package` and `provision` commands.
* Upload box image (qcow2 format) to Libvirt storage pool.
* Create volume as COW diff image for domains.
* Create private networks.
* Create and boot Libvirt domains.
* SSH into domains.
* Setup hostname and network interfaces.
* Provision domains with any built-in Vagrant provisioner.
* Synced folder support via `rsync`, `nfs` or `9p`.
* Snapshots via [sahara](https://github.com/jedi4ever/sahara).
* Package caching via [vagrant-cachier](http://fgrehm.viewdocs.io/vagrant-cachier/).
* Use boxes from other Vagrant providers via [vagrant-mutate](https://github.com/sciurus/vagrant-mutate).

## Future work

* Take a look at [open issues](https://github.com/pradels/vagrant-libvirt/issues?state=open).

## Installation

First, you should have libvirt installed if you plan to run VMs on your local system. For instructions, refer to your linux distribution's documentation,

Next, you must have [Vagrant installed](http://docs.vagrantup.com/v2/installation/index.html). Vagrant-libvirt supports Vagrant 1.5, 1.6 and 1.7.

 Now you're ready to install vagrant-libvirt using standard [Vagrant plugin](http://docs.vagrantup.com/v2/plugins/usage.html) installation methods.

```
$ vagrant plugin install vagrant-libvirt
```

### Possible problems with plugin installation on Linux

In case of problems with building nokogiri and ruby-libvirt gem, install
missing development libraries for libxslt, libxml2 and libvirt.

In Ubuntu, Debian, ...
```
$ sudo apt-get install libxslt-dev libxml2-dev libvirt-dev zlib1g-dev
```

In RedHat, Centos, Fedora, ...
```
# yum install libxslt-devel libxml2-devel libvirt-devel libguestfs-tools-c
```

If have problem with installation - check your linker. It should be ld.gold:
```
sudo alternatives --set ld /usr/bin/ld.gold
# OR
sudo ln -fs /usr/bin/ld.gold /usr/bin/ld
```

## Vagrant Project Preparation

### Add Box

After installing the plugin (instructions above), the quickest way to get
started is to add Libvirt box and specify all the details manually within
a `config.vm.provider` block. So first, add Libvirt box using any name you
want. This is just an example of Libvirt CentOS 6.4 box available:

```
vagrant box add fedora21 http://citozin.com/fedora21.box
# or
vagrant box add centos64 http://citozin.com/centos64.box
```

### Create Vagrantfile

And then make a Vagrantfile that looks like the following, filling in your
information where necessary. In example below, VM named test_vm is created from
centos64 box.

```ruby
Vagrant.configure("2") do |config|
  config.vm.define :test_vm do |test_vm|
    test_vm.vm.box = "centos64"
  end
end
```

### Start VM

In prepared project directory, run following command:

```
$ vagrant up --provider=libvirt
```

Vagrant needs to know that we want to use Libvirt and not default VirtualBox.
That's why there is `--provider=libvirt` option specified. Other way to tell
Vagrant to use Libvirt provider is to setup environment variable

`export VAGRANT_DEFAULT_PROVIDER=libvirt`.

### How Project Is Created

Vagrant goes through steps below when creating new project:

1.	Connect to Libvirt localy or remotely via SSH.
2.	Check if box image is available in Libvirt storage pool. If not, upload it to
	remote Libvirt storage pool as new volume.
3.	Create COW diff image of base box image for new Libvirt domain.
4.	Create and start new domain on Libvirt host.
5.	Check for DHCP lease from dnsmasq server.
6.	Wait till SSH is available.
7.	Sync folders and run Vagrant provisioner on new domain if
	setup in Vagrantfile.


### Libvirt Configuration

### Provider Options

Although it should work without any configuration for most people, this provider exposes quite a few provider-specific configuration options. The following options allow you to configure how vagrant-libvirt connects to libvirt, and are used to generate the [libvirt connection URI](http://libvirt.org/uri.html):

* `driver` - A hypervisor name to access. For now only kvm and qemu are supported.
* `host` - The name of the server, where libvirtd is running.
* `connect_via_ssh` - If use ssh tunnel to connect to Libvirt.
* `username` - Username and password to access Libvirt.
* `password` - Password to access Libvirt.
* `id_ssh_key_file` - If not nil, uses this ssh private key to access Libvirt. Default is $HOME/.ssh/id_rsa. Prepends $HOME/.ssh/ if no directory.
* `socket` - Path to the libvirt unix socket (eg: /var/run/libvirt/libvirt-sock)
* `uri` - For advanced usage. Directly specifies what libvirt connection URI vagrant-libvirt should use. Overrides all other connection configuration options.

Connection-independent options:

* `storage_pool_name` - Libvirt storage pool name, where box image and instance snapshots will be stored.

Here is an example of how to set these options.

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :libvirt do |libvirt|
    libvirt.host = "example.com"
  end
end
```

### Domain Specific Options

* `disk_bus` - The type of disk device to emulate. Defaults to virtio if not set. Possible values are documented in libvirt's [description for _target_](http://libvirt.org/formatdomain.html#elementsDisks).
* `nic_model_type` - parameter specifies the model of the network adapter when you create a domain value by default virtio KVM believe possible values, see the documentation for libvirt
* `memory` - Amount of memory in MBytes. Defaults to 512 if not set.
* `cpus` - Number of virtual cpus. Defaults to 1 if not set.
* `nested` - [Enable nested virtualization](https://github.com/torvalds/linux/blob/master/Documentation/virtual/kvm/nested-vmx.txt). Default is false.
* `cpu_mode` - What cpu mode to use for nested virtualization. Defaults to 'host-model' if not set.
* `volume_cache` - Controls the cache mechanism. Possible values are "default", "none", "writethrough", "writeback", "directsync" and "unsafe". [See driver->cache in libvirt documentation](http://libvirt.org/formatdomain.html#elementsDisks).
* `kernel` - To launch the guest with a kernel residing on host filesystems. Equivalent to qemu `-kernel`.
* `initrd` - To specify the initramfs/initrd to use for the guest. Equivalent to qemu `-initrd`.
* `random_hostname` - To create a domain name with extra information on the end to prevent hostname conflicts.
* `cmd_line` - Arguments passed on to the guest kernel initramfs or initrd to use. Equivalent to qemu `-append`.
* `graphics_type` - Sets the protocol used to expose the guest display.  Defaults to `vnc`.  Possible values are "sdl", "curses", "none", "gtk", "vnc" or "spice".
* `graphics_port` - Sets the port for the display protocol to bind to.  Defaults to 5900.
* `graphics_ip` - Sets the IP for the display protocol to bind to.  Defaults to "127.0.0.0.1".
* `graphics_passwd` - Sets the password for the display protocol. Working for vnc and spice. by default working without passsword.
* `video_type` - Sets the graphics card type exposed to the guest.  Defaults to "cirrus".  [Possible values](http://libvirt.org/formatdomain.html#elementsVideo) are "vga", "cirrus", "vmvga", "xen", "vbox", or "qxl".
* `keymap` - Set keymap for vm. default: en-us
* `video_vram` - Used by some graphics card types to vary the amount of RAM dedicated to video.  Defaults to 9216.
* `machine` - Sets machine type. Equivalent to qemu `-machine`. Use `qemu-system-x86_64 -machine help` to get a list of supported machines.
* `machine_arch` - Sets machine architecture. This helps libvirt to determine the correct emulator type. Possible values depend on your version of qemu. For possible values, see which emulator executable `qemu-system-*` your system provides. Common examples are `aarch64`, `alpha`, `arm`, `cris`, `i386`, `lm32`, `m68k`, `microblaze`, `microblazeel`, `mips`, `mips64`, `mips64el`, `mipsel`, `moxie`, `or32`, `ppc`, `ppc64`, `ppcemb`, `s390x`, `sh4`, `sh4eb`, `sparc`, `sparc64`, `tricore`, `unicore32`, `x86_64`, `xtensa`, `xtensaeb`.
* `machine_virtual_size` - Sets the disk size in GB for the machine overriding the default specified in the box. Allows boxes to defined with a minimal size disk by default and to be grown to a larger size at creation time. Will ignore sizes smaller than the size specified by the box metadata. Note that currently there is no support for automatically resizing the filesystem to take advantage of the larger disk.
* `boot` - Change the boot order and enables the boot menu. Possible options are "hd" or "network". Defaults to "hd" with boot menu disabled. When "network" is set first, *all* NICs will be tried before the first disk is tried.
* `nic_adapter_count` - Defaults to '8'. Only use case for increasing this count is for VMs that virtualize switches such as Cumulus Linux. Max value for Cumulus Linux VMs is 33.


Specific domain settings can be set for each domain separately in multi-VM
environment. Example below shows a part of Vagrantfile, where specific options
are set for dbserver domain.

```ruby
Vagrant.configure("2") do |config|
  config.vm.define :dbserver do |dbserver|
    dbserver.vm.box = "centos64"
    dbserver.vm.provider :libvirt do |domain|
      domain.memory = 2048
      domain.cpus = 2
      domain.nested = true
      domain.volume_cache = 'none'
    end
  end

  # ...
```

The following example shows part of a Vagrantfile that enables the VM to
boot from a network interface first and a hard disk second. This could be
used to run VMs that are meant to be a PXE booted machines.

```ruby
Vagrant.configure("2") do |config|
  config.vm.define :pxeclient do |pxeclient|
    pxeclient.vm.box = "centos64"
    pxeclient.vm.provider :libvirt do |domain|
      domain.boot 'network'
      domain.boot 'hd'
    end
  end

  # ...
```

## Networks

Networking features in the form of `config.vm.network` support private networks
concept. It supports both the virtual network switch routing types and the point to
point Guest OS to Guest OS setting using TCP tunnel interfaces.

http://wiki.libvirt.org/page/VirtualNetworking
https://libvirt.org/formatdomain.html#elementsNICSTCP

Public Network interfaces are currently implemented using the macvtap driver. The macvtap
driver is only available with the Linux Kernel version >= 2.6.24. See the following libvirt
documentation for the details of the macvtap usage.

http://www.libvirt.org/formatdomain.html#elementsNICSDirect


An examples of network interface definitions:

```ruby
  # Private network using virtual network switching
  config.vm.define :test_vm1 do |test_vm1|
    test_vm1.vm.network :private_network, :ip => "10.20.30.40"
  end

  # Private network. Point to Point between 2 Guest OS using a TCP tunnel
  # Guest 1
  config.vm.define :test_vm1 do |test_vm1|
    test_vm1.vm.network :private_network,
          :libvirt__tcp_tunnel_type => 'server',
          # default is 127.0.0.1 if omitted
          # :libvirt__tcp_tunnel_ip => '127.0.0.1',
          :libvirt__tcp_tunnel_port => '11111'

  # Guest 2
  config.vm.define :test_vm2 do |test_vm2|
    test_vm2.vm.network :private_network,
          :libvirt__tcp_tunnel_type => 'client',
          # default is 127.0.0.1 if omitted
          # :libvirt__tcp_tunnel_ip => '127.0.0.1',
          :libvirt__tcp_tunnel_port => '11111'


  # Public Network
  config.vm.define :test_vm1 do |test_vm1|
    test_vm1.vm.network :public_network,
          :dev => "virbr0",
          :mode => "bridge",
          :type => "bridge"
  end
```

In example below, one network interface is configured for VM test_vm1. After
you run `vagrant up`, VM will be accessible on IP address 10.20.30.40. So if
you install a web server via provisioner, you will be able to access your
testing server on http://10.20.30.40 URL. But beware that this address is
private to libvirt host only. It's not visible outside of the hypervisor box.

If network 10.20.30.0/24 doesn't exist, provider will create it. By default
created networks are NATed to outside world, so your VM will be able to connect
to the internet (if hypervisor can). And by default, DHCP is offering addresses
on newly created networks.

The second interface is created and bridged into the physical device 'eth0'.
This mechanism uses the macvtap Kernel driver and therefore does not require
an existing bridge device. This configuration assumes that DHCP and DNS services
are being provided by the public network. This public interface should be reachable
by anyone with access to the public network.

### Private Network Options

*Note: These options are not applicable to public network interfaces.*

There is a way to pass specific options for libvirt provider when using
`config.vm.network` to configure new network interface. Each parameter name
starts with 'libvirt__' string. Here is a list of those options:

* `:libvirt__network_name` - Name of libvirt network to connect to. By default,
  network 'default' is used.
* `:libvirt__netmask` - Used only together with `:ip` option. Default is
  '255.255.255.0'.
* `:libvirt__host_ip` - Adress to use for the host (not guest).
  Default is first possible address (after network address).
* `:libvirt__dhcp_enabled` - If DHCP will offer addresses, or not. Used only
  when creating new network. Default is true.
* `:libvirt__dhcp_start` - First address given out via DHCP.
  Default is third address in range (after network name and gateway).
* `:libvirt__dhcp_stop` - Last address given out via DHCP.
  Default is last possible address in range (before broadcast address).
* `:libvirt__dhcp_bootp_file` - The file to be used for the boot image.
  Used only when dhcp is enabled.
* `:libvirt__dhcp_bootp_server` - The server that runs the DHCP server.
  Used only when dhcp is enabled.By default is the same host that runs the DHCP server.
* `:libvirt__adapter` - Number specifiyng sequence number of interface.
* `:libvirt__forward_mode` - Specify one of `veryisolated`, `none`, `nat` or `route` options.
  This option is used only when creating new network. Mode `none` will create
  isolated network without NATing or routing outside. You will want to use
  NATed forwarding typically to reach networks outside of hypervisor. Routed
  forwarding is typically useful to reach other networks within hypervisor.
  `veryisolated` described [here](https://libvirt.org/formatnetwork.html#examplesNoGateway).
  By default, option `nat` is used.
* `:libvirt__forward_device` - Name of interface/device, where network should
  be forwarded (NATed or routed). Used only when creating new network. By
  default, all physical interfaces are used.
* `:libvirt_tcp_tunnel_type` - Set it to "server" or "client" to enable TCP
  tunnel interface configuration. This configuration type uses TCP tunnels to
  generate point to point connections between Guests. Useful for Switch VMs like
  Cumulus Linux. No virtual switch setting like "libvirt__network_name" applies with TCP
  tunnel interfaces and will be ignored if configured.
* `:libvirt_tcp_tunnel_ip` - Sets the source IP of the TCP Tunnel interface. By
  default this is `127.0.0.1`
* `:libvirt_tcp_tunnel_port` - Sets the TCP Tunnel interface port that either
  the client will connect to, or the server will listen on.
* `:mac` - MAC address for the interface.
* `:model_type` - parameter specifies the model of the network adapter when you create a domain value by default virtio KVM believe possible values, see the documentation for libvirt


When the option `:libvirt__dhcp_enabled` is to to 'false' it shouldn't matter
whether the virtual network contains a DHCP server or not and vagrant-libvirt
should not fail on it. The only situation where vagrant-libvirt should fail
is when DHCP is requested but isn't configured on a matching already existing
virtual network.

### Public Network Options
* `:dev` - Physical device that the public interface should use. Default is 'eth0'.
* `:mode` - The mode in which the public interface should operate in. Supported
  modes are available from the [libvirt documentation](http://www.libvirt.org/formatdomain.html#elementsNICSDirect).
  Default mode is 'bridge'.
* `:type` - is type of interface.(`<interface type="#{@type}">`)
* `:mac` - MAC address for the interface.
* `:ovs` - Support to connect to an open vSwitch bridge device. Default is 'false'.

### Management Network

Vagrant-libvirt uses a private network to perform some management operations
on VMs. All VMs will have an interface connected to this network and
an IP address dynamically assigned by libvirt. This is in addition to any
networks you configure. The name and address used by this network are
configurable at the provider level.

* `management_network_name` - Name of libvirt network to which all VMs will be connected. If not specified the default is 'vagrant-libvirt'.
* `management_network_address` - Address of network to which all VMs will be connected. Must include the address and subnet mask. If not specified the default is '192.168.121.0/24'.

You may wonder how vagrant-libvirt knows the IP address a VM received.
Libvirt doesn't provide a standard way to find out the IP address of a running
domain. But we do know the MAC address of the virtual machine's interface on
the management network. Libvirt is closely connected with dnsmasq, which acts as
a DHCP server. dnsmasq writes lease information in the `/var/lib/libvirt/dnsmasq`
directory. Vagrant-libvirt looks for the MAC address in this file and extracts
the corresponding IP address.

## Additional Disks

You can create and attach additional disks to a VM via `libvirt.storage :file`. It has a number of options:

* `path` - Location of the disk image. If unspecified, a path is automtically chosen in the same storage pool as the VMs primary disk.
* `device` - Name of the device node the disk image will have in the VM, e.g. *vdb*. If unspecified, the next available device is chosen.
* `size` - Size of the disk image. If unspecified, defaults to 10G.
* `type` - Type of disk image to create. Defaults to *qcow2*.
* `bus` - Type of bus to connect device to. Defaults to *virtio*.
* `cache` - Cache mode to use, e.g. `none`, `writeback`, `writethrough` (see the [libvirt documentation for possible values](http://libvirt.org/formatdomain.html#elementsDisks) or [here](https://www.suse.com/documentation/sles11/book_kvm/data/sect1_chapter_book_kvm.html) for a fuller explanation). Defaults to *default*.
* `allow_existing` - Set to true if you want to allow the VM to use a pre-existing disk.  This is useful for sharing disks between VMs, e.g. in order to simulate shared SAN storage. Shared disks removed only manualy.If not exists - will created. If exists - using existed.

The following example creates two additional disks.

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :libvirt do |libvirt|
    libvirt.storage :file, :size => '20G'
    libvirt.storage :file, :size => '40G', :type => 'raw'
  end
end
```

## CDROMs

You can attach up to four (4) CDROMs to a VM via `libvirt.storage :file, :device => :cdrom`. Available options are:

* `path` - The path to the iso to be used for the CDROM drive.
* `dev` - The device to use (`hda`, `hdb`, `hdc`, or `hdd`). This will be automatically determined if unspecified.
* `bus` - The bus to use for the CDROM drive. Defaults to `ide`

The following example creates three CDROM drives in the VM:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :libvirt do |libvirt|
    libvirt.storage :file, :device => :cdrom, :path => '/path/to/iso1.iso'
    libvirt.storage :file, :device => :cdrom, :path => '/path/to/iso2.iso'
    libvirt.storage :file, :device => :cdrom, :path => '/path/to/iso3.iso'
  end
end
```

## Input

You can specify multiple inputs to the VM via `libvirt.input`. Available options are
listed below. Note that both options are required:

* `type` - The type of the input
* `bus` - The bust of the input

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :libvirt do |libvirt|
    # this is the default
    # libvirt.input :type => "mouse", :bus => "ps2"

    # very useful when having mouse issues when viewing VM via VNC
    libvirt.input :type => "tablet", :bus => "usb"
  end
end
```

## SSH Access To VM

vagrant-libvirt supports vagrant's [standard ssh settings](https://docs.vagrantup.com/v2/vagrantfile/ssh_settings.html).

## Forwarded Ports

vagrant-libvirt supports Forwarded Ports via ssh port forwarding.  Please note that due to a well known limitation only the TCP protocol is supported.  For each `forwarded_port` directive you specify in your Vagrantfile, vagrant-libvirt will maintain an active ssh process for the lifetime of the VM.

vagrant-libvirt supports an additional `forwarded_port` option
`gateway_ports` which defaults to `false`, but can be set to `true` if
you want the forwarded port to be accessible from outside the Vagrant
host.  In this case you should also set the `host_ip` option to `'*'`
since it defaults to `'localhost'`.

## Synced Folders

vagrant-libvirt supports bidirectional synced folders via nfs or 9p and
unidirectional via rsync. The default is nfs. Vagrant automatically syncs
the project folder on the host to */vagrant* in the guest. You can also
configure additional synced folders.

You can change the synced folder type for */vagrant* by explicity configuring
it an setting the type, e.g.

    config.vm.synced_folder './', '/vagrant', type: 'rsync'

    or

    config.vm.synced_folder './', '/vagrant', type: '9p', disabled: false, accessmode: "squash", owner: "vagrant"

**SECURITY NOTE:** for remote libvirt, nfs synced folders requires a bridged public network interface and you must connect to libvirt via ssh.


## Customized Graphics

vagrant-libvirt supports customizing the display and video settings of the
managed guest.  This is probably most useful for VNC-type displays with multiple
guests.  It lets you specify the exact port for each guest to use deterministically.

Here is an example of using custom display options:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :libvirt do |libvirt|
    libvirt.graphics_port = 5901
    libvirt.graphics_ip = '0.0.0.0'
    libvirt.video_type = 'qxl'
  end
end
```

## Box Format

You can view an example box in the [example_box/directory](https://github.com/pradels/vagrant-libvirt/tree/master/example_box). That directory also contains instructions on how to build a box.

The box is a tarball containing:

* qcow2 image file named `box.img`.
* `metadata.json` file describing box image (provider, virtual_size, format).
* `Vagrantfile` that does default settings for the provider-specific configuration for this provider.

## Create Box
To create a vagrant-libvirt box from a qcow2 image, run `create_box.sh` (located in the tools directory):

```$ create_box.sh ubuntu14.qcow2```

You can also create a box by using [Packer](https://packer.io). Packer templates for use with vagrant-libvirt are available at https://github.com/jakobadam/packer-qemu-templates. After cloning that project you can build a vagrant-libvirt box by running:

``` ~/packer-qemu-templates/ubuntu$ packer build ubuntu-14.04-server-amd64-vagrant.json```

## Development

To work on the `vagrant-libvirt` plugin, clone this repository out, and use
[Bundler](http://gembundler.com) to get the dependencies:

```
$ git clone https://github.com/pradels/vagrant-libvirt.git
$ cd vagrant-libvirt
$ bundle install
```

Once you have the dependencies, verify the unit tests pass with `rake`:

```
$ bundle exec rake
```

If those pass, you're ready to start developing the plugin. You can test
the plugin without installing it into your Vagrant environment by just
creating a `Vagrantfile` in the top level of this directory (it is gitignored)
that uses it. Don't forget to add following line at the beginning of your
`Vagrantfile` while in development mode:

```ruby
Vagrant.require_plugin "vagrant-libvirt"
```

Now you can use bundler to execute Vagrant:

```
$ bundle exec vagrant up --provider=libvirt
```

IMPORTANT NOTE: bundle is crucial. You need to use bundled vagrant.

## Contributing

1. Fork it.
2. Create your feature branch (`git checkout -b my-new-feature`).
3. Commit your changes (`git commit -am 'Add some feature'`).
4. Push to the branch (`git push origin my-new-feature`).
5. Create new Pull Request.
