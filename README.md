# OpenDJ

- [openidentityplatform.org](https://www.openidentityplatform.org/openam)
  - [Open Directory Service](https://www.openidentityplatform.org/opendj)
- [OpenIdentityPlatform/OpenDJ](https://github.com/OpenIdentityPlatform/OpenDJ)

OpenDJ is an **LDAPv3** compliant **directory service**, which has been developed for the **Java platform**, provides a high performance, highly available, and secure store for identities, that managed by your organization. Its easy installation process, combined with the power of the **Java platform** makes OpenDJ the simplest, fastest directory to deploy and manage.

----

## Before Install

[Release Notes](https://github.com/OpenIdentityPlatform/OpenDJ/wiki/Release-Notes)

- Java Environment: Java SE 7+
- Maximum Open FilesOpenDJ can use at least 64K (65536) file descriptors.

`opendj` user's level limits

#### example of limits

```bash
cat /etc/security/limits.conf

opendj soft nofile 65536
opendj hard nofile 131072
```

the Linux system overall maximum:

```bash
cat /proc/sys/fs/file-max

47462
```

### Operating System

- Linux 2.6 and later
- Microsoft Windows Server 2008, 2008 R2, 2012, and 2012 R2
- Oracle Solaris 10, 11

Enable file system write barriers and make sure that the file system journaling mode is ordered.

For details: [man mount](https://linux.die.net/man/8/mount)

### Application Servers

- OpenDJ directory server: a standalone Java service.
- OpenDJ REST to LDAP gateway, and OpenDJ DSML gateway: run on Apache Tomcat and Jetty.

### FQDNs For Replication

OpenDJ replication requires that you use fully qualified domain names

### Hardware

#### Memory Requirements

- 32 bit: 256 MB
- 64 bit: 1 GB
- 100MB free disk space

In production:

- 2 GB memory
- four times the disk space needed to house initial production data in LDIF format

#### Processor Alternatives

Read is fast. Write is slow.

#### Storage Requirements

Not currently support network file systems such as NFS for database storage.

---

## Install

### Kubernetes

- docker image: [openidentityplatform/opendj](https://hub.docker.com/r/openidentityplatform/opendj/)
- [How To Run OpenDJ in Kubernetes](https://github.com/OpenIdentityPlatform/OpenDJ/wiki/How-To-Run-OpenDJ-in-Kubernetes)

#### Create OpenDJ Service

```bash
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

#### Pods

```bash
kubectl get pods -l="app=ldap"

NAME                    READY   STATUS    RESTARTS   AGE
ldap-756cc94559-vcbzr   1/1     Running   0          8m34s
```

#### Status

```bash
kubectl exec -it $(kubectl get pods -l="app=ldap" -o name) \
-- /opt/opendj/bin/status --bindDN "cn=Directory Manager" --bindPassword password
```

```bash
          --- Server Status ---
Server Run Status:        Started
Open Connections:         1

          --- Server Details ---
Host Name:                ldap-756cc94559-vcbzr
Administrative Users:     cn=Directory Manager
Installation Path:        /opt/opendj
Instance Path:            /opt/opendj/data
Version:                  OpenDJ Server 4.4.7
Java Version:             1.8.0_265
Administration Connector: Port 4444 (LDAPS)

          --- Connection Handlers ---
Address:Port : Protocol               : State
-------------:------------------------:---------
--           : LDIF                   : Disabled
0.0.0.0:1389 : LDAP (allows StartTLS) : Enabled
0.0.0.0:1636 : LDAPS                  : Enabled
0.0.0.0:1689 : JMX                    : Disabled
0.0.0.0:8080 : HTTP                   : Disabled

          --- Data Sources ---
Base DN:     dc=example,dc=com
Backend ID:  userRoot
Entries:     0
Replication:
```

#### Add a directory

```bash
kubectl exec -it $(kubectl get pods -l="app=ldap" -o name) -- bash
```

```bash
/opt/opendj/bin/dsconfig \
--port 4444 \
--hostname $(/bin/hostname -f) \
--bindDN "cn=Directory Manager" \
--bindPassword password \
--no-prompt \
--trustAll \
--batch <<EOF
create-backend --backend-name newBackend \
--type pdb \
--set base-dn:"dc=example,dc=org" \
--set db-cache-percent:20 \
--set enabled:true

create-backend-index --backend-name newBackend \
--type generic \
--set index-type:equality \
--set index-type:substring \
--index-name cn

create-backend-index --backend-name newBackend \
--type generic \
--set index-type:equality \
--set index-type:substring \
--index-name sn

create-backend-index --backend-name newBackend \
--type generic \
--set index-type:equality \
--index-name uid

create-backend-index --backend-name newBackend \
--type generic \
--set index-type:equality \
--set index-type:substring \
--index-name mail
EOF
```

`sample.ldif`:

```bash
echo "dn: dc=example,dc=com
dc: example
objectClass: top
objectClass: domain

dn: ou=people,dc=example,dc=com
objectClass: organizationalUnit
ou: people

dn: ou=group,dc=example,dc=com
objectClass: organizationalUnit
ou: group

dn: cn=Keanu Reeves,ou=people,dc=example,dc=com
objectClass: top
objectClass: posixAccount
objectClass: inetOrgPerson
cn: Keanu Reeves
uid: keanu
sn: Reeves
givenName: Keanu
uidNumber: 1001
gidNumber: 500
homeDirectory: /home/users/keanu
loginShell: /bin/bash" > /tmp/sample.ldif
```

```bash
/opt/opendj/bin/import-ldif \
--bindDN "cn=Directory Manager" \
--bindPassword password \
--includeBranch dc=example,dc=com \
--backendID userRoot \
--ldifFile /tmp/sample.ldif \
--trustAll
```

#### List all indexes

```bash
/opt/opendj/bin/dsconfig \
list-backend-indexes \
--port 4444 \
--hostname $(/bin/hostname -f) \
--bindDN "cn=Directory Manager" \
--bindPassword password \
--backend-name userRoot \
--no-prompt \
--trustAll
```

```bash
kubectl exec -it $(kubectl get pods -l="app=ldap" -o name) \
-- /opt/opendj/bin/backendstat show-index-status --backendID userRoot --baseDN dc=example,dc=com
```
