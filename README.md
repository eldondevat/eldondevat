### Hi there!

I'm a seasoned software engineer and sysadmin, and you've found my github profile!

So far I've spent the last 15 years writing software and building infrastructure
for startups. I am a strong believer in the power of effective remote work, and
the creative and technical benefits of having strong human bonds on technical teams,
be they community-driven, open source, or business-based.


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
