Deployment
**********

As stated in the above section, to use MiCADO, you need to deploy the MiCADO services on a (separate) virtual machine, called MiCADO master. We recommend doing the installation remotely i.e. to download the Ansible playbook on your local machine and run the deployment on an empty virtual machine dedicated for this purpose on your preferred cloud.

Prerequisites
=============

Git & Ansible 2.4 or greater are needed on your (local) machine to run the Ansible playbook.

**The version of Ansible in the Ubuntu 16.04 APT repository is outdated and insufficient**

Ansible
-------

Install Ansible on Ubuntu 16.04.
::

   sudo apt-get update
   sudo apt-get install software-properties-common
   sudo apt-add-repository ppa:ansible/ansible
   sudo apt-get update
   sudo apt-get install ansible

To install Ansible on other operation system follow the `official installation guide <#https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html>`__.

Installation
============

Perform the following steps on your local machine.

Step 1: Download the ansible playbook.
--------------------------------------

Currently, MiCADO v5 version is available.

::

   git clone https://github.com/micado-scale/ansible-micado.git ansible-micado
   cd ansible-micado
   git checkout master

Step 2: Specify credential for instantiating MiCADO workers.
------------------------------------------------------------

MiCADO master will use this credential to start/stop VM instances (MiCADO workers) to realize scaling. Credentials here should belong to the same cloud as where MiCADO master is running. We recommend making a copy of our predefined template and edit it. The ansible playbook expects the credential in a file, called credentials.yml. Please, do not modify the structure of the template!

::

   cp sample-credentials.yml credentials.yml
   vi credentials.yml

Edit credentials.yml to add cloud credentials. You will find predefined sections in the template for each cloud interface type MiCADO supports. Fill only the section belonging to your target cloud.

Optionally you can use the `Ansible Vault <#https://docs.ansible.com/ansible/2.4/vault.html>`_ mechanism to keep the credential data in an encrypted format. To achieve this, create the above file using Vault with the command

::

    ansible-vault create credentials.yml


This will launch *vi* or the editor defined in the ``$EDITOR`` environment variable to make changes to the file. If you wish to make any changes to the previously encrypted file, you can use the command

::

    ansible-vault edit credentials.yml

Step 3a: (Optional) Specify security settings and credentials.
--------------------------------------------------------------

MiCADO master will use these security-related settings and credentials during provisioning.

::

   cp sample-security-cred.yml security-cred.yml
   vi security-cred.yml

Specify the provisioning method for the x509 keypair used for TLS encryption of the management interface in the ``tls`` subtree:

* The 'self-signed' option generates a new keypair with the specified hostname as subject (or 'micado-master' if omitted).
* The 'user-supplied' option lets the user add the keypair as plain multiline strings (in unencrypted format) in the ansible_user_data.yml file under the 'cert' and 'key' subkeys respectively.

Specify the default username and password for the administrative we user in the the ``authentication`` subtree.

Specify the IPs or subnets which allowed to access the management interfaces. It can be subnet or IP or both. 

Optionally you may use the Ansible Vault mechanism as described in Step 2 to protect the confidentiality and integrity of this file as well.


Step 3b: (Optional) Specify details of your private Docker repository.
----------------------------------------------------------------------

Set the Docker login credentials of your private Docker registries in which your personal containers are stored. We recommend making a copy of our predefined template and edit it. The ansible playbook expects the docker registry details in a file, called docker-cred.yml. Please, do not modify the structure of the template!

::

   cp sample-docker-cred.yml docker-cred.yml
   vi docker-cred.yml

Edit docker-cred.yml and add username, password, and repository url. To login to the default docker_hub, leave DOCKER_REPO as is (a blank string).

Optionally you may use the Ansible Vault mechanism as described in Step 2 to protect the confidentiality and integrity of this file as well.

Step 4: Launch an empty cloud VM instance for MiCADO master.
------------------------------------------------------------

This new VM will host the MiCADO master core services. Use any of aws, ec2, nova, etc command-line tools or web interface of your target cloud to launch a new VM. We recommend a VM with 2 cores, 4GB RAM, 20GB disk. Make sure you can ssh to it (password-free i.e. ssh public key is deployed) and your user is able to sudo (to install MiCADO as root). Store its IP address which will be referred as ``IP`` in the following steps. The following ports should be open on the virtual machine:

::

   TCP: 22,2377,7946,8300,8301,8302,8500,8600
   UDP: 4789,7946,8301,8302,8600

Step 5: Customize the inventory file for the MiCADO master.
-----------------------------------------------------------

We recommend making a copy of our predefined template and edit it. Use the template inventory file, called sample-hosts for customisation.

::

   cp sample-hosts hosts
   vi hosts

Edit the ``hosts`` file to set ansible variables for MiCADO master machine. Update the following parameters: ansible_host=\ *IP*, ansible_connection=\ *ssh* and ansible_user=\ *YOUR SUDOER ACCOUNT*. To specify the listening port of the MiCADO web interface, please edit the *web_listening_port* parameter, which defaults to the default HTTPS port (443/TCP). Please, revise the other parameters as well, however in most cases the default values are correct. To change the default port of the MiCADO web management interface edit the value of the ``web_listening_port`` variable.


Step 6: Start the installation of MiCADO master.
------------------------------------------------

::

   ansible-playbook -i hosts micado-master.yml

If you have used Vault to encrypt your credentials, you have to add the path to your vault credentials to the command line as described in the `Ansible Vault documentation <#https://docs.ansible.com/ansible/2.4/vault.html#providing-vault-passwords>`_ or provide it via com mand line using the command
::

    ansible-playbook -i hosts micado-master.yml --ask-vault-pass


Health checking
===============

At the end of the deployment, core MiCADO services will be running on the MiCADO master machine. Here are the commands to test the operation of some of the core MiCADO services:

*  Prometheus:

::

    curl --insecure -s https://[username]:[password]@[IP]:[port]/prometheus/api/v1/status/config | jq '.status'

Check the logs
==============

Alternatively, you can SSH into MiCADO master and check the logs at any point after MiCADO is succesfully deployed. All logs are kept under ``/var/log/micado`` and are organised by component. Scaling decisions, for example, can be inspected under ``/var/log/micado/policykeeper``
