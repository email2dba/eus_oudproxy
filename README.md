


# <p style="color:Green"> Install Enterprise User Security (EUS) using OUDProxy/OUD and OUDSM in Docker </p>

# <p style="color:red"> Steps to Install OUD

NOTE: Basic OUD server and OUDSM server installation on Docker code has been downloaded/copied from
      "Stefan Oehrli (oes) stefan.oehrli@trivadis.com" github source.

For more detailed information about OUD/OUDSM installation you can visit their page.

This Document will explain How to configured EUS using OUDproxy/OUD/OUDSM.

###  <p style="color:red"> httpd container creation.

Create httpd container to host installation software using docker.

#### create persistent storage directory for httpd container

Change HOST_DIR_NAME and OUD_SOFTWARE_DIR based on your environments.

```
OUD_SOFTWARE_DIR=/home/enduser/oudproject/docker_oud/oud/software
HOST_DIR_NAME=/vmdata2/nfsshare/oudproject/software
```

I have downloaded following software to create OUD/OUDSM images in OUD_SOFTWARE_DIR .

Note: Oracle Unified Directory version 12.2.1.4

* p28186730_139425_Generic.zip
* p30188352_122140_Generic.zip
* p32464056_180291_Linux-x86-64.zip
* p32730494_122140_Generic.zip
* p32750004_1601_Linux-x86-64.zip

#### copy software into httpd server Directory

```bash
cp ${OUD_SOFTWARE_DIR}/*.zip  ${HOST_DIR_NAME}
```
#### create httpd container

```bash
##create the container

docker run -dit --hostname orarepo --name orarepo  -p 8080:80 \
     -v  ${HOST_DIR_NAME}:/usr/local/apache2/htdocs/ \
      httpd:alpine
 ```
#### Verify httpd container

```bash
##get IP address of the container
orarepo_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' orarepo)
echo ${orarepo_ip}

curl http://${orarepo_ip}:80/index.html
```

###   <p style="color:red"> Create OUD Server



####  Create Dockerfile to build  OUD image


####  Build OUD image using Dockerfile and Softwares from httpd docker container  

```bash
cd ./oud

##Get the IP address of the local HTTP server:
orarepo_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' orarepo)
echo $orarepo_ip
#example : 72.17.0.2

docker build --add-host=orarepo:${orarepo_ip} --build-arg ORAREPO="${orarepo_ip}" -t oracle/oud:12.2.1.4.0 .

```

#### Execute(create) an Oracle Unified Directory POD/Container

NOTE:  oud-setup is executing by "62_create_oud_instance.sh" which has been created by "Stefan Oehrli".
In that script we do not have "--integration eus" parameter.
I have added this parameter before start container first time.
otherwise, after creation, we can include using the following command
manage-suffix update -h host -p adminPort -D "cn=directory manager" -j pwd.txt -X -n -b baseDN --integration eus

* creating OUD instance inside the Container

```bash
##Change HOST_DIR_INST1, HOST_DIR_SCR and HOST_DIR_CORE based on your environments.
HOST_DIR_INST1=/vmdata2/nfsshare/oudproject/inst1
HOST_DIR_SCR=/vmdata2/nfsshare/oudproject/inst1/scripts
HOST_DIR_CORE=/vmdata2/nfsshare/oudproject/inst1/corescripts

mkdir -p ${HOST_DIR_INST1}
mkdir -p ${HOST_DIR_SCR}
mkdir -p ${HOST_DIR_CORE}

cp ./oud/opt_oradba_bin_files  ${HOST_DIR_CORE}

docker run --name   ouddapps -d \
--hostname ouddapps \
-p 1389:1389 -p 1636:1636 -p 4444:4444 \
-e ADMIN_PASSWORD="mypassword25" \
-e ADMIN_USER="cn=directorymanager" \
-e BASEDN="dc=pelican,dc=home" \
-e OUD_INSTANCE=oud_dapps1 \
--volume  ${HOST_DIR_INST1}:/u01 \
--volume  ${HOST_DIR_SCR}:/u01/scripts \
--volume  ${HOST_DIR_CORE}:/opt/oradba/bin \
oracle/oud:12.2.1.4.0

## Add sample users.[we must run this ]
## update dc=pelican,dc=home to your base DN.


oud_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ouddapps)
echo $oud_ip

## add Sample users.
ldapmodify -p 1389 -D "cn=directorymanager" -w mypassword25 -a -c -f ./oud/EUSFiles/example4eususers.ldif

## Create Password Policy  for EUSAdmins

dsconfig create-password-policy --policy-name EUSAdmins \
--set password-attribute:userpassword \
--set default-password-storage-scheme:AES \
--set default-password-storage-scheme:Salted\ SHA-512 \
--type generic \
--hostName ${oud_ip} \
--port 4444 \
--bindDN "cn=directorymanager" \
--bindPasswordFile pwd.txt \
--trustAll \
--no-prompt

## Add eusadmin user account
## update dc=pelican,dc=home to your base DN.
ldapmodify -p 1389 -D "cn=directorymanager" -w mypassword25 -f ./oud/EUSFiles/eusadmin.ldif

ldapsearch -h  $oud_ip -p 1389 -D "cn=eusadmin,cn=oraclecontext" -w Welcome1 \
      -b cn=OracleContext,dc=pelican,dc=home   -s base "(objectclass=*)"

```

###  <p style="color:red"> Create OUDSM Server

#### copy OUDSM software installation files into the httpd mount points

As per Stefan Oehrli notes, we have to download the following software to install OUDSM.

```bash
OUD_SOFTWARE_DIR=/vmdata2/nfsshare/oudproject/oudsm/software
mkdir -p ${OUD_SOFTWARE_DIR}

##copy downloaded the following files into ${OUD_SOFTWARE_DIR}
## Following files are require to install OUDSM
#fmw_12.2.1.4.0_infrastructure_Disk1_1of1.zip
#p28186730_139425_Generic.zip
#p30188352_122140_Generic.zip
#p32464056_180291_Linux-x86-64.zip
#p32581859_122140_Generic.zip
#p32698246_122140_Generic.zip
#p32730494_122140_Generic.zip

```
#### Build OUDSM image using Dockerfile

NOTE: Softwares copied into httpd docker container volume

```bash

##Get the IP address of the local HTTP server:
orarepo_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' orarepo)
echo $orarepo_ip
#example: 172.17.0.2
cd ./oudsm
docker build --add-host=orarepo:${orarepo_ip} --build-arg ORAREPO="${orarepo_ip}" -t oracle/oudsm:12.2.1.4.0 .

```

#### Configure(Setup) an OUDSM Container

```bash
## change HST_DOMAIN_HOME based on your environment.
HST_DOMAIN_HOME="/vmdata2/nfsshare/oudproject/oudsm"
mkdir -p ${HST_DOMAIN_HOME}

docker run --name   oudsmdapps \
--hostname oudsmdapps \
-p 7001:7001 -p 7002:7002 \
-e ADMIN_PASSWORD="mypassword25" \
-e ADMIN_USER="weblogic" \
-e DOMAIN_HOME="/u01/domains/oudsm_dapps_domain" \
-e DOMAIN_NAME="oudsm_dapps_domain" \
--volume ${HST_DOMAIN_HOME}:/u01 \
oracle/oudsm:12.2.1.4.0


oudsmdapps_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' oudsmdapps)
echo $oudsmdapps_ip


docker exec -u oracle -it oudsmdapps bash --login

##--------------------------------------------------------------
## Instance Name      : oudsm_dapps_domain
## Instance Home (ok) : /u01/domains/oudsm_dapps_domain
## Oracle Home        : /u00/app/oracle/product/fmw12.2.1.4.0
## Instance Status    : up
## Console            : http://oudsmdapps:7001/oudsm
## HTTP               : 7001
## HTTPS              : 7002
##--------------------------------------------------------------

```

#### OUDSM  Console login details

```
oudsmdapps_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' oudsmdapps)
echo $oudsmdapps_ip

ouddapps_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ouddapps)
echo $ouddapps_ip


## use browser to login
http://$oudsmdapps_ip:7001/oudsm

Name=oud_dapps1
Server= $ouddapps_ip    <<$ouddapps_ip>>
AdminPort = 4444
Username = cn=directorymanager
Password = mypassword25


##Error : Java.security.certificateException: No Subject alternative names present
## solution
###copy file from container to local server
docker cp oudsmdapps:/u01/domains/oudsm_dapps_domain/bin/setDomainEnv.sh /tmp/setDomainEnv.sh

update line 525 like this
JAVA_OPTIONS="${JAVA_OPTIONS} -Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true "
export JAVA_OPTIONS
###copy file from local server to  container
docker cp /tmp/setDomainEnv.sh oudsmdapps:/u01/domains/oudsm_dapps_domain/bin/setDomainEnv.sh

##/u00/app/oracle/product/fmw12.2.1.4.0/wlserver/server/bin/
```
---------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------
### <p style="color:red"> Create OUDProxy Server

#### Execute(create) an OUD Proxy POD/Container

NOTE: oud-proxy-setup is executing by "62_create_oud_instance.sh" created by Stefan Oehrli.
In that script we do not have "--eusContext dc=example,dc=com" parameter.
I have added this before start container first time.

```bash

HST_PRXY_DIR=/vmdata2/nfsshare/oudproject/proxy1
HST_PRXY_SCR=/vmdata2/nfsshare/oudproject/proxy1/scripts
HST_PRXY_CORE=/vmdata2/nfsshare/oudproject/proxy1/corescripts

mkdir -p ${HST_PRXY_DIR}
mkdir -p ${HST_PRXY_SCR}
mkdir -p ${HST_PRXY_CORE}

cp oud/opt_oradba_bin_files/*  ${HST_PRXY_CORE}

docker run --name   oudproxy -d  \
--hostname oudproxy  \
-e OUD_INSTANCE=oud_proxy1 \
-e OUD_PROXY=TRUE \
-e PORT=2389 \
-e PORT_SSL=2636 \
-e PORT_REP=8989 \
-e PORT_ADMIN=5555 \
-e ADMIN_USER="cn=directorymanager" \
-e ADMIN_PASSWORD="mypassword25" \
-e BASEDN="dc=pelican,dc=home" \
-e SAMPLE_DATA=FALSE \
-p 2389:2389 -p 2636:2636 -p 5555:5555 \
--volume ${HST_PRXY_DIR}:/u01 \
--volume ${HST_PRXY_SCR}:/u01/scripts \
--volume ${HST_PRXY_CORE}:/opt/oradba/bin \
oracle/oud:12.2.1.4.0  

docker logs oudproxy
```

```
 ---------------------------------------------------------------
   OUD instance is ready to use:
   Instance Name      : oud_proxy1
   Instance Home (ok) : /u01/instances/oud_proxy1
   Oracle Home        : /u00/app/oracle/product/fmw12.2.1.4.0
   Instance Status    : up
   LDAP Port          : 2389
   LDAPS Port         : 2636
   Admin Port         : 5555
   Replication Port   : 8989
   REST Admin Port    : 8444
   REST http Port     : 1080
   REST https Port    : 1081

 ---------------------------------------------------------------
```

```bash
oudproxy_ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' oudproxy)
echo ${oudproxy_ip}
##172.17.0.5
#
```

#### Add Workflow in OUDProxy server instance

Step by step and association is as follows -

* 1. Creation of WorkFlow
* 2. Creation of Join WorkFlow Element
* 3. Creation of Proxy LDAP WorkFlow Element
* 4. Creation of LDAP Server Extension
* 5. Creation of Join Participant
* 6. Add workflow to a Network Group

##### Creation of WorkFlow  and  Creation of Join WorkFlow Element
```
docker exec -it ${oudproxy_ip} /bin/bash

## change dc=pelican,dc=home based on you base_dn.  

dsconfig

>>>> Oracle Unified Directory configuration console main menu
##Choose
General Configuration

>>>> General Configuration management menu
## choose
"workflow"


>>>> Workflow management menu
##choose
Create a new Workflow

>>>> Enter a name for the Workflow that you want to create: WorkFlow1
>>>> Configuring the "base-dn" property
Specifies the base DN of the data targeted by the Workflow .
Syntax: DN
Enter a value for the "base-dn" property: dc=pelican,dc=home

>>>> Configuring the "enabled" property
Indicates whether the Workflow is enabled for use in the server.
##choose
true

>>>> Configuring the "workflow-element" property
## choose
Create a new Workflow Element

>>>> Select the type of Workflow Element that you want to create:
##choose
Join Workflow Element

>>>> Enter a name for the Join Workflow Element>>>> Configuring the "enabled" property
that you want to create: JoinWorkFlowElement1
##choose
True

>>>> Configuring the "join-suffix" property
The virtual DN that will be exposed by the Join Workflow Element
Syntax: DN
Enter a value for the "join-suffix" property: dc=pelican,d=home
>>>> Configure the properties of the Join Workflow Element
choose
f) finish - create the new Join Workflow Element

>>>> Configure the properties of the Workflow
##choose
f) finish - create the new Workflow

-----------------------------------------------------------------------------
```
##### Creation of Proxy LDAP WorkFlow Element and Creation of LDAP Server Extension
```
dsconfig

>>>> Oracle Unified Directory configuration console main menu
##Choose
Remote Data Source

>>>> Remote Data Source management menu
##Choose
LDAP Server Extension

>>>> LDAP Server Extension management menu
##Choose
Create a new LDAP Server Extension

>>>> Enter a name for the LDAP Server Extension that you want to create: LDAPServerExtension1
>>>> Configuring the "enabled" property
#choose
True

>>>> Configuring the "remote-ldap-server-address" property
Enter a value for the "remote-ldap-server-address" property: <HOSTNAME>
1) enabled                        true
2) remote-ldap-server-address     <HOSTNAME>
3) remote-ldap-server-port        <LDAP_PORT>
 <<<<<<<<<<<<<<<<< NOTE here once the Remote LDAP Server
Hostname is given by default it takes non-ssl port <LDAP_PORT>
which you must change accordingly >>>>>>>

>>>> Configuring the "remote-ldap-server-port" property
#type
Correct port number

>>>> Configure the properties of the LDAP Server Extension
#choose
f) finish - create the new LDAP Server Extension
#------------------------------------------------
dsconfig

>>>> Remote Data Source management menu
#choose
Proxy LDAP Workflow Element

>>>> Proxy LDAP Workflow Element management menu
#choose
Create a new Proxy LDAP Workflow Element

>>>> Enter a name for the Proxy LDAP Workflow Element that you want to create: ProxyLDAPWorkFlowElement1

>>>> Configuring the "client-cred-mode" property
##choose
use-client-identity

>>>> Configuring the "enabled" property
##Choose
True

>>>> Configuring the "ldap-server-extension" property
#choose
LDAPServerExtension1
>>>> Configure the properties of the Proxy LDAP Workflow Element

##choose
finish - create the new Proxy LDAP Workflow Element

##-----------------------------------------------------------------------------
```
##### Creation of Join Participant  
```
dsconfig
>>>> Oracle Unified Directory configuration console main menu
#choose
Virtualization

>>>> Virtualization management menu
##choose
Join Workflow Element

>>>> Join Workflow Element management menu
##choose
>Join Participant management menu

>>>> Join Participant management menu
##choose
Create a new Join Participant

>>>> Select the Join Workflow Element from the following list:
##Choose
JoinWorkFlowElement1

>>>> Enter a name for the Join Participant that you want to create: JoinParticipant1
>>>> Configuring the "participant-dn" property

Enter a value for the "participant-dn" property: dc=pelican,dc=home

>>>> Configuring the "participating-workflow-element" property

##Choose
ProxyLDAPWorkFlowElement1

>>>> Configure the properties of the Join Participant
##choose
primary-participant

##choose
True

>>>> Configure the properties of the Join Participant
f) finish - create the new Join Participant

##-----------------------------------------------------------------------------
```
##### Add workflow to a Network Group

```
dsconfig
##choose
General Configuration

>>>> General Configuration management menu
##choose
Network Group

>>>> Network Group management menu
##choose
View and edit an existing Network Group

>>>> There is only one Network Group: "network-group". Are you sure that this
is the correct one? (yes / no) [yes]:

>>>> Configure the properties of the Network Group
##choose
workflow

>>>> Configuring the "workflow" property
##choose
Add one or more values

Select the Workflows you wish to add:
##Choose
WorkFlow1

>>>> Configuring the "workflow" property (Continued)
##Choose
Use the value: WorkFlow1

>>>> Configure the properties of the Network Group
##choose
f) finish - apply any changes to the Network Group

##-----------------------------------------------------------------------------
```
##### Verify proxy worklow configuration
```
###---------------------------------------------------------------------
##Test the SSL connection using the following command:

##working code
##------------
## Following code must work

ldapsearch --useSSL -h ${oudproxy_ip} -p 2636  -b cn=OracleContext,dc=pelican,dc=home --trustall \
 -D "cn=eusadmin,cn=OracleContext"  -w Welcome1 -s base "(objectclass=*)" -v

ldapsearch --useSSL -h ${oudproxy_ip} -p 2636  -b dc=pelican,dc=home --trustall \
   -D "cn=eusadmin,cn=OracleContext"   -w Welcome1 -s sub "uid=user.1"  -v

ldapsearch --useSSL -h ${oudproxy_ip} -p 2636  -b dc=pelican,dc=home --trustall \
   -D "cn=directorymanager"   -w mypassword25 -s sub "uid=user.1"  -v

ldapsearch --useSSL -h ${oudproxy_ip} -p 2636  -b dc=pelican,dc=home --trustall \
   -D "cn=directorymanager"   -w mypassword25 -s sub "uid=user.1"  -v

###---------------------------------------------------------------------
```
#### Configuring the User and Groups Location

After Oracle Unified Directory has been configured for EUS or Oracle E-Business Suite, you must configure the naming context used to store the users and the groups by performing the following steps:

Locate the LDIF template file at install_directory/config/EUS/modifyRealm.ldif.
Edit the modifyRealm.ldif file as follows:

Replace dc=example,dc=com with the correct naming context for your server instance.

Replace ou=people and ou=groups with the correct location of the user and group entries in your DIT.

```

ldapmodify -h ${oudproxy_ip}  -p 2389 -D cn=directorymanager -j pwd.txt \
   -a -f ./modifyRealm.ldif

```


#### Add gcrep OracleNetService in oudproxy instance


```
cd  ./oud/EUSFiles

## Use this sample file and make changes as per your environment information.
vi gcrep_info.ldif
dn: cn=gcrep,cn=OracleContext,dc=pelican,dc=home
changetype: add
objectclass: top
objectclass: orclService
objectclass: orclDBServer
objectclass: orclDBServer_92
objectclass: orclApplicationEntity
orclservicetype: DB
userpassword: testusernathan14
cn: gcrep
orclsid: gcrep
orclsystemname: gcrep.pelican.home
orclnetdescname: 000:cn=DESCRIPTION_0
orcldbglobalname: gcrep
orcloraclehome: /apps/oracle/product/12.1.0.2
orclnetdescstring: (DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=testserver.dev.pelican.home)(PORT=1551)))(CONNECT_DATA=(SERVICE_NAME=testdb.pelican.home)))
orclversion: 122000
orclguid: 4EA7591CD133CA91E05018AC88607EA6


docker cp ./gcrep_info.ldif  oudproxy:/home/oracle/gcrep_info.ldif

docker exec -it oudproxy /bin/bash

ldapmodify --hostname  ${oudproxy_ip}  --port 2389 --bindDN "cn=directorymanager" \
  --bindPassword mypassword25 --filename ./gcrep_info.ldif

##Processing ADD request for cn=gcrep,cn=OracleNetAdmins,cn=OracleContext,dc=pelican,dc=home
##ADD operation successful for DN cn=gcrep,cn=OracleNetAdmins,cn=OracleContext,dc=pelican,dc=home

## From OUDSM display
## Distinguished Name  cn=gcrep,cn=OracleNetAdmins,cn=OracleContext,dc=pelican,dc=home
##  Created by  cn=directorymanager           Modified by cn=directorymanager
##  Created at  June 11, 2021 11:28:09 PM GMT Modified at June 11, 2021 11:28:09 PM GMT

```
##### verify gcrep netservice from oudproxy container.

```bash
ldapsearch --useSSL -h  ${oudproxy_ip} -p 2636  -b cn=OracleContext,dc=pelican,dc=home  --trustall \
   -D "cn=directorymanager"   -w mypassword25 "(cn=gcrep)"

dn: cn=gcrep,cn=OracleNetAdmins,cn=OracleContext,dc=pelican,dc=home
orclVersion: 122000
cn: gcrep
orclServiceType: DB
objectClass: orclApplicationEntity
objectClass: orclDBServer
objectClass: orclService
objectClass: top
objectClass: orclDBServer_92
orclOracleHome: /apps/oracle/product/12.1.0.2
orclNetDescString: (DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=tests
 erver.dev.pelican.home)(PORT=1551)))(CONNECT_DATA=(SERVICE_NAME=testdb.pelican.
 home)))
orclDBGlobalName: gcrep
userPassword: {SSHA}aePFebP2Q7eLjZgX8oEmBKgIItk95gdWBWlwBg==
orclcommonrpwdattribute: {SASL-MD5}v8L1YP5MKBmhBL8hDio6qA==
orclNetDescName: 000:cn=DESCRIPTION_0

```
---------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------

###  <p style="color:red"> Create Oracle Database

#### Pull Oracle 19c Images

```bash

## login into docker registry  container-registry.oracle.com

docker login container-registry.oracle.com
Username:
Password:
WARNING! Your password will be stored unencrypted in /home/enduser/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
###NOTE : goto container-registry.oracle.com website and accept "terms/conditions" before pull otherwise wonot allow.
[OR]
 docker login -u username@company.com  container-registry.oracle.com
Password:

docker pull   container-registry.oracle.com/database/enterprise:19.3.0.0
```

#### start (Create) Database container 19c

#### host level configuration
```bash
## following parameters at host level in /etc/sysctl.conf
vi  /etc/sysctl.conf
fs.aio-max-nr = 1048576
fs.file-max = 6815744
net.core.rmem_max = 4194304
net.core.rmem_default = 262144
net.core.wmem_max = 1048576
net.core.wmem_default = 262144
net.core.rmem_default = 262144

# sysctl -a
# sysctl -p
```
#### Start Container 19c

```bash
ORA_DISK_BASE=/vmdata2/nfsshare/oudproject/database/my19c

mkdir -p ${ORA_DISK_BASE}/oradata
mkdir -p ${ORA_DISK_BASE}/startup
mkdir -p ${ORA_DISK_BASE}/setup

chmod 777 ${ORA_DISK_BASE}/oradata
chmod 777 ${ORA_DISK_BASE}/startup
chmod 777 ${ORA_DISK_BASE}/setup


##ORACLE PASSWORD FOR SYS, SYSTEM AND PDBADMIN: Docker#2020

ORACLE_SID=PSTGCDB
# default value ORCLPDB1
ORACLE_PDB=PSTGPDB1
# default value random
ORACLE_PWD=docker2020
# enterprise or standard default enterprise
ORACLE_EDITION=enterprise
# default value AL32UTF8
ORACLE_CHARACTERSET=AL32UTF8
#
ORADATA=${ORA_DISK_BASE}//oradata
ORASTARTUP=${ORA_DISK_BASE}/startup
ORASETUP=${ORA_DISK_BASE}//setup
#
CON_NAME=my19c
LISTENER=1521
OEM=5500
IMAGE=container-registry.oracle.com/database/enterprise:19.3.0.0

## Working
docker run  --name ${CON_NAME}   \
   -p ${LISTENER}:1521 -p ${OEM}:5500 \
   -v ${ORADATA}:/opt/oracle/oradata \
   -v ${ORASTARTUP}:/opt/oracle/scripts/startup \
   -v ${ORASETUP}:/opt/oracle/scripts/setup ${IMAGE}


##not Tested
docker run  --name ${CON_NAME}   \
   -p ${LISTENER}:1521 -p ${OEM}:5500 \
   -e ORACLE_SID=${ORACLE_SID} \
   -e ORACLE_PDB=${ORACLE_PDB} \
   -e ORACLE_PWD=${ORACLE_PWD} \
   -e ORACLE_EDITION=${ORACLE_EDITION} \
   -v ${ORADATA}:/opt/oracle/oradata \
   -v ${ORASTARTUP}:/opt/oracle/scripts/startup \
   -v ${ORASETUP}:/opt/oracle/scripts/setup ${IMAGE}


docker exec -u oracle -it my19c /bin/bash

## Verify if we can reach ldap server.
ldapsearch -h  ${oudproxy_ip} -p 2389  -b dc=pelican,dc=home  \
    -D "cn=directorymanager" -w mypassword25   -s base -s sub "uid=user.1"

```

#### Create Oracle wallet to connect OUDProxy

```

LSNR_LCTN=/opt/oracle/product/19c/dbhome_1/network/admin

mkdir p ${LSNR_LCTN}/wallet

mkstore -wrl ${LSNR_LCTN}/wallet -create
##password =mypassword25

mkstore -wrl ${LSNR_LCTN}/wallet \
          -createEntry ORACLE.SECURITY.DN cn=gcrep,cn=OracleContext,dc=pelican,dc=home

mkstore -wrl ${LSNR_LCTN}/wallet \
          -createEntry ORACLE.SECURITY.PASSWORD testusernathan14

mkstore -wrl ${LSNR_LCTN}/wallet -viewEntry ORACLE.SECURITY.DN
mkstore -wrl ${LSNR_LCTN}/wallet -viewEntry ORACLE.SECURITY.PASSWORD  


## get Root certificate from ouddapps server
keytool -export -alias server-cert -file ./ouddapps_ca.crt -keystore ./ouddapps_keystore -storepass `cat keystore.pin` -v

## Import Root certificate into wallet in oracle server [which is got it from ouddaps server]
orapki wallet add -wallet ./wallet -trusted_cert -cert ./ouddapps_ca.crt  -pwd mypassword25

```

#### Create ldap.ora file

```
## use ip address of OUDProxy docker node
{
LSNR_LCTN=/opt/oracle/product/19c/dbhome_1/network/admin
cd $LSNR_LCTN
echo "DIRECTORY_SERVERS=(xxx.xxx.xxx.xxx:2389:2636)" >./ldap.ora
echo "DIRECTORY_SERVER_TYPE=OID" >>./ldap.ora
echo '## DEFAULT_ADMIN_CONTEXT must be parent of  cn=oracleContext and do not inclide oracleContext here "' >>./ldap.ora
echo 'DEFAULT_ADMIN_CONTEXT="dc=pelican,dc=home"' >>./ldap.ora
echo '##DEFAULT_ADMIN_CONTEXT="ou=People,dc=pelican,dc=home"' >>./ldap.ora
}
cat ldap.ora

cat sqlnet.ora
NAMES.DIRECTORY_PATH=(tnsnames,ldap,ezconnect) >./sqlnet.ora

{
LSNR_LCTN=/opt/oracle/product/19c/dbhome_1/network/admin
cd $LSNR_LCTN
echo "NAMES.DIRECTORY_PATH=(LDAP,TNSNAMES,EZCONNECT)" >./sqlnet.ora
echo "WALLET_LOCATION = ">>./sqlnet.ora
echo "  (SOURCE =  ">>./sqlnet.ora
echo "    (METHOD = File)   ">>./sqlnet.ora
echo "    (METHOD_DATA = (DIRECTORY = ${LSNR_LCTN}/wallet))) ">>./sqlnet.ora
}

cat sqlnet.ora

```

#### Create connect user

```
## dn: uid=user.1,ou=People,dc=pelican,dc=home
alter system set ldap_directory_access=PASSWORD sid='*' scope=both ;       
alter system set  RDBMS_SERVER_DN="dc=pelican,dc=home"  scope=spfile ;

show parameter ldap_directory_access


alter session set container = ORCLPDB1 ;

create user testuser identified globally as  'uid=user.1,ou=People,dc=pelican,dc=home' ;

alter user testuser identified globally as  'cn=testuser,ou=People,dc=pelican,dc=home' ;

grant connect,resource,dba to testuser ;
alter user testuser default tablespace users ;
alter user testuser temporary  tablespace temp ;


col EXTERNAL_NAME FORMAT A50
col USERNAME      FORMAT A12
SELECT external_name, username FROM dba_users  WHERE username like 'testuser' ;

col GUID FORMAT A100
col NAME FORMAT A20
set linesize 200
SELECT guid,name FROM  gv$pdbs WHERE name like 'ORCLPDB1';

mkdir -p  $ORACLE_HOME/admin/ORCLCDB/C48C1AE066590EFCE053060011AC8C67
cd  $ORACLE_HOME/admin/ORCLCDB/C48C1AE066590EFCE053060011AC8C67
ln -s $ORACLE_HOME/network/admin/wallet wallet
ls -lrt $ORACLE_HOME/admin/ORCLCDB/C48C1AE066590EFCE053060011AC8C67/wallet/*

mkdir -p $ORACLE_HOME/network/admin/wallet/C48C1AE066590EFCE053060011AC8C67
cd $ORACLE_HOME/network/admin/wallet/C48C1AE066590EFCE053060011AC8C67
ln -s ../ewallet.p12 ewallet.p12
ln -s ../cwallet.sso cwallet.sso


sqlplus -L  testuser/mypassword25@ORCLPDB1

## For trace login issue, use the following to trace on
alter system set events '28033 trace name context forever, level 9';

alter system set events '28033 trace name context off';

```

####  <p style="color:green">  FOR Debug errors vist 10_oracle_OUD_administration.pdf.

#####  <p style="color:green"> 31.7   Understanding Enterprise User Security Password Warnings
#####  <p style="color:green"> 31.8   Troubleshooting Issues after Integrating OUD and Enterprise User Security
```
ldapsearch -h ${oudproxy_ip} -p 2389 \
-b cn=common,cn=products,cn=oraclecontext,dc=pelican,dc=home  "(objectclass=*)"

ldapsearch -h ${oudproxy_ip} -p 2389 \
  -b  cn=oraclecontext,dc=pelican,dc=home -s one "(objectclass=orcldbserver)"

ldapsearch -h ${oudproxy_ip} -p 2389 \
 -b cn=common,cn=products,cn=oraclecontext,dc=pelican,dc=home \
"(objectclass=*)" orclcommonusersearchbase \
orclcommongroupsearchbase orclcommonnicknameattribute orclcommonnamingattribute

ldapsearch -h ${oudproxy_ip} -p 2389  -b ou=People,dc=pelican,dc=home \
   -D "cn=gcrep,cn=OracleContext,dc=pelican,dc=home" -w testusernathan14 "(uid=user.1)"
```         
---------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------
