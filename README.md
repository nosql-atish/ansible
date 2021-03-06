Ansible
=========

Setup servers with Ansible like OS Hardening and base packages installation  

First we installed OS, then do OS hardening and package installation etc., this is tradition way of doing, now let's see how to do it by simplest way using Ansible. 


##What is Ansible ?


Ansible is a radically simple IT automation platform that makes your applications and systems easier to deploy. Avoid writing scripts or custom code to deploy and update your applications— automate in a language that approaches plain English, using SSH, with no agents to install on remote systems   

##Goal 

We will install ansible on your local machine and setup following things in remote machine 

  - 1. OS Hardening 
  - 2. Install base packages 


Install Ansible on local machine

```
sudo apt-add-repository -y ppa:ansible/ansible
sudo apt-get update
sudo apt-get install -y ansible
```

##Creating Playbook 

We will separate our task using 3 different playbook 

  - 1. os_hardening.yml  :  All things related to OS hardening 
  - 2. base_package.yml :  Install or compile required packages
  - 3. main.yml              :  above both task include in main 

Also there will be one single file to manage global variables, like in future if we need to change version of any application or links etc. 

Now create one dir and cd into it then create yml files

```
mkdir deploy-nodes && cd deploy-nodes
```

##Create our yml file

```
touch vars.yml main.yml base_packages.yml ansible_hosts os_hardening.yml 
```

So our final structure as below :

```
|-- vars.yml
|-- main.yml
|-- ansible_hosts
|-- base_packages.yml
`-- os_hardening.yml
```

Now let's configure the os_hardening.yml

There are many things you can do to secure your OS but for learning ansible, we will do basic things, like disable root user in ssh and adding MaxAuthTries 3

Let's first defined ssh related variables in vars.yml file

Edit "vars.yml" and configure as below :

```
---
sshd_config: '/etc/ssh/sshd_config'
```


Configure the os_hardening.yml

```
---
  - name: SSHD# Disable Root login
    lineinfile:
        backup=yes
        state=present
        dest={{ sshd_config }}
        regexp='^PermitRootLogin'
        line='PermitRootLogin no'

  - name: SSHD# Updating MaxAuthTries to 3
    lineinfile:
        backup=yes
        state=present
        dest={{ sshd_config }}
        regexp='^MaxAuthTries' 
        line='MaxAuthTries 3'

  - name: SSHD# Restarting ssh service
    service:
      name=ssh
      state=restarted
```

###Explanation :

   - 1. First it will take backup destination file then search line starting with "PermitRootLogin" and replace with "PermitRootLogin no" 
   - 2. Again search and replace but if search not found then it will add "MaxAuthTries 3"
   - 3. Finally it will restart ssh to affect the changes. 

Configure base_package.yml

```
---
  - name: Install list of packages
    action:
       apt
       update_cache=yes
       cache_valid_time=600
       pkg={{item}}
       state=installed
    with_items:
    - unzip
    - build-essential
    - openssl
    sudo: true
    when:
       ansible_distribution == 'Debian' or
       ansible_distribution == 'Ubuntu'
```

##Explanation :

It will first do apt-get update if it is not run last 10 min ( cache_valid_time=600 ), then it will install mention packages. 

Configure main.yml file

```
---
  - hosts: all
    vars_files:
       - vars.yml
    sudo: true
    tasks:
       - include: os_hardening.yml
       - include: base_packages.yml
```

Add your hosts to ansible_hosts file 

```
[servers]
remote-ip-address ansible_ssh_user=remote-user ansible_sudo_pass=password ansible_ssh_pass=password
```

Note: if there is any special character in your password then you need to escape it Ex. if password is: p@ssw1rd then use: p\@ssw1rd

Now let's run the ansible command to setup remote host

```
ansible-playbook -vvv -i ansible_hosts main.yml
```

If you face "ssh fingerprint" issue  then do following thing and run above command again:

```
sudo sed -i.bkp 's/^#host_key_checking = False/host_key_checking = False/' /etc/ansible/ansible.cfg
```


