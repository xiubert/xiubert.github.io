---
permalink: /noiseSurfing/
title: "Noise Surfing"
toc: true
toc_label: "contents:"
last_modified_at: 6/1/2021
---

This here is concerned with surfing the noise that is the internet. Let's say you've got root shell access some place in the internet. The below are tools that make use of that for somewhat sub rosa surfing or surfing over hurdles and around obstacles.

# Surfing over hurdles and around obstacles

## Reverse tunneling
**OBJECTIVE**: To create a secure liaison so that you may talk to someone within an otherwise inaccessible castle and do so with relatively little overhead (eg. access a remote jupyter server that's behind a moat).

### Overview

- Castle port that the serf needs to access:  9999
- Gateway port used at liaison to go from serf to nobility:  4455
- Local serf port that is forwarded to liaison gateway:  7777
- Port at which liaison listens for SSH:  2233
- Username at liaison:  viaNoise
- liaison location: liaison.domain.org

1.  Nobility within the castle connects to the liaison: the castle port is remote forwarded to the liaison gateway port (`ssh -R` flag)
    ```
    ssh -N -R 4455:localhost:9999 viaNoise@liaison.domain.org -p 2233 -i /path/to/privateKey
    ```
2.  Serf outside the castle connects to the liaison:  the local serf port is forwarded to the liaison gateway port (`ssh -L` flag)
    ```
    ssh -N -L 7777:localhost:4455 viaNoise@liaison.domain.org -p 2233 -i /path/to/privateKey
    ```
3.  The serf outside the castle may now connect to the nobility port at 9999 via the local serf port 7777: `https://localhost:7777`


It ends there if serf requires access to solely one castle port.  If the serf needs access to other ports they may consider it too presumptuous to ask the nobility to make multiple remote forward connections.  

If the nobility can run an ssh daemon (openSSH server), the nobility can remote forward the ssh port to the liaison: 
```
ssh -N -R 4455:localhost:22 viaNoise@liaison.domain.org -p 2233 -i /path/to/privateKey
```

The serf must then locally forward the liaison gateway port corresponding to the ssh port within the castle:
```
ssh -N -L 7777:localhost:4455 viaNoise@liaison.domain.org -p 2233 -i /path/to/privateKey
```
Now the serf may forward a local port to whichever port they please within the castle via an ssh connection with the nobility via the liaison.  Let's say the serf would like their port 8888 to correspond to the nobility port 5555:
```
ssh -N -L 8888:localhost:5555 nobility@localhost -p 7777 -i /path/to/privateKey
```

**Note**: If the castle doesn't allow any outgoing ports other than HTTP and HTTPS (80 and 443) and the liaison hosts a webserver requiring 80 and 443, one solution would be to use openVPN on 443 on the liaison with port sharing.

### A few hardening notes

-  On liaison, create a user without shell access:

    ```
    # useradd -s /usr/bin/nologin viaNoise
    ```

   Also, create ssh key pairs for the serf and nobility (persons inside and outside castle) and push the public keys to the liaison (where `viaNoise` exists).

- Use `AllowUsers` in `/etc/ssh/sshd_config` to solely allow specific users to make ssh connections. Make sure viaNoise is listed there.

- Define user specific settings in `/etc/ssh/sshd_config` with `Match User`:
    ```
    Match User viaNoise
       AuthenticationMethods publickey
       AllowAgentForwarding no
       PermitTTY no
       #PermitOpen localhost:[relevant port]
       #ForceCommand echo 'this is only a liaison'
    ```
    The gateway ports open via forwarding to the liaison may be limited by listing them after `PermitOpen` and uncommenting that line.

    This may be especially worth considering within the castle (`/etc/ssh/sshd_config` at castle):
    ```
    Match User nobility
       AuthenticationMethods publickey
       AllowAgentForwarding no
       PermitTTY no
       PermitOpen localhost:9999 localhost:22 localhost:5555
    ```

### A few streamlining notes

- To make the ssh commands easier to remember, parameters may instead be listed in `~/.ssh/config`:

    `nobility@castle $ cat ~/.ssh/config`

    ```
    Host liaison
            Hostname liaison.domain.com
            Port 2233
            User viaNoise
            IdentityFile "/private/key/location/"
            RemoteForward 4455 localhost:9999
            ServerAliveInterval 240
            ServerAliveCountMax 2
    ```
    
    Nobility then issues:  `ssh -N liaison`

    
    `serf@village $ cat ~/.ssh/config`

    ```
    Host liaison
            Hostname liaison.domain.com
            Port 2233
            User viaNoise
            IdentityFile "/private/key/location/"
            LocalForward 7777 localhost:4455
            ServerAliveInterval 240
            ServerAliveCountMax 2
    ```
    
    serf then issues:  `ssh -N liaison`

- **NOTE**: this would require an open nobility shell at the castle to be running indefinitely if the serf wants anytime access. For stability and more function, one may instead wish to use a VPN.
    
## VPN (todo)
**OBJECTIVE**: 

- To again create a secure liaison so that you may talk to someone within an otherwise inaccessible castle

- To find your way over a prohibitive moat
- To surf in a reasonably sub rosa fashion (both web traffic and non [with a few exceptions])

   

# Somewhat sub rosa surfing

When you'd rather some or all of your traffic not be apparent to the local network. This is also useful if you're in a castle that prohibits your access to certain places outside the castle.

## Dynamic forwarding via SOCKS tunnel

**OBJECTIVE**: Securely route web traffic through a liaison outside your local network to circumvent a moat or sure up your banking traffic.

1. Configure your browser (easiest with Firefox):  Options>>General>>Network Settings

    Manual proxy configuration>
    
    SOCKS host: 127.0.0.1

    Port:  8080

    SOCKS v5

2. Route to SOCKSv5 port (`ssh -D` flag) with:
    ```
    ssh -C -N -D 8080 viaNoise@liaison.domain.org -p 2233 -i /path/to/privateKey
    ```

   or use `~/.ssh/config`
   ```
   Host liaisonSOCKS
      Hostname liaison.domain.org
      Port 2233
      User viaNoise
      IdentityFile /path/to/privateKey
      DynamicForward 8080
      Compression yes
      ServerAliveInterval 240
      ServerAliveCountMax 2

   ```
   and issue: `ssh -N liaisonSOCKS`

   googling "what's my ip address" should now show the public IP of the liaison

   **NOTE**:  While the tunnel is via 8080 (in this case), dynamic port numbers are used on the liaison.  Thus if you use the `PermitOpen` setting in `sshd_config` for the ssh destination user on the liaison, the connections will likely be 'administratively prohibited'.

## via VPN

todo