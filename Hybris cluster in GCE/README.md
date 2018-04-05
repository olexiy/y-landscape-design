## Step by step creation of the small load Hybris test cluster with B2C Accelerator and system monitoring

#Setting the goal
The outcome of this Step-by-Step TODO is going to be a basic Hybris cluster with to simple, B2C Accelerator based, shops ready for a testing or to be used for Hybris demo.
The described process is mainly actions steps described in the [Build an SAP Hybris Clustered Landscape](https://www.udemy.com/build-an-sap-hybris-environment/learn/v4/overview) minus Hybris theory and explanations plus my thoughts and tips.
A nice Hybris landscape analytics based on the concurrent user count could be found here: [SAP HYBRIS – PLATFORM SIZING](https://www.hybhub.com/sap-hybris-platform-sizing/).
Our implementation is a simplified medium solution with back- and front-hybris-server, SOLR cluster, and MySQL Database but without Load Balancing, Apache and MySQL Cluster. Deployment recipes for complete solutions from this page could follow later.
For this course you are going to need SAP Hybris 6.0+ (I am using version 6.6)

#Landscape
+ 2 x Hybris (Fron- and Back-Office)
+ 2 x Apache Solr (Master & Slave)
+ 1 x MySQL Database

GCP instance type:  n1-standard-2 (2 vCPUs, 7.5 GB memory)

OS: Ubuntu 16.04

Solr and MySQL are going to be placed inside one common instance and Hybris servers in two separate.


# 1. Setting up the GCE access and  creating the service insatances.
Today using the cloud for the study, hobby or professional IT project is more convenient and cost/time efficient as doing it locally on a dedicated server or in VM on the developer machine. All of the solutions described are applicable with some minor changes by the all biggest cloud platform providers of today: AWS, Google Compute Cloud, Digital Ocean. I have selected Google Cloud Platform as a cloud of choice for this and bigger part of the future tutorials based on the following thoughts:
* I am starting based on the [Build an SAP Hybris Clustered Landscape](https://www.udemy.com/build-an-sap-hybris-environment/learn/v4/overview) Udemy course, and it was used there.
* GCP gas a [free 300$ credit](https://cloud.google.com/free) for the first year. And if your trial period or credit expiers you can just create the new user account for the new credit.
* GCP IMHO ist easier to start with compared to the AWS.
* If your instance is shut down it costs almost nothing (Digital Ocean have the same price for the active and suspended instance)

1.1 [Go to the Google Cloud Platform console](https://console.cloud.google.com/)

1.2 Create the new project or select one you allready have.

1.3 Set up your optimal [zone](https://cloud.google.com/compute/docs/regions-zones/). In my case it is **europe-west3-b**. Please replace it in all scripts with the zone of your chois.

1.3 Create new instance with folowing settings
 - Name : database-solr-insatance
 - Machine type: 2 vCPUs
 - Operating system : Ubuntu 16.04
 - Allow HTTP Trafick : checked
 - Allow HTTPS Trafick : checked

Tip: If you whant to save some costs turn on the Preemptibility setting. In this case you instance is going to be accusionally turned of (every 24 h) but will cost you mach chiper.

1.4 Create the second instance with the name ´hybris-back-office´ and the same properties.

1.5 Create two new firewall rules to allow ingress and egress trafic on the 0.0.0.0/0  with "Allow all" setting. !!! This is not the secure way to set acces settings, just esiest.

1.6 Download and install [GCloud SDK](https:cloud.google.com/sdk/downloads). Set it up with you account and project data.

# 2 Setting up MySQL
2.1 Conect to the database instance. Where is a waraity of posabilities sujested by GCP, but i am going to use gcloud commands in this tutorial.
```
 gcloud compute --project "hybris-project-198721" ssh --zone "europe-west3-b" "database-solr-instance"
```
!!! Do not forget to change the availability zone
2.2 Install MySQL, SOLR and JRE on the server.
```
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install mysql-server -y && sudo apt-get install unzip -y && sudo apt-get install default-jre -y
```
Remember password you have set for root.
2.3 Secure MySQL with command **sudo mysql_secure_installation** and anwer all questions but about root passwort with ***y***.

2.4 Create the hybris user and database shema.
```
mysql -u root -p
create database hybris;
create user 'hybrisusr'@'%' identified by 'hybrispwd';
grant all on hybris.* to 'hybrisusr';
```
# 3. Setting up SOLR
For the SOLR source we are going to use embedded in Hybris Solr version.
3.1 Unzip Hybris locally.
3.2 Create a Zip for the Solr Master instance by editing Solr configuration file ***\hybris\bin\ext-commerce\solrserver\resources\solr\server\solr\configsets\default\conf\solrconfig.xml*** by removing comment around master configuration. To find this part of the configuration search for the **<requestHandler name="/replication" class="solr.ReplicationHandler" >**.
3.3 Zip the content of the ***\hybris\bin\ext-commerce\solrserver\resources\solr\server\*** folder into the **solr_master.zip**.
3.4 Now coment master part of the configuration back and uncomment slave part. Edit path to the master loke this :
```
<lst name="slave">
  <str name="masterUrl">http://localhost:8983/solr/${solr.core.name}/replication</str>
  <str name="pollInterval">00:00:60</str>
</lst>
```
3.5 Zip the **solr_slave.zip**
3.6 Create folders solr_master and solr_slave in the database-solr-instance.
3.7 Upload the solr_master.zip and solr_slave.zip to the server like this:
```
gcloud compute scp solr_master.zip database-solr-instance:/home/olsak/solr_master --zone europe-west3-b
gcloud compute scp solr_slave.zip database-solr-instance:/home/olsak/solr_slave --zone europe-west3-b
```
3.8 Start the Solr services
```
//starting master
cd solr_master
unzip solr_master.zip
chmod a+x bin/solr
./bin/solr start

//starting slave
cd solr_slave
unzip solr_slave.zip
chmod a+x bin/solr
```
3.8 Optional:  Configeure the autostart for the Solr services:
  TODO!!!

# 4. Setting up Hybris backend server.
4.1 Upload and unpack HYBRIS
```
gcloud compute --project "hybris-project-198721" ssh --zone "europe-west3-b" "hybris-back-office"
mkdir hybris
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install unzip -y && sudo apt-get install  default-jre -y
gcloud compute scp commerce-suite-6.6.0.0-20171206.211610-24554.zip hybris-back-office:/home/olsak/hybris --zone europe-west3-b
unzip commerce-suite-6.6.0.0-20171206.211610-24554.zip
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64/"
```
4.2 Execute B2C Accelerator recepie
```
./installer/install.sh -r b2c_ass
cd installer
./install.sh -r b2c_acc
./install.sh -r b2c_acc initialize
./install.sh -r b2c_acc start
```
4.3 Add a MySQL  java connector:
* [Download mysql connector jar](https://dev.mysql.com)
* Copy it to the hybris/bin/platform/lib/dbdriver

4.4 Change databese for the Hybris backoffice
* Update local.properties whit Db config part of the local.properties muster from this project. Update datza **db.url** with internal IP of the ***database-solr-instance***
* run ***ant aclean all***
* run ***./hybrisserver.sh***
* by connection issues during the server start update MySQL config file under ***/etc/mysql/mysql.conf.d/mysql.cnf*** by commenting the **bin-address** propertie.
* open HMC on https://{server public ip}:9002 and initialize Hybris

4.5 Update the Solr configuration
* Disable Solr autostar in local.properties like this and rebuild Hybris:
```
## Disable Solr Server Autostart ##jöhlg.bvbvnöä bm-vn. 
solrserver.instances.default.autostart=false
```
4.6 Configure new Solr instances with solr import in HMC with solr.impex
4.7 Run full import job in backoffice: System -> Search and Navigation

# 5. Creating the Hybris cluster.

# 6. Setting up the Hybris monitoring.
