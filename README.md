---
<!-- Create formatted output with:
    pandoc --embed-resources --standalone -f markdown -t latex -o README.pdf README.md
-->

# On-air FM Monitor and Silence Detector From (almost) Scratch #

## Overview ##

This set of [Ansible](https://ansible.com/) playbooks aims to build a general-purpose FM broadcast monitoring device from a bare-bones Ubuntu installation.

The Ansible playbooks are organized into two roles: "_common_" and "_fm-monitor_".

## Sensitive Data ##

Note that the files in _host\_vars_ likely contain passwords and other sensitive information. The example in this repository uses nonsensical values for these. You should take due precautions when creating a public copy of this repository.

## Organization ##

The top-level playbook is _all.yml_. This playbook simply "includes" the roles and, at the end, reboots the target device. Run the entire playbook with the command

    ansible-playbook -v --limit=<target-host-name> all.yml

### _ansible.cfg_ ###

Ansible is configured with the file _[ansible.cfg](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)_. The copy in the top-level directory of this has the highest precedence.

### _inventory.yml_ ###

Ansible "[inventories](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)" come in different forms. This repo uses a simple YAML file for its inventory. This file is optional; you may specify the name or IP address of your device on the `ansible-playbook` command line with something like `--inventory=192.168.1.3,` or `--inventory=my-host.name,`.

### Tags ###

The playbooks use Ansible tags to enable execution of subsets of the whole. Tags defined at the top level (and lower) include:

- timezone - set the device time zone
- packages - install and update system packages
- device_user - ensure the device user is configured correctly
- icecast - install and configure a local instance of the [Icecast](https://icecast.org/) stream server
- ffmpeg-icecast - install and configure [FFmpeg](https://ffmpeg.org/) to stream to an Icecast server
- rtl-fm-play - install the _rtl-sdr_ package and configure it to send the audio to [Pipewire](https://pipewire.org/)
- jack-plumbing - configure audio routing using [JACK](https://jackaudio.org/)
- silentjack - install the _silentjack_ package and configure it to do something silences is detected
- vu-meters - install and configure the _meterbridge_ package to display audio levels on the locally connected display
- reboot - reboot

Run the playbooks with one or more tags with the command (assuming "_target-host-name_" is included in the file _inventory.yml_ see [above](#inventory.yml) for details):

    ansible-playbook -v --limit=<target-host-name> --tags=comma,separated,list,of,tags all.yml

## Device-specific Data ##

Per-device data are kept in YAML files in the _host\_vars_ directory. Files are named by their inventory (and SSH) hostnames. See the example in _host\_vars/example.yml_ (I find it extremely helpful to keep host names, inventory names and SSH device names consistent).

Here are some parameters you will likely want to change:

- _device\_user_ - the Linux username that will "own" the scripts and services

- _device\_user\_group_ - the Linux group name for the above user

- _station\_timezone_ - set to the name of the time zone where your device lives (e.g., _America/Los\_Angeles_, _America/Chicago_, etc.)

- _icecast\_ports_ - a YAML array containing one or more (probably just one) port numbers the Icecast server listens on; the first item in this array is used in the _ffmpeg-icecast_ script when connecting to the Icecast server. Used in _roles/fm-monitor/templates/ffmpeg-icecast.j2_.

- the following parameters are used in _roles/fm-monitor/templates/rtl-fm-play-environment.j2_:

  - _station\_rtl\_fm\_gain_ - the gain setting used in _rtl\_fm_; defaults to 20

  - _station_frequency_ - example: _89.9M_

  - _sample_rate_ - example: _48000_

## Roles ##

The two existing roles are _common_ and _fm-monitor_. These roles contain playbooks, Jinja2 templates and files to be installed on the target device.

The bullet items below are taken directly from the Ansible task names.

### _common_ ###

_common_ includes the following tasks (listed in _roles/common/tasks/main.yml_):

- **Timezone.** - set the time zone on the target device

- **Packages.** - update packages; this playbook does not install any new packages

- **The device user.** - adds a username and associated environment (ZSH), SSH keys, and directories

  This playbook also uses the _loginctl_(1) command to enable "_lingering_" for the user. According to the _loginctl_ manual page, "This allows users who are not logged in to run long-running services," which we need for systemd services to remain running if the user is inadvertently logged out.

### _fm-monitor_ ###

_fm-monitor_ includes the following tasks (listed in _roles/fm-monitor/tasks/main.yml_):

- **Role-specific variables.** - this task brings in the some common default settings

- **The systemd service unit files.** - configures and installs the systemd user services. The list of services includes

  - ffmpeg-icecast
  - rtl-fm-play
  - silentjack
  - jack-plumbing

- **The jack-plumbing configuration and service.** - installs and configures the JACK plumbing service

- **The rtl-fm-play stuff.** - installs and configures the parts needed for running _rtl\_fm_ and sending its audio to the Pipewire audio subsystem

- **Icecast2.** - installs and configures Icecast on the monitor device; you may also specify an external (public) Icecast server if desired

- **FFmpeg-icecast.** - installs and configures FFmpeg to "package" the audio and send it to the desired Icecast server

- **Silentjack script.** - installs _silentjack_ and an associated _systemd_ user service, along with a script that is run when silence is detected

- **VU meters.** - installs the _meterbridge_ package and an associated autostart _XDG desktop_ file to start _meterbridge_ when the device user logs in

## Running the Playbooks ##

In order to successfully run _all.yml_ you need a baseline Ubuntu system. You need an Ubuntu Desktop at least 24.04 installation in order to make use of the Pipewire audio subsystem. Configure your preferred username during the base installation. This user must have SSH access and _sudo_(1) access (see next paragraph). It is most convenient to configure the initial user to use sudo without a password, but you can also use the _ansible-playbook_ option `--ask-become-pass` to provide this at playbook runtime.

Many of the Ansible "plays" require elevated privileges on the target device (`become: true`). Not included in the "roles" playbooks, but included in the top-level directory of this repository is an example playbook named _pam-ssh-agent-auth.yml_. This playbook and the associated template files install and configure the _libpam-ssh-agent-auth_ package. This package provides a method for authorizing _sudo_(1) users using their SSH keys and the SSH "_agent_". For all the details, see the _pam-ssh-agent-auth.yml_ playbook. See [this writeup](https://jpmens.net/2021/11/21/pam-ssh-agent-authentication-with-ansible/) for a good description of this technique; remember to add `ForwardAgent = yes` in the appropriate places in your _~/.ssh/config_.

## Changing the Playbooks ##

Of course these playbooks are not "one size fits all". After making changes to the playbooks, consider running _ansible-lint_ to verify the correct syntax of the playbooks, and `ansible-playbook -v --check --limit=<target-host-name> all.yml` to check the validity of your plays.

As of the date of this document, here is the output of `ansible-lint --strict --project-dir=. -v all.yml`:

	INFO     Collection paths was patched to include extra directories /home/dklann/.ansible/collections,/usr/share/ansible/collections,/usr/lib/python3.13/site-packages,/usr/lib64/python3.13/site-packages
	INFO     Set ANSIBLE_LIBRARY=/home/dklann/d/projects/spec/fm-monitor/ansible/.ansible/modules:/home/dklann/.ansible/plugins/modules:/usr/share/ansible/plugins/modules
	INFO     Set ANSIBLE_COLLECTIONS_PATH=/home/dklann/d/projects/spec/fm-monitor/ansible/.ansible/collections:/home/dklann/.ansible/collections:/usr/share/ansible/collections:/usr/lib/python3.13/site-packages:/usr/lib64/python3.13/site-packages
	INFO     Set ANSIBLE_ROLES_PATH=/home/dklann/d/projects/spec/fm-monitor/ansible/.ansible/roles:roles:/home/dklann/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles
	INFO     Executing syntax check on playbook all.yml (0.31s)

	Passed: 0 failure(s), 0 warning(s) on 15 files. Last profile that met the validation criteria was 'production'.
