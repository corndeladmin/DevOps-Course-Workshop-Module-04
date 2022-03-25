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
* Each line is a job, consisting of a cron expression (i.e. when it should run) followed by a shell command to run.
* Each user has their own crontab. It will execute as them, and in their home directory.
* An important difference between the cronjob execution and running the command yourself in bash is that the cronjob will not have run the `~/.bash_profile` file beforehand, which sets up some important environment variables.
* You can experiment with the meaning of cron expressions [here](https://crontab.guru/)
* You might be wondering how to check the output of your cron jobs. After it runs, and you next interact with the terminal, you will see a message in the terminal saying "You have new mail" and a filepath. You can read the file with `cat` or `tail`. E.g. `cat /var/spool/mail/ec2-user`.
<details><summary>Hints</summary>

If you're not seeing your cronjob succeeding, do you know if it's running at all?

<details><summary>How can we tell if the job is being attempted?</summary>

There are a few things we could try:
* Keep an eye out for messages stating you have mail - if you do then take a look to see if it has any hints on what's happening!
* Check any outputs that your script has - are they updating? For example, you can see the modified time of files using `ls -l`
* If you're not seeing either of the above, check that your cron expression is described by [crontab guru](https://crontab.guru/) as what you'd expect
</details>

<details><summary>How can I output the logs to my own log file?</summary>

Good question, let's [ask the internet!](https://stackoverflow.com/questions/4811738/how-to-log-cron-jobs)

</details>

<details><summary>The job is running but it fails</summary>

First of all, double check that the script runs successfully when you run it directly. If it doesn't, start by fixing it up.

If it's running directly but not through cron, check the error messages that you receive. Clearly we have some differences in how cron is running the job compared to you; we may need to provide some extra specifics within our script to help cron know how to run the job.

In particular, the following may be of assistance:
* The `whereis` command can help you find the absolute path for commands
* You can set environment variables within a script using `export VAR="value"`

</details>
</details>

We've got some requirements from the CEO:
* A dataset should be generated every five minutes, containing earthquakes in the last hour. It should be displayed on the site at `/latest`. <details><summary>Hint</summary> Use the --dataset-name option mentioned in the [cliapp_reference.md](./cliapp_reference.md) to specify a dataset name of "latest".</details>
* The same data should also be available with a dataset name containing the date and time it was generated. <details><summary>Hint</summary>Use the `date` command. E.g. `date +"%y"` would give you the year.</details> 
* Any datasets older than 24 hours should be automatically deleted. More recent ones should be kept accessible. <details><summary>Hint</summary>Use the `find` command on the folder containing the datasets. It has options to filter by date/time last modified, and an option to delete the files it finds. The `DATA_FOLDER` environment variable will tell you where the datasets are stored.</details>
<br>

When you think your cronjob is working, check! Either look at the "dataset generated" timestamp on the website, or check the logs in `/var/spool/mail/ec2-user`.

## Part 2

*If you haven't finished "Automation Part 1" yet, do so before starting the next steps*

### Scaling up

Because of your great work, the Chimera website has become hugely popular - it's been featured on the TV news and you have thousands of users every day.
In fact, it's become so popular that it's starting to slow down due to the sheer number of users.

We need to scale up!

We're going to set up two more VMs so we can share the website traffic between them.  
For this exercise, we're going to focus on setting up the new VMs. We won't deal with how to share the traffic between them.

You should have been provided with **hostnames** for the two new VMs.  
The credentials to log in are the same as for your first VM.

Now, we *could* set up these VMs by SSHing into them and running some commands.  
But, instead we're going to use Ansible.  

### Why Ansible?

The main reasons for using Ansible are:
* Ansible makes it just as easy to set up 100 machines as it is to setup the first 1.
So, when we need to scale up further, this will be simple.

* Ansible is **declarative** rather than **imperative**.  
  * Declarative: we say *what we want the end state to be*  
  * Imperative: we say *the steps we want to take*  

  This makes it easier to manage. It gives us a lot of reassurance because Ansible files describe the state of the servers and can be managed in source control, like code.
  When it runs, Ansible can tell us what steps it has already taken, whether steps succeeded or failed (we'll see this in practice later).

### How to Ansible

Ansible needs two machines (either physical machines or VMs) to run:
* **Control node**  
  Any machine with Ansible installed. You can run Ansible commands and playbooks by invoking the `ansible` or `ansible-playbook` command from any control node. You can use any computer that has a Python installation as a control node - laptops, shared desktops, and servers can all run Ansible.  
  **Note:** However, you cannot use a Windows machine as a control node.  

* **Managed nodes**  
  The servers you manage with Ansible. Managed nodes are also sometimes called “hosts”. Ansible is not installed on managed nodes. They only need a way for the control node to connect (e.g. SSH). Most operations require Python but Ansible can use the "raw module" to install Python itself.

We've given you 3 VMs: the 1 original VM and the 2 new VMs.  
Control Node: Use the original VM as the Control Node.  
Managed Nodes: Use the 2 new VMs as the Managed Nodes.

If you use Mac or Linux, you can choose to use your own computer as the Control Node (you'll need to install Ansible on it first).  
If you use Windows, you'll have to use the original VM or your own VM as the Control Node, since Ansible doesn't run on Windows.

### **Let's get going**

**Step 1.** SSH into your Control Node (the original VM) if you aren't already connected.

**Step 2.** Check that Ansible is installed

Run the command `ansible --version`. If this prints out a version number + some other info, then Ansible is installed.  
Don't worry if it says `[DEPRECATION WARNING]` etc.

If Ansible isn't installed, then we can install it with `sudo pip install ansible`.  
This might take a minute to run. It might show a warning/error message at the end.  
Even if it does show a warning/error, run `ansible --version` to check if it succeeded.

**Step 3.** Test the SSH connection from the Control Node to each of the Managed Nodes.  
You are currently SSH'd from your own machine to the Control Node.  
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
From your SSH session on the control node, SSH onto the Managed Node:<br>
<code>ssh ec2-user@HOSTNAME-OR-IP-ADDRESS-OF-MANAGED-NODE</code><br>
It should ask you for a password (it's the same as the password to the first VM)<br>
<br>
To exit each SSH session, run:<br>
<code>exit</code>
</details>
<br>

**Step 4.** Create an SSH key and add it to the Managed Nodes.

All this typing in of passwords is a bit boring, isn't it!  
Let's save ourselves some hassle by creating an SSH key instead. This is the same as what we did at the start of the workshop but this time we'll create the key on the Control Node and copy the public part onto the two Managed Nodes. 

That way, the Managed Nodes will allow the Control Node to connect.

- SSH from your laptop onto the Control Node (if you aren't already connected).
- Create an SSH key with `ssh-keygen` (accept the default options)
- Copy the SSH key onto the Managed Nodes, but rather than doing it manually, use the `ssh-copy-id` command this time. Run this command twice, once for each node:  
`ssh-copy-id ec2-user@HOSTNAME-OF-MANAGED-NODE`  
You will be asked for the password to the Managed Node.

**Step 5.** Tell Ansible which machines we want it to control.

Ansible works against multiple Managed Nodes or “hosts” in your infrastructure at the same time, using a list known as [Inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#intro-inventory). The most common formats are INI and YAML.

We will create our own inventory file.  
Here's an example INI format inventory:  
```
[my_group_name]
host-name-of-server-1.example.com
host-name-of-server-2.example.com
3.10.179.156
```

In an Ansible inventory file, you can create groups, each with a group name.  
Within each group, you add lines, each with a hostname or IP address.  

Have a go creating your own inventory file - perhaps save it in your home directory and name it `my-ansible-inventory`

<details markdown="1">
<summary markdown="1">Hint:</summary>
SSH from your laptop onto the Control Node (if you aren't already connected).

Check you're in the home directory with the command `pwd`. This stands for "print working directory" and should print `/home/ec2-user`. If not, you can change to your home directory with `cd ~` and then run `pwd` again to check.

Create the inventory file with vim or nano: <code>vim my-ansible-inventory</code><br>

Add the inventory details:
- Create a group called "webservers"
- Add the two hostnames (or IP addresses) of your new VMs

```ini
[webservers]
1.2.3.4
5.6.7.8
```
</details>
<br>

**Step 6.** Give Ansible our instructions

We give Ansible instructions by creating a [Playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#about-playbooks).  
If you need to execute a task with Ansible more than once, write a playbook and put it under source control. Then you can use the playbook to push out new configuration or confirm the configuration of remote systems.

First, create the playbook, let's call it `my-ansible-playbook.yml`

<details>
<summary>Hint:</summary>

Create the file in the home directory of your Control Node, next to your inventory file. Again, use vim or nano, e.g. <code>vim my-ansible-playbook.yml</code>
</details>

Playbooks are expressed in [YAML format](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html#yaml-syntax).  

A playbook is composed of one or more ‘plays’ in an ordered list.  
For now you just need one playbook, containing one ‘play’. 
In each play, we specify:
- the **name** of the play
- the group of **hosts** to run the play on (using the group name from the Inventory)
- the **remote_user** that Ansible should use to connect to the hosts (Ansible connects to the hosts using SSH)

For example:
```
- name: Install Chimera on new web servers
  hosts: webservers
  remote_user: ec2-user
```

Each play runs one or more tasks. Each task calls an Ansible module.  
We will have several tasks in our playbook.  
A playbook runs plays in order from top to bottom. Within each play, tasks also run in order from top to bottom.

Let's create our first task, then inspect what it's doing:
```
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
Each task has a name (e.g. "Create log folder").  
The task then picks a Module (e.g. "ansible.builtin.file") and uses that as a key.

> There are a huge number of Ansible Modules, grouped into Collections.  
> Here is the list of all [Ansible Collections](https://docs.ansible.com/ansible/latest/collections/index.html).  
> And here is the list of [Modules within the "ansible.builtin" collection](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html#modules).
> 
> In this case, we are using the [ansible.builtin.file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html) module,
which can manage files, folders and file properties.

Each Module performs a specific task and takes a different set of parameters.  
You can check on the Ansible website to find out which parameters to use.

In this case:
* `path: /var/log/chimera` means... act on the file or folder `/var/log/chimera`
* `state: directory` means... make this path be a directory
* `mode: '777'` means... give everyone full permissions to this directory

`become: yes` is an option on all Modules.  
It tells Ansible to run the Module as the root user (like putting `sudo` before a shell command). 
[See the Ansible website for more details about "become"](https://docs.ansible.com/ansible/latest/user_guide/become.html).
<br>

**Step 7.** Time to run the playbook for the first time

Run your ansible playbook using this command:  
`ansible-playbook my-ansible-playbook.yml -i my-ansible-inventory`

The output should look a bit like this:  
<pre>
PLAY [Install Chimera on new web servers] ********************

TASK [Gathering Facts] ********************
ok: [3.10.179.156]
ok: [35.178.196.136]

TASK [Create log folder] ********************
changed: [35.178.196.136]
changed: [3.10.179.156]

PLAY RECAP *********************
35.178.196.136 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
3.10.179.156 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
</pre>

Notice that, within the "Create log folder" task, it says `changed` for both nodes.  
This is how we can tell that Ansible has made a change.

Now run the playbook again...  
`ansible-playbook my-ansible-playbook.yml -i my-ansible-inventory`

Now, for the "Create log folder" task, it says `ok` instead of `changed`:
<pre>
PLAY [Install Chimera on new web servers] ********************

TASK [Gathering Facts] ********************
ok: [3.10.179.156]
ok: [35.178.196.136]

TASK [Create log folder] ********************
ok: [35.178.196.136]
ok: [3.10.179.156]
</pre>

But why?  
Simply because the folder already exists (Ansible created it the first time you ran the playbook).

So, Ansible checks the state of the folder before trying to create it.  
If the folder already exists, and has the correct permissions, it won't make any changes.
Isn't that reassuring? 😊

You can run the playbook just to check that everything is in the correct state, safe in the knowledge that it won't change anything that doesn't need changing.


**Step 8.** Write the rest of the playbook

Now it's over to you to write the rest of the Ansible playbook.  
I'll give you a list of tasks to add below.

For each task, you'll need to:  
* Work out which Module you need
* Work out which options you need for the module  
  You can look this up on the Ansible docs
* Work out whether you need to run the module as root user (`become: yes`)  
  Hint: you can just try without first - it will fail and tell you if it doesn't have permission to perform the task

For reference, here's the Ansible Modules that we'll be using:
* [ansible.builtin.copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
* [ansible.builtin.file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
* [ansible.builtin.systemd](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html)

Write your playbook one task at a time.  
**After each new task, run the playbook** so you can check if it works and you can see Ansible making the changes.  
Notice how each task shows `changed` the first time and `ok` subsequently.

Here's the list of tasks you'll need to add:

**Task a: Create folder for the Chimera webapp binary**  
We want to create the folder `/opt/chimera/bin`  

<details>
<summary>Hints:</summary>

  <details>
    <summary>Hint 1: Give your task a name</summary>
    The name could be <b>Create folder for the Chimera webapp binary</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
      - name: Create folder for the Chimera webapp binary</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 2: Which module should you use?</summary>
    We want to create a folder.<br>
    So, we should use the <a href="https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html">ansible.builtin.file</a> module.
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Create folder for the Chimera webapp binary
    ansible.builtin.file:</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 3: Which folder do we want to create?</summary>
    We want to create the folder <b>/opt/chimera/bin</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Create folder for the Chimera webapp binary
    ansible.builtin.file:
      path: /opt/chimera/bin</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 4: What state do we want the folder to be in?</summary>
    We need to specify that it's a <b>folder</b> that we're creating, not a <b>file</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Create folder for the Chimera webapp binary
    ansible.builtin.file:
      path: /opt/chimera/bin
      state: directory</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 5: Do we need to run this as a root user?</summary>
    Try it out? If it fails, try adding <b>become: yes</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Create folder for the Chimera webapp binary
    ansible.builtin.file:
      path: /opt/chimera/bin
      state: directory
    become: yes</pre>
    </details>
  </details><br>

</details><br>

**Task b: Copy webapp program from Control Node to Managed Nodes**  
The Control Node (the VM that you're running Ansible on) has the webapp program running on it.  
It's file name is `webapp` and it's in the folder `/opt/chimera/bin/`.  
We want to copy this file to the Managed Nodes.  
We want to put it in the same folder on the Managed Nodes (`/opt/chimera/bin/`).  
The file we're copying is a program, so make sure it is executable.

<details>
<summary>Hints:</summary>

  <details>
    <summary>Hint 1: Give your task a name</summary>
    The name could be <b>Copy webapp program from Control Node to Managed Nodes</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp program from Control Node to Managed Nodes</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 2: Which module should you use?</summary>
    We want to copy a file from the local to remote machine.<br>
    So, we should use the <a href="https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html">ansible.builtin.copy</a> module.
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp program from Control Node to Managed Nodes
    ansible.builtin.copy:</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 3: Which file do we want to copy?</summary>
    It's file name is `webapp` and it's in the folder `/opt/chimera/bin/`.<br>
    You'll need to join the folder and file name to give the full path<br>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp program from Control Node to Managed Nodes
    ansible.builtin.copy:
      src: /opt/chimera/bin/webapp</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 4: Where do we want the file to end up?</summary>
    We want to put it in the same folder on the Managed Nodes: <b>/opt/chimera/bin/</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp program from Control Node to Managed Nodes
    ansible.builtin.copy:
      src: /opt/chimera/bin/webapp
      dest: /opt/chimera/bin/</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 5: Does the file need any special permissions?</summary>
    Remember to make the file executable.<br>
    The docs for this aren't very clear - a nice way of doing this is using the mode <b>'+x'</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp program from Control Node to Managed Nodes
    ansible.builtin.copy:
      src: /opt/chimera/bin/webapp
      dest: /opt/chimera/bin/
      mode: '+x'</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 6: Do we need to run this as a root user?</summary>
    Try it out? If it fails, try adding <b>become: yes</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp program from Control Node to Managed Nodes
    ansible.builtin.copy:
      src: /opt/chimera/bin/webapp
      dest: /opt/chimera/bin/
      mode: '+x'
    become: yes</pre>
    </details>
  </details><br>

</details><br>


**Task c: Copy start-webapp.sh from Control Node to Managed Nodes**  
Here's another file to copy from the Control Node to the Managed Nodes.  
It's file name is `start-webapp.sh` and it's in the folder `/opt/chimera/bin/`.  
We want to put it in the same folder on the Managed Nodes (`/opt/chimera/bin/`).  
The file we're copying is a program, so make sure it is executable.

<details>
<summary>Hints:</summary>
This task will look a lot like the previous one. You just need a different "name" for the task and the correct "src" filename 
</details><br>


**Task d: Copy webapp.service from Control Node to Managed Nodes**  
Here's another file to copy from the Control Node to the Managed Nodes.  
It's file name is `webapp.service` and it's in the folder `/opt/chimera/bin/`.  
**This time we want to put it in a different folder**  
We want to put it in folder (`/usr/lib/systemd/system/`) on the Managed Nodes.  
The file we're copying **is not** a program, so it doesn't need to be executable.

<details>
<summary>Hints:</summary>

  <details>
    <summary>Hint 1: Give your task a name</summary>
    The name could be <b>Copy webapp.service from Control Node to Managed Nodes</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp.service from Control Node to Managed Nodes</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 2: Which module should you use?</summary>
    We want to copy a file from the local to remote machine.<br>
    So, we should use the <a href="https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html">ansible.builtin.copy</a> module.
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp.service from Control Node to Managed Nodes
    ansible.builtin.copy:</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 3: Which file do we want to copy?</summary>
    It's file name is `webapp.service` and it's in the folder `/opt/chimera/bin/`.<br>
    You'll need to join the folder and file name to give the full path<br>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp.service from Control Node to Managed Nodes
    ansible.builtin.copy:
      src: /opt/chimera/bin/webapp.service</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 4: Where do we want the file to end up?</summary>
    We want to put it in folder <b>/usr/lib/systemd/system/</b> on the Managed Nodes.
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp.service from Control Node to Managed Nodes
    ansible.builtin.copy:
      src: /opt/chimera/bin/webapp.service
      dest: /usr/lib/systemd/system/</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 5: Does the file need any special permissions?</summary>
    The file we're copying **is not** a program, so it doesn't need to be executable.<br>
    So, there's nothing to add here.
  </details><br>

  <details>
    <summary>Hint 6: Do we need to run this as a root user?</summary>
    Try it out? If it fails, try adding <b>become: yes</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Copy webapp.service from Control Node to Managed Nodes
    ansible.builtin.copy:
      src: /opt/chimera/bin/webapp.service
      dest: /usr/lib/systemd/system/
    become: yes</pre>
    </details>
  </details><br>

</details><br>

**Task e: Start the webapp service**  
The previous task copied `webapp.service` into the `/usr/lib/systemd/system/` folder.  
This installs `webapp` as a <b>systemd service</b>. We now want to start this service.  

> `systemd` is a tool on most Linux distributions for managing user processes. We are using it to start the webapp more reliably than simply running the executable directly. If we needed to restart the app after a change, we would be able to do so easily with the systemd module.  

<details>
<summary>Hints:</summary>

  <details>
    <summary>Hint 1: Give your task a name</summary>
    The name could be <b>Start the webapp service</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Start the webapp service</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 2: Which module should you use?</summary>
    We want to start a systemd service.<br>
    So, we should use the <a href="https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html">ansible.builtin.systemd</a> module.
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Start the webapp service
    ansible.builtin.systemd:</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 3: Which service do we want to start?</summary>
    The service name is <b>webapp.service</b> (which we can shorten to <b>webapp</b> if we like.).
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Start the webapp service
    ansible.builtin.systemd:
      name: webapp</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 4: What state should the service end up in?</summary>
    We want the service to be "started".<br>
    If the service is already running, then we want to leave it running (no need to restart it). You could set state to "restarted" instead, if you want to ensure it is restarted after a change to the app or its configuration.
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Start the webapp service
    ansible.builtin.systemd:
      name: webapp
      state: started</pre>
    </details>
  </details><br>

  <details>
    <summary>Hint 6: Do we need to run this as a root user?</summary>
    Try it out? If it fails, try adding <b>become: yes</b>
    <details>
    <summary>Show me the code</summary>
    <pre>
  - name: Start the webapp service
    ansible.builtin.systemd:
      name: webapp
      state: started
    become: yes</pre>
    </details>
  </details><br>

</details><br>


**Step 9** Check the webapp is working!

Once Ansible says everything has succeeded, the webapp should be working on each of your 2 new VMs. Let's check this.  
Try to visit the hostname (or public IP address) of your managed nodes and you should hopefully see the earthquake map (try this for each of your 2 managed nodes).

## Stretch Goals

You've done the core part of the exercise so feel free to pick and choose stretch goals that interest you more.

### Ansible Part 2

You now have the **webapp** running on the 2 new VMs but not with any real data. Let's fix that, but note the cronjob to generate datasets is **not** something we want to scale in the same way: we want the same datasets available on all of the replica webservers. So the data processing job should only run once. Then after it generates datasets, Ansible can synchronise the data folder from that machine with any other webservers.

**Update your playbook:**
- Add a new task to create a folder at `/opt/chimera/data` on each host (to store datasets). You can create it similarly to the log folder.
- Run your updated playbook.

**Write a new playbook**

This will be for generating and sharing datasets, so call it something like "datasets-playbook.yml".

In the new playbook, add a play that runs on just **one** server. It should do the following:
- Copy two files from the control node onto the target server: the cliapp executable and the script you wrote earlier today.
- Use the [ansible.builtin.yum](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html) module to install `jq`.
- Run your shell script for generating the "latest" dataset. Note that you may need to set DATA_FOLDER and PATH variables for your script to work.
- Use the [ansible.posix.synchronize](https://docs.ansible.com/ansible/latest/collections/ansible/posix/synchronize_module.html) module to copy the data from the host onto the control node. Alternatively, you could try running an rsync command directly with the "ansible.builtin.command" module.

Add a second play to that playbook. It should run on all webservers and just needs one task - copy the up to date datasets folder from the control node onto each host.

**Test it out**

Running this playbook should generate a "latest" dataset and synchronise the same data folder across all hosts. Try it out with an `ansible-playbook` command! Check it works by opening up "/latest" in your browser using the hosts' addresses. 

> If your shell script also deletes old datasets, you will want the "delete" option of syncrhonize/rsync to make sure that they get deleted on the control node/other hosts.

**Run it on a schedule**

To run this playbook on a schedule, you could manually modify your existing crontab, but let's keep going with Ansible. Use the [ansible.builtin.cron](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/cron_module.html) module to manage a cronjob.

<details markdown="1"><summary>Hint</summary>

Add a second play to your original playbook. It should target the control node itself and just needs one task, using that "cron" module to manage a cronjob. The cronjob should run your new "datasets-playbook.yml" playbook via Ansible. You should manually remove any existing cronjobs on your control node.

> Yes, your control node can manage itself! You can do this by specifying `hosts: localhost` when writing a play.

</details>

### Automation Part 2

The CEO has come back with some more requirements for you.

* At midnight everyday the script should create a summary of the last 24 hours. It should be called `yesterday`.
* Every hour during the day a new dataset called `today` should be produced. It should show all earthquakes since the most recent midnight.

#### Hints
* The USGS has more than just the hourly data feed.
* You might want to add some parameters to your script, if you haven't already.

### Document `cliapp`

If you run `cliapp --help` you'll see lots of different options that can be passed to the program. We've documented some of these in [cliapp_reference.md](./cliapp_reference.md) but some are a complete mystery.

Can you work out what they do and complete the documentation?

### Create an API

While command line programs are very useful in certain circumstances they do have their limitations. We would like to create an API that makes the data produced by `cliapp` available as JSON, rather than through the rather clunky `webapp`.

Use your existing knowledge of Flask (and documentation from the internet) to create an API to run alongside the webapp. You can work on it locally to start with (remembering any extra requirements e.g. to set a DATA_FOLDER environment variable). Your initial goal should be to create an endpoint to `GET` a dataset based on its name:
```
// e.g. GET the dataset called `edge-throw-except`
GET http://localhost/dataset/edge-throw-except
```

To deploy it to the Ansible hosts, you could publish your app to a public repository and then get Ansible to pull it onto the hosts, install dependencies and start up the app.

Once you have this working consider other improvements. A `GET` endpoint to list all the currently available datasets? A `POST` endpoint that accepts input data and options and invokes `cliapp` with them? There are lots of possibilities.
