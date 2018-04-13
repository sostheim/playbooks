# Overview
This is a simple method for administering a users's access to a jump host behind a bastion in an AWS VPC.  The bastion host exists in a public subnet with an EIP/IGW access enabled.  Some number of jump hosts, 1 or more, exist in a private subnet and enable access to a VGW.  

This method never requires the private half of a key pair.  There is no need to expose it to being transferred to an administrator or risk having it compromised on an exploited server.

## Install an RSA Key on the Bastion and Jump Hosts
This process takes the public half of the RSA key pair only.  Be sure that you only install your public key!

To execute these steps you need to:
1. have a working Ansible environment \[[0](#reference-0)\].
2. have to have previously configured the bastion and jump hosts to have the vpn authorized user added. If you need to create the VPN user for the first see the steps below to [Create an Authorized User](#create-an-authorized-user)
3. have an RSA key pair, or a suitable alternative. If you need to create an RSA key first see the steps below to [Create an RSA Key](#create-your-rsa-key)
4. have physical network access and an administrative server (admin laptop/worksation) that has existing ssh access (for Ansible) to the bastion and jump hosts


Example:
```
# do all work in a temporary admin directory
$ mkdir tmp_admin
$ cd tmp_admin/
 
# note: intentionally overwrite any existing file of the same name
~/tmp_admin$ cat > install_id_rsa.pub <<EOF
> ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQClw+LbnhxmBnb+HmzNOK0fnpd+Ldn/3yfi463SFLtMK4Tsi9lxNXPfn9ECUh3na/Bk2wiubdMM0H/H4ZLl5fAkgkZozthQOGieVh7x5wTFxIC/0v8VoBjNHZu9j9POKo470q3kDAEBpQGM1VezQYqirJijLiJQh4dQnMur12+DIALuJMLT6pLDT79Q0sEcwdBXk2aLEKkh2I+v76ZKRXnbw3UMkRRekBAt247bzx0/kTfrtUVHXYxFRO6mY+ZlFISMSoGJmic53xwsWqFoYV9jVcFxUwFQ2tOOWgkPepPKr3g0N41UVxM8iW3Js2MLrZ+XLjt77bQJy6fQiGNK2K+n sostheim@rastop
> EOF
 
# run the ansible playbook to add the authorized key to the target hosts
~/tmp_admin$ ansible-playbook add_authorized_key.yml --inventory-file=./hosts --user=ubuntu
 
PLAY [all] *******************************************************************************************************************************************************************
 
TASK [Gathering Facts] *******************************************************************************************************************************************************
ok: [10.10.1.192]
ok: [10.7.128.254]
ok: [10.7.128.250]
 
TASK [add the public rsa key to the ftl user's authorized_keys file] *********************************************************************************************************
changed: [10.10.1.192]
changed: [10.7.128.254]
changed: [10.7.128.250]
 
PLAY RECAP *******************************************************************************************************************************************************************
10.7.128.250 : ok=2 changed=1 unreachable=0 failed=0
10.7.128.254 : ok=2 changed=1 unreachable=0 failed=0
10.1.10.192 : ok=2 changed=1 unreachable=0 failed=0
```
## Configure Your SSH Client
Back on your system, add/append the following entries to your ssh configuration file \[[1](#reference-1)\]\[[2](#reference-2)\].  You need to replace the value for the `IdentityFile` value with a path that reflects the private key half of the public key you sourced in the previous step.  The User name, here `ftl`, is required match the name of the authorized user on the bastion and jump hosts.

```
# Use this host for External Internet -> Bastion
Host my-bastion
  HostName my-bastion.example.com
  Port 22
  User ftl
  IdentityFile /Users/sostheim/.ssh/id_rsa_vpn
 
# Use this host for External Internet -> Bastion -> Private Instance 1 -> VGW -> VPN -> private network
Host jump-1
  HostName 10.7.128.254
  Port 22
  User ftl
  IdentityFile /Users/sostheim/.ssh/id_rsa_vpn
  ProxyCommand ssh my-bastion -W %h:%p
 
# Use this host for External Internet -> Bastion -> Private Instance 2 -> VGW -> VPN -> private network
Host jump-2
  HostName 10.7.128.250
  Port 22
  User ftl
  IdentityFile /Users/sostheim/.ssh/id_rsa_vpn
  ProxyCommand ssh my-bastion -W %h:%p
```
## Test Your Configuration
At this point, using the authorized user identity, you can ssh directly to either `jump-1` or `jump-2` that are instances that have VGW, VPN, private network access enabled. The user *can* access the bastion (although you shouldn't be able to get anywhere from the bastion host).  Only the jump hosts have access to the internal network.  There is no difference between the `jump-1` and `jump-2` hosts, they exist mainly for redundancy sake.

```
# ssh directly to jump host
$ ssh jump-1
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-1052-aws x86_64)
 
 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
 
  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud
 
0 packages can be updated.
0 updates are security updates.
 
 
Last login: Wed Apr  4 09:02:37 2018 from 10.7.1.184 
ftl@ip-10-7-128-254:~$ 

#
# ssh to an internal lab host (private network ip behind vpn)
# NOTE: requires password auth to avoid installing users private key here
#
ftl@ip-10-7-128-254:~$ ssh sostheim@192.168.48.9
sostheim@192.168.48.9's password:
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic x86_64)
 
 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
 
0 packages can be updated.
0 updates are security updates.
 
Last login: Wed Apr 4 11:35:03 2018 from 192.168.1.81
sostheim@labnode-9:~$
```

# Create An Authorized User
The bastion and jump hosts use a non-privileged user account to enable access.  The account used is the default from the playbook: `ftl`.  Regardless of the name you choose, this account has no special access, especially not sudo access.  It is a member to as few groups as possible, and has no privileges.

To execute this steps you need to:
1. Have a working Ansible environment \[[0](#reference-0)\].

```
# This step only needs to be executed once to setup a new bastion host, jump box, or both.

# Check administrator's ansible configuration (optional if you are sure this already works).
$ ansible all -m ping --inventory-file=./hosts -u ubuntu
10.1.10.192 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
10.7.128.254 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
10.7.128.250 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

# This adds the `ftl` user to a new bastion, jump server, or both.
~/tmp_admin$ ansible-playbook add_user.yml --inventory-file=./hosts --user=ubuntu
```

# Create Your RSA Key
To create a new key pair, it's generally a good idea to do so in a temporary directory so as not to inadvertently overwrite any existing keys, especially the default key ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub.  If you already have these files in place and don't mind reusing them for your VPN access, then you can do that in place of the next step as well.

```
# Create a new RSA key pair
# - When prompted for a passphrase and confirmation just hit <enter>
# - You can use any name that you like in place of  `id_rsa_vpn` (or accept the default value: id_rsa)
rastop:keys sostheim$ ssh-keygen -t rsa -f id_rsa_vpn
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in id_rsa_vpn.
Your public key has been saved in id_rsa_vpn.pub.
The key fingerprint is:
SHA256:ibfq9TSCqcqJH7+bmjA8A0cSvEvE/g+Gq2EBFn741no sostheim@rastop
The key's randomart image is:
+---[RSA 2048]----+
|+.               |
|.=o              |
|=+o.             |
|o*o .  . .       |
|o.=o .. S        |
|o+o+.  + .       |
|+=+.oEo + o      |
|.Bo=.+ o + .     |
|+.Bo*+o   .      |
+----[SHA256]-----+
 
# set the permissions and copy/move the new key to your local .ssh directory
# if yours are different, they should be set to be at least this restricted 
rastop:keys sostheim$ ls -l
total 16
-rw-------  1 sostheim  staff  1675 Apr  4 10:58 id_rsa_vpn
-rw-r--r--  1 sostheim  staff   397 Apr  4 10:58 id_rsa_vpn.pub
 
rastop:keys sostheim$ mv id_rsa_vpn* ~/.ssh/
 
# optional: display the contents of the public half of the key pair for use later
rastop:keys sostheim$ cat ~/.ssh/id_rsa_vpn.pub ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDGa7ZUqgJTOf58wM266rFjA2T4/CklJUEIiyiNcaosz1OujzKSEqIgF0igIU6crZnDuGqo8DG03tMh7kQaYezNJ4XTGFV6O+87yTBadvdG+aar2+x8MRtSVS2bPCQ/axAa6hc0pKcxp64hv7gnv2hj0SVbQdLQtDQArU/djGICrTr3YLnkDZhYiZjdgrhF9O+JfwAyl0VIuY6v3e0HWTvQUc1eKkS3skeH8PhyhnW/Ndl27pvZpKFJjZiC+00HTfPpYV51LNYWhI/MHHUbigQa+quqV1ni8NwYxAAgscNUh08AinIjtjUfcFe5D5nSS4xQDXcK27RwJ7Y4S5dOQlMX sostheim@rastop
```

## References
### Reference 0
Setting up Ansible for your environment is outside of the scope of this document.  For assistance refer to the [Ansible Installation Guide](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
### Reference 1 
ssh_config(5) - Linux man page
### Reference 2 
https://www.ssh.com/ssh/config/ 