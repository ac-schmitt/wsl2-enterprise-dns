# wsl2-enterprise-dns

How to get the wsl2 dns working in an enterprise environment with localPolicyMerge disabled on the windows firewall.

## Issue

Resolving names has been an issue for enterprise users since WSL1 where the resolv.conf was regularly filled with den Windows DNS config. And in difference to your plastic router private DNS some enterprise grade DNS Servers will only resolve their private records but not public records. And linux just tries the first entry in the resolv.conf to get an answer **if** the nameserver is responding.

WSL2 gets around that issue by using a nameserver within the hyper-v network it creates. But that built-in hyper-v switch is public, so your classic enterprise firewall rule will not allow a policy merge for public interfaces.

Result: name resolution will stop working. :(

## Workaround

Using the nameservers directly from within WSL2 will work, though. So having a custom resolv.conf again.

As multiple nameserver will be surely involved in big corp, lets also use dnsmasq to create a simple multi-nameserver environment that'll work regardless wether you are trying to resolv some internet stuff or some internal ressources.

## Manual steps

### Disable autogeneration of the resolv.conf

Add

```
[network]
generateResolvConf = false
```

to `/etc/wsl.conf` or create a new `/etc/wsl.conf` with only these lines.

I guess restarting your wsl-linux might be necessary to make this setting persistent. Don't know how? Rebooting your box will also do the job.

### Get your nameservers

Check out the script [Get-Nameservers.ps1](Get-Nameservers.ps1). You can run it from your linux shell like this:

```
alexander@DESKTOP  powershell.exe $(wslpath -w Get-Nameservers.ps1)
nameserver 10.xxx.0.1
nameserver 10.xxx.0.2
nameserver 192.168.178.1
nameserver 8.8.8.8
```

This gives you your nameservers in the format needed for `/etc/resolv.conf`. Take any one nameserver that is able to resolv public internet names.

Or simply just use a well known public nameserver like `nameserver 8.8.8.8` - the google one. 

:warning: But beware. If the script is located in the wsl filesystem (which you should do performance and otherwise) you'll probably get an error that the script cannot be executed. Remember, we are talking about big corp enterprise setups here, and usally unsigned scripts from remote hosts are not allowed on your box. Put the script on one of your windows drives and it'll work.

### Install dnsmasq

Equiped with a nameserver that'll resolv public dns records you should probably update your linux subsystem first.

:warning: This is the first draft of this description and I'll provide the necessary steps here for an alpine linux. There are some issues for more complexe environments like Ubuntu as you need to use sudo a lot and dnsmasq being a "service" that'll simply not automatically run when you start your environment. Probably I'll describe the workarounds for something like Ubuntu in a later release of this guide.

Install dnsmasq using your package manager. For Alpine this is `apk add dnsmasq`

Add your configuration to the server section in `/etc/dnsmasq.conf`:

```
# Add other name servers here, with domain specs if they are for
# non-public domains.
#server=/localnet/192.168.0.1
server=8.8.8.8
server=/fritz.box/192.168.178.1
```

For my example I'll continue using the google namesserver and add the nameserver for the plastic router here as well. You'll have to adapt that to your enterprise environment. So usually there must be an entry like `server=/mycompany.example/10.xxx.y.z`

Make sure to disable DHCP and TFTP for dnsmasq by uncommenting the following line:

```
# If you want dnsmasq to provide only DNS service on an interface,
# configure it as shown above, and then use the following line to
# disable DHCP and TFTP on it.
no-dhcp-interface=
```

:warning: The following might be a bad hack.

Exit and shutdown the wsl distro. In my case that'll be `wsl --shutdown Alpine`

Start your wsl distro again. Yep, for me that'll be `wsl -d Alpine`

The resolv.conf is replaced by a link to the regular dnsmasq setup. Remove the link with `rm /etc/resolv.conf` and create a new `resolv.conf` using this command:

```
echo "nameserver 127.0.0.1" >> /etc/resolv.conf
```

You may add `dnsmasq` as command to your `.profile` or different file for your command interpreter to launch the service at startup.

## Wrap-Up

You can check if its working either by accessing your services or using something like `dig`.

```
DESKTOP:~# dig dl-cdn.alpinelinux.org +short
dualstack.d.sni.global.fastly.net.
151.101.2.133
151.101.66.133
151.101.130.133
151.101.194.133

DESKTOP:~# dig shurrock.fritz.box +short
192.168.178.121
```

The sample configuration files used in this readme can be found in the folder [sample-config](sample-config) in this repository.

## Todo

- create a python script that does all of this automatically
