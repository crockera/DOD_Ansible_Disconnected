# DOD-Ansible-Disconnected



This process was tested in a virtualized environment with the following details:

- Virtualization: KVM on RHEL 8
- OS: RHEL 8.5
- AAP: 2.1.1-2

## Host Deployment

Documenting the tested machine configuration for repeatablility.

### Software Selection:

- Minimal Install
- Additional software selected:
  - Guest Agents
  - Standard (Standard installation of Red Hat Enterprise Linux)

### Installation Destination

Due to the requirements of the RHEL 8 Installer's DISA STIG for RHEL 8 Security Profile additional volumes are required.

Required volumes (partitions):

- `/var`
- `/var/log`
- `/var/log/audit`
- `/var/tmp`
- `/tmp`

Example (tested) disk configuration:

- 120 GiB Volume
- Storage Configuration > Custom
- Disk encryption enabled

| Mount Point      | Example/Tested Size |
| ---------------- | ------------------- |
| `/`              | 56 GiB              |
| `/boot`          | 1024 MiB            |
| `/home`          | 10 GiB              |
| `/var`           | 24 GiB              |
| `/var/log`       | 5 GiB               |
| `/var/log/audit` | 10 GiB              |
| `/var/tmp`       | 3 GiB               |
| `/tmp`           | 5 GiB               |
| swap             | 5.99 GiB            |

### Kdump

Disabled, per the Security Profile requirements

### Network

- Hostname set
- Interface configured with static IP and set to connect automatically

### Security Policy

With the `Apply security policy` option `on`, highlight the `DISA STIG for Red Hat Enterprise Linux 8` security profile and click `Select profile`.

### User settings

Create an admin user and password as desired and set the root password.

---

# Preparing host for AAP installation

## Create offline repositories for RHEL 8

Using the RHEL DVD ISO as a source for the offline repositories, create a directory and mount the ISO.

```
$ sudo mkdir /media/rheldvd
$ sudo mount /dev/sr0 /media/rheldvd
```

Create a new repo file defining the RHEL 8 BaseOS and AppStream repositories.

```
$ cat << EOF | sudo tee -a /etc/yum.repos.d/rheldvd.repo
[dvd-BaseOS]
name=DVD for RHEL - BaseOS
baseurl=file:///media/rheldvd/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[dvd-AppStream]
name=DVD for RHEL - AppStream
baseurl=file:///media/rheldvd/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF
```

For good measure, clean the yum cache.

```
$ sudo yum clean all
```

Verify that that you get the packages list from the DVD repositories.

```
$ sudo yum --noplugins list
```

### References and Additional Resources

- [How Do I Install Ansible Automation Platform 2.1 in a Disconnected Environment from the Internet in a Single Node? (RH account login required)](https://access.redhat.com/solutions/6635021)
- [Need to set up yum repository for locally-mounted DVD on Red Hat Enterprise Linux 8 and above versions (RH account login required)](https://access.redhat.com/solutions/3776721)

---

# Prepare the bundled installation inventory

Extract the bundled installer and configure the inventory file.

```
$ pwd
/home/admin

$ tar xvf ansible-automation-platform-setup-bundle-*.tar.gz
$ mv ansible-automation-platform-setup-bundle-*/ setup-bundle/
$ cd setup-bundle/
$ vi inventory
```

### Example inventory

<pre><code>
# Automation Controller Nodes
# There are two valid node_types that can be assigned for this group.
# A node_type=control implies that the node will only be able to run
# project and inventory updates, but not regular jobs.
# A node_type=hybrid will have the ability to run everything.
# If you do not define the node_type, it defaults to hybrid.
#
# control.example node_type=control
# hybrid.example  node_type=hybrid
# hybrid2.example <- this will default to hybrid
<strong>[automationcontroller]</strong>
<strong>10.0.0.1</strong>

[automationcontroller:vars]
peers=execution_nodes


# Execution Nodes
# There are two valid node_types that can be assigned for this group.
# A node_type=hop implies that the node will forward jobs to an execution node.
# A node_type=execution implies that the node will be able to run jobs.
# If you do not define the node_type, it defaults to execution.
#
# hop.example        node_type=hop
# execution.example  node_type=execution
# execution2.example <- this will default to execution
[execution_nodes]

[automationhub]

[database]

[servicescatalog_workers]

# Single Sign-On
# If sso_redirect_host is set, that will be used for application to connect to
# SSO for authentication. This must be reachable from client machines.
#
# ssohost.example sso_redirect_host=&lt;host/ip&gt;
[sso]

[all:vars]
#required_ram=true
<strong>admin_password='XXXXXXXX'</strong>

pg_host=''
pg_port=''

pg_database='awx'
pg_username='awx'
<strong>pg_password='XXXXXXXX'</strong>
pg_sslmode='prefer'  # set to 'verify-full' for client-side enforced SSL

# Execution Environment Configuration
# Credentials for container registry to pull execution environment images from,
# registry_username and registry_password are required for registry.redhat.io
<strong>#registry_url='registry.redhat.io'</strong>
<strong>registry_url=''</strong>
registry_username=''
registry_password=''

# Receptor Configuration
#
receptor_listener_port=27199

# Automation Hub Configuration
#

<strong>automationhub_admin_password='XXXXXXXX'</strong>

automationhub_pg_host=''
automationhub_pg_port=''

automationhub_pg_database='automationhub'
automationhub_pg_username='automationhub'
automationhub_pg_password=''
automationhub_pg_sslmode='prefer'

# When using Single Sign-On, specify the main automation hub URL that
# clients will connect to (e.g. https://&lt;load balancer host&gt;).
# If not specified, the first node in the [automationhub] group will be used.
#
# automationhub_main_url = ''

# By default if the automation hub package and its dependencies
# are installed they won't get upgraded when running the installer
# even if newer packages are available. One needs to run the ./setup.sh
# script with the following set to True.
#
# automationhub_upgrade = False

# By default when one uploads collections to Automation Hub
# an admin needs to approve it before it is made available
# to the users. If one wants to disble the content approval
# flow, the following setting should be set to False.
#
# automationhub_require_content_approval = True

# At import time collections can go through a series of checks.
# Behaviour is driven by galaxy-importer.cfg configuration.
# Example are ansible-doc, ansible-lint, flake8, ...
#
# The following parameter allow one to drive this configuration.
# This variable is expected to be a dictionnary.
#
# automationhub_importer_settings = None

# The default install will deploy a TLS enabled Automation Hub.
# If for some reason this is not the behavior wanted one can
# disable TLS enabled deployment.
#
# automationhub_disable_https = False

# The default install will deploy a TLS enabled Automation Hub.
# Unless specified otherwise the HSTS web-security policy mechanism
# will be enabled. This setting allows one to disable it if need be.
#
# automationhub_disable_hsts = False

# The default install will generate self-signed certificates for the Automation
# Hub service. If you are providing valid certificate via automationhub_ssl_cert
# and automationhub_ssl_key, one should toggle that value to True.
#
# automationhub_ssl_validate_certs = False

# SSL-related variables

# If set, this will install a custom CA certificate to the system trust store.
# custom_ca_cert=/path/to/ca.crt

# Certificate and key to install in nginx for the web UI and API
# web_server_ssl_cert=/path/to/tower.cert
# web_server_ssl_key=/path/to/tower.key

# Certificate and key to install in Automation Hub node
# automationhub_ssl_cert=/path/to/automationhub.cert
# automationhub_ssl_key=/path/to/automationhub.key

# Server-side SSL settings for PostgreSQL (when we are installing it).
# postgres_use_ssl=False
# postgres_ssl_cert=/path/to/pgsql.crt
# postgres_ssl_key=/path/to/pgsql.key

# Keystore file to install in SSO node
# sso_custom_keystore_file='/path/to/sso.jks'

# The default install will deploy SSO with sso_use_https=True
# Keystore password is required for https enabled SSO
# sso_keystore_password=''

# Single-Sign-On configuration
sso_console_admin_password=''
</code></pre>

After saving the edited `inventory` file we can now move on to relocating the bundled installer to a directory where we can run the installation.

# Move the bundled installer and prepare the installation

To allow for the insallation to run we need to reloate the extracted bundled installer directory to the `/opt` directory, temporarily (until the system is rebooted) stop the `fapolicyd` service, remount the `/tmp` and `/var/tmp` volumes with exec priviledges, and install the `ansible-core` package from the bundled installer prior to running the `setup.sh` install script.

### Relocate the bundled installer

```
$ cd
$ pwd
/home/admin

$ sudo mv setup-bundle /opt/
```

### Stop the fapolicyd service

```
$ sudo systemctl stop fapolicyd.service
```

To check that fapolicy and been temporarily disabled run:

```
$ sudo systemctl status fapolicyd.service
```

### Remount the tmp directories with exec priveledges

```
$ sudo mount -o remount,exec /tmp
$ sudo mount -o remount,exec /var/tmp
```

### Allow user namespaces

For podman to work, we need to allow user namespaces as the STIG profile set this to `user.max_user_namespaces = 0`.

To fix this:

```
$ echo 10000 | sudo tee -a /proc/sys/user/max_user_namespaces
```

User namespaces need to be allowed following a system reboot. To accomplish this:

```
$ echo "user.max_user_namespaces = 10000" | sudo tee -a /etc/sysctl.d/99-usernamespaces.conf
```

### Install the ansible-core package

The bundled installer utilizes the `ansible-playbook` command to conduct the installation. First we create a yum repo pointing to the bundled installer's repos directory which contains the needed `ansible-core` rpm which privides the `ansible-playbook` command.

```
$ cat << EOF | sudo tee -a /etc/yum.repos.d/bundled-installer.repo
[bundle-repo]
name=AAP Bundled Installer Repo
baseurl=file:///opt/setup-bundle/bundle/el8/repos
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF
```

Clean the yum cache and install the `ansible-core` package

```
$ sudo yum clean all
$ sudo yum install -y ansible-core
```

### Setup an `ansible.cfg` file the installer

Due to the security profile we are not able to ssh to the host directly with the root user. To work around this we need to define a few settings in an `ansible.cfg` file inside of the bundled installer directory.

First lets get into the directory of the installer.

```
$ cd /opt/setup-bundle
```

Next, create the required `ansible.cfg` file.

```
$ cat << EOF > /opt/setup-bundle/ansible.cfg
[defaults]
remote_user = admin
ask_pass = true

[privilege_escalation]
become = true
become_ask_pass = true
EOF
```

And finally connect to the host IP with ssh since we are utilizing password authentication. This creates the needed entry in the `known_hosts` file.

```
$ sudo ssh admin@10.0.0.1
[sudo] password for admin: XXXXXXXX
The authenticity of host '10.0.0.1 (10.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.0.1' (ECDSA) to the list of known hosts.
...
admin@10.0.0.1's password: XXXXXXXX
...
$ exit
```

We need the `/var/log/tower` directory to exist for the installer. Let's create that.

```
$ sudo mkdir /var/log/tower
```

# setup.sh

We are now ready to run the installer.

```
$ cd /opt/setup-bundle
$ sudo ./setup.sh -- -k
[sudo] password for admin: XXXXXXXX
Using /opt/setup-bundle/ansible.cfg as config file
SSH password: XXXXXXXX
BECOME password[defaults to SSH password]: XXXXXXXX
```

### Patience

The following task takes quite a while to run, please be patient:

```
TASK [ansible.automation_platform_installer.repo_management : Copy bundle packages to repos source directory (legacy)]
```

After being stuck of this task for what seemed to be an abnormal amount of time I thought the process had timed our had entered a hung state.

To check on the progress of the copying of the bundled packages, in a different terminal session:

```
$ ls /opt/setup-bundle/bundle/el8/repos/{,repodata/,source/} | wc -l
```

will privied the number of items being copied to the destination directory.

The destination directory can be queried via:

```
$ sudo ls /var/lib/ansible-automation-platform-bundle/{,repodata/,source/} | wc -l
```

e.g.:

```
$ sudo ls /var/lib/ansible-automation-platform-bundle/{,repodata/,source/} | wc -l
73
$
```

Re-running this command a few moments later showed:

```
$ sudo ls /var/lib/ansible-automation-platform-bundle/{,repodata/,source/} | wc -l
75
$
```

On my system, the total runtime of this task was around 30-45 minutes.

Once the ansible-playbook has completed successfully, move on to the remainder of the steps to complete the installation.

# Complete the installation

For the remainder of these configurations we will become the root user.

```
$ sudo su -
```

### Place the execution environment container images

You need to place the execution environment container images into the `awx` user account. The images are bundled in the installer, `images/` directory. Copy them to the `awx` user home directory and load them into podman.

```
# cp -Rp /opt/setup-bundle/images /var/lib/awx/
# chown -R awx:awx /var/lib/awx/images
```

Now become the `awx` user.

```
# su - awx
```

As the `awx` user, load the images with podman using one of the two methods below:

#### Method 1

To accomplish this with a single command:

```
$ for image in images/ee-*-rhel8.tgz; do gzip -dc $image | podman load; done
```

Then confirm the images have been loaded and clean up:

```
$ podman images
REPOSITORY                                                            TAG         IMAGE ID      CREATED      SIZE
registry.redhat.io/ansible-automation-platform-21/ee-supported-rhel8  latest      ffdd50a269e6  5 weeks ago  1.08 GB
registry.redhat.io/ansible-automation-platform-21/ee-29-rhel8         latest      c41083287673  5 weeks ago  785 MB
registry.redhat.io/ansible-automation-platform-21/ee-minimal-rhel8    latest      3711e0ea1d33  5 weeks ago  396 MB

$ rm -rf images
```

#### Method 2

Individually load the execution environment images, confirm, and clean up:

<pre><code>
<strong>$ gzip -dc images/ee-29-rhel8.tgz | podman load</strong>
Getting image source signatures
Copying blob af092941766c done
Copying blob 9e93b678ebf2 done
Copying blob a65a1b01a4d2 done
Copying blob c02d758c2215 done
Copying config c410832876 done
Writing manifest to image destination
Storing signatures
Loaded image(s): registry.redhat.io/ansible-automation-platform-21/ee-29-rhel8:latest
<strong>$ gzip -dc images/ee-minimal-rhel8.tgz | podman load</strong>
Getting image source signatures
Copying blob a65a1b01a4d2 skipped: already exists
Copying blob 4fe50fe3a3b7 done
Copying blob c02d758c2215 skipped: already exists
Copying blob af092941766c skipped: already exists
Copying config 3711e0ea1d done
Writing manifest to image destination
Storing signatures
Loaded image(s): registry.redhat.io/ansible-automation-platform-21/ee-minimal-rhel8:latest
<strong>$ gzip -dc images/ee-supported-rhel8.tgz | podman load</strong>
Getting image source signatures
Copying blob a65a1b01a4d2 skipped: already exists
Copying blob c02d758c2215 skipped: already exists
Copying blob 4fe50fe3a3b7 skipped: already exists
Copying blob 9b2e1ea8a49b done
Copying blob af092941766c skipped: already exists
Copying config ffdd50a269 done
Writing manifest to image destination
Storing signatures
Loaded image(s): registry.redhat.io/ansible-automation-platform-21/ee-supported-rhel8:latest
<strong>$ podman images</strong>
REPOSITORY                                                            TAG         IMAGE ID      CREATED      SIZE
registry.redhat.io/ansible-automation-platform-21/ee-supported-rhel8  latest      ffdd50a269e6  5 weeks ago  1.08 GB
registry.redhat.io/ansible-automation-platform-21/ee-29-rhel8         latest      c41083287673  5 weeks ago  785 MB
registry.redhat.io/ansible-automation-platform-21/ee-minimal-rhel8    latest      3711e0ea1d33  5 weeks ago  396 MB
<strong>$ rm -rf images</strong>
</code></pre>

Exit out of the `awx` account, and back into the `root` account.

```
$ exit
```

Now edit the corrupted file `/etc/tower/conf.d/execution_environment.py` in order to configure the definitions of the execution environments.

Before:

```
# cat /etc/tower/conf.d/execution_environments.py
GLOBAL_JOB_EXECUTION_ENVIRONMENTS = [
    {'name': 'Default execution environment', 'image': '/ansible-automation-platform-21/ee-supported-rhel8:latest'},
    {'name': 'Ansible Engine 2.9 execution environment', 'image': '/ansible-automation-platform-21/ee-29-rhel8:latest'},
    {'name': 'Minimal execution environment', 'image': '/ansible-automation-platform-21/ee-minimal-rhel8:latest'},
]
CONTROL_PLANE_EXECUTION_ENVIRONMENT = '/ansible-automation-platform-21/ee-supported-rhel8:latest'
```

Add the registry hostname to each line according to the above execution environment images.

After:

<pre><code>
# cat /etc/tower/conf.d/execution_environments.py
GLOBAL_JOB_EXECUTION_ENVIRONMENTS = [
    {'name': 'Default execution environment', 'image': '<strong>registry.redhat.io</strong>/ansible-automation-platform-21/ee-supported-rhel8:latest'},
    {'name': 'Ansible Engine 2.9 execution environment', 'image': '<strong>registry.redhat.io</strong>/ansible-automation-platform-21/ee-29-rhel8:latest'},
    {'name': 'Minimal execution environment', 'image': '<strong>registry.redhat.io</strong>/ansible-automation-platform-21/ee-minimal-rhel8:latest'},
]
CONTROL_PLANE_EXECUTION_ENVIRONMENT = '<strong>registry.redhat.io</strong>/ansible-automation-platform-21/ee-supported-rhel8:latest'
</code></pre>

Finally reflect the file.

```
# awx-manage register_default_execution_environments
'Minimal execution environment' Default Execution Environment updated.
'Ansible Engine 2.9 execution environment' Default Execution Environment updated.
'Default execution environment' Default Execution Environment updated.
(changed: True)
```

You can now logout as the root user.

```
# exit
```

# Confirm the installation

Login to the Web UI of the automation controller. You will need to upload a subscription manifest file to activate the installation.

Refer to the product documentation for more details about importing the manifest file.

- [Import a Subscription](https://docs.ansible.com/automation-controller/latest/html/quickstart/import_license.html)

Your installation is complete and you can now run the `Demo Job Template` to see wheter it works as expected or not.
