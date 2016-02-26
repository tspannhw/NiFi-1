# Using HDF for Site to site data ingestion into HDP

## Contents
  - [Prebuilt sandbox VM](https://github.com/abajwa-hw/ambari-workshops/blob/master/contributed-views.md#ambari-20-sandbox)
  - [Views screenshots](https://github.com/abajwa-hw/ambari-workshops/blob/master/contributed-views.md#views-screenshots)
  - [Setup views on existing cluster](https://github.com/abajwa-hw/ambari-workshops/blob/master/contributed-views.md#setup-instructions-for-contributed-views)
  - [Install Ambari 2.0 from scratch](https://github.com/abajwa-hw/security-workshops/blob/master/Setup-kerberos-Ambari.md)
  - [Try new Ambari 2.0 security wizard](https://github.com/abajwa-hw/security-workshops/blob/master/Setup-kerberos-Ambari.md#run-ambari-security-wizard)

---------------
  
### Demo scenario
Site to site data ingestion is a common problem for compagnies doing big data with geographically distributed business. 
Site to site data ingestion is collecting data generated at different regional sites and bringing it to a core site for further analysis. 
An example of this is a manifacturing company having facories in several countries. 
At a factory level, each machine generates sensors data stored in files that should be sent to the core site. 
At the core site, data coming from all factories is aggregated and used for supervision and maintenance. 
For instance, data can be stored in a Hadoop cluster (HDFS or HBase) to implement predective maintenance application with Spark.

![Scenario] (https://raw.githubusercontent.com/ahadjidj-hw/NiFi/master/Site2Site.png)

### Challenges with site to site ingestion
Implementing a site 2 site data flow with traditional Hadoop solutions is a hard, time consuming task. 
Network communications can be instable and offer limited bandwidth. 
IoT systems where 3G or 4G technologies are used to collect sensors data is an exemple of these conditions.
Adapting the transmission rate to the available bandwidth is a real challenge when managin data in motion.
Generated files can be huge which makes it impossible to send them directly. 
Finally, securing data and providing provenance when data travels on public networks is required due to the sensitive nature of data.

### NiFi for site 2 site ingestion
The previously described scenario is a piece of cake for NiFi. With NiFi, we easily implement a data flow to collect data from several sites with security and data provenance.
Nifi can split generated files into smaller pieces and compress them to optimize network transmissions. 
These pieces can be merged into bigger files on the core site to optimize data storage and processing (eg. avoid small files on HDFS).
All these features and others are available out of the box.

Let see how we can build this dataflow in only few minutes with NiFi. Let's consider that we have a regional site (site 1) and a core site (site 2).
Site 1 has a NiFi instance to collect, split, compress and send generated files to site 2. Site 2 has a NiFi instance to collect, decompress and merge these splits.
Site 2 has also an HDP cluster to store and process the data. The following picture shows this process at a very high level.

![NiFi] (https://raw.githubusercontent.com/ahadjidj-hw/NiFi/master/NiFi-S2S.png)

### Demo setup

- **Preparing site 1:** installing NiFi
  - Download Hortonworks DataFlow [here](http://hortonworks.com/hdp/downloads/#hdf)
	- Decompress the content of the file in a local directory (NiFi_path)
	- Edit nifi.properties in /NiFi_path/conf/
	- Set nifi.web.http.port to 8686 (the default 8080 port will be used by HDP in Site 2)
	- Start NiFi /NiFi_path/bin/nifi.sh start
	- Make sure that you have access to NiFi on http://localhost:8686/nifi/
	
- **Preparing site 2:** installing HDF and HDP
  - The easiest way to get site 2 up and running is to use Hortonworks Sandbox VM
  - Download the Sandbox [here](http://hortonworks.com/products/hortonworks-sandbox/#install) and start it with VirtualBox. This will give a working Hadoop installation
  - Follow the same instructions above to install NiFi on the Sandbox VM
    - Set nifi.web.http.port to 8787
    - Set nifi.remote.input.secure to false
  - Make sure that you have access to NiFi on http://localhost:8787/nifi/ and to Ambari on http://localhost:8080
    - In VirtualBox, you have to add forwarding for port 8787
    
### Design data workflow for site 1
  - 
