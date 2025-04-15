# kubernetes-nginx-hardening
![Nginx](https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

### Hardened NGINX deployment strategies in Kubernetes using least privilege, capabilities, and non-root best practices.
> RND purpose only

# The Origin

The idea of this repo came into existence from a [LinkedIn post of Bibin Wilson](https://www.linkedin.com/posts/bibinwilson_devops-activity-7291012399187312640-EIB8/) and my comment there.

#### Will this pod run without any issues?

![k8s-challenge.png](/images/k8s-challenge.png "k8s-challenge")

![comment](/images/comment.png "my-comment")


# This repository contains
- 🛡️ NGINX with **minimal Process capabilities**
- 👤 NGINX running as **non-root** using **file capabilities**
- 🔒 NGINX as **non-root** without any capabilities (using unprivileged ports and pre-set file permissions)

# A Recap on Linux Capabilities

## 1. Process Capabilities
- Set on a process
- Set on the ENTRYPOINT in case of containers
- pid is 1 most of the cases

#### Commands:
##### `capsh` [Check shell capabilities]
```
capsh --print
capsh --print --has-p=cap_net_bind_service
```
- capsh shows the capabilities of the current process, which can differ — even if the file has them.
- This means if you run 'capsh --print' it will show capabilities of the current shell you are in, not of any other intended process.

#### Check capabilities of any process using  `status`
```
# Check current shell capabilities
grep Cap /proc/self/status
CapInh:	0000000000000000
CapPrm:	00000002a80425db
CapEff:	00000002a80425db
CapBnd:	00000002a80425db
CapAmb:	0000000000000000
```

```
# For process id 1
cat /proc/1/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000000000000400
CapEff:	0000000000000400
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
```
Then decode the desired Hexadecimal value
```
capsh --decode=0000000000000400
0x0000000000000400=cap_net_bind_service
```

##### `pscap`
```
pscap -a

ppid  pid   name        command           capabilities
0     1     root        nginx             chown, setgid, setuid, net_bind_service
```

### What are these fields?
- `CapInh` (Inheritable):
Capabilities a process can inherit across execve() if the target binary is inheritable-aware.
- `CapPrm` (Permitted):
The maximum set of capabilities a process can make effective or add to its inheritable set.
- `CapEff` (Effective):
The capabilities actively in use by the process — only these are actually enforced by the kernel.
- `CapBnd` (Bounding set):
The ceiling of capabilities a process can ever obtain — acts as a global limiter for the process.
- `CapAmb` (Ambient):
Capabilities retained across execve() if explicitly added — mostly relevant for non-root processes.

## 2. File Capabilities
- Set on a binary/executable file.
- Process should be started directly using the binary/executable
- This means not under the shell process like `sh -c ["nginx -g 'daemon off;'"]`

#### Commands
##### `setcap` and `getcap`
```
setcap 'cap_net_bind_service=+ep' /usr/sbin/nginx

getcap /usr/sbin/nginx
/usr/sbin/nginx cap_net_bind_service=ep
```

## 📁 Structure

```bash
.
├── using-process-capabilities-root
    ├── Dockerfile.nginx_debug       # Root-based NGINX with only required process capabilities with debug tools
    ├── pod.using-process-capabilities-root.yaml  # k8s manifest to run the image
    ├── pod.using-process-capabilities-root_debug.yaml  # k8s manifest to run the image having debug tools
    ├── pod.using-process-capabilities-root_debug_under_shell.yaml # k8s manifest to run the image under a shell process
├── using-file-capabilities-non-root
    ├── Dockerfile             # Non-Root NGINX with required file capabilities
    ├── Dockerfile_debug       # Non-Root NGINX with required file capabilities with debug tools
    ├── pod.using-file-capabilities-root_debug.yaml  # k8s manifest to run the image having debug tools
    ├── pod.using-file-capabilities-root_debug.yaml  # k8s manifest to run the image
└── without-capabilities-non-root
    ├── default.conf		   # port changed to 8080
    ├── Dockerfile                 # Non-Root NGINX with proper permissions and no capabilities
    ├── Dockerfile_debug           # Non-Root NGINX with debug tools
    ├── nginx.conf
    ├── pod.without-capabilities-non-root_debug.yaml      # k8s manifest to run the image having debug tools
    └── pod.without-capabilities-non-root.yaml		  # k8s manifest to run the image
```
# Running this project

## 🛡️ NGINX with **minimal Process capabilities**

```
git clone https://github.com/arindamgb/kubernetes-nginx-hardening.git
cd using-process-capabilities-root
```

```
docker build -t arindamgb/kubernetes-nginx-hardening:1.24.0-pcap-debug -f Dockerfile.nginx_debug .
docker push arindamgb/kubernetes-nginx-hardening:1.24.0-pcap-debug
k apply -f pod.using-process-capabilities-root_debug.yaml
```
Check the capabilites of the `shell` and the `nginx` process, it's the same.

```
# shell
k exec -it nginx-using-process-capabilities-root-debug -- capsh --print
Current: cap_chown,cap_setgid,cap_setuid,cap_net_bind_service=ep

```

```
# nginx process
k exec -it nginx-using-process-capabilities-root-debug -- pscap -a
ppid  pid   name        command           capabilities
0     1     root        nginx             chown, setgid, setuid, net_bind_service
```

## 👤 NGINX running as **non-root** using **file capabilities**

```
git clone https://github.com/arindamgb/kubernetes-nginx-hardening.git
cd using-file-capabilities-non-root
```
```
docker build -t arindamgb/kubernetes-nginx-hardening:1.24.0-fcap-debug -f Dockerfile_debug .
docker push arindamgb/kubernetes-nginx-hardening:1.24.0-fcap-debug
k apply -f pod.using-file-capabilities-non-root_debug.yaml
```
Note that the `shell` has no capabilies as running as non-root(`Current` is empty)

```
k exec -it nginx-using-file-capabilities-non-root-debug -- capsh --print
Current: =
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
<output-omitted>
```
But the binary file i.e. `nginx` has the capability as assigned through the `Dockerfile`
```
k exec -it nginx-using-file-capabilities-non-root-debug -- getcap /usr/sbin/nginx
/usr/sbin/nginx cap_net_bind_service=ep
```
So does the `nginx` process as it was invoked directly using the binary file. Runs as `pid 1`

```
k exec -it nginx-using-file-capabilities-non-root-debug -- cat /proc/1/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000000000000400
CapEff:	0000000000000400
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000

k exec -it nginx-using-file-capabilities-non-root-debug -- capsh --decode=0000000000000400
0x0000000000000400=cap_net_bind_service
```
Can be also checked using `pcap`

```
k exec -it nginx-using-file-capabilities-non-root-debug -- pscap -a
ppid  pid   name        command           capabilities
0     1     nginxuser   nginx             net_bind_service
1     21    nginxuser   nginx             net_bind_service
1     22    nginxuser   nginx             net_bind_service
```

## 🔒 NGINX as **non-root** without any capabilities

```
git clone https://github.com/arindamgb/kubernetes-nginx-hardening.git
cd without-capabilities-non-root
```

```
docker build -t arindamgb/kubernetes-nginx-hardening:1.24.0-nocap-debug -f Dockerfile_debug .
docker push arindamgb/kubernetes-nginx-hardening:1.24.0-nocap-debug
k apply -f without-capabilities-non-root.yaml
```

There are no `process` or `file` capabilities in play.

```
k exec -it nginx-without-capabilities-non-root-debug -- capsh --print
Current: =

k exec -it nginx-without-capabilities-non-root-debug -- cat /proc/1/status | grep Cap
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000

k exec -it nginx-without-capabilities-non-root-debug -- getcap /usr/sbin/nginx
k exec -it nginx-without-capabilities-non-root-debug -- pscap -a
```

> **Signing off, [Arindam Gustavo Biswas](https://www.linkedin.com/in/arindamgb/)**
>
> 16th April 2025, 04:00 AM
