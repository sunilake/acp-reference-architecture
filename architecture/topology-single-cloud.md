# Reference Architecture Topology

The following is a holistic look at the topological construction of our Apprenda Platform reference architecture including machine, clustering, and networking.  To visualize the overall architecture, [see this diagram](../resources/acp-reference-architecture.png).

## The Entire System

Here’s a look at the entire stack for our reference architecture.  Each Platform role is discussed in greater detail elsewhere in this guide.  Not pictured here are tangential systems such as domain controllers or DNS servers.  These roles may be served by existing infrastructure outside the Apprenda Platform architecture.

## Load Managers

On the Apprenda Platform, the Load Manager (best described as a reverse proxy) serves as the initial receptor of an inbound HTTP request.  IIS configuration on the Load Manager is managed by the Apprenda Load Manager Service, a Windows service that creates and modifies URL Rewrite rules as various guest app components are deployed to internal frontend content servers.  The Load Manager Service ensures the appropriate routing of inbound requests based on request URL patterns.  The Load Manager does not host any application workloads (unless the role of Load Manager is given to a worker node with the necessary prerequisites - although this is not recommended in production).

For high availability purposes, we recommend a topology consisting of two Load Manager instances.  [See here](../planning/considerations-for-availability.md) for detailed availability considerations.

## Platform Coordinator Nodes

Coordination of guest application workload deployment is handled by these nodes. The machines run Apache ZooKeeper (installed and configured by Apprenda installation), wrapped in an Apprenda Windows service. More on the Apache Zookeeper project here: [https://zookeeper.apache.org/](https://zookeeper.apache.org/). On Apprenda, the Platform Coordinator Nodes maintain knowledge of the topology of guest application workloads across all nodes, as well as any shared configuration in use by Apprenda components.  The Zookeeper endpoints are not for use by guest applications.

A properly configured Apprenda installation requires an odd number of Platform Coordination nodes because a majority of Platform Coordination nodes must be available in order for the Platform to function properly (since this is dictated by Zookeeper, see Zookeeper documentation for details).  Our production reference architecture prescribes _five_ Platform Coordination nodes, which will allow the Platform to function as long as any 3 nodes are running; additional nodes can be added as needed once the environment is up and running.  For proper availability [see here](../planning/considerations-for-availability.md) for node failure considerations.

## Cache Nodes

These nodes house the Apprenda Caching Windows service, a distributed Redis-based in-memory caching implementation for all Platform Windows and Linux servers.  Each Cache node can support one instance of the Caching service per processor core; the number of processor cores/Caching service instances should be used as the number of service ports specified for the Cache in the Apprenda Installer. We recommend 2 dedicated Cache nodes with 2 cores/Caching service instances each to help with load distribution and mitigate infrastructure failure risk.

## Worker Nodes

### Windows Machines

All Windows Application Servers host the Apprenda Windows Host service. The Windows Host enables the hosting of WCF and Windows services, thereby allowing both key Platform and guest application service components to be hosted on these servers.

It is necessary that at least one Windows Application Server per cloud host Apprenda’s Storage Controlling Services, which interfaces with SQL Server and Oracle to configure databases. These servers are required to have SQL Server Management Objects (SMO) 2012 installed.  At installation, the Platform will mark any Windows Application Servers with SMO installed as capable of hosting the Storage Controlling Services and will deploy this component to those servers.  It will install the required SMO version on a single Application Server if no suitable host is found.  In order to ensure that the Storage Controlling Services is highly available, we recommend installing a supported version of SMO (version 11.0 or higher) on two servers that will be designated as Application servers prior to running the Apprenda Installer, as this will result in both servers being designated as Storage Controlling Services hosts.  As needed after installation, additional Application servers can be configured as Storage Controlling Services hosts by installing SMO on the servers and then designating them as such in the System Operation Center (SOC).

Windows Web Servers are frontend web servers that host .NET based UIs through IIS. Via portals, the Apprenda Platform allows developers and Platform operators to create ad-hoc web farms for .NET guest application UI workloads at any time (as long as there is sufficient infrastructure).  The sum of your Windows Web Servers represents the compute power of your Platform specifically for hosting .NET guest application UI workloads.

It should be noted that all Windows Web Servers are capable of hosting WCF and Windows services. This is necessary for the nodes to host the Presentation Controlling Services (see the Apprenda Software Inventory below), which allows management of .NET UIs in IIS. As such, all Windows Web Servers will also be marked as Windows Application Servers (described in the section above) after installation even if they were designated as Web Servers only in the Apprenda Installer.  By default, however, the Platform will deploy WCF/Windows services to Web Servers only if there are no dedicated Application Servers available.

### Linux Machines

Linux Servers host Java Web Application components, which are deployed and managed on top of individual Tomcat/JBoss instances by the Apprenda Linux Container.

### Docker

Linux machines with the Docker engine installed are Linux worker nodes capable of also running Docker images deloyed via Apprenda.

### Kubernetes

As of version 7.1.0 of the Apprenda platform, operators may attach existing [Kubernetes](https://www.kubernetes.io) clusters to a running instance of the platfrom and gain operational control and pod deployment capabilities against that cluster through the Apprenda toolset.  Clusters should be installed using the [Kismatic Enterprise Toolkit](https://www.github.com/apprenda/kismatic), which uses best practices for enterprise-ready Kubernetes clusters.

### Database Servers

**MS SQL**
The Platform manages SQL Server instances on your behalf to provision and configure guest application databases. Any number of SQL Server instances can be managed by a single Platform, and SQL instances can be added to the Platform at any time for capacity.  Our reference architecture includes a single SQL Server instance that is configured as a SQL Server failover cluster; such a configuration typically relies on shared storage on a SAN or NAS.  We’ve included two SQL nodes in the cluster as a standard recommendation for simple redundancy.  The Platform will manage this cluster as it would a normal SQL Server instance.  Expansion of the database tier of our reference architecture would come in two forms:

Adding SQL Server nodes to existing cluster(s) will increase redundancy capabilities of existing Platform-managed SQL instance.
Adding SQL Server nodes independent of existing cluster(s) or adding entirely new Platform-managed clusters will increase the capacity of the database hosting layer of your Apprenda Platform.

In addition to providing storage for guest applications, an Apprenda managed SQL Server instance is necessary for housing the Apprenda Core Database, which contains data necessary for Platform functionality.

**Oracle**
The Platform manages Oracle RDBMS installations on your behalf to provision and configure storage for guest applications.

## Repository

The Platform Repository is a network share that will serve as the central storage location for all Platform and guest application binaries. This location must be configured prior to installation, should be specified as a network path in the Apprenda Installer, and must contain the following three folders:

* Applications
* Apprenda
* SAC

It is also acceptable to configure each folder as a separate share as long as they are accessible through the same base path.

All guest application binaries, once uploaded to the Platform by developers, are stored in the Applications folder in the Platform Repository (in some parts of the Platform, such as the Repository Browser in the SOC, this folder is called the “Application Repository”).  The Applications folder will also include binaries for Platform components that are themselves hosted on the Platform as guest applications, such as the Developer Portal and the System Operation Center. All other Platform binaries are stored in the “Apprenda” folder (which is sometimes referred to as the “System Repository”).  Upon workload deployment, binaries that are needed for local execution are copied to the target server from their respective locations in the Platform Repository.

## Federation Nodes

As an install-time option, the Platform can be configured to support identity federation using Active Directory Federation Services (AD FS). If you choose to use AD FS for identity federation, we recommend creating an AD FS Web Farm consisting of no less than 2 AD FS Nodes backed by a SQL Server failover cluster.  Please contact support at apprenda dot com for information on setting up an AD FS Web Farm prior to running the Apprenda Installer.

It should be noted that all AD FS nodes—including those that constitute an AD FS Web Farm—will automatically be designated as Windows worker nodes. This is necessary for the nodes to host the Federation Service (see the Apprenda Software Inventory), which allows the Platform to interface with AD FS.  *Note*: If these nodes are hosted in a DMZ or you otherwise don't want them to be part of the general Windows worker node pool, application deployment policies can prevent them from being valid targets for any application workload __aside__ from the Federation Service.
