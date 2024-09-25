# Security-Automation-with-Ansible

## Ansible Configuration

Ansible only needs to be installed on the control node. Managed hosts (remote systems) do not require Ansible installed, as communication between the control node and the managed hosts occurs over SSH.

#### Requirements for the Control Node:
- A valid RedHat Automation Platform subscription is required on the control node.
- The control node must have Ansible installed and configured, as it will manage all automation tasks across the infrastructure.

### Requirements for Managed Hosts:
- **Linux Managed Hosts:**
    - Must have Python 3.6+ or Python 2.7+ installed, as Ansible relies on Python for executing tasks.
    - Ensure that a user with administrative privileges (such as a user with sudo access) is available on the managed hosts to be used by Ansible for running tasks.
- **Windows Managed Hosts:**
    - Must have PowerShell 3.0 and .NET Framework 4.0+ installed to support Ansible modules that operate on Windows systems.
    - Ansible communicates with Windows hosts using WinRM (Windows Remote Management).
    
In all cases, the control node must have appropriate network access to the managed hosts for successful communication.

### Project Architecture
```     
                               Server A
                                 |
                                 |
Control Node -------- Switch ----- 
                                 |
                                 |
                               Server B

```
Create a new user 'automation' on the control node and set its password
```
[root@Host /]# useradd automation
[root@Host /]# passwd automation

```
SSH into servera from the control node to create the same user
```
[root@servera /]# ssh servera

[root@servera /]# useradd automation

[root@servera /]# passwd automation
```

Set administrative privileges to the user automation
```
[root@servera ~]# vim /etc/sudoers.d/automation
```

Provides the ability to run all commands as root without requiring any password. 
The NOPASSWD directive allows the user automation to run commands as root without being prompted for a password. This is essential for enabling Ansible to execute tasks on remote servers without requiring manual password input.
```
automation ALL=(ALL) NOPASSWD:ALL
```
Switch over to the automation user
```
[root@servera ~]# su automation

[automation@servera root]#  sudo systemctl status sshd
```
It should show **active (running)**

Exit the session
```
[automation@servera root]# exit
[root@servera ~]# exit
```

Repeat the same process for Server B:
```
[root@Host /]# ssh serverb
[root@serverb ~]# useradd automation
[root@serverb ~]# passwd automation
```

Set administrative privileges to the user automation
```
[root@serverb ~]# vim /etc/sudoers.d/automation
```
Provides the ability to run all commands as root without requiring any password
```
automation ALL=(ALL) NOPASSWD:ALL
```
Switch to automation user
```
[root@servera ~]# su automation

[automation@servera root]#  sudo systemctl status sshd
```
It should show **active (running)**

Exit the session
```
[automation@servera root]# exit
[root@servera ~]# exit
```

Check the sudoers files on the control node to see if the user automation can run administrative commands without any password
```
[root@Host /]# vim /etc/sudoers.d/automation
```
It should be like:
```
automation ALL=(ALL) NOPASSWD:ALL
```

```
[root@Host /]# su automation
[automation@Host /]# cd ~
```

Create ssh key. Do not use any password or passphrase for this key for now.
SSH Key Setup: SSH key-based authentication allows secure, passwordless logins between the control node and managed hosts. This is critical for automation tools like Ansible to run commands remotely without needing manual password input.
```
[automation@Host /]# ssh-key
[automation@Host /]# ssh-keygen
```
Send the key over to servera
```
[automation@Host /]# ssh-copy-id servera

[automation@Host /]# cd ~
[automation@Host /]# ssh-copy-id serverb
```
Let's see if key-based authentication works connecting to servera and then to serverb
```
[automation@Host /]# ssh servera
[automation@servera /]# exit

[automation@Host /]# ssh serverb
[automation@serverb /]# exit
```
We could connect to both servera and serverb without any password prompt.

#### Installing Ansible
**Installing Ansible:** Ansible can be installed using the package manager available on your Linux distribution. On a RedHat-based system, you can install it using yum:
```
[automation@Host /]# yum install ansible

[automation@Host /]# sudo cat /etc/hosts
127.0.0.1 localhost localhost.localadmin localhost4 localhost4.localdomain4
::1       localhost localhost.localadmin localhost4 localhost4.localdomain4
172.16.3.110 servera
172.16.3.110 serverb
```
An inventory file tells Ansible on which hosts tasks are done. It is necessary to have an inventory file or Ansible will not know on which hosts the task should be done. As you can see I have grouped servera and serverb under the catergory web servers.
```
[automation@Host /]# cat inventory
[webservers]
servera
serverb

[automation@Host /]# ansible webservers -m ping -i inventory
servera | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/isr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
serverb | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/isr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
```
## Ansible Inventories
Ansible inventories define the hosts or groups of hosts where commands, modules, and tasks in a playbook or ad-hoc command are executed. The inventory can specify individual hosts or logically grouped hosts, allowing for organized management of systems. This capability enables you to run specific tasks on targeted sets of machines.

By default, Ansible looks for the inventory file at **/etc/ansible/hosts**. However, you can create project-specific inventory files and store them in alternate locations. These custom inventory files can be referenced through the Ansible configuration file, included directly in a playbook, or passed using the -i option when running ad-hoc commands.

Ansible inventories can be written in either INI or YAML format, providing flexibility based on user preference or project requirements.

```yaml
#Inventory file

# You can list hosts by IP or hostname one per each line
192.168.0.1 
Example.com
servera

# You can group managed hosts by creating a group using "[]"
[group1]
Servera
Serverb


[group2]
Serverc
Serverd

# You can create a group of groups using the ":children" suffix and specifying the sub-groups
[group12:children]
group1
group2

# You can specify ranges in both hostnames or IP addresses. You can specify both numeric or alphabetic ranges. 
192.168.0.[1:255]
```
