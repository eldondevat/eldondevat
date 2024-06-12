### Hi there!

I'm a seasoned software engineer and sysadmin, and you've found my github profile!

So far I've spent the last 15 years writing software and building infrastructure
for startups. I am a strong believer in the power of effective remote work, and
the creative and technical benefits of having strong human bonds on technical teams,
be they community-driven, open source, or business-based.

# FAQs 2024-06-11

> How do I enable/disable autologin for a tty?

If you're using a linux machine, you are likely using either systemd, or
possibly rc-init. Some systems enable autologin in a specific TTY, or allow
configuration to do so. A good example in Flatcar Linux. If deploying to
baremetal, or experimenting in a VM, there may be some scenarios where you want
to be able to administer the system in the short term without configuring SSH
up front, or even enabling networking.

Another scenario might be persistent VM infrastructure on a laptop. Usually your
primary OS will be protected by authentication, with encrypted partitions, etc. In
this scenario, enabling authentication on an internal VM is likely overkill from
a security perspective, and can be an annoying unnecessary step in your workflow.

It's possible to enable autologin with `agetty` using the `-a <username>` flag, or with
`login` with `login -f <username>`. Of course, things like gdm also have pam module configurations
to enable login-by-default.

I would expect most user-facing linux systems today (except android) to utilize systemd,
in which case you probably have the getty@ service, which can be edited with `systemctl edit getty@.service` for
any tty, or for a specific tty by putting it after the @ sign (ie, `systemctl edit getty@tty3.service`. This doesn't
edit the underlying file, but instead is a helper method for generating override.conf files
in `/etc/systemd/system`. After editing the file, it will be necessary to call `systemctl daemon-reload` and restart the service.

The specific use case that triggered this FAQ was specifying autologin in the
command line for a particular flatcar linux system booted over ipxe.  Out of
caution/paranoia, I wanted to disable login after setting up (manually,
unfortunately) the network on the system. The override file I generated looks like this, and after
doing the reload dance, the terminal was no longer automatically logging in:
```
#/etc/systemd/system/getty\@tty1.service.d/override.conf 
ExecStart=
ExecStart=-/sbin/agetty --noclear %I $TERM
```

I do have some non-systemd systems, and when login is managed by `rc-init`, the
tty login behavior will likely be managed in the file `/etc/inittab`. Here is
an exaple of agetty managing the second tty2 for console 2:
```
c2:2345:respawn:/sbin/agetty 38400 tty2 linux
```

To test modifying this behavior, change the file and issue the `telinit q` command. Then
switch to the console (ctrl+alt+f2 for me), and you will see the change (autologin or whatever).

One useful application of this is USB serial device access to a system with problematic networking,
which I hope to follow up on in a subsequent FAQ.


# Triplog 2024-06-03 Ubuntu

Recently I upgraded a machine from Ubuntu release 20.04 to release 22.04. This
install was migrated to this laptop, where it lives alongsied another Ubuntu
22.04 release. The core reason (in addition to receiving better regular
updates,i) was some stuttering that I experienced on google meet calls. It
seemed less prevalent on zoom, but who knows if that was accurate, or just luck
of the draw. In the end I switched several things to try to get up to date on
my Thinkpad P1 Gen3 laptop.

First, and the biggest surprise: despite having run through a
`do-release-upgrade`, I was still on a 5.15 kernel! I had to uninstall the old
hardware enablement package and install the new kernel 6.5 hardware enablement
package.

Second, adding proprietary NVIDIA drivers. The nouveau driver seems to be
better on the 6.X line of kernels, but I was still experiencing occasional
artifacts. The NVIDIA drivers seem to have resolved that, but this laptop also
has an intel graphics card for when battery life is more important than
performance. I need to verify that the appropriate graphics card is being
utilized properly, when I was checking the resources in use, it seems that there
are two possible acceleration APIs, VDPAU and VA-API.

I will test drive this setup for a few meetings (video calls seem to be the
only things that suffer, and primarily when I am sharing video). However, if I
don't see improvements, I may need to consider switching to an alternative device
specifically for these sorts of video calls.

# FAQs 2024-06-02

> How do I enable TLS on my k3s cluster without certmanager?

CertManager is a great utility, but it's not necessary to install it to get letsencrypt certificates in k3s. Traefik, which provides ingress capabilities for k3s, is perfectly capable of negotiating
`acme` for certificates for you. Additional configuration can be provided to k3s by dropping the following in your installation (with your own email):

```
$ cat /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    logs:
      access:
        enabled: true
    certResolvers:
      letsencrypt:
        email: "$(certificate email)"
        caServer: https://acme-v02.api.letsencrypt.org/directory # Production server
        tlsChallenge: true
```

Then, using the defined certResolver on traefik ingresses (be sure to specify both FQDNs!):

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt
    traefik.ingress.kubernetes.io/router.tls.domains.0.main: $(your fqdn)
spec:
  rules:
  - host: $(your fqdn)
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-k8s
            port:
              number: 9090

```

Note the certresolver in the ingress annotation is a reference to the certresolver defined in the HelmChartConfig (the token
being letsencrypt in this case).

Also, notice that this enables access logs for the traefik proxies.

Is this easier than installing CertManager? Not sure, but it's definitely one less moving piece if you
don't need fullblown certificate management, but just want to expose a few websites.


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
