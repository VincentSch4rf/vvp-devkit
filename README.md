## Prerequisites ##

Versions displayed below are required except for the OS

- Ansible :      2.2.1.0
- Vagrant :      1.9.4
- VirtualBox:    5.1.14
- kubectl:       1.10.0+
- OS:            OS X 10.13.3 / Debian 9.4


## Getting the required packages (Debian only) ##

The following commands should get everything set up:

``` bash
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt update -y
apt install -y build-essential linux-headers-$(uname -r) apt-transport-https dkms ansible git kubectl
wget https://download.virtualbox.org/virtualbox/5.1.38/virtualbox-5.1_5.1.38-122592~Debian~stretch_amd64.deb
dpkg -i virtualbox-5.1_5.1.38-122592~Debian~stretch_amd64.deb
apt-get install -f -y
/sbin/vboxconfig #If this fails, make sure you got the correct linux-headers and reboot
wget https://releases.hashicorp.com/vagrant/1.9.4/vagrant_1.9.4_x86_64.deb
dpkg -i vagrant_1.9.4_x86_64.deb
mkdir -p /opt/vvp
cd /opt/vvp
git clone https://github.com/nacho3c/vvp-devkit devkit
cd devkit
chmod +x ./setenv
```
 
## Installation ##

Add the following line into your local hosts file:
```
10.252.0.12 coreos-01.development.vvp.example.com
```

Select the required environment from the list when requested:
```
$ ./setenv
```

install the vvp custom box:
```
$ bin/vvp-install-box
```

start the infrastructure deployment 
```
$ vagrant up
```
Wait for vagrant to finish, then ssh into the coreos-01 vm.
You can watch the status of the coreos installation with:
```
watch -n10 "ps ax | grep coreos-install"
```
As soon as it finishes, the machine will reboot.

After the reboot, run the following commands on the coreos machine:
```bash
$ sudo tee -a /etc/systemd/network/static.network << 'EOF'
[Match]
Name=eth1

[Network]
Address=10.252.0.12
EOF
```

```
$ sudo tee -a /etc/hosts << 'EOF'
10.252.0.12 coreos-01.development.vvp.example.com
EOF
```

Reboot coreos

After rebooting, once again login to the coreos box.
Wait (about 15 min) for all of the docker containers and services to become fully up and ready.
You can monitor the syslogs using the command "journalctl -xef". Wait for activity in the syslogs to die down for a few minutes.
It's usually ready to go as soon as it repeatedly shows something like:
```
Jun 15 09:38:43 coreos-01 kubelet-wrapper[883]: W0615 09:38:43.577543     883 helpers.go:793] eviction manager: no observation found for eviction signal allocatableNodeFs.available
Jun 15 09:38:53 coreos-01 kubelet-wrapper[883]: W0615 09:38:53.607446     883 helpers.go:793] eviction manager: no observation found for eviction signal allocatableNodeFs.available
```
Exit the ssh session and install kubectl on your host machine if you haven't already.
Run `kubectl version` to check if kubectl can establish a connection.
The output should be somewhat similar to this:

```
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.4", GitCommit:"5ca598b4ba5abb89bb773071ce452e33fb66339d", GitTreeState:"clean", BuildDate:"2018-06-06T08:13:03Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"7+", GitVersion:"v1.7.16+coreos.0", GitCommit:"8daba243261316d9d15de0db88ac286e5ba1e09e", GitTreeState:"clean", BuildDate:"2018-04-06T16:06:15Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}
```

Then run the following command:
```bash
$ bin/vvp-deploy
```

> After the above deploy, it can take around **30 minutes** for everything to finish.

- To access the **ICE dashboard**, go to `https://10.220.220.12/#/`
- To access the **gitlab dashboard**, go to `http://10.252.0.12/`
- To access the **Jenkins dashbaord**, go to `http://10.252.0.12:8080`

> It is also recommended to setup port forwarding for port 22 on the coreos box

## USER GUIDE ##

To create an account
- Sign up like normal (email, pwd)
- No email will be sent, you can find the activation link with the command `kubectl logs <em-uwsgi pod name>`
- Paste the activation link into your browser to activate the new account
- login to your new account
- Add a public ssh key your profile

Before creating an engagement, you need to manually configure the jenkins container
- login as root to the running jenkins container
`$ docker exec -it -u root <jenkins container id> "/bin/sh"`
- add `10.252.0.12 dev-git.vvp.example.com` to `/etc/hosts`

From here, you have two options to setup the Jenkins validation script
1) If you are running locally and gitlab is not accessible from the internet, 
you can stop here and once a repo is created in gitlab you can mark it public.
2) If you need to keep the heat template repos private, you can modify `/usr/local/bin/ice-testengine` to use authentication
`http://$domain/$repo" master` change to `http://<administrator username>:<administrator password>@$domain/$repo" master`
- administrator username by default is `root`. `gitlab_admin_password` is in the unencrypted vault file.

> At this point, users can login to the ICE portal and create engagements.

Once an engagement has been created with your new user, it will create a corresponding project in gitlab.

- login to gitlab **with the admin credentials**
- You can grab the link that a user will need to clone and upload heat templates. Most likely ssh will not work, only http.
- In the previous step when you set up the jenkins container,
If you did not add authentication to the ice-testengine script, you need to mark the repo public.
**be aware** if youre not using a private instance, this could allow anyone to see the uploaded heat templates.
- clone the repo. Cloning may only work over http.
- add your heat templates and commit them to the repo
- This will start a jenkins job to validate your templates, and results will be posted to the portal once complete.

Login to the Engagement Manager portal as an administrator
- move engagement along
- check heat tempalte validation status
- approve, reject, check jenkins logs, etc...

