# Using HDF for multisite data ingestion into HDP

## Contents
  - [Scenario](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#scenario)
  - [Challenges with multisite data ingestion](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#challenges-with-multisite-data-ingestion)
  - [HDF for multisite data ingestion](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#nifi-for-multisite-data-ingestion)
  - [Setup](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#setup)
  - [Design data workflow for site 1](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#design-data-workflow-for-site-1)
  - [Design data workflow for site 2](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#design-data-workflow-for-site-2)
  - [Connect the two sites](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#connect-the-two-sites)
  - [Start the workflows](https://github.com/ahadjidj-hw/NiFi/blob/master/Multisite-Data-Ingestion/README.md#start-the-workflows)
  
---------------
  
### Scenario
Multisite data ingestion is a common problem for companies doing big data with geographically distributed business. 
Multisite data ingestion is collecting data generated at different regional sites and bringing it to a core site for further analysis. 
An example of this is a manufacturing company having factories in several countries. 
At a factory level, each machine generates sensors data and stores it in files that should be sent to the core site. 
At the core site, files coming from all factories are merged and stored for further processing. 
For instance, data can be stored in a Hadoop cluster (HDFS or HBase) to implement predictive maintenance application with Spark.

![Scenario] (https://raw.githubusercontent.com/ahadjidj-hw/NiFi/master/Multisite-Data-Ingestion/images/Site2Site.png)

### Challenges with multisite data ingestion
Implementing a multisite data flow with traditional Hadoop solutions is a tedious and time consuming task. 
Network communications can be instable and have limited bandwidth. 
IoT systems where 3G or 4G technologies are used to collect sensor data is an example of this situation.
Adapting the transmission rate to the available bandwidth has the potential to improve data transmission in such conditions. 
Also, generated files can be huge which makes it impossible to send them directly. 
Finally, securing data and providing provenance while data is travelling on internet is a must due to the sensitive nature of data.

### HDF for multisite data ingestion
The previously described scenario is a piece of cake for HDF powered by Apache Knife. With HDF, we easily implement a dataflow to collect data from several sites with security and data provenance.
HDF can split generated files into smaller pieces and compress them to optimize data transmissions. 
These pieces can be merged into bigger files on the core site to optimize data storage and processing (eg. avoid small files on HDFS).
All these features and others are available out of the box.

Let see how we can build this dataflow in only few minutes with HDF. Let's consider that we have a regional site (site 1) and a core site (site 2).
Site 1 has a HDF instance to collect, split, compress and send generated files to site 2. Site 2 has a HDF instance to collect, decompress and merge these splits.
Site 2 has also an HDP cluster to store and process the data. The following picture shows this process at a very high level.

![HDF] (https://raw.githubusercontent.com/ahadjidj-hw/NiFi/master/Multisite-Data-Ingestion/images/NiFi-S2S.png)

### Setup

- **Preparing site 1:** installing HDF
  - Download Hortonworks DataFlow [here](http://hortonworks.com/hdp/downloads/#hdf)
	- Decompress the content of the file in a local directory NIFI_PATH
	- Edit nifi.properties in NIFI_PATH/conf/
	- Set nifi.web.http.port to 8686 (the default 8080 port will be used by HDP in Site 2)
	- Start NiFi `NIFI_PATH/nifi.sh start`
	- Make sure that you have access to NiFi on http://localhost:8686/nifi/
	
- **Preparing site 2:** installing HDF and HDP
  - The easiest way to get site 2 up and running is to use Hortonworks Sandbox VM
  - Download the Sandbox [here](http://hortonworks.com/products/hortonworks-sandbox/#install) and start it with VirtualBox. This will give you a working Hadoop installation
  - Follow the same instructions above to install NiFi on the Sandbox VM
    - Set nifi.web.http.port to 8787
    - Set nifi.remote.input.secure to false
  - Make sure that you have access to NiFi on http://localhost:8787/nifi/ and to Ambari on http://localhost:8080
    - In VirtualBox, you have to add port forwarding for port 8787
    
### Design data workflow for site 1

Starting with a blank NiFi canvas in Site 1. Drag and drop processors to the canvas to design the workflow:

1. Add a GetFile processor
	2. Right click on the processor, choose configure and go to the properties tab
	3. Set Input Directory to /tmp/input/
	4. Create the /tmp/input directory on your machine and make sure NiFi has access to it `sudo chown nifi:nifi /tmp/input`
1. Add a SegmentContent processor and configure it
	2. Set Segment Size to 1 MB. This will split each flow file into 1 MB segments
1. Add an UpdateAttribut processor
	2. Press the plus sign in the top right of the dialog to add a new property
	3. Add a property called "mime.type" and set this to "application/gzip"
1. Add a compress content processor
1. Add a Remote Process Group
	2. Set URL to http://localhost:8687/nifi (URL of the second NiFi instance)

Connect the processors to build the data flow logic. To connect two processors, hover over the source processor until the begin connection icon appears, drag this onto the destination processor and release it. This brings up the connection configuration dialog.

1. Connect the GetFile to the SegmentContent processor
	2. Leave the default properties and press add
2. Connect the SegmentContent to the UpdateAttribut processor
	3. Check only `segments` in the Relations option and press add. The UpdateAttribut will receive the segment and not the original file
	4. Right click on the SegmentContent processor, choose configure and go to the Settings tab
	5. In the Auto terminate relations, choose `Original` to drop the original file
3. Connect the UpdateAttribut to the CompressContent processor
	4. Leave the default properties and press add

We won't be able to connect the UpdateAttribut to the CompressContent processor for now. We need to design the site 2 workflow before.

### Design data workflow for site 2

Starting with a blank NiFi canvas in Site 2. Drag and drop processors to the canvas to design the workflow:

1. Add an InputPort and name it RemoteIngest
2. Add a CompressContent processor and configure it
	3. Set mode to decompress and validate
3. Add a MergeContent processor and configure it 
	4. Here we have a choice between two strategies. (1) Merge the segments to get the exact same original files (2) Merge the segments independently of their original files. Set Merge Strategy to `Defragment` to choose the first option.
5. Add a PutHDFS processor and configure it
	6. Set Hadoop Configuration Resources to `/etc./hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml` 
	7. Set Directory to `/tmp/nifi`
	8. Set Conflict Resolution Strategy to `Replace`

Now connect the different processors as following

1. Connect the Input port to the CompressContent processor and press add
2. Connect the CompressContent to the MergeContent processor
	3. Choose `Success` in the relation type
	4. Right click on the CompressContent processor, choose configure and go to the settings tab
	5. In the Auto terminate relations, check `Failure` to drop the flow file in case the decompression fails
3. Connect the MergeContent to the PutHDFS processor
	4. Check only `Merged` in the Relations option and press add. The PutHDFS will receive the merged file and not the segments
	5. Right click on the MergedContent processor, choose configure and go to the Settings tab
	6. Choose auto terminate `Failure` and `Original` relations
	7. Right click on the PutHDFS processor, choose configure and go to the settings tab
	8. Choose auto terminate `Failure` and `Success` relations

At this level, we should have all the processors configured. This means that their status is stopped (red square) as in the following screenshot.

! [Site2] (https://raw.githubusercontent.com/ahadjidj-hw/NiFi/master/Multisite-Data-Ingestion/images/Site2.png)

### Connect the two sites

Now that we have the dataflow designed in site 2, we can continue our design in site 1. In the first NiFi workflow designer, do the following:

1. Connect the CompressContent to the Remote Process Group
	2. Choose the `Success` relation and press add
	3. Make sure that the RemoteIngest input is selected in the `To Input` property
4. Right click on the CompressContent processor, choose configure and go to the Settings tab
	8. Choose auto terminate `Failure` relation
9. Right click on the Remote Group Process and click on enable transmission

At this level, we should have all the processors configured in site 1 as in the following screenshot.

![Site2] (https://raw.githubusercontent.com/ahadjidj-hw/NiFi/master/Multisite-Data-Ingestion/images/Site1.png)

### Start the workflows

Select all the processors in site 1 and press play at the top of the screen. Do the same thing in site 2. To test the workflow, copy several files to the input folder in site 1 `/tmp/input`. NiFi will pick those files, splits them into 1 MB pieces, compress each piece and sends them to Site 2. NiFi in Site 2 receives the compressed pieces, decompress them, and merges pieces from them same file together to reconstruct the original files. Finally, those files are stored in HDFS in `/tmp/nifi`. Take a minute to verify this behavior and observe the number of input/output files in each processor as well as the number of input/output bytes. Finally, verify that you have received the files in HDFS through Ambari HDFS view.