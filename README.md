# rpi-k3s-sonarqube
Deploy Sonarqube on k3s on a Raspberry Pi
#### NOTE: If you change the image, this will also work on standard kubernetes, which is where I derived this from

Runs on arm64 (if you run the 64-bit Ubuntu raspberry pi image it will run in 64-bit)

Uses an external nfs server as the pi's sd card dies faster with high r/w's


## NFS Server

Create the following directories as per the paths in PV.yaml:

```[root@k3snfs1 k3s_export1]# mkdir -p sonarqube/conf```
```[root@k3snfs1 k3s_export1]# mkdir -p sonarqube/data```
```[root@k3snfs1 k3s_export1]# mkdir -p sonarqube/extensions/downloads```
```[root@k3snfs1 k3s_export1]# mkdir -p sonarqube/extensions/plugins```
```[root@k3snfs1 k3s_export1]# mkdir -p sonarqube/logs```
```[root@k3snfs1 k3s_export1]# mkdir -p sonarqube/temp```
```[root@k3snfs1 k3s_export1]# mkdir -p sonarqube/postgres```

Then set the right permission on the folder on the nfs server:

```[root@k3snfs1 k3s_export1]# chown 999:0 sonarqube -R```

## sonar.properties
Path: 
```
./conf/sonar.properties
```

As of Sonarqube 8.X you need to add the following 3 lines (change as appropriate), or create the file if it doesn't exist

```
sonar.jdbc.username=postgres
sonar.jdbc.password=mypassword
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonar
```

## sonarqube-community-branch-plugin
NOTE: The plugin version to download depends on which version of Sonarqube you are running, for V8.9.1LTS I am using the below, for all others see: https://github.com/mc1arke/sonarqube-community-branch-plugin/blob/master/README.md 

For the container to start you need to download this plugin:
```
cd sonarqube/extensions/plugins
wget https://github.com/mc1arke/sonarqube-community-branch-plugin/releases/download/1.8.0/sonarqube-community-branch-plugin-1.8.0.jar .
chown 999:0 sonarqube-community-branch-plugin-1.8.0.jar
```

### LDAP
Update sonar.properties to configure LDAP to point to the right LDAP Server, located at:
```
./conf/sonar.properties
```

### Example file working as of 8.9LTS:
```
 # use ldap.server1... and ldap.server2... prefixes to use multiple ldap servers
 # e.g. ldap.server1.url=ldap1
 # ldap.server2.url=ldap2
 # you would then change the other ldap. lines with ldap.server<1/2> as appropriate
 sonar.security.realm=LDAP
 ldap.url=ldap://ldap1:389
 ldap.bindDn=CN=user,CN=Users,dc=contoso,dc=com
 ldap.bindPassword=<CHANGEME>
 ldap.user.baseDn=dc=contoso,dc=com
 ldap.group.baseDn=dc=contoso,dc=com
 ldap.group.request=(&(objectClass=group)(member={dn}))
 ldap.user.request=(&(objectClass=user)(sAMAccountName={login}))
 # ldap.windows.auth=false

 # fixes ldap timeout issues, do not remove
 ldap.followReferrals=false
```
Now make sure the service account credentials exist in the OU you specified above.

You also need sonarqube-administrators and sonarqube-users Security Groups so you can control who gets what access

## Kernel parameters
Note: need to change the sysctl kernel parameters on the nodes, e.g.

```sudo sysctl -w vm.max_map_count=524288```

```sudo sysctl -w fs.file-max=131072```

Then add a file under /etc/sysctl.d with the following:
(or modify sysctl.conf directly)

```
vm.max_map_count=524288
fs.file-max=131072
```

## Upgrade steps:

### postgresql

Dump postgresql database to sql dump file Upgrade postgres container to a supported version

Restore postgresql databse from sql dump file then verify the restore worked:
```
su - sonar
psql 
\d # should see the sonar db here
```
delete the sql dump file

### sonarqube

#### You can upgrade from an older Sonar LTS version directly to the next LTS without jumping from each minor release in between, see the documentation (https://docs.sonarqube.org/latest/setup/upgrading/)

* Create a seperate *clean* sonarqube pod deployment using latest LTS version of sonar and latest supported postgresql version with no data and a clean nfs path seperate to the existing pod (and create the folder structure as per the existing sonar on the nfs mount and dont forget to chown 999:999 the sonar pv and 999:0 postgresql pv). Use different container names, pv's, pvc's, svc, ing, etc to prevent conflicts with the existing version (i.e if it wasn't already clear, do *NOT* break/corrupt the existing pod, keep it seperate, possibly using a test namespace)
* Import the LDAP config from the existing sonar (./conf/sonar.properties)
* Start the containers by creating the deployment.yaml, then once its up and running delete the deployment to kill the containers/pod.
* Compare the plugins directory to the existing sonar and copy over plugins from the older version, but do NOT overwrite any of the plugins as they may be newer which would break sonar if overwritten, location is ./extensions/plugins/
* Start the containers again, ensure it comes up to verify no conflicts, (check the logs)
* Kill the container again
* Now create a dump of the pgsql and restore the dump as per the postgresql section above
* Bring the containers back up
* Go to the https://<sonar URL>/setup and perform the database schema upgrade
* After the upgrade is complete, you should be able to login and see the projects and rules, etc.
* Next perform a full vacuum as below on the db.

## Full Vacuum
* Now run a vacuum by exec-ing to the postgres container and running the following:
* vacuumdb --dbname sonar --verbose --full

## Update sonar plugins
* You can update the plugins via the marketplace in Administration settings which will require a reboot, some plugins may need to be upgraded/replaced manually by searching online.

## Troubleshooting
* You can set logging to DEBUG under the Administation tab. # NOTE: revert back to INFO once the issue you are troubleshooting is resolved
* Inspect the log files via the nfs share, ./logs/sonar.log ./logs/web.log, etc
* You can also check logs via kubectl in the container crashes/fails to figure out why.
