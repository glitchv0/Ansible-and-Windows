Ansible and Windows

In this post, I'm going to go over how I use ansible to manage Windows Servers.  I'm going to start with a playbook that I use to clone a server 2019 template in VMWare to a new domain-joined machine.  I'll try to make the playbook as easy to use so you can take it with little modification and use it yourself.

Don't worry this is going to be super easy, barely an inconvenience.

The first thing we need to do is create a Windows Server 2019 (or 2016 if you want).

The main thing you need to do for prep is to put the powershell script to enable ansible to connect on WinRM

https://github.com/ansible/ansible/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1

Save the file above in the C:\Windows\Temp directory.  Don't change the name of the file (if you do you'll have to change it in the playbook)

I'm not going to go into great detail on this.  I simply did a standard GUI install of 2019.  I installed VMWare Tools and renamed the machine Server-2019-template.  I did also fully update it with Windows Updates.  Make sure you leave the default administrator account enabled and remember what you set the password to as you will need it later.

DO NOT JOIN IT TO YOUR DOMAIN.

Once you have the template made, shut the machine down.  You don't need to Sysprep it or anything, simply shut it down.  Make sure it is a clean shutdown with no updates pending.

Once shutdown, in VMWare, right-click the machine and go to "Template > Convert to Template".  Click Yes on the confirmation box.
<image of a convert to template>

We also need to run that powershell on at least one of your domain controllers.  You can pick one to be the one you run your ansible tasks against or you can do it to all of them.  Up to you.

Now we have a bit of work to do in AWX before we can use the playbooks.  We will need to create new credential types and then add the credentials (I'll be adding mine to HashiCorp Vault).  Then we need to set up the project and then the template.


Let's get the project setup.  Go to projects in AWX then fill yours out like mine below.

<image of project>
```bash
SCM URL: https://github.com/glitchv0/Ansible-and-Windows.git

Hit save and it should grab the playbook.

Now we need to set up two new "Credential Types".  We have to do this because we need to use multiple sets of credentials to manipulate the machine and AWX/Tower only allows for one "Machine" password per template, so a new "Credential Type" is how you get around that.

Go to "Credential Types" in the left menu.  Click the plus in the upper right hand corner.  Now we are going to do two of these.

First is for our domain user.  Name is something like Domain User or yourdomain User.
In the "Input Configuration" box enter what I have below:
```
---
fields:
  - id: domain_username
    type: string
    label: Domain User
  - id: domain_password
    type: string
    label: Domain Password
    secret: true
required:
  - username
  - password

In "Injector Configuration" enter:
```
---
extra_vars:
  domain_password: '{{ domain_password }}'
  domain_username: '{{domain_username }}'

Now we need to create another credential type.  This one we are going to call localwin_password. Go back to Credential Types and click the plus again.
This time name it something like Local Windows Password.
In "Input Configuration" enter what I have below:
```
---
fields:
  - id: localwin_password
    type: string
    label: Local Windows Password
    secret: true
required:
  - password

In "Injector Configuration" enter the following:
```
---
extra_vars:
  localwin_password: '{{ localwin_password }}'

You may notice this one is a little different.  It only has a password and no username.

Now that we have the credential types created, we need to actually put the credentials into AWX.
On the left select "Credentials".  Click the plus in the top right.
For the name enter VMWare vCenter (or something similar).
Under "Credential Type" click the search button.  We want to select the type of VMWare vCenter.
This gives us 3 fields to fill in.  vCenter host is obviously your vCenter server, then the username and password.
I'm storing all my info in HashiCorp Vault so I'm going to map mine like that.  If you aren't using vault just enter the information.

The account you enter here needs to have enough access to clone templates, change hardware etc.  I'm using an admin account I made for ansible.

<screenshot of vcenter creds>

Now we need to create another credential.  This time we are going to make the Domain credentials.  This time when you select "Credential Type" make sure you select Domain User (or whatever you named your new "Credential Type" you made above.)

<image of domain user cred>

This user needs to have permission to join machines to your domain and to move computer objects to different OUs.  I made a service account for ansible and made it a domain admin.  The few places I've worked has done the same.  You need to decide if that works for your environment.  I would say at a minium make it a service account that can not log on interactively to the machines.

We need to create one last Credential.  Name this one something like Local Windows Password.  For the "Credential Type" select Local Windows Password (or whatever you named it above)
You'll notice this one only asks for a password.  Enter the password you set in the template for the admin user.
<image of local windows cred>


Now we need to create the template.  Go to Templates, click the plus in the top right corner and select Job Template.  Name it whatever you want.  I'll name mine New Windows VM.  Add a description if you like.  Set the job type to run.  Inventory can be anything, we are using localhost to do this work.  If you still have the "Demo Inventory" you can use that.  For Project select NewVM, then for Playbook, select NewVM.yml

Now we need to select those new credentials we created.  Click the search button.
In the pop up, change the "Credential Type" to "VMWare vCenter".  You should see your newly created credential, select it.
<image of select vcenter>

Click the search button beside credential again.  This time select Local Windows Password as the "Credential Type".  You should see the credential you created, select it.
<image of local winpassword select>

Click the search beside Credentials one more time.  This time select Domain User in the "Credential Type".  Select the credential you created earlier.
<image of domain user select>

Click Save.

Now comes the fun part.  We are going to build a Survey.  A survey is a form that will feed the job template the information as variables.  If you've looked at the playbook, you'll notice there are a lot of variables.

Click the "ADD SURVEY" button in the top middle of the job template page.
<img of add survey button>

We are going to create a question for all the variables we have in our playbook.  We will set defaults where we can.
These are the variables we need to create a question for:
```
server_name:
os_version:
cluster:
datacenter:
vmfolder:
datastore:
dns1:
dns2:
vmnetname:
serverip:
netmask:
gateway:
ram:
cpu_cores:
description:
domain:
tz:
dcname:
OU:

<img of server_name>
<img of os_version survery>
<cluster survey - put note about putting any clusters you have, I only have the one.>
<datacenter survey>
<vmfolder - put note that you can make it a multipel choice or text, if text folder must exist>
<image of datastore - recommend multple choice>
<image of dns1>
<image of dns2>
<image of vmnet name>
<image of server ip - note about getting this from their ipam/network team>
<image of netmask>
<image of gateway>
<image of ram>
<image of cpu_cores>
<image of description>
<image of domain>
<image of TZ - link to where to get TZ - https://docs.microsoft.com/en-us/previous-versions/windows/embedded/ms912391(v=winembedded.11)?redirectedfrom=MSDN>
<image of dcname>
<image of OU>
<image of company>
Now you could statically set some of these things in the playbook if you wanted to set them there and don't have multiple domains.

Now when you click launch, you will fill in all the survey questions and it should start cloning you a new VM.

If everything works correctly, you should get a successfully cloned machine, joined to your domain and ready to manage with Ansible.  In a much shorter blog post I'll cover how to setup an inventory for your Windows hosts and some of the setup you can do to ease the use of ansible and windows.

Feel free to contact me
<contact link>