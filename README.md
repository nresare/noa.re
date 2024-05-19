# How to update the apt repository

* Make sure that the yubikey is in place. If it needs to be set up on a 
new computer, import the public key from http://noa.re/resare-signing-key-2024.asc and insert the yubikey. Should be enough, at least on ubuntu
* Use aptly to update the local repo state and publish an updated repo
** The [multi dist patch](https://github.com/nresare/aptly/tree/multi-dist)  must be used when building 
** `aptly repo create -distribution focal resare-focal`
** `aptly repo add resare-focal pam-ssh-agent_0.5.0_amd64.deb`
** `aptly snapshot create resare-focal-snapshot from repo resare-focal`
** `aptly publish snapshot -multi-distro resare-focal-snapshot`
** `rsync -a --progress --delete /home/noa/.aptly.2/public/ rachel.noa.re:/var/www/noa.re/static/repo`
