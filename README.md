# Security-Automation-with-Ansible

# Table of Contents
1. [Installation](#installation)
2. [Inventories](#inventories)
3. [Configuration Files](#configuration-files)
4. [Ansible Ad-hoc Commands](#ansible-ad-hoc-commands)
5. [Ansible Playbooks](#ansible-playbooks)

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

## Installation
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
## Inventories
Ansible inventories define the hosts or groups of hosts where commands, modules, and tasks in a playbook or ad-hoc command are executed. The inventory can specify individual hosts or logically grouped hosts, allowing for organized management of systems. This capability enables you to run specific tasks on targeted sets of machines.

By default, Ansible looks for the inventory file at **/etc/ansible/hosts**. However, you can create project-specific inventory files and store them in alternate locations. These custom inventory files can be referenced through the Ansible configuration file, included directly in a playbook, or passed using the -i option when running ad-hoc commands.

Ansible inventories can be written in either INI or YAML format, providing flexibility based on user preference or project requirements.

**Example:**
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

**Demo**

To create an inventory file in Ansible, use the following command to edit or create the file:
```
vim inventory
```
The content of the inventory file should look like this:
```
servera
[group1]
serverb
[group2]
servera
[group12:children]
group1
group2
```
This defines the following groups and hosts:

servera and serverb are individual hosts.
group1 contains serverb.
group2 contains servera.
group12 is a group of groups, containing both group1 and group2.

### Listing Hosts in Inventory
Run the following command

**List All Hosts**

To list all the hosts defined in the inventory, run the following command:
```
ansible all --list-hosts -i inventory
```
*Response:*

    hosts (2):

        serverb
        servera

This lists all the hosts (serverb and servera) defined in the inventory.

**List Ungrouped Hosts**

To list hosts that are not assigned to any group:
```
ansible ungrouped --list-hosts -i inventory
```
*Response:*

    hosts (0):

There are no ungrouped hosts because all the hosts (servera and serverb) are part of groups (either group1, group2, or group12).

### **List Hosts in Specific Groups**

List Hosts in `group1`
```
ansible group1 --list-host -i inventory
```
*Response*

    hosts (1):
        serverb

List Hosts in `group2`

```
ansible group2 --list-hosts -i inventory
```

*Response:*

    hosts (1):
    servera

List Hosts in group12 (Children of group1 and group2)

```
ansible group12 --list-hosts -i inventory
```

Response:

    hosts (1):
    serverb
    servera

`group12` includes both `serverb` (from `group1`) and servera (from `group2`).

### **Pinging Hosts in Inventory**
You can use the ping module in Ansible to verify connectivity with the hosts in the inventory.

**Ping All Hosts**

To ping all the hosts:
```
ansible all -m ping -i inventory
```
This will ping all the hosts listed in the inventory file.

### **Ping Hosts in Specific Groups**
Ping Hosts in `group1`
```
ansible group1 -m ping -i inventory
```
*Response*
```
serverb | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}

```

Ping Hosts in `group2`
```
ansible group1 -m ping -i inventory
```

Response:
```
servera | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/isr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
```


Ping Hosts in `group12`
```
ansible group12 -m ping -i inventory
```
*Response*
```
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


## Configuration Files
Ansible's behavior is controlled by its configuration file. By default, this file is located at `/etc/ansible/ansible.cfg`. However, the configuration file used by Ansible can be overridden by the following files or environment variables, in order of precedence:

1. **Environment Variable(`ANSIBLE_CONFIG`):** If the ANSIBLE_CONFIG environment variable is defined, it will take precedence over all other configuration files.
2. **Working Directory:** If an `ansible.cfg` file exists in the current working directory from which the Ansible command is run, it will override the default configuration.
3. **Home Directory:** If an `.ansible.cfg` file exists in the home directory of the user running the command, it will take precedence over the default /etc/ansible/ansible.cfg file.
4. **Default Configuration:** If none of the above is set, Ansible will use the default configuration at `/etc/ansible/ansible.cfg`.

**Note:** Ansible does not merge settings from multiple configuration files. If a setting is not explicitly defined in the *active configuration file*, the default value from Ansible will be used.


**Example: Configuration and Inventory**
```yaml
#Inventory file

# You can list hosts by IP or hostname one per each line
[defaults] # -> Default settings section
inventory=/home/automation/inventory #Path to the inventory file used by Ansible
remote_user=automation # User that Ansible will use to connect to managed hosts

ask_pass=false # Whether Ansible should prompt for a password during connection

[privilege_escalation] # -> Privilege escalation settings section
become=true # Enables privilege escalation
become_method=sudo # The method used to escalate privileges (e.g., sudo)
become_user=root # The user to switch to after connection (root in this case)
become_ask_pass=false # Whether Ansible should prompt for a password for privilege escalation
```

**Demo**

To create our own coniguration file:

```
[automation@Host ~]$ mkdir configurationfile
[automation@Host ~]$ vim /etc/ansible/ansible.cfg
[automation@Host ~]$ cd configurationfile/
[automation@Host configurationfile]$ vim ansible.cfg

```
**ansible.cfg file**
```
[defaults]
inventory= /home/automation/configurationfile/inventory
remote_user= automation

[privilege_escalation]
become= true
become_method = sudo
become_user= root
become_ask_pass= false
```

```
[automation@Host configurationfile]$ vim inventory
```
**inventory file**
```
servera
serverb
```
1. Run a Ping Command Using the Configuration
    ```
    [automation@Host configurationfile]$ ansible all -m ping
    ```
    This command pings all hosts listed in the inventory file. Since the inventory is defined in the ansible.cfg file, there's no need to use the -i option.
    ```
    servera | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/libexec/platform-python"
        },
        "changes": false,
        "ping": "pong"
    }
    serverb | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/libexec/platform-python"
        },
        "changes": false,
        "ping": "pong"
    }
    ```

    This command pings all hosts listed in the inventory file. Since the inventory is defined in the ansible.cfg file, there's no need to use the -i option.

    **Explanation:** The output shows that Ansible is running as the root user on both servers due to the privilege escalation (become=true).

2. **Check the User ID on Remote Hosts**
    This command gives us the ID under which ansible is running on the manage hosts.
    ```
    [automation@Host configurationfile]$ ansible all -m command -a 'id'
    servera | CHANGED | rc=0 >>
    uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
    serverb | CHANGED | rc=0
    uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
    ```

    **Explanation:** The output shows that Ansible is running as the root user on both servers due to the privilege escalation (become=true).

    As you can see it has a UID of 0 that means we are running as the root user.

**Updating the Configuration File**
1. Modify the Inventory File to `inventory2`
    ```
    [automation@Host configurationfile]$ vim ansible.cfg
    ```

    Update the inventory setting in `ansible.cfg`
    ```
    [defaults]
    inventory= /home/automation/configurationfile/inventory2
    remote_user= automation

    [privilege_escalation]
    become= true
    become_method = sudo
    become_user= root
    become_ask_pass= false
    ```
    Create the new inventory file: `inventory2.cfg`
    ```
    [automation@Host configurationfile]$ vim inventory2.cfg
    ```

    ```
    servera
    ```

    Now, run the ping command again:

    ```
    [automation@Host configurationfile]$ ansible all -m ping

    servera | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/libexec/platform-python"
        },
        "changes": false,
        "ping": "pong"
    }

    ```
    **Explanation:** Only servera responds because serverb is not in inventory2.

 2. Disable Privilege Escalation
     Update the privilege escalation setting:
     ```
     [defaults]
     inventory= /home/automation/configurationfile/inventory2
     remote_user= automation

     [privilege_escalation]
     become= false       # <=======
     become_method = sudo
     become_user= root
     become_ask_pass= false
     ```

    ```
    [automation@Host configurationfile]$ ansible all -m command -a 'id'
    ```

    ```
    servera | CHANGED | rc=0 >>
    uid=1001(automation) gid=1001(automation) groups=1001(automation) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
    ```
    **Explanation:** Ansible is now running as the automation user because privilege escalation is disabled (become=false).

**Moving the Configuration to the Home Directory**
1. **Copy `ansible.cfg` to the Home Directory:**

    ```
    [automation@Host configurationfile]$ cp ansible.cfg /home/automation/ansible.cfg
    ```
2. **Remove the Existing Configuration File in the Working Directory:**

    After copying the configuration file, remove the existing ansible.cfg from the current working directory to avoid confusion:
   ```
   rm -f ansible.cfg
   ```

3. **Run the Ansible Command:**
    Now, attempt to run the same Ansible command to check the user ID on the remote hosts:
    ```
    [automation@Host configurationfile]$ ansible all -m command -a 'id'
    ```
    This will likely produce the following warning:
    ```
    [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
    ```
    **Why does this happen?**
    This happens because Ansible is looking for a file named .ansible.cfg in the home directory, not ansible.cfg. Since we copied the configuration file under the name ansible.cfg, it isn’t being used by Ansible.

4. **Verify the Files in the Home Directory:**

    Move to the home directory and check for the copied configuration file:
    ```
    [automation@Host configurationfile]$ cd ..
    [automation@Host ~]$ ls

    ansible.cfg configurationfile

    ```
    You’ll see that the file is named ansible.cfg, but Ansible expects it to be .ansible.cfg.

5. **Rename the File to `.ansible.cfg`:**
   Rename the configuration file to .ansible.cfg, which is the correct name for Ansible to recognize the file in the user’s home directory:
    ```
    [automation@Host ~]$ mv ansible.cfg .ansible/
    cp/ tmp/
    [automation@Host ~]$ mv ansible.cfg .ansible.cfg
    [automation@Host ~]$ ansible all -m command -a 'id'
    servera | CHANGED | rc=0 >>
    uid=1001(automation) gid=1001(automation) groups=1001(automation) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
    [automation@Host ~]$ cd configurationfile/
    ```
6. **Run the Ansible Command Again:**
    Now, run the Ansible command again to check the user ID on the remote hosts:

    ```
    [automation@Host configurationfile]$ ansible all -m command -a 'id'
    servera | CHANGED | rc=0 >>
    uid=1001(automation) gid=1001(automation) groups=1001(automation) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
    [automation@Host ~]$ 
    ```
7. **Verify That the Home Directory Configuration is Being Used:**

    The output indicates that Ansible is running under the automation user (UID 1001). If the output matches your expectations, it confirms that Ansible is using the .ansible.cfg configuration file from the user's home directory, which takes precedence over the default configuration at /etc/ansible/ansible.cfg.

## Ansible Ad-hoc Commands

Ansible ad-hoc commands are used for single-use tasks that are typically simple and quick to execute. They are commonly employed for:

- **Testing** specific actions or system conditions
- **Performing quick changes** on managed hosts
While ad-hoc commands are useful for straightforward system administration tasks, their functionality is limited compared to the full capabilities of Ansible. To harness Ansible's full power as an automation engine, you would typically use playbooks, which offer more control and reusability.


    **Ad-hoc Command Syntax**

    ```
    #Ansbile ad-hoc commands

    ansible <hosts> -m <module> -a '<arguments>' -i <inventory> -u <user>

    ```
**Breakdown of the Command:**
- <**hosts**>: Specifies the target hosts or host groups, which must be defined in your inventory. You can also use all to target all hosts or ungrouped to target hosts that are not part of any group.

- **-m** **<**module**>** : Specifies the Ansible module to be executed. Modules are the units of work in Ansible (e.g., ping, command, copy).

- **-a** **<**arguments**>**: (Optional) Provides arguments to the specified module. This allows you to pass specific commands or options to the module.

- **-i** **<**inventory**>:** Specifies the inventory file where the list of hosts and groups are defined. If this option is omitted, Ansible will look for the default inventory.

- **-u** **<**user**>:** Defines the user Ansible will use to connect to the hosts. If omitted, Ansible will use the default user specified in the configuration

    Let's look at some sample of Ansible ad-hoc commands

    ```yaml
    #Sample Ansible ad-hoc commands

    ansible all -m ping # This command will ping all hosts within the inventory to check connectivity.

    ansible servera -m copy -a 'content="Hello world"'
    dest=/home/automation/helloworld # This command will copy the content "Hello world" to the file /home/automation/helloworld on "servera".

    ansible servers -m command -a "hostname" # This command will run the "hostname" command on all hosts in the "servers" group, as defined in the inventory.


    ansible serverb -m user -a 'name=test state=present' -i /home/automation/invenotory # This command will create a user called "test" on "serverb" using the inventory file located at /home/automation/inventory.

    Ansible-doc user # This command will display detailed documentation about the "user" module and how to use it.

    ```

**Key Explanations:**
- ansible all -m ping: The ping module is used to check connectivity with all hosts in the inventory.
- ansible servera -m copy: The copy module is used to copy the content "Hello world" to the specified file on servera.
- ansible servers -m command -a "hostname": The command module runs the specified shell command (in this case, hostname) on all hosts in the servers group.
- ansible serverb -m user: The user module is used to create a new user named test on serverb.
- ansible-doc user: This shows the built-in documentation for the user module, helping you understand how to use it effectively.

**Basic Modules to Know**

- **Ping:** It pings and you can check host accessibility.
- **User:** Manages users on the managed host.
- **Service:** Manages services on the managed host.
- **Copy:** Copy a file to the managed host but can also used to create content.
- **Yum:** Manage packages on the managed host.
- **File:** Manage files on the managed host.
- **Firewalld:** Manage the firewalld service on managed hosts.
- **Reboot:** Reboot the managed hosts.
- **URI:** Interact with web content.
- **Nmcli:** MAnage networking on the managed hosts.


## Ansible Playbooks

### Playbooks Overview
Playbooks are a sequence of plays, and each play is a series of tasks performed on managed hosts specified in the inventory. The purpose of playbooks is to simplify administrative tasks by organizing them into easy, repeatable routines.

- Tasks in a Playbook: Tasks work together within a playbook to document the steps needed to achieve a desired outcome. These tasks are executed sequentially, from top to bottom.

- Repeatability: Playbooks are designed to be reusable, meaning they can be executed multiple times for consistent results.

- YAML Format: Playbooks are written in YAML format and are typically saved with the .yaml or .yml file extension. YAML ensures human-readable and structured files:

  - The start of a playbook is denoted by ---.
  - Strings do not require quotation marks, but using them improves readability.
- Multiple Plays: You can define multiple plays within a single playbook.

- Task Execution: By default, if a task fails during execution, the playbook will stop, preventing subsequent tasks from running.

### Output Color Coding

When running a playbook, Ansible provides color-coded output to indicate the status of each task. The colors represent the following states:

- **🟢 Green**: The system is already at the desired state, so no changes were required.
  
- **🟡 Yellow**: A change has been made to the system to bring it to the desired state.

- **🔴 Red**: An error occurred, and the task could not be completed successfully.


```yml
#Sample playbook This playbook will make sure the user called "test" exists on servera

---   # <== Start of playbook with 3 lines

-   name: Create new user # Name of play

    hosts: servera                          # Specify the target host(s) for this play (in this case, servera)
    tasks:                                  # Begin the list of tasks to be executed on the target host(s)
        - name: Ensure the user is created  # Descriptive name of the task
          user:                             # Ansible user module to manage users
            name: test                      # Specify the username ('test') to ensure it exists
            state: present                  # Ensure the user is in the 'present' state (i.e., created)

```

```yml
---
---
# This playbook performs two tasks on serverb:
# 1. Installs the firewalld service using the yum package manager.
# 2. Ensures the firewalld service is started.

-   name: Ensure firewalld is installed and running   # Name of the play

    hosts: serverb                                    # Target host(s) for the play (in this case, serverb)
    tasks:                                            # List of tasks to execute
        - name: Install firewalld service             # Task to install the firewalld package
          yum:                                        # Ansible yum module for package management on RedHat-based systems
            name: firewalld                           # Specify the firewalld package to install
            state: latest                             # Ensure the latest version of firewalld is installed

        - name: Start the firewalld service           # Task to start the firewalld service
          service:                                    # Ansible service module for managing services
            name: firewalld                           # Specify the service to manage (firewalld)
            state: started                            # Ensure the service is in the 'started' state


```

This is how you can customize configuration options for individual playbooks. These options will take precedence over the global Ansible configuration file (`ansible.cfg`).

Remote User: You can specify the remote user Ansible will use to connect to the managed host.
Privilege Escalation: You can use the `become` option to elevate privileges, switch to a different user (`become_user`), and specify the method (e.g., `sudo`).

```yml
---
# Sample playbook to illustrate remote user and privilege escalation configuration

- name: Myplay  # Name of the play

  become: true                # Enable privilege escalation
  become_user: <user>         # Specify the user to switch to (e.g., root)
  become_method: sudo         # Use sudo as the privilege escalation method
  remote_user: <user>         # Remote user Ansible will use to connect to the managed host
  hosts: serverb              # Specify the target host(s)
  tasks:                      # List of tasks (no tasks defined in this sample)

```

### Ansible Playbook Commands: 

- **Run a playbook:** 
  
  ``` ansible-playbook <playbookfile.yml> ```

- **Verify the syntax of a playbook**

  ``ansible-playbook --syntax-check <playbookfile.yml>``

- **Dry run a playbook (see what changes would be made without making them)**
  
  ```ansible-playbook -C <playbookfile.yml>```


