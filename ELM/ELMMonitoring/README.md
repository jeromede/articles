# ELM Monitoring

## Introduction

The [Jazz.net wiki](https://jazz.net/wiki/bin/view/Deployment/DeploymentMonitoring) shares a lot of information about what is to monitor for maintaining the healh of an ELM deployment. The purpose of this document is to provide a bit of background information and to summarize the need.

## Estimating is difficult

The resources (memory, CPU) to be given to an ELM application depend on many factors and the amount of them may need to be changed after the first deployment because there are more and more users, more and more complex data shapes… The "good" size (RAM, CPU) of a machine hosting an application is difficult to estimate before a first installation. It will be based on experience, and some guessing. With an effort put in monitoring, however, it is possible to check whether the starting hypothesis really stands, and to anticipate the changes that might be necessary: a machine could have been oversized, or on the contrary, should receive more resources.

## Memory Heaps

As far as topology is concerned, there is a lot of freedom. A given Jazz application runs inside a Liberty Java application server. One Liberty may deploy several applications. There might be several Liberty running on a given machine. To be remembered, a Liberty is run by a JVM: 1 Liberty = 1 JVM.

In production for a user counts after ~50 and growing, the best is to go for an "enterprise topology" where every applications is given a dedicated machine.

When started (for a Jazz application, it would be running `server/server.startup`), a chunk of memory is given to the JVM. Usually for ELM applications, the maximum (`Xmx`) and minimum (`Xms`) memory sizes parameters are set with equal values so that the JVM starts with a pre-allocated and contiguous amount of memory for the Java programs; this is the JVM **Java heap**. The JVM is also an OS process that needs some memory (and CPU) for itself; this memory is the JVM **native heap**. Only the Java heap can be limited, the total JVM memory is not limited (unless some OS-dependent specific actions are taken): in a bad case scenario (a bug in the application for example), or if, by nature, the application needs a lot of memory (implementing JNI for example), the native heap could increase a lot and the JVM process take all the space. Measuring the JVM memory (as a process) and subtracting the Java heap size gives the native heap size. The memory actually used in the Java heap can be low (allocated memory resources may have been overestimated) or saturated (in that case underestimated); if saturated, garbage collection tasks (that free the unreferenced objects within the Java Heap space) will be executed often and lower the overall JVM efficiency.

1 Liberty executes 1 JVM which is an Operating System process:

![](img/jvm-simple.svg)

* Java heap: Xmx = size limit for the Java heap, allocated by the JVM when it is started. The Java program in the JVM won’t be able to consume more.
* Native heap: no limitation (other than the physical or possible OS ones) —  in case of a problem (bug, unlikely use-cases…), the native heap could saturate the machine.

Note: if `Xmx ≠ Xms`, then the JVM will dynamically resize the Java Heap, following a configured algorithm.

For a machine with a single Liberty with a single application, the ELM recommendation as a starting point is:

* Xmx = Xms = ½ machine RAM
* Xmx = Xms = ⅓ machine RAM for /rm applications

To be monitored.

## Topology

How to decide a topology in the {machine > Liberty=JVM > application} hierarchy?

An application by Liberty application server means that the resources are separated. When two or more applications run in a single Liberty (they are installed together), the Java heap, the Java native heap, the CPU for the JVM process are indistinctly used by these applications; it is therefore not possible to measure and understand which application consumes what. In addition, too much memory for the Java Heap can become counterproductive for specific mechanisms like garbage collection. There are therefore good reasons to define a rule so that in a production environment there is only one application in a given Liberty (= JVM), especially considering demanding applications like the Jazz ones. Anyway —see below— monitoring the JVM will give useful indications.

Now, one big machine running several Liberty application servers? Or one Liberty per machine? The easiest route to take is to continue with the isolation principle in mind. With one Liberty per machine, monitoring the machine means more or less monitoring the only JVM process running on it. A general rule is also to separate applications on different machines based on criticality: if an application saturates a machine, it might prevent another running on the same machine to serve its purpose. In a Jazz context, that would mean isolate JTS (Jazz Team Server) from the other applications. In a world of VMs, too, there are often limits imposed on the max size of a VM. All good reasons to provide one machine per Liberty for "big" applications. **But** many machines means more complexity and more resources. Even if this consideration is balanced by how easy it is to setup and monitor VMs on a good infrastructure, the lesser machines to manage, the lesser complexity… There is no definite answer.

## VMs need physical resources

Speaking of VM infrastructure, the vCPUs and memory allocated to VMs must match something real: there is a layer of virtualization that is executed by a physical machine or a cluster of them. We have witnessed situations in stovepipes orgnizations where the team responsible for providing Jazz applications to system/software engineers has been promised resources that were not actually delivered, or was, without knowledge, put in a random competition against entirely different functional teams running their own applications on the same infrastructure. In production, good virtualization practices are mandatory; they include: staying below certain ratios (1!) between virtual "promises" and real resources, preallocating certain resources; and monitoring!

## Monitoring overview

Monitoring can be done at several layers of abstraction, and it is very useful to have a dashboard that correlates the different measurements, with sufficient historical data. These layers are:

* virtualization system level: vCPUs / φCPUs ratio, and total VM memory / actual memory ratio, VM migration rates in the cluster (if applicable).
* VM level, provided by the virtualization system, or OS level: vCPUs and memory consumption.
* OS level: JVM process CPU and memory; disk consumption.
* JVM level Java heap behavior (memory, garbage collecting)
* Liberty probes for more detailed observation
* Jazz application level, with MBeans: specifics like DB ping time, # of simultaneous users…
* RBDMS level: machine(s) (CPU, memory), number of connections, execution time of queries, disks performances…

The JVM level can be monitored through Liberty probes.
For example (used memory in the RM application JVM):

```
https://…/IBMJMXConnectorREST/mbeans/WebSphere%3Atype%3DJvmStats/attributes/UsedMemory
```

Jazz application probes can also be unlocked and then provided by the Liberty running the application. For example (DB ping time from the RM application):

```
https://…/IBMJMXConnectorREST/mbeans/com.ibm.team.foundation.server%3Aname%3Drm%2Ctype%3DserverMetrics/attributes/dbPingTime
```

These both sets of probes (based on the Java MBeans mechanism) can be monitored by periodically reading a Json object from an HTTP GET.

Note:

* JVM can be instrumented my monitoring tools extensions to observe them directly
* most monitoring tools know how to interface to MBeans directly.

Everything that is measurable should not be necessarily activated to be measured, because the very task of collecting information may have a performance penalty at the source of information. Other possible measures have not been mentioned here: network efficiency, reverse proxy response time, LDAP server response time, etc.

The key is to be able to

* select a significant set but not too many measures, each at any level
* see all measures on the same dashboard so that correlations can be analyzed
* adjust measurement frequencies
* revisit historical data at a given past time interval.

## Basic health indicators

You can get a good initial assessment of deployment health by monitoring usage metrics at the operating system level. This includes:

* % CPU utilization
* % CPU time spent waiting for disk
* % disk busy
* Free disk space 

Collect this information for all servers that are part of the deployment (including proxy servers and database servers).

## Monitoring details above OS layer

Examples given with [Instana Observability](https://www.ibm.com/docs/en/instana-observability), to be transposed for any over monitoring tool that may be used.

## Monitoring details at OS level

On each server: CPU and RM usage of the JVM process running Liberty and its application.

### JVM

#### JvmMXbean

The JvmMXbean MXBean allows the monitoring of performance data emitted by the JVM: 

* Heap: Heap size used for current JVM.
* FreeMemory: Free heap available for current JVM.
* UsedMemory: Used heap for current JVM.
* ProcessCPU: Percentage of CPU used by JVM process.
* GcCount: Number of times GC has happened since JVM starts.
* GcTime: Total accumulated value of GC time.
* UpTime: Time in milliseconds, since JVM has started.

There will be only one JvmStats MXBean for a Liberty profile instance. 
The object name for identifying JVMStats MXBean is: 

```
WebSphere:type=JvmStats.
```

The following table summarizes the key metrics to monitor:

| Metric | Expected value | Comments |
|:-------|:---------------|:---------|
| **UsedMemory** | Variable. Not steadily close to 100% of Heap Size | A high value of Used Memory doesn’t necessarily mean that there is a risk of an out of memory. Monitoring this value over time should not show the Used Memory steadily close of equal to the heap size. If the UsedMemory is steadily close to 100% of Heap Size, consider reviewing the JVM configuration (Xmx, Xms) and analyze the garbale garbarge collector for additional tuning. |
| **GC Time** |We would expect a GC Time of less than 500 ms or 1 sec. (GcTime/GcCount)|If the GC time is consistently high, you should investigate the Garbage Collector.|

#### Tooling Example

[Monitoring Java virtual machine (JVM)](https://www.ibm.com/docs/en/instana-observability/current?topic=technologies-monitoring-java-virtual-machine)

### Liberty

#### Thread Pool

The following performance metrics are available

* Active Threads: The number of active threads
* PoolSize: The pool size of the thread pool which is named Default Executor

The ObjectName for identifying each servlet MXBean is:

```
WebSphere:type=ServletStats,name=<AppName>.<ServletName>
```

The following table summarizes the key metrics to monitor:

| Metric | Expected value | Comments |
|:-------|:---------------|:---------|
| **Active Threads** | The expected value depends on the number of concurrent requests and the type of workloads processed by Liberty. You should define a threshold value, and when exceeded, trigger an alert. | The Liberty thread pool autonomic tuning process runs every 1.5 seconds to adjust automatically the PoolSize with the coreThreads and maxThreads values. If necessary, tune the coreThreads/maxThreads in cases of very large number of active threads. |

#### Web Applications

Web Applications can monitored by using  ServletStats MXBean. The performance data is available for each Servlet in the Web Application. Each servlet has its own MXBean. The following key data is available for ServletStats MXBean:

* Request Count
* Response Time
* Servlet Name
* Application Name

The ObjectName for identifying each servlet MXBean is:

```
WebSphere:type=ServletStats,name=<AppName>.<ServletName>
```

The following table summarizes the key metrics to monitor:

| Metric | Expected value | Comments |
|:-------|:---------------|:---------|
| **Response Time** | Less than the target response time defined per application | Large response times may suggest bottlenecks within Liberty or backend servers. |

#### Session monitoring

The SessionStats MXBean can be used to monitor the performance data of sessions in the Liberty profile. Performance data of sessions for each application are available as an MXBean which can be accessed through JMX. The sessions associated with a single web application have their own SessionStats MXBean (that is, one SessionStats MXBean for each web application). The following key data is available for SessionStats MXBean:

* CreateCount
* LiveCount
* ActiveCount
* InvalidatedCount
* InvalidatedCountbyTimeout

The ObjectName for identifying each Session MXBean is:

```
WebSphere:type=SessionStats,name=*
```

The following table summarizes the key metrics to monitor:

| Metric | Expected value | Comments |
|:-------|:---------------|:---------|
| **ActiveCount** | Close to the expected number of concurrent users actively using the application | |
| **LiveCount** | The number of sessions that are cached in memory | Large number of Idle sessions (LiveCount – ActiveCount) may suggest that additional tuning is required in Liberty (session invalidation timeout) |


#### Tooling Example

[Monitoring WebSphere Liberty](https://www.ibm.com/docs/en/instana-observability/current?topic=technologies-monitoring-websphere-liberty)

### ELM application

#### LKSART

The License Key Server Administration and Reporting Tool (LKSART) will provide dahsboards and reports to monitor license consumption based on real application user roles, for example concurrent usage peaks.

The available reports are documented in [Creating a report definition](https://www.ibm.com/docs/en/common-licensing/9.0.0?topic=reports-creating-report-definition).

Note that user count is per application type, no per application instance. For example, 3 RM instances would be merged in a single RM measure.

`Concurrent peak usage` remains an interesting data to follow. To be completed with the `concurrentUsers` and `activeUsersCount` MBeans on each instance.

#### MBeans

All available MBeans (Java observation points, like for the JVM or Liberty) for ELM are documented here: [JMX MBeans for ELM Application Monitoring](https://jazz.net/wiki/bin/view/Deployment/JMXMBeans).

From there, got to [MBeans Reference Guides](https://jazz.net/wiki/bin/view/Deployment/MBeansReference) then to the reference for your version, for example [ELM 7.0.3](https://jazz.net/wiki/bin/view/Deployment/MBeansReference703).

##### User count

See [Collecting user related metrics using JMX MBeans](https://jazz.net/wiki/bin/view/Deployment/CollectingUserMetrics).

Concurrent users is an important data point to collect over time for growth planning/projection, to ensure you stay within known user scale maximums and to give insight into the load on an application when triaging a performance anomaly.

**concurrentUsers**: represents the number of unique users associated with an application's active services running at the time the bean data is collected

It is useful to track activeUsersCount to get a sense of what the 'normal' range of activity is for an application.

**activeUsersCount**: represents the number of unique users associated with all of an application's active services that were running some time during the bean's collection interval (not the same as concurrent users).

##### Application specific MBeans

The MBeans reference (see above) for each version gives a list of MBeans specific for each application. These are to be activated from each application Advanced Properties page. This could be for a temporary duration for some, as activating them costs a small performance penalty each time.

The precise list of ELM application specific MBeans to deploy is beyond the scope of this document, as it is very context-dependant.

### Example

MBeans Reference for Jazz: <https://jazz.net/wiki/bin/view/Deployment/MBeansReference703>

For each set of MBeans, how to activate them is documented above.

Activate for JTS from this page:

* <https://local.jazz:9443/jts/admin#action=com.ibm.team.repository.admin.serviceability>

Observe the JVM with:

* <https://local.jazz:9443/IBMJMXConnectorREST/mbeans/WebSphere%3Atype%3DJvmStats
* <https://local.jazz:9443/IBMJMXConnectorREST/mbeans/WebSphere%3Atype%3DJvmStats/attributes/UsedMemory>

Observe an application with:

* `https://local.jazz:9443/IBMJMXConnectorREST/mbeans/com.ibm.team.foundation.server:name=jts,type=serverMetrics`
  * that includes the DB ping time: `https://local.jazz:9443/IBMJMXConnectorREST/mbeans/com.ibm.team.foundation.server:name=jts,type=serverMetrics/attributes/dbPingTime`
* `https://local.jazz:9443/IBMJMXConnectorREST/mbeans/com.ibm.team.foundation.activeservices:name=jts,type=activeServicesSummaryMetrics`
  * that includes the number of concurrent users: `https://local.jazz:9443/IBMJMXConnectorREST/mbeans/com.ibm.team.foundation.activeservices%3Aname%3Djts%2Ctype%3DactiveServicesSummaryMetrics/attributes/concurrentUsers`

If needed, configure your reverse proxy:

* `https://<ELM public subdomain+port>/jtsmb` → `https://<JTS private subdomain+port>/IBMJMXConnectorREST/mbeans`
* `https://<ELM public subdomain+port>/ccmmb` → `https://<CCM private subdomain+port>/IBMJMXConnectorREST/mbeans`
* etc.

(Change `jtsmb` and `ccmmb` with whatever root name you prefer.)


----
_(End of Document.)_
