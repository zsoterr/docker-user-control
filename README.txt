# docker-user-control
Control docker command usage/user activity on docker node(s)

Goal:
Contol docker command usage on docker node(s) where awx containers are running.


Deployment:

I. Requirements, FYI:
You should start here: https://docs.docker.com/engine/extend/plugins_authorization/#basic-principles
I've validated everything on Centos v7, using a default installation, the OS up-to-date: with fresh packages (at 31.08.2020).


II. Installation:
1.) Install and configure auth. plugin:
We will start here: https://github.com/twistlock/authz

You have two option: you can create a service or you can build and run a container for this purpose. Check the git repository above.
IMPORTANT: I realized if neither service nor container re-read the policy.json file (after a changes in the file). That means, if you edit this file you have to restart the service or container.

Create the policy file in desired path, for example: /var/lib/authz-broker/policy.json with all permissions (no control regardig users' activity):
cat /var/lib/authz-broker/policy.json
{"name":"policy_1","users":[""],"actions":[""]}


Installation, options:
a.)Authz as service:
IMPORTANT:
- the installation's steps (based on the Readme in the git repository) doesn't work.
- The binary is not available in the cloned repository.
There seems to be a problem with install.sh file: I had to fix it. You can use my updated file or you sould edit by yourself - before you run that.
--> first, you have to build the container image (meanwhile the building, the necessary binary file will be placed the right place,automaticaly)
-->after that you have to edit the install.sh and copy the binary to /usr/bin/ from .../src/github.com/twistlock/authz/bin/authz-broker and run the script
For example:
My updated install.sh file:
<
$ cat install.sh
#!/usr/bin/env bash
name=twistlock_authz_broker
cp /home/$USERNAME/git-external/docker/auth/src/github.com/twistlock/authz/bin/authz-broker /usr/bin/
cat <<SERVICE > "/lib/systemd/system/twistlock-authz.service"
[Unit]
Description=Twistlock docker authorization plugin
After=syslog.target
[Service]
Type=simple
ExecStart=/usr/bin/authz-broker
[Install]
WantedBy=multi-user.target
SERVICE
>
- further steps  ( based on the readme in git repository, under "Running as a stand-alone service"), like update system file of docker daemon: It seems, work normally.
PS: I'm working on Centos v7, so the daemon configuration file, in my case: /etc/systemd/system/docker.service.  Relevant part in the file: ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --authorization-plugin=authz-broker
PS2: Don't forget to run these commands - after you edited the configuration file of docker daemon-: systemctl daemon-reload&&systemctl restart docker.service
Maybe useful for you:
systemctl status twistlock-authz.service
ll /run/docker/plugins/authz-broker.sock


b.)Authz as container:
Build the container:
- I had to install some packages on docker node: yum install -y go gcc gcc-c++ kernel-devel
- download the repository, link: above,
export GOPATH=/home/$USERNAME/git-external/docker/auth/authz
(Of course, thet GOPATH depends on your requirement.)
export PATH=$PATH:$GOPATH/bin
cd /home/$USERNAME/git-external/docker/auth/authz
go get github.com/twistlock/authz : I got this message: "package github.com/twistlock/authz: no Go files in /home/$USERNAME//git-external/docker/auth/other2/src/github.com/twistlock/authz" - but it seems it is not critical, beacause - at the end- the image has been built succesfully
go get github.com/tools/godep
cd src/github.com/twistlock/authz
// godep restore: I can't run it. I got this message (it seems we can skip this step): "godep: open Godeps/Godeps.json: no such file or directory
make all
- check if the image exist: docker images
PS: If you want you can use the pre-built image, you can find here: docker pull zsoterr/docker:twistlock-authz-broker-1-0

Run the container:
docker run -d --name=authz-broker --restart=always -v /var/lib/authz-broker/policy.json:/var/lib/authz-broker/policy.json -v /run/docker/plugins/:/run/docker/plugins twistlock/authz-broker
or (if you use our "pre-built" image):  docker run -d --name=authz-broker --restart=always -v /var/lib/authz-broker/policy.json:/var/lib/authz-broker/policy.json -v /run/docker/plugins/:/run/docker/plugins zsoterr/docker:twistlock-authz-broker-1-0

Update the configuration files:
- /var/lib/authz-broker/policy.json
- /etc/systemd/system/docker.service
relevant part: ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --authorization-plugin=authz-broker
- systemctl daemon-reload
- systemctl restart docker.service

If everything goes well, go to next step.


2.) Remote API,TLS - for docker daemon:
You should start here:
https://docs.docker.com/engine/extend/plugins_authorization/#basic-principles
https://docs.docker.com/engine/security/https/

Installation:
1. Create a CA:
openssl genrsa -aes256 -out ca-key.pem 4096
//When you are aksed, enter pass phrase for ca-key.pem and keep it in a safe place!
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
//You can set different value like 365 days.

//You have to add the necessary informations like Country Name,etc. Common Name: add your server's hosname/DNS name (where docker daemon is running).
openssl genrsa -out server-key.pem 4096
2. Create a server key:
openssl genrsa -out server-key.pem 4096
Create a CSR:
HOST=`hostname`
openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr
SERVERIP=$PutHereYourServerIPAddress
echo subjectAltName = DNS:$HOST,IP:$SERVERIP,IP:127.0.0.1 >> extfile.cnf
//for TLS: You have to add docker host name/DNS name and IP addres(ses): for example, in this case the connections will be allowed using these IP addresses
echo extendedKeyUsage = serverAuth >> extfile.cnf
Sign the certificate:
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf
//You can set different value like 365 days.

3. Create one (or more: client key/user) client key:
Run this command(s) as appropiate user!:
mkdir $HOME/.docker
cd $HOME/.docker
openssl genrsa -out key.pem 4096
echo extendedKeyUsage = clientAuth > extfile-client.cnf
Create a CSR for user:
openssl req -subj '/CN=$PutHereTheUserName' -new -key key.pem -out client.csr
Sign the CSR:
Switch to right user (like root) - if needed -:
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile-client.cnf
//If needed change the paths or copy user's CSR and extfile-client.cnf files to "CA directory" for the signing.
If everything goes well:
- remove the unnecessary files: rm -v client.csr server.csr extfile.cnf extfile-client.cnf
- set the right permission on key files:
as root: chmod -v 0400 ca-key.pem server-key.pem;chmod -v 0444 ca.pem server-cert.pem cert.pem
as user: chmod -v 0400 ~/.docker/key.pem
- copy the client certificate (cert.pem) and ca.pem to right user directory, like : /home/$USERNAME/.docker/
and change the file owner of signed certificate to right user:
chown $AffectedUSERNAME. /home/$AffectedUSERNAME/.docker/cert.pem

Load necessary variables: fo example: export DOCKER_HOST=tcp://127.0.0.1:2376&&export DOCKER_TLS_VERIFY=1&&export DOCKER_CERT_PATH=~/.docker

4. Configure docker daemon for TLS authentication:
Have a look this documentation: https://tech.paulcz.net/blog/secure-docker-with-tls/
Steps:
- Predetermine the location of docker daemon configuration file. For example: systemctl status docker
- Edit this file, for example:
// Backup the original service file,please.
in my case, on Centos v7.: vi /etc/systemd/system/docker.service or  vi /usr/lib/systemd/system/docker.service
Update the right part of configuration file. You should get similar to this: ExecStart=/usr/bin/dockerd -H=tcp://127.0.0.1:2376 --tlsverify --tlscacert=/srv/awx/config/ca.pem --tlscert=/srv/awx/config/server-cert.pem --tlskey=/srv/awx/config/server-key.pem --authorization-plugin=authz-broker
// PS: Maybe useful (if you would like to enhance the "security level of docker-enviroment" but not mandatory: control the access to socket file of docker daemon!: in this case (as my example contains above) you can remove this option! If you dont want, you can use this example: ExecStart=/usr/bin/dockerd -H=tcp://127.0.0.1:2376 -H fd:// --containerd=/run/containerd/containerd.sock --tlsverify --tlscacert=/srv/awx/config/ca.pem --tlscert=/srv/awx/config/server-cert.pem --tlskey=/srv/awx/config/server-key.pem --authorization-plugin=authz-broker
// PS2: In this case we also LIMIT the availability of docker daemon, using control via IP address. In my case, we don't want if the daemon will be reachable outside, remotely. If you dont want to limit this capability, use this sample: ExecStart=/usr/bin/dockerd -H=tcp://0.0.0.0:2376 -H fd:// --containerd=/run/containerd/containerd.sock --tlsverify --tlscacert=/srv/awx/config/ca.pem --tlscert=/srv/awx/config/server-cert.pem --tlskey=/srv/awx/config/server-key.pem --authorization-plugin=authz-broker
- reload the daemon:
systemctl daemon-reload&&systemctl restart docker.service

5. Update the json file (if needed):
For example:
vi /var/lib/authz-broker/policy.json
{"name":"policy_1","users":["test1user"],"actions":[""]}
{"name":"policy_2","users":["test2user"],"actions":["container_exec_start","container_exec_create","container_exec_inspect","container_inspect","container_start","container_stop","container_restart"]}

test1user: run all docker commands, without any control
test2user: he has control regarding which docker command are available for him/her

Options (which commands are useable?):
cat core/types.go |grep container_
ActionContainerArchive = "container_archive"
ActionContainerArchiveExtract = "container_archive_extract"
ActionContainerArchiveInfo = "container_archive_info"
ActionContainerAttach = "container_attach"
ActionContainerAttachWs = "container_attach_websocket"
ActionContainerChanges = "container_changes"
ActionContainerCommit = "container_commit"
ActionContainerCopyFiles = "container_copyfiles"
ActionContainerCreate = "container_create"
ActionContainerDelete = "container_delete"
ActionContainerExecCreate = "container_exec_create"
ActionContainerExecInspect = "container_exec_inspect"
ActionContainerExecStart = "container_exec_start"
ActionContainerExport = "container_export"
ActionContainerInspect = "container_inspect"
ActionContainerKill = "container_kill"
ActionContainerList = "container_list"
ActionContainerLogs = "container_logs"
ActionContainerPause = "container_pause"
ActionContainerRename = "container_rename"
ActionContainerResize = "container_resize"
ActionContainerRestart = "container_restart"
ActionContainerStart = "container_start"
ActionContainerStats = "container_stats"
ActionContainerStop = "container_stop"
ActionContainerTop = "container_top"
ActionContainerUnpause = "container_unpause"
ActionContainerWait = "container_wait"


6. Enchanced the security:
Reason: there are a few Colleagues whom will be able to become to root - in the future - , too. Not to much, but we have to also handle this situation.
6.1: Protect the CA's and server's files:
Goal: We protect the created files (for using TLS with docker)  with password.
Solution:
 Add the desired users to adminmembers group.
 Change the permisson on folder where you store the files: chgrp adminmembers /srv/awx/config/
 Create a compressed file and set the password on that:
  for example:
   tar -czvf ca4docker.tar.gz /srv/awx/config
   openssl enc -AES-256-CBC -in  ca4docker.tar.gz -out ca4docker.tar.gz.enc
   rm -rf /srv/awc/config/*
   mv ca4docker.tar.gz.enc /srv/awc/config/
   chgrp adminmembers /srv/awx/config/ca4docker.tar.gz.enc
If you need to decrypt the file (for example: you need to sign a new client certicate):
 openssl enc -AES-256-CBC -d -in ca4docker.tar.gz.enc -out ca4docker.tar.gz
 tar -xzf ca4docker.tar.gz
sign the client certificate and remove/delete all unpacked files excetption one, : ca4docker.tar.gz.enc

6.2: Protect user's .docker folder:
Goal: We store the necessary files - for using docker with TLS) in this folder: ~/.docker/ . We have to encrypt this folder to ensure if only who knows this password will be able to decrypt the folder and use docker command.
Solution:
 Install the necessary package on the node(s): yum install encfs 
 Create the folders within all affected users's folder and set the right permisson on those.
 Anything you will store in ~/de will be automatically encrypted and put into ~/en directory:
  encfs  /home/$AffectedUserName/en /home/$AffectedUserName/de
  → choose the "Paranoid configuration"  as "p" and set the dedicated password/user for ENCFS. Share this password with right user via safe-channel (like encrypted email).
  chown $AffectedUserName /home/$AffectedUserName/en /home/$AffectedUserName/de
  move necessary files (client and user cert, for example: ca.pem,cert.pem ) to /home/$AffectedUserName/de directory. Check the permission on these files (if needed set the right permission on those.)
ps: If you want to unmount the directory - manually- you should run this command, as right user: fusermount -u ~/de
Add the necessary steps to right files: we would like to ensure if the encrypted directory will be mounted and dismounted automatically:
 vi /home/$AffectedUserName/.bash_profile:
→ add this line to file: encfs -o nonempty ~/en ~/de
Don't forget to update the variable of docker ! : export DOCKER_HOST=tcp://127.0.0.1:2376&&export DOCKER_TLS_VERIFY=1&&export DOCKER_CERT_PATH=~/de
and set the timeout (the user's session will be disconected after 900 seconds inactivity:
TMOUT=900
 vi /home/$AffectedUserName/.bash_logout
→ add this line to file: fusermount -u ~/de

6.3: Limit usage of su command:
Goal: limit who is able to use su command (become to another user):
Solution:
6.3.1: Only wheel group's member:
Add the claimed user to wheel group: usermod -aG wheel $UserName
Edit the pam file: vi /etc/pam.d/su and ensure if this row is exist/enabled: auth required pam_wheel.so use_uid
6.3.2: Only defined group's member (in our case: adminmembers group):
Add the desired users to adminmembers group.
vi /etc/security/su-adminmembers-access
→ add the sshawx user to this file.
check if the permission (on the file) is right or not (0644 basically enough, it is owned by root): ls -l /etc/security/su-adminmembers-access
Edit the pam file: vi /etc/pam.d/su and add these rows to file:
 auth required pam_wheel.so use_uid group=adminmembers debug
 auth required pam_listfile.so item=user sense=allow onerr=fail file=/etc/security/su-adminmembers-access
