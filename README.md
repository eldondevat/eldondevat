### Hi there!

I'm a seasoned software engineer, and you've found my github profile!

So far I've spent the last 15 years writing software and building infrastructure
for startups. I am a strong believer in the power of effective remote work, and
the creative and technical benefits of having strong human bonds on technical teams,
be they community-driven, open source, or business-based.


# FAQs 2024-05-31

> How do I set the hostname of a system fom the kernel command line?

Recently, I was using qemu to run some VMs, which were connected directly to IPv4, with no dhcp in place. It is possible to configure the network information via a parameter that is set on the kernel command line. Usually if you're using linux, the kernel command line is set by your bootloader, likely GRUB. In my case, I was using QEMU to run the kernel directly, which allows the specification of the command like via a kernel argument, something like this:

`qemu-system-x86_64 -machine accel=kvm -cpu host,vmx=on -m 12G -smp 2 -kernel flatcar_production_image.vmlinuz -initrd flatcar_production_pxe_image.cpio.gz -append flatcar.autologin=ttyS0 console=ttyS0 ip=192.168.0148::192.168.0145:255.255.255.0:k3s:eth0:none:1.1.1.1 rootfstype=btrfs sshkey="'"$(cat .ssh/id_rsa.pub)"'"' -nographic`

I was slightly surprised that the resulting system was using 'localhost' as a hostname, when the client hostname is specified as a parameter in the network configuration:

` ip=192.168.0148::192.168.0145:255.255.255.0:k3s:eth0:none:1.1.1.1`

I went digging and found that the only hostname systemd will extract from the commandline is one specified with systemd.hostname, like `systemd.hostname=k3s`. An easy addition, but I think that adding the possibility to utilize the hostname set in the ip argument might be interesting as a fallback. Of course there is always the possibility for surprising resullts for people with different usecases, so it's worth considering such a change very thoughtfully.

# FAQs 2024-05-30

> How do I boot QEMU to test an IPXE setup?

I like to build an EFI usb, and test via this command:

` qemu-system-x86_64 -machine accel=kvm -bios bin-x86_64-efi/OVMF_CODE.fd  -hda bin-x86_64-efi/ipxe.usb  -monitor stdio -m 4g  `


<!--
**eldondevat/eldondevat** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
