# Using Ansible to Provision without Callbacks

When utilizing provisioning callbacks in Ansible Automation Platform (AAP), the main issue is that the host is required to be in the Inventory for it to work. This can pose an issue if you are provisioning from another utility that is unable to call into Ansible to add the host beforehand or if you are not using a dynamic inventory to sync the new host in.

There is an example playbook given to add the host to the inventory beforehand, but it comes with a few security concerns. First, it requires a username / password for AAP that needs a lot of access. This password is stored on the server image that is being provisioned in the clear. This access includes the ability to Admin an inventory, and directly launch your job template or workflow against the host (which may include passing in various data). The playbook is also fairly dated as it uses the URI module which requires it to look up all the proper IDs for things.

[Example Playbook](https://github.com/redhat-manufacturing/device-edge-workshops/tree/main/exercises/rhde_aw_120/2.1-kickstart-template#step-4---creating-a-call-home-playbook)

A slightly more secure approach is to create a playbook that does all the heavy lifting for you, and then have your server call it during provisioning. This way you can lock it behind a user that only has access to run the single main provisiong template and nothing else. This template then adds the calling server to the inventory and calls the appropriate provisioning template for that server type (assuming you have different provisioning playbooks for different server OSes or even application servers).

## Inventories
To start we will create 2 Inventories in this example
 - Blank
 - Provisioning Servers
 
The Blank inventory contains absolutely no servers. We will use it for the main provisioning playbook. This is necessary to have this playbook running via a different inventory (really you could pick any inventory) as we will be cleaning up the provisioning host afterwards.  If you use the same inventory for the provisioning hosts, you will run into the nonsensical error that you can't delete the host because it is being used by a running job, even though it isn't (the inventory itself is whats locked).

The other inventory will be blank also but we will be provisioning our hosts into here. I like to keep my hosts being provisioned separate until they are done, and then I make my specific provisioning playbook add them to the appropriate inventory later (or run a sync).

## Playbooks
There are 2 playbooks provided in this example.  Both require the [AWX.AWX](https://docs.ansible.com/ansible/latest/collections/awx/awx/index.html) collection. The first playbook is callback.yml. This playbook is what is ran from the server that is to be provisioned. You don't really need this playbook, as you can do the same thing with curl on Linux (or create a systemd service) or powershell on Windows. The jist of it is that you are going to call AAP and launch a job. You are going to use a username / password or you can do it with an OAuth token. We will not be using the Provisioning Callback feature of AAP (otherwise we wouldn't need these playbooks).  In this example we are storing the password in /etc/ and it should be ownable only by root. You could even make it clean up this file afterwards.

If you use my example playbook, it is basically looking for an INI file located at /etc/ansible/provision.yml that looks like this
```
[provision]
hostname=my_app_hostname
username=provision
password=provision
job_template=Provision Servers
server_type=RHEL
```

The 2nd playbook is the main provisioner, provision.yml. It requires you to create a template with a few things in it.

- Name: Provision Servers (or you can change to anything as long as your callback.yml calls the correct one)
- Inventory: Blank (this is the blank inventory or really any other inventory besides the provisioner will work)
- Credentials: (A credential that utilizes the type: Red Hat Ansible Automation Platform that has access to add to the appropriate inventory and run any of your existing provisioning playbooks)
- Variables: (set to prompt on launch)
- Concurrent Jobs: Checked

This main provisioner is going to need a few variables passed into the extra_vars.  You could make it pass it in via the survey instead (which is a bit more secure), but I am being a bit lazy for this example. We will be passing in 3 variables.
- server_type: This can be anything, whether the OS or Application type for the server that will be provisioned. I stick to OS in this example but it really just allows us to call different playbooks or pass in different credentials for different servers types. You could even do it based upon location, as I use completely different default credentials for my Azure, AWS, and VMWare hosts (but the same playbook).
- ipaddress: This is the IP of the server currently (even if you may change it later during the provisioning process)
- macaddress: We use the mac to set a temporary hostname in AAP.  The actual hostname of the server might not be set and we don't want to use localhost if so.

Inside our provision.yml we have some variables at the top that we will need to edit to match our environment. This is the stypes variable. Here you will set all your different server types and the appropriate playbook to call and what credentials to pass in.

Now we just need to fix our original Provisioning Playbooks that may be called from the stypes variables.  They will need a few things set to "Prompt on Launch" so that we can change things on the fly.
- Inventory: Set to Prompt on Launch. We need this because we are specifying the Inventory in the playbook, if you remove that, you can set this to the Provisioning Inventory and remove the check. By leaving it Prompt on Launch you can technically make the provision.yml change the inventory name for each server type also with a small modification.
- Variables: Prompt on Launch
- Credentials: Prompt on Launch (so we can specify different credentials for different servers types if we want)
- Limit: Prompt on Launch (so we can run it just against the server we are adding)
- Concurrent Jobs: True

So now this is order of events that things will happen
1) Server calls into AAP to provision itself
2) provision.yml adds server to a Provisioning Inventory with a temporary hostname based upon the mac address and sets the IP as a variable
3) provision.yml looks up the server_type in our vars data and determines which Job Template and Credentials to run it with
4) provision.yml launches real Provisioning job template with appropriate settings and sets the limit to the new host it created
5) Once the job is complete (it waits) it will remove the new host from the inventory

This exmaple could be expanded upon much further, but I will leave it to you to customize it for your needs
