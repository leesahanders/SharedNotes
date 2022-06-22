Connect to it at: 
ec2-18-188-191-176.us-east-2.compute.amazonaws.com:3939

jen (jen123) <- user that is in >16 groups
julie (julie123)
john (john123)

RSC Cheat Sheet: 
```
sudo systemctl stop rstudio-connect
sudo systemctl start rstudio-connect
sudo systemctl status rstudio-connect
```
 - sudo nano /etc/rstudio-connect/rstudio-connect.gcfg
 - View logs with sudo tail -n 50 /var/log/rstudio-connect.log

sssd Cheat Sheet: 
```
sudo systemctl stop sssd.service
sudo systemctl start sssd.service
sudo systemctl status sssd.service
```
 - sudo nano /etc/sssd/sssd.conf
 - sudo tail -n 50 /var/log/sssd/sssd_LDAP.log


## What was run: 

#### Purge 
Stop the service with sudo systemctl stop rstudio-connect 

sudo apt-get purge rstudio-connect

Remove logs from `/var/log/rstudio-connect*`
```
cd /var/log/
sudo rm rstudio-connect-setup.install.log 
sudo rm rstudio-connect.access.log 
sudo rm rstudio-connect.access.log.1 
```
We can also do this programmatically: 
test it first:
```bash
find . -type f -name rstudio-connect\*
```
doing risky things: 
```bash
find . -type f -name rstudio-connect\* -exec rm {} \;
```
(will need to sudo)

Purge the database. When using PostgreSQL, drop the database used by Connect. You may also wish to remove the PostgreSQL user associated with Connect. 

Remove the [`Server.DataDir`](https://docs.rstudio.com/connect/admin/appendix/configuration/#Server.DataDir) directory. By default, this is `/var/lib/rstudio-connect`.
 - We moved it to DataDir = "/efs/rstudio-connect" 
 - Oohhh I think the folder I messed up with chown and now can't delete it, not even as root (because the user it was transferred to no longer exists?) I don't really want to unload and reload the efs storage, maybe I can just create a new folder name with a new install? 
```
sudo su - 
rm -rf /efs/rstudio-connect
```

Remove configuration files from `/etc/rstudio-connect` if they still exist.
```
rm -rf /etc/rstudio-connect
```

#### Reinstall
##### Base install stuff

Refer to: https://docs.rstudio.com/rsc/configuration/high-availability/

code run for installing more system dependencies: 
```
sudo apt-get update

sudo apt install libssl-dev libcurl4-openssl-dev openssl nano libxml2 libxml2-dev

sudo apt-get install gdebi-core
sudo apt-get install -y libpng-dev

sudo apt-get install realmd sssd sssd-tools samba-common  samba-common-bin samba-libs adcli ntp

sudo apt-get install nfs-common
```

Just in general this is a good one to install (needed for vs code): 
```
sudo apt install libuser1-dev
```

R:
```
export R_VERSION=4.1.3
export R_VERSION=4.2.0
export R_VERSION=3.6.3

curl -O https://cdn.rstudio.com/r/ubuntu-2004/pkgs/r-${R_VERSION}_1_amd64.deb 
sudo gdebi r-${R_VERSION}_1_amd64.deb

/opt/R/${R_VERSION}/bin/R --version
sudo ln -s /opt/R/${R_VERSION}/bin/R /usr/local/bin/R 
sudo ln -s /opt/R/${R_VERSION}/bin/Rscript 

/usr/local/bin/R --version

Optional: 
sudo /opt/R/${R_VERSION}/bin/Rscript -e 'install.packages(c("haven","forcats","readr","lubridate","shiny", "DBI", "odbc", "rvest", "plotly","rmarkdown", "rsconnect","pins","png","tidyverse", "Rcpp"), repos = "http://cran.us.r-project.org")'
```

quarto:
``` 
curl -LJO https://github.com/quarto-dev/quarto-cli/releases/download/v0.9.414/quarto-0.9.414-linux-amd64.deb

sudo gdebi quarto-0.9.414-linux-amd64.deb
```

Finally rsc itself: This time let's do it manually on rsc2
https://docs.rstudio.com/rsc/manual-install/
```
sudo apt-get install gdebi-core 
curl -O https://cdn.rstudio.com/connect/2022.05/rstudio-connect_2022.05.0~ubuntu20_amd64.deb 
sudo gdebi rstudio-connect_2022.05.0~ubuntu20_amd64.deb
```

Install system dependencies: 
```
sudo apt install -y make libpng-dev tcl tk tk-dev tk-table default-jdk libxml2-dev git libssl-dev libcurl4-openssl-dev libjpeg-dev imagemagick libmagick++-dev gsfonts unixodbc-dev zlib1g-dev libfreetype6-dev libfribidi-dev libharfbuzz-dev libicu-dev libmysqlclient-dev libgeos-dev libudunits2-dev libfontconfig1-dev libcairo2-dev libtiff-dev libgdal-dev gdal-bin libproj-dev libssh2-1-dev libglpk-dev libgmp3-dev python3 perl libv8-dev cmake libsodium-dev libglu1-mesa-dev libgl1-mesa-dev
```

activate license: 
```
sudo /opt/rstudio-connect/bin/license-manager activate F9SH-TMB9-9A4U-DK5N-I4QX-JYFF-T2TA
```

##### Share drive 

Mount via IP: <- Do this option (arbitrary chosen) 
```
sudo mkdir /efs

sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,lookupcache=pos,retrans=2,noresvport 172.31.9.183:/ /efs
```


##### Database

Let's install aws cli on my wsl ubuntu: 
```
sudo apt-get update
sudo apt-get install libc6
sudo apt install glibc groff less

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install

/usr/local/bin/aws --version

aws --version
```

Step 3 of: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-prereqs.html 

Skipped to: https://rstudiopbc.atlassian.net/wiki/spaces/ENG/pages/36343631/AWS+Single+Sign+On

```
aws configure sso
```

To use this profile specify it with: 
```
aws s3 ls --profile PowerUser-637485797898
```

Basically I will need to add  `--profile PowerUser-637485797898` at the end of each command I run unless I configure it to use my poweruser as default. 

sudo nano ~/.aws/config

For testing aws cli: 
```
aws sts get-caller-identity --profile PowerUser-637485797898

aws sts get-caller-identity
```

So now let's try running Trevor's commands: 
```
aws sso login

Original, don't run: 
aws rds create-db-instance --db-instance-identifier tn-test-ha-db --db-instance-class db.t3.medium --engine postgres --tags Key=rs:owner,Value=trevor.nederlof@rstudio.com Key=rs:project,Value=solutions --master-user-password XXXXXXXX --master-username postgres --allocated-storage 30 --vpc-security-group-ids sg-0ceef5f8950f6b8b1

Run this one: 
aws rds create-db-instance --db-instance-identifier lisa-training-postgres --db-instance-class db.t3.medium --engine postgres --tags Key=rs:owner,Value=lisa.anders@rstudio.com Key=rs:project,Value=solutions --master-user-password XXXXXXXX --master-username postgres --allocated-storage 30 --vpc-security-group-ids sg-0d1bbc189fa2e3a42

```

Well looks like it is created if I go to the rds dashboard I can see it. 

Create a database called connect inside our postgres instance (set tablespace to default during creation) - can use dbeaver for this. Be sure to select the "show all databse" option under the postgres tab when setting up the connection. 




##### LDAP and PAM 
Let's assume the ldap server is already set up (with openldap)

```
sudo apt install sssd-ldap ldap-utils

sudo apt install realmd

sudo apt install oddjob oddjob-mkhomedir sssd adcli samba-common 

sudo apt install sssd
```

sudo adduser ubuntu
sudo passwd ubuntu (ubuntu)

```
sudo apt install libnss-ldapd
systemctl restart systemd-logind
```

apt install libnss-ldapd
ldap://18.116.220.235
dc=example,dc=org
enter

sudo nano /etc/sssd/sssd.conf
```
[sssd]
config_file_version = 2
services = nss, pam
domains = LDAP

[nss]
filter_users = root,named,avahi,haldaemon,dbus,radiusd,news,nscd
filter_groups =

[pam]

[domain/LDAP]
id_provider = ldap
auth_provider = ldap
chpass_provider = ldap
sudo_provider = ldap
enumerate = true
# ignore_group_members = true
cache_credentials = false
ldap_schema = rfc2307
ldap_uri = ldap://18.116.220.235
ldap_search_base = dc=example,dc=org
ldap_user_search_base = dc=example,dc=org
ldap_user_object_class = posixAccount
ldap_user_name = uid

ldap_group_search_base = dc=example,dc=org
ldap_group_object_class = posixGroup
ldap_group_name = cn
ldap_id_use_start_tls = false
ldap_tls_reqcert = never
ldap_tls_cacert = /etc/ssl/certs/ca-certificates.crt
# ldap_default_bind_dn = cn=admin,dc=example,dc=org
# ldap_default_authtok = admin
access_provider = ldap
ldap_access_filter = (objectClass=posixAccount)
min_id = 1
max_id = 0
ldap_user_uuid = entryUUID
ldap_user_shell = loginShell
ldap_user_home_directory = homeDirectory
ldap_user_uid_number = uidNumber
ldap_user_gid_number = gidNumber
ldap_group_gid_number = gidNumber
ldap_group_uuid = entryUUID
ldap_group_member = memberUid
ldap_auth_disable_tls_never_use_in_production = true
use_fully_qualified_names = false
ldap_access_order = filter
debug_level=6
```
sudo chmod 600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf

sudo nano /etc/nsswitch.conf
```
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files systemd sss
group:          files systemd sss
shadow:         files sss
gshadow:        files

hosts:          files dns
networks:       files

protocols:      db files
services:       db files sss
ethers:         db files
rpc:            db files

netgroup:       nis sss
sudoers:        files sss
automount:      sss

```


In /etc/ldap.conf, set your client machine to use SSL to connect to LDAP and also allow the self-signed certificate.
sudo nano /etc/ldap/ldap.conf
```
TLS_CACERT      /etc/ssl/certs/ca-certificates.crt

URI ldaps://18.116.220.235:389
TLS_REQCERT never
```

```
sudo pam-auth-update --enable mkhomedir

sudo systemctl stop sssd.service
sudo systemctl start sssd.service
sudo systemctl status sssd.service
```

test it
getent passwd john
should return: 
john:*:1003:500:john:/home/john:/bin/sh


##### Config

Update our config: 
sudo nano /etc/rstudio-connect/rstudio-connect.gcfg
alt a, right arrow, ctrl k to delete multiple lines all at once 
```
[Server]
RVersionScanning = false
RVersion = "/opt/R/3.6.3"
RVersion = "/opt/R/4.2.0"
RVersion = "/opt/R/4.1.3"
DataDir = "/efs/connect"
Address = "http://ec2-18-188-191-176.us-east-2.compute.amazonaws.com"

[Metrics]
DataPath = "/efs/connect/metrics/rsc2"

[Python]
Enabled = FALSE
;Executable = "/opt/Python/3.10.4/bin/python"
;Executable = "/opt/Python/3.7.7/bin/python"
;Executable = "/opt/Python/3.9.6/bin/python"

[Quarto]
Enabled = true
Executable = "/usr/local/bin/quarto"

[HTTP]
; RStudio Connect will listen on this network address for HTTP connections.
Listen = ":3939"
NoWarning = true

[Applications]
RunAsCurrentUser = true
; limits number of bundles retained per application
BundleRetentionLimit = 2

[Authentication]
Provider = "pam"

[PAM]
Service = rstudio-connect
UseSession = true
ForwardPassword = true
PasswordLifetime = 12h
;AuthenticatedSessionService = rstudio-connect
;SessionService = rstudio-connect-session
# Allows root to su without passwords (required)
;auth sufficient pam_rootok.so

[Database]
Provider = "postgres"

[Postgres]
URL = "postgres://postgres:XXXXXXXX@lisa-training-postgres.cpbvczwgws3n.us-east-2.rds.amazonaws.com/connect>
;URL = "postgres://username:password@db.seed.co/connect"
;URL = "postgres://postgres:XXXXXXXX@lisa-training-postgres.cpbvczwgws3n.us-east-2.rds.amaz>

;[SQLite]
;Dir = "/var/lib/rstudio-connect/db"

[RPackageRepository "CRAN"]
;URL = https://packagemanager.rstudio.com/all/__linux__/focal/2022-05-31+Y3Jhbi>
URL = https://packagemanager.rstudio.com/all/__linux__/focal/latest
;URL = http://3.13.68.185:4242/prod-cran/__linux__/focal/latest

[Packages]
;External = shiny
;External = Rcpp

```

##### Migrate

Make sure the server is stopped: sudo systemctl stop rstudio-connect 

Run the migration with: 
```
sudo /opt/rstudio-connect/bin/migrate db -yes
```

If need to wipe database run: 
```
sudo /opt/rstudio-connect/bin/migrate db --drop-all -yes
```

If the terminal ever gets funky try stty sane (not correctly moving to a new line)

Let's add a path for metrics 
```
sudo mkdir /efs/connect/
sudo mkdir /efs/connect/metrics/rsc1
sudo mkdir /efs/connect/metrics/rsc2
```
Enable path in the above config

```
sudo systemctl stop rstudio-connect
sudo systemctl start rstudio-connect
sudo systemctl status rstudio-connect
```
View logs with `sudo tail -n 50 /var/log/rstudio-connect.log`


##### Load Balancer 

This can be done through the amazon GUI. The key things are: 
 - Target group with our instances 
	 - Set the health check to go to **/__ping__**
	 - Register each target to port 3939
	 - Overall protocol : port is HTTP: 3939
 - Load balancer (application)
	 - Listener on port 80 set to forward to our target group

Be sure to check these settings: 
 - Sticky sessions are added to the target group (under the attributes tab) 
 - Set Desync mitigation mode to 'Monitor' in the load balancer settings 


This could also be done with the cli. Refer to this which was used for package manager: 
So now just to configure the load balancer and hook it up. If there is a way to do this with the amazon cli I think that would be a life saver. Package manager uses 4242. This documentation is really good: https://github.com/awsdocs/elb-application-load-balancers-user-guide/blob/master/doc_source/tutorial-application-load-balancer-cli.md 

rspm instance id's: 
i-054aca1f9c64b10d2 
i-08dad4166c7ef1136

```
aws sso login

#create target group
aws elbv2 create-target-group \
    --name lisa-training-rspm-target \
    --protocol HTTP \
    --port 80 \
    --target-type instance \
    --tags Key=rs:owner,Value=lisa.anders@rstudio.com Key=rs:project,Value=solutions \
    --vpc-id vpc-98c857f3 

#It returns us the id that we need to plug in later: 
arn:aws:elasticloadbalancing:us-east-2:637485797898:targetgroup/lisa-training-rspm-target/01ae44edf12da7c4

#register our targets
aws elbv2 register-targets \
    --target-group-arn arn:aws:elasticloadbalancing:us-east-2:637485797898:targetgroup/lisa-training-rspm-target/01ae44edf12da7c4 \
    --targets Id=i-054aca1f9c64b10d2 Id=i-08dad4166c7ef1136

#create load balancer
aws elbv2 create-load-balancer --name lisa-training-rspm-lb \
--subnets subnet-60d4291d subnet-66f87f0d \
--security-groups sg-0d1bbc189fa2e3a42 sg-45b41633 \
--tags Key=rs:owner,Value=lisa.anders@rstudio.com Key=rs:project,Value=solutions

#It returns the id that we need to plug in later: 
arn:aws:elasticloadbalancing:us-east-2:637485797898:loadbalancer/app/lisa-training-rspm-lb/b38169d56441a930

#create a listener (4242, for rspm)
aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:us-east-2:637485797898:loadbalancer/app/lisa-training-rspm-lb/b38169d56441a930 --protocol HTTP --port 4242  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-2:637485797898:targetgroup/lisa-training-rspm-target/01ae44edf12da7c4

#create a listener (80)
aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:us-east-2:637485797898:loadbalancer/app/lisa-training-rspm-lb/b38169d56441a930 --protocol HTTP --port 80  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-2:637485797898:targetgroup/lisa-training-rspm-target/01ae44edf12da7c4
```


##### Testing
the rsc1 instance is broken for some reason, but rsc2 is chugging along. 

Connect to it at: 
ec2-18-188-191-176.us-east-2.compute.amazonaws.com:3939

jen (jen123)
julie (julie123)
john (john123)





