# Description

Makes `getifaddrs()` work again inside proot-distro, which fixes a lot of programs.

Like fixing Home-Assistant, Node.js, Python (ifaddr, psutil, etc.), JupyterLab, etc.

<br>

# What It Does
User apps on Android have limited permissions. Moreover, Android has it's own implementation for `getifaddrs()` in Bionic LibC that considers these limited capabilities.

This result in programs compiled for Android Bionic LibC to still be able to work.

But, in proot-distro, it's either GNU LibC, Musl LibC, etc.

These LibC implementations do not consider the limited permissions on Android, and thus programs crash when they call `getifaddrs()`.

To overcome this, we'll use `LD_PRELOAD` to override the incompatible `getifaddr()` and make it conform to Android standards.

Like this, programs will now call the Android compatible `getifaddrs()` and work.

Thanks to DeepSeek, which created the LD_PRELOAD.

<br>

### Here are some relevant issues it fixes

https://github.com/termux/proot/issues/248

https://community.home-assistant.io/t/simple-and-fast-installing-home-assistant-core-and-matter-server-on-android-no-root-no-qemu/788933/11

https://www.reddit.com/r/LinuxOnAndroid/comments/1fuanv2/linux_on_android_running_spyder_ide_on_nomone/

https://www.reddit.com/r/termux/comments/143y69f/patching_getifaddrs_permission_denied/

https://www.reddit.com/r/termux/comments/1kopovl/comment/msuij85/

https://github.com/termux/proot-distro/issues/438

https://discourse.ros.org/t/discussion-ros2-on-mobile-devices/15289/30

<br>

# Install

Download Termux from F-Droid and open it -

https://f-droid.org/repo/com.termux_1021.apk

Copy-Paste and enter -

```
apt update && apt install -y proot-distro
```

Then according to the proot-distro chosen, Copy-Paste and enter -

<br>

### Ubuntu/Debian

```
proot-distro install ubuntu
```

```
proot-distro login --isolated ubuntu -- bash -c ' \
    export DEBIAN_FRONTEND=noninteractive && \
    apt update && apt upgrade -y && \
    apt install -y dialog apt-utils && \
    apt install -y --no-install-recommends gcc libc6-dev python3 python3-pip patch ca-certificates curl sudo && \
    mkdir -p /.libnetstub && \
    curl -sL -o /.libnetstub/libnetstub.sh "https://raw.githubusercontent.com/George-Seven/Termux-Proot-Utils/refs/heads/main/libnetstub.sh" && \
    echo ". /.libnetstub/libnetstub.sh" | tee -a ~/.bashrc ~/.zshrc >/dev/null \
'
```

That's done. Now, exit and login again to reflect the changes.

<br>

### Alpine

```
proot-distro install alpine
```

```
proot-distro login --isolated alpine -- ash -c ' \
    apk upgrade && \
    apk add gcc musl-dev python3  py3-pip python3-dev linux-headers patch ca-certificates curl sudo && \
    mkdir -p /.libnetstub && \
    curl -sL -o /.libnetstub/libnetstub.sh "https://raw.githubusercontent.com/George-Seven/Termux-Proot-Utils/refs/heads/main/libnetstub.sh" && \
    echo "ENV=~/.rc" >> ~/.profile && \
    echo ". /.libnetstub/libnetstub.sh" | tee -a ~/.rc ~/.bashrc ~/.zshrc >/dev/null \
'
```

That's done. Now, exit and login again to reflect the changes.

<br>

# Comparison

To check if it's working, enter one proot-distro -

```
proot-distro login --isolated ubuntu
```

<br>

## Before

This is how it looks without the fix. Many programs crashes -

```
root@localhost:~# python3 -c "import psutil, socket; [print(f'{iface}: {[addr.address for addr in psutil.net_if_addrs()[iface] if addr.family in (socket.AF_INET, socket.AF_INET6)]}') for iface in psutil.net_if_addrs()]"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/usr/local/lib/python3.12/dist-packages/psutil/__init__.py", line 2215, in net_if_addrs
    rawlist = _psplatform.net_if_addrs()
              ^^^^^^^^^^^^^^^^^^^^^^^^^^
PermissionError: [Errno 13] Permission denied
```

```
root@localhost:~# python3 -c "import ifaddr, socket; [print(f'{adapter.name}: {[ip.ip for ip in adapter.ips if ip.is_IPv4 or ip.is_IPv6]}') for adapter in ifaddr.get_adapters()]"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/usr/local/lib/python3.12/dist-packages/ifaddr/_posix.py", line 56, in get_adapters
    raise OSError(eno, os.strerror(eno))
PermissionError: [Errno 13] Permission denied
```

```
root@localhost:~# node -e "console.log(require('os').networkInterfaces())"
node:os:68
      throw new ERR_SYSTEM_ERROR(ctx);
      ^

SystemError [ERR_SYSTEM_ERROR]: A system error occurred: uv_interface_addresses returned Unknown system error 13 (Unknown system error 13)
    at Object.networkInterfaces (node:os:277:16)
    at [eval]:1:27
    at Script.runInThisContext (node:vm:122:12)
    at Object.runInThisContext (node:vm:298:38)
    at node:internal/process/execution:82:21
    at [eval]-wrapper:6:24
    at runScript (node:internal/process/execution:81:62)
    at evalScript (node:internal/process/execution:103:10)
    at node:internal/main/eval_string:30:3 {
  code: 'ERR_SYSTEM_ERROR',
  info: {
    errno: 13,
    code: 'Unknown system error 13',
    message: 'Unknown system error 13',
    syscall: 'uv_interface_addresses'
  },
  errno: [Getter/Setter],
  syscall: [Getter/Setter]
}
```

<br>

## After

This is after the fix -

```
root@localhost:~# python3 -c "import psutil, socket; [print(f'{iface}: {[addr.address for addr in psutil.net_if_addrs()[iface] if addr.family in (socket.AF_INET, socket.AF_INET6)]}') for iface in psutil.net_if_addrs()]"
wlan2: ['192.168.39.30', 'fe80::XXXX:XXXX:fe9c:151a%28']
wlan0: ['192.168.29.211', 'fe80::XXXX:XXXX:fe69:6db1%24', '2405:201:a801:2134:XXXX:XXXX:fe69:6db1', '2405:201:a801:2134:XXXX:XXXX:XXXX:1fa']
lo: ['127.0.0.1', '::1']
r_rmnet_data0: ['fe80::XXXX:XXXX:fe6f:14aa%20']
rmnet_data1: ['fe80::XXXX:XXXX:fe10:422f%18', '2401:4900:3e0d:e88a:XXXX:XXXX:fe10:422f']
rmnet_data0: ['fe80::XXXX:XXXX:fe94:97da%17']
dummy0: ['fe80::XXXX:XXXX:fee9:672a%2']
```

```
root@localhost:~# python3 -c "import ifaddr, socket; [print(f'{adapter.name}: {[ip.ip for ip in adapter.ips if ip.is_IPv4 or ip.is_IPv6]}') for adapter in ifaddr.get_adapters()]"
wlan2: [('fe80::XXXX:XXXX:fe9c:151a', 0, 28), '192.168.39.30']
wlan0: [('fe80::XXXX:XXXX:fe69:6db1', 0, 24), ('2405:201:a801:2134:XXXX:XXXX:fe69:6db1', 0, 0), ('2405:201:a801:2134:XXXX:XXXX:XXXX:1fa', 0, 0), '192.168.29.211']
r_rmnet_data0: [('fe80::XXXX:XXXX:fe6f:14aa', 0, 20)]
rmnet_data1: [('fe80::XXXX:XXXX:fe10:422f', 0, 18), ('2401:4900:3e0d:e88a:XXXX:XXXX:fe10:422f', 0, 0)]
rmnet_data0: [('fe80::XXXX:XXXX:fe94:97da', 0, 17)]
dummy0: [('fe80::XXXX:XXXX:fee9:672a', 0, 2)]
lo: [('::1', 0, 0), '127.0.0.1']
```

```
root@localhost:~# node -e "console.log(require('os').networkInterfaces())"
{
  wlan2: [
    {
      address: 'fe80::XXXX:XXXX:fe9c:151a',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: 'fe80::XXXX:XXXX:fe9c:151a/64',
      scopeid: 28
    },
    {
      address: '192.168.39.30',
      netmask: '255.255.255.0',
      family: 'IPv4',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: '192.168.39.30/24'
    }
  ],
  wlan0: [
    {
      address: 'fe80::XXXX:XXXX:fe69:6db1',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: 'fe80::XXXX:XXXX:fe69:6db1/64',
      scopeid: 24
    },
    {
      address: '2405:201:a801:2134:XXXX:XXXX:fe69:6db1',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: '2405:201:a801:2134:XXXX:XXXX:fe69:6db1/64',
      scopeid: 0
    },
    {
      address: '2405:201:a801:2134:XXXX:XXXX:XXXX:1fa',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: '2405:201:a801:2134:XXXX:XXXX:XXXX:1fa/64',
      scopeid: 0
    },
    {
      address: '192.168.29.211',
      netmask: '255.255.255.0',
      family: 'IPv4',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: '192.168.29.211/24'
    }
  ],
  r_rmnet_data0: [
    {
      address: 'fe80::XXXX:XXXX:fe6f:14aa',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: 'fe80::XXXX:XXXX:fe6f:14aa/64',
      scopeid: 20
    }
  ],
  rmnet_data1: [
    {
      address: 'fe80::XXXX:XXXX:fe10:422f',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: 'fe80::XXXX:XXXX:fe10:422f/64',
      scopeid: 18
    },
    {
      address: '2401:4900:3e0d:e88a:XXXX:XXXX:fe10:422f',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: '2401:4900:3e0d:e88a:XXXX:XXXX:fe10:422f/64',
      scopeid: 0
    }
  ],
  rmnet_data0: [
    {
      address: 'fe80::XXXX:XXXX:fe94:97da',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: 'fe80::XXXX:XXXX:fe94:97da/64',
      scopeid: 17
    }
  ],
  dummy0: [
    {
      address: 'fe80::XXXX:XXXX:fee9:672a',
      netmask: 'ffff:ffff:ffff:ffff::',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: false,
      cidr: 'fe80::XXXX:XXXX:fee9:672a/64',
      scopeid: 2
    }
  ],
  lo: [
    {
      address: '::1',
      netmask: 'ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff',
      family: 'IPv6',
      mac: '00:00:00:00:00:00',
      internal: true,
      cidr: '::1/128',
      scopeid: 0
    },
    {
      address: '127.0.0.1',
      netmask: '255.0.0.0',
      family: 'IPv4',
      mac: '00:00:00:00:00:00',
      internal: true,
      cidr: '127.0.0.1/8'
    }
  ]
}
```

<br>

###  Notes

The above are examples of interactive sessions, where you login to the proot-distro and manually "type" the commands. The shell automatically sources the script and sets LD_PRELOAD in interactive sessions, so everything is set to work by default.

For non-interactive sessions, you need to set the path to source the script, as the shell will not do it.

Example of non-interactive session -

```
proot-distro login --isolated ubuntu -- bash -c " \
    . /.libnetstub/libnetstub.sh && \
    python3 -c \"import psutil, socket; [print(f'{iface}: {[addr.address for addr in psutil.net_if_addrs()[iface] if addr.family in (socket.AF_INET, socket.AF_INET6)]}') for iface in psutil.net_if_addrs()]\" \
"
```

<br>
