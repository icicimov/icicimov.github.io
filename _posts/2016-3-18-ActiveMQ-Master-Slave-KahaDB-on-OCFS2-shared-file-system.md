---
layout: post
comments: true
author:
  name: Igor Cicimov
  email: igorc@encompasscorporation.com
description: 'Activemq locks with OCFS2'
keywords: 'activemq,ocfs2'
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
title: ActiveMQ Master/Slave KahaDB on OCFS2 shared file system
category: Server
tags: [activemq, ocfs2]
---

During my tests of shared storage clusters I wondered if ActiveMQ supports file locking on OCFS2 file system which I used on couple of occasions. While looking into it I came accross the following warning on the [Apache project site](http://activemq.apache.org/shared-file-system-master-slave.html):

> **OCFS2 Warning**
> 
> Was testing using OCFS2 and both brokers thought they had the master lock - this is because OCFS2 only supports locking with `fcntl` and not `lockf and flock`, therefore mutex file locking from Java isn't supported.

From [RedHat faq](http://sources.redhat.com/cluster/faq.html#gfs_vs_ocfs2) :

> OCFS2: No cluster-aware flock or POSIX locks
> 
> GFS: fully supports Cluster-wide flocks and POSIX locks and is supported.

See this JIRA for more discussion: [AMQ-4378](https://issues.apache.org/jira/browse/AMQ-4378)

And true enough, when tested ActiveMQ-5.11.1 on one of the OCFS2 clusters, what happened is that both servers started as master due to lack of lock file on the share. Investigating further it came up that some good man created a patch that Apache never bothered to apply from some unknown reason. The case link [AMQ-4694](https://issues.apache.org/jira/browse/AMQ-4694). The patch was attached to the ticket so thought I would try it and apply to latest 5.11.4 version.

Downloaded the source:

```
root@drbd01:~# aptitude install zip
root@drbd01:~# wget http://mirror.ventraip.net.au/apache/activemq/5.11.4/activemq-parent-5.11.4-source-release.zip
root@drbd01:~# unzip activemq-parent-5.11.4-source-release.zip
root@drbd01:~# cd activemq-parent-5.11.4/
```
and applied the patch:

```
root@drbd01:~/activemq-parent-5.11.4# vi ocfs2-locks.patch
root@drbd01:~/activemq-parent-5.11.4# patch -p0 -i ocfs2-locks.patch
patching file NOTICE
Hunk #1 succeeded at 45 with fuzz 2 (offset -4 lines).
patching file activemq-broker/pom.xml
patching file activemq-broker/src/main/java/org/apache/activemq/store/AbstractResourceLocker.java
patching file activemq-broker/src/main/java/org/apache/activemq/store/SharedFileLocker.java
Hunk #1 FAILED at 16.
1 out of 1 hunk FAILED -- saving rejects to file activemq-broker/src/main/java/org/apache/activemq/store/SharedFileLocker.java.rej
patching file activemq-broker/src/main/java/org/apache/activemq/store/SharedNativeFileLocker.java
patching file activemq-broker/src/main/java/org/apache/activemq/util/LockFile.java
patching file activemq-broker/src/main/java/org/apache/activemq/util/LockResource.java
patching file activemq-broker/src/main/java/org/apache/activemq/util/NativeLockFile.java
patching file activemq-unit-tests/src/test/java/org/apache/activemq/util/NativeLockFileTest.java
patching file assembly/pom.xml
Hunk #1 succeeded at 342 (offset 14 lines).
patching file assembly/src/main/descriptors/common-bin.xml
Hunk #1 succeeded at 191 (offset -9 lines).
patching file pom.xml
Hunk #1 succeeded at 881 with fuzz 1 (offset 44 lines).
```

Ok, so all good except it failed for the SharedFileLocker.java file. Easily fixed by applying that part of the patch manually:

```
root@drbd01:~/activemq-parent-5.11.4# vi activemq-broker/src/main/java/org/apache/activemq/store/SharedFileLocker.java
/**
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.activemq.store;
 
import java.io.File;
import org.apache.activemq.util.LockFile;
import org.apache.activemq.util.LockResource;
 
/**
 * An {@link AbstractResourceLocker} that utilizes a {@link LockFile}.
 *
 * @org.apache.xbean.XBean element="shared-file-locker"
 *
 */
public class SharedFileLocker extends AbstractResourceLocker {
 
    @Override
    protected LockResource newLockResource(File lockFileName) {
        return new LockFile(lockFileName, true);
    }
}
```

Next installed Maven:

```
root@drbd01:~# wget http://apache.uberglobalmirror.com/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
root@drbd01:~# tar -xzvf apache-maven-3.3.9-bin.tar.gz
root@drbd01:~# mv apache-maven-3.3.9 /usr/local/
root@drbd01:~# export PATH=/usr/local/apache-maven-3.3.9/bin:$PATH
```

Back to AMQ directory and build the project:

```
root@drbd01:~/activemq-parent-5.11.4# mvn -Dtest=false -DfailIfNoTests=false clean install -e -X
```

Install the built binaries:

```
root@drbd01:~# tar -xzvf /root/.m2/repository/org/apache/activemq/apache-activemq/5.11.4/apache-activemq-5.11.4-bin.tar.gz -C /opt/
```

and copy the setup from the previous AMQ version into the new one:

```
root@drbd01:/opt# cp -a apache-activemq-5.11.1/conf/{log4j.properties,jmx.access,jmx.password,jetty.xml,encompass-activemq.xml,star_encompasshost_com.jks} apache-activemq-5.11.4/conf/
root@drbd01:/opt# chown -R activemq\: apache-activemq-5.11.4
root@drbd01:/opt# rm activemq
root@drbd01:/opt# ln -s apache-activemq-5.11.4 activemq
```

The last bit is modifying the KahaDB locker to use the newly introduced shared-native-file-locker one:

```
root@drbd01:/opt# vi activemq/conf/activemq.xml
[...]
    <persistenceAdapter>
        <kahaDB directory="/share/activemq-data"
               indexCacheSize="40000"
               checkForCorruptJournalFiles="true">
               <locker>
                  <shared-native-file-locker lockAcquireSleepInterval="5000"/>
               </locker>
         </kahaDB>
    </persistenceAdapter>
[...]
```

After copying over the binary tarball to the other server, installing and performing the above steps, I started the server and the lock file was indeed created this time:

```
root@drbd01:/opt# ls -ltr /share/activemq-data/
total 160
drwxr-xr-x 3 activemq activemq     3896 Mar 17 15:27 com.broker.name
-rw-r--r-- 1 root     root            5 Mar 17 15:30 test
-rw-r--r-- 1 activemq activemq        0 Mar 17 18:16 lock
-rw-r--r-- 1 activemq activemq 33030144 Mar 17 18:17 db-1.log
-rw-r--r-- 1 activemq activemq    12304 Mar 17 18:17 db.redo
-rw-r--r-- 1 activemq activemq    12288 Mar 17 18:17 db.data
```

and the other server can see it:

```
root@drbd02:/opt# start activemq && tail -f /opt/activemq/data/activemq.log | grep -i lock
2016-03-18 10:00:15,375 | INFO  | Database /share/activemq-data/lock is locked... waiting 5 seconds for the database to be unlocked. Reason: java.io.IOException: failed to acquire lock: [11] ( | org.apache.activemq.store.SharedNativeFileLocker | main
...
```
