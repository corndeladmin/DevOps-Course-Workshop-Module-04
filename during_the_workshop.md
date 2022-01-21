# Workshop Instructions

## Part 1

### SSH to your VM

* You should have been provided with a **hostname** for a remote VM and some **credentials**.
* Use these credentials to connect to your VM over ssh.

```bash
  ssh user@hostname
```

You should now be connected to the remote VM. Try a few simple shell commands to confirm you're connected.

The preferred method of connecting is with SSH keys. This removes the need to provide a username and password every time, is more secure and is easier to administrate. (If you've forgotten how to setup an SSH key please follow [this guide](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key), following only the four steps under "Generating a new SSH key").

You should add your **public key** on a new line of the `authorized_keys` file on the remote VM.

```bash
# Opening the authorized_keys file for editing
vim ~/.ssh/authorized_keys
```

> How to edit and save a file with `vim`:  
> - You start in **Command mode** and cannot simply type text, though you can still paste from the clipboard, with `Shift + Insert`.  
> - Press `i` to enter **Insert mode** to type text. Press `Escape` to return to **Command mode**.  
> - From Command mode you can save & exit by typing `:wq` and then pressing `Enter`.

### Getting started with Chimera
The VM has a (partially) working version of Chimera on it. If you navigate to the URL provided by your trainer you should be able to see the page it produces. The `webapp` part appears to be functioning correctly, so you should leave it alone.

On the VM you should be able to find `cliapp`. This is a command line program with minimal documentation (see [cliapp_reference.md](./cliapp_reference.md)). Its purpose is to generate "datasets" for the webapp to display. You should be able to run it like so:

```bash
cliapp --version
```

This should print the version number of the program to the console. If you see this, you can move onto the next section.

### Input Data
If you run `cliapp` without any arguments it will print its output to the console. You should see something like this:

```json
{
  "datasetName": "edge-throw-except",
  "data": [],
  "generationTime": 1596458170057,
  "centre": "[0, 0]",
  "zoom": null
}
```

This is an empty dataset with a randomly generated name. To produce a useful dataset we need to provide an input file in the correct format.

Create a new file using the `touch` command and pass it to `cliapp` using the `-i` flag.

```bash
touch some_file_name
cliapp -i some_file_name
```

The result should be very similar to running the program without any arguments. This isn't surprising as our input file is currently blank.

Open your input file using `vim` and add the following line. Then run `cliapp -i some_file_name` again. What happens?
```bash
59.0916|-137.8717|111 km WSW of Covenant Life, Alaska|6.4
```

Hopefully `cliapp` generated a non-empty dataset. If you copy the dataset name and paste it onto the end of your URI in your web browser, you should see something on the map.

```bash
http://{hostname}/{datasetName}
```

### Data Manipulation
Now we know how to feed data into Chimera, we need to work out how to do this more efficiently.

We want to use [USGS](https://earthquake.usgs.gov/earthquakes/feed/v1.0/geojson.php) as our source. They have various regularly updated feeds. Start off using the [hourly feed](https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_hour.geojson). Try using the command line to fetch the data and then convert it to the pipe delimited format that `cliapp` can read.

#### **Tips:**
* Use `curl` to save a copy of the JSON feed locally (it's quicker than downloading it every time). Use the `-o` option to specify a file to save it to.
* The command line program `jq` is very useful for manipulating JSON data. You can find its manual [here](https://stedolan.github.io/jq/manual/).
  * There is also this [Cheatsheet](https://lzone.de/cheat-sheet/jq) that provides helpful examples.
  * String interpolation is a convenient way to build a string. When operating on some data like: `{"stringProp": "foobar", "intProp": 1}`, then the jq expression `"\(.stringProp) raw text \(.intProp)"` will evaluate to `"foobar raw text 1"`.
  * The jq command itself has an option `-r` to return string outputs "raw", meaning without wrapping it in quotes.
* Use `>` to direct the output of a command to a file. E.g. `echo "something" > output.txt`

<details>
<summary>If you are really stuck, expand this for more detailed help.</summary>

Read your json file and feed it into jq:

```
cat earthquakes.json | jq
```
  
<details>
<summary>Next step</summary>

The downloaded JSON contains far more data than you need. The outer object has a field called "features" which is the actual list of earthquakes.

```
cat earthquakes.json | jq '.features'
```

<details>
<summary>Next step</summary>

To loop over the list and evaluate an expression on each, you can use `[]` and jq's own pipe operator.

```
cat earthquakes.json | jq '.features[] | .geometry'
```

<details>
<summary>Next step</summary>

Access nested values by chaining dots. Combine values with string interpolation.

```
cat earthquakes.json | jq '.features[] | "\(.geometry.coordinates[0])|\(.properties.place)"'
```
<details>
<summary>Complete command</summary>

```
cat earthquakes.json | jq -r '.features[] | "\(.geometry.coordinates[1])|\(.geometry.coordinates[0])|\(.properties.place)|\(.properties.mag)"' > earthquakes.psv
```

</details>
</details>
</details>
</details>
</details>
<br>

When you've worked out the correct commands, put them in a bash script. This can simply be a text file containing your commands. but the file extension should be `.sh`.

> You will often see a "shebang" such as `#!/bin/bash -e` as the first line in a .sh file. This explicitly specifies what should run the file, and the `-e` is a useful option that means the script will exit if any command exits with an error

Make it executable (run `chmod +x your_script.sh`) and then try executing it (run `./your_script.sh`) to check it works. The script should:
* Fetch fresh data
* Convert it to the correct format
* Use `cliapp` to generate a dataset that the web app can display.

### Automation Part 1
When you've written your script we need to automate executing it. You'll want to use `crontab` to call the script on a schedule.

A summary of crontab and some tips:
  
* Use `crontab -e` to edit the crontab, which is simply a file listing scheduled jobs. This will open the file in `vim`.
* Each line is a job, given by a cron expression (specifying when it should run) and then a command to run.
* Each user has their own crontab. It will execute as them, and in their home directory
* An important difference between the cronjob execution and running the command yourself in bash is that the cronjob will not have run the `~/.bash_profile` file beforehand.
* You can experiment with the meaning of cron expressions [here](https://crontab.guru/)
* You might be wondering how to check the output of your cron jobs. After it runs, and you next interact with the terminal, you will see a message in the terminal saying "You have new mail". You can read the file it tells you about with `cat` or `tail`. E.g. `cat /var/spool/mail/ec2-user`.

We've got some requirements from the CEO:
* A dataset should be generated every five minutes, containing earthquakes in the last hour. It should be displayed on the site at `/latest`. <details><summary>Hint</summary> Use the --dataset-name option mentioned in the [cliapp_reference.md](./cliapp_reference.md) to specify a dataset name of "latest".</details>
* The same data should also be available with a dataset name containing the date and time it was generated. <details><summary>Hint</summary>Use the `date` command. E.g. `date +"%y"` would give you the year.</details> 
* Any datasets older than 24 hours should be automatically deleted. More recent ones should be kept accessible. <details><summary>Hint</summary>Use the `find` command on the folder containing the datasets. It has options to filter by date/time last modified, and an option to delete the files it finds.</details>
<br>

## Part 2

### Scaling up

Because of your great work, the Chimera website has become hugely popular - it's been featured on the TV news and you have thousands of users every day.  
In fact, it's become so popular that it's starting to slow down due to the sheer number of users.

We need to scale up!

We're going to setup 2 more VMs so we can share the website traffic between them.  
For this exercise, we're going to focus on setting up the new VMs. We won't deal with how to share the traffic between them.

You should have been provided with a **hostname** for the two new VMs.  
The credentials to login to these new VMs should be the same as for your first VM.

Now, we *could* setup these VMs by SSHing into them and running some commands.  
But, instead we're going to use Ansible.  
<br>

#### **Why Ansible?**

The main reasons for using Ansible are:
* Ansible makes it just as easy to setup 100 machines as it is to setup the first 1
  So, when we need to scale up further, this will be simple

* Ansible is **declarative** rather than **imperative**  
  Declarative: we say *what we want the end state to be*  
  Imperative: we say *the steps we want to take*  
  This can make our code easier to write, and can give us a lot of reassurance.  
  e.g. Ansible can tell us what steps it has already taken, whether steps succeeded or failed (we'll see this in practice later).

<br>

#### **How to Ansible**

Ansible needs two machines (either physical machines or VMs) to run:
* **Control node**  
  Any machine with Ansible installed. You can run Ansible commands and playbooks by invoking the `ansible` or `ansible-playbook` command from any control node. You can use any computer that has a Python installation as a control node - laptops, shared desktops, and servers can all run Ansible.  
  **Note:** However, you cannot use a Windows machine as a control node.  

* **Managed nodes**  
  The servers you manage with Ansible. Managed nodes are also sometimes called “hosts”. Ansible is not installed on managed nodes.

We've given you 3 VMs: the 1 original VM and the 2 new VMs.  
Control Node: Use the original VM as the Control Node.  
Managed Nodes: Use the 2 new VMs as the Managed Nodes.

If you use Mac or Linux, you can choose to use your own computer as the Control Node (you'll need to install Ansible on it first).  
If you use Windows, you'll have to use the original VM as the Control Node, since Ansible doesn't run on Windows.

<br>

#### **Let's get going**

**Step 1.** SSH into your Control Node (the original VM)

<details>
<summary>Hint:</summary>
<code>ssh ec2-user@HOSTNAME-OR_IP_ADDRESS-OF-VM</code><br>
(e.g. <code>ssh ec2-user@35.177.117.226</code> )
</details>
<br>

**Step 2.** Check that Ansible is installed  
Run the command `ansible --version`. If this prints out a version number + some other info, then Ansible is installed.  
Don't worry if it says `[DEPRECATION WARNING]` etc.

If Ansible isn't installed, then we can install it with `sudo pip install ansible`.  
This might take a minute to run. It might show a warning/error message at the end.  
Even if it does show a warning/error, run `ansible --version` to check if it succeeded.
<br><br>

**Step 3.** Test the SSH connection from the Control Node to each of the Managed Nodes.  
You are currently SSH'd from your laptop onto the Control Node.  
You are now going to SSH from the Control Node into each of the Managed Nodes in turn.  
```
                                      .--------------->  Managed
                                    .'   SSH tunnel       Node 1
 Your   ----------------->  Control
laptop      SSH tunnel       Node  .     
                                    `----------------->  Managed
                                         SSH tunnel       Node 2
```

<details>
<summary>Hint:</summary>
SSH from your laptop onto the Control Node (if you aren't already connected)<br>
<code>ssh ec2-user@HOSTNAME-OR-IP_ADDRESS-OF-CONTROL-NODE</code><br>
<br>
Then, from within that SSH session, SSH onto the Managed Node:<br>
<code>ssh ec2-user@HOSTNAME-OR-IP-ADDRESS-OF-MANAGED-NODE</code><br>
It should ask you for a password (it's the same as the password to the first VM)<br>
<br>
To exit each SSH session, run:<br>
<code>exit</code>
</details>
<br>


<details>
<summary>Here's what I saw as I typed in and ran the commands:</summary>

<pre>
JamGri@manatee MINGW32 /
$ 
</pre>

<pre>
$ ssh ec2-user@35.177.117.226
ec2-user@35.177.117.226's password:
</pre>

<pre>
ec2-user@35.177.117.226's password: ***********
Last login: Thu Jan 20 16:33:50 2022 from cpc73840-dals21-2-0-cust250.20-2.cable.virginm.net

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
56 package(s) needed for security, out of 110 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-172-31-31-253 ~]$
</pre>

<pre>
[ec2-user@ip-172-31-31-253 ~]$ ssh ec2-user@3.10.179.156
ec2-user@3.10.179.156's password:
</pre>

<pre>
ec2-user@3.10.179.156's password: ***********
Last login: Wed Jan 19 15:15:38 2022 from 35.177.117.226

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
56 package(s) needed for security, out of 110 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-172-31-24-197 ~]$
</pre>

<pre>
[ec2-user@ip-172-31-24-197 ~]$ exit
logout
Connection to 3.10.179.156 closed.
[ec2-user@ip-172-31-31-253 ~]$
</pre>

<pre>
[ec2-user@ip-172-31-31-253 ~]$ exit
logout
Connection to 35.177.117.226 closed.

JamGri@manatee MINGW32 /
$
</pre>
</details>
<br><br>

**Step 4.** Create an SSH key and add it to the Managed Nodes.  
All this typing in of passwords is a bit boring, isn't it!  
Let's save ourselves some hassle by creating an SSH key instead.

An SSH key has two parts (public and private).  
We'll keep the private part on the Control Node and copy the public part onto the two Managed Nodes.  
That way, the Managed Nodes will know who we are when we ask to login.  

SSH from your laptop onto the Control Node (if you aren't already connected)  
`ssh ec2-user@HOSTNAME-OF-CONTROL-NODE`  
We will continue to use a password for this connection.

Create an SSH key.  
`ssh-keygen` (accept the default options)

Copy the SSH key onto the Managed Nodes (you'll have to run this command twice, once for each node).  
`ssh-copy-id ec2-user@HOSTNAME-OF-MANAGED-NODE`  
You will be asked for the password to the Managed Node.
<br><br>


**Step 5.** Tell Ansible which machines we want it to control.  
Ansible works against multiple Managed Nodes or “hosts” in your infrastructure at the same time, using a list known as [Inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#intro-inventory).

We will create our own inventory file.  
Here's an example:  
```
[my-group-name]
host-name-of-server-1.example.com
host-name-of-server-2.example.com
3.10.179.156
```

In an Ansible inventory file, you can create groups, each with a group name.  
Within each group, you add lines, each with a hostname or IP address.  

Have a go creating your own inventory file - perhaps save it in your home directory and name it `my-ansible-inventory`

<details>
<summary>Hint:</summary>
SSH from your laptop onto the Control Node (if you aren't already connected):<br>
<code>ssh ec2-user@HOSTNAME-OF-CONTROL-NODE</code><br><br>

Check we're in the home directory: <code>pwd</code><br>
This should print <code>/home/ec2-user</code><br>

Create the inventory file: <code>touch my-ansible-inventory</code><br>

Edit the inventory file: <code>nano my-ansible-inventory</code><br>

Add the inventory details<br>
\- Let's crerate a group - perhaps call it "webservers"<br>
\- Add the hostnames or IP addresses of your VMs<br>
(remember to use the actual IP addresses of your VMs if you're copying the example below!)<br>
<code>
[webservers]<br>
1.2.3.4<br>
5.6.7.8
</code><br>
</details>
<br>

**Step 6.** Give Ansible our instructions
We give Ansible instructions do by creating a [Playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#about-playbooks).  
If you need to execute a task with Ansible more than once, write a playbook and put it under source control. Then you can use the playbook to push out new configuration or confirm the configuration of remote systems.

First, create the playbook, let's call it `my-ansible-playbook.yaml`

<details>
<summary>Hint:</summary>
SSH from your laptop onto the Control Node (if you aren't already connected):<br>
<code>ssh ec2-user@HOSTNAME-OF-CONTROL-NODE</code><br><br>

Check we're in the home directory: <code>pwd</code><br>
This should print <code>/home/ec2-user</code><br>

Create the playbook file: <code>touch my-ansible-playbook.yaml</code><br>

For the following sections, you can edit the playbook file using: <code>nano my-ansible-playbook.yaml</code><br>
</details>
<br>


Playbooks are expressed in YAML format.  
YAML files always start with 3 hyphens:  
```
---
```

A playbook is composed of one or more ‘plays’ in an ordered list.  
We will just have one ‘play’ in ours.  
In each play, we specify:
- the name of the play
- the group of hosts to run the play on (using the group name from the Inventory)
- the user that Ansible should use to connect to the hosts (Ansible connects to the hosts using SSH)

```
---
- name: Install Chimera on new web servers
  hosts: webservers
  remote_user: ec2-user
```

Each play runs one or more tasks. Each task calls an Ansible module.  
We will have several tasks in our playbook.  
A playbook runs in order from top to bottom. Within each play, tasks also run in order from top to bottom.

Let's create our first task, then inspect what it's doing:
```
---
- name: Install Chimera on new web servers
  hosts: webservers
  remote_user: ec2-user

  tasks:
  - name: Create log folder
    ansible.builtin.file:
      path: /var/log/chimera
      state: directory
      mode: '777'
    become: yes
```
Each task name a name (e.g. "Create log folder").  
The task then has a Module (e.g. "ansible.builtin.file").  

There are a huge number of Ansible Modules, grouped into Collections.  
Here is the list of all [Ansible Collections](https://docs.ansible.com/ansible/latest/collections/index.html).  
And here is the list of [Modules within the "ansible.builtin" collection](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html#modules).

In this case, we are using the [ansible.builtin.file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html) module,
which we can use to manage files, folders and file properties.

Each Module performs a specific task and take a different set of parameters.  
You can check on the ansible website to find out which paramters to use.

In this case:
* `path: /var/log/chimera` means... act on the file or folder `/var/log/chimera`
* `state: directory` means... make this path be a directory
* `mode: '777'` means... give full permissions to this directory

`become: yes` is an option on all Modules.  
It tells Ansible to run the Module as the root user (e.g. the same as putting `sudo` before a shell command).  
There's [more details about become on the Ansible website](https://docs.ansible.com/ansible/latest/user_guide/become.html).
<br><br>


**Step 7.** Time to run the playbook for the first time

Run your ansible playbook using this command:  
`ansible-playbook my-ansible-playbook.yaml -i my-ansible-inventory`

The output should look a bit like this:  
<pre>
PLAY [Install Chimera on new web servers] ******************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [3.10.179.156]
ok: [35.178.196.136]

TASK [Create log folder] ***********************************************************************************************************************************
changed: [35.178.196.136]
changed: [3.10.179.156]
</pre>

Notice that, within the "Create log folder" task, is says `changed` for both nodes.  
This is how we can tell that Ansible has made a change.

Now run the playbook again...  
`ansible-playbook my-ansible-playbook.yaml -i my-ansible-inventory`

The output should look different:  
<pre>
PLAY [Install Chimera on new web servers] ******************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [3.10.179.156]
ok: [35.178.196.136]

TASK [Create log folder] ***********************************************************************************************************************************
ok: [35.178.196.136]
ok: [3.10.179.156]
</pre>

Now, for the "Create log folder" task, it says `ok` instead of `changed`.  
But why?  
Simply because the folder already exists (Ansible created it the first time you ran the playbook).

So, Ansible checks the state of the folder before trying to create it.  
If the folder already exists, and has the correct permissions, it won't make any changes.  
Isn't that reassuring :-)  
You can run the playbook just to check that everything is in the correct state, safe in the knowledge that it won't change anything that doesn't need changing.
<br><br>


**Step 8.** Write the rest of the playbook

Below is the rest of the playbook.  
Copy it into your playbook a section or two at a time.  
**After each section, run the playbook** so you can see Ansible making the changes.  
Notice how each task moves shows `changed` the first time and `ok` subsequently.

Have a look through the Ansible documentation to see what all the Modules do and what all the options mean.  
We're using these Modules:
* [ansible.builtin.file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
* [ansible.builtin.yum](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html)
* [ansible.builtin.pip](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pip_module.html)
* [amazon.aws.aws_s3](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_s3_module.html)
* [ansible.builtin.systemd](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html)

Here's the playbook:  
<pre>
---
- name: Install Chimera on new web servers
  hosts: webservers
  remote_user: ec2-user

  tasks:
  - name: Create log folder
    ansible.builtin.file:
      path: /var/log/chimera
      state: directory
      mode: '777'
    become: yes

  - name: Install jq (command-line JSON processor)
    ansible.builtin.yum:
      name: jq
      state: present
    become: yes

  - name: Create folder for the Chimera webapp binary
    ansible.builtin.file:
      path: /opt/chimera/bin
      state: directory
    become: yes

  - name: Install pip (python package manager)
    ansible.builtin.yum:
      name: pip
      state: present
    become: yes

  - name: Install a python library that we will need for AWS file copying
    ansible.builtin.pip:
      name: boto3
      state: present
    become: yes

  - name: Copy webapp from AWS S3 bucket to new binaries folder
    amazon.aws.aws_s3:
      mode: get
      bucket: jwg-devops-module4   #################### CHANGE THIS (although it will work if needed) ####################
      object: webapp
      dest: /opt/chimera/bin/webapp
      overwrite: false
    become: yes

  - name: Make webapp executable
    ansible.builtin.file:
      path: /opt/chimera/bin/webapp
      state: file
      mode: '+x'
    become: yes

  - name: Copy start-webapp.sh from AWS S3 bucket to new binaries folder
    amazon.aws.aws_s3:
      mode: get
      bucket: jwg-devops-module4   #################### CHANGE THIS (although it will work if needed) ####################
      object: start-webapp.sh
      dest: /opt/chimera/bin/start-webapp.sh
      overwrite: false
    become: yes

  - name: Make start-webapp.sh executable
    ansible.builtin.file:
      path: /opt/chimera/bin/start-webapp.sh
      state: file
      mode: '+x'
    become: yes

  - name: Copy webapp.service from AWS S3 bucket to systemd folder
    amazon.aws.aws_s3:
      mode: get
      bucket: jwg-devops-module4   #################### CHANGE THIS (although it will work if needed) ####################
      object: webapp.service
      dest: /usr/lib/systemd/system/webapp.service
      overwrite: false
    become: yes

  - name: Start the systemd service
    ansible.builtin.systemd:
      name: 'webapp.service'
      state: started
    become: yes
</pre>
<br>

**Optional extra**

You may have noticed that, in the playbook, we:
* copied a "webapp.service" file to a "systemd" folder
* used the `systemd` module to start a service

Take a look at the `webapp.service` file and have a read online about how systemd works.  
`systemd` is a very powerful tool that can, amongst many other things, be used to start / stop / monitor long-running services like our web app.
<br><br>


### Automation Part 2
*If you haven't finished "Automation Part 1" yet, do so before starting the next steps*

The CEO has come back with some more requirements for you.

* At midnight everyday the script should create a summary of the last 24 hours. It should be called `yesterday`.
* Every hour during the day a new dataset called `today` should be produced. It should show all earthquakes since the most recent midnight.

#### Hints
* The USGS has more than just the hourly data feed.
* You might want to add some parameters to your script, if you haven't already.

### (Stretch goal) Document `cliapp`

If you run `cliapp --help` you'll see lots of different options that can be passed to the program. We've documented some of these in [cliapp_reference.md](./cliapp_reference.md) but some are a complete mystery.

Can you work out what they do and complete the documentation?

### (Stretch goal) Create an API

While command line programs are very useful in certain circumstances they do have their limitations. We would like to create an API that makes the data produced by `cliapp` available as JSON, rather than through the rather clunky `webapp`.

Use your existing knowledge of Flask (and documentation from the internet) to create an API on your local Vagrant box. Your initial goal should be to create an endpoint to `GET` a dataset based on its name:
```
// e.g. GET the dataset called `edge-throw-except`
GET http://localhost/dataset/edge-throw-except
```
Once you have this working consider other improvements. A `GET` endpoint to list all the currently available datasets? A `POST` endpoint that accepts input data and options and invokes `cliapp` with them? There are lots of possibilities.
