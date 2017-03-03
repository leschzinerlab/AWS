# Cryo-EM Cloud Tools
Software to interface with AWS through convenient command line inputs, ultimately connecting into Relion-2.0's GUI to launch cloud processing from local Relion-2.0 GUI. 

To learn more: [cryoem-tools.cloud] (cryoem-tools.cloud)

*Contents:*
* [Typical Workflow] (https://github.com/leschzinerlab/AWS#typical-workflow)
* [Getting started] (https://github.com/leschzinerlab/AWS#getting-started)
	* [Software dependences] (https://github.com/leschzinerlab/AWS#software-dependencies)
	* [Environment setup] (https://github.com/leschzinerlab/AWS#environment-setup)
* [Usage] (https://github.com/leschzinerlab/AWS#usage)
* [Removing temporary data storage on AWS] (https://github.com/leschzinerlab/cryoem-cloud-tools#removing-temporary-data-storage-on-aws)

*Overview:*

Shortcut AWS commands found in this Github repo:

**Popular commands**

* **awshelp** - List available AWS commands from this repo

* **awsls** - List volumes and instances (Include -i to list only instances)

* **awskill** - Terminate instance OR cluster

* **awslaunch** - Boot up single instance

**Other commands**

* **awslaunch_cluster** - Boot up cluster of instances using STARcluster 

* **aws_spot_price_history** - List spot prices for a given instance and availability zone

* **aws_ebs_create** - Create EBS volume

* **aws_ebs_delete** - Delete EBS volume

* **aws_ebs_attach** - Attach EBS volume to running instance

* **aws_ebs_detach** - Detach EBS volume from instance

* **delete_temp_s3_ebs** - Delete all S3 bucket and EBS volumes with temporary status (name starts with rln-aws-tmp)
 
## Typical Workflow
In using these wrappers, we typically use only a few of the commands for booting up & terminating instances.

### Create new EBS volume for data upload

You will store and process your data on EBS volumes, so to create an EBS volume you specify the volume size (in Gigabytes) and the availability zone (keep in mind that your initial setup above assumes a certain region). This amount of data storage is static, so, after creating it is very difficult to expand the size. Choose the size wisely! 
<pre>$ aws_ebs_create

Usage: aws_ebs_create [size in GB] [zone] "Description in double quotes"</pre>

<pre>$ aws_ebs_create 100 us-west-2c "My shiny data"

Create volume in us-west-2c that is 100 GB? [Y/n] Y

Creating volume ...
</pre>

You will now see this in your list of AWS resources:

<pre>$ awsls
----------------------------------------------------------------------------------------------------------
Volume ID       Description             Avail. Zone     Size    User            Status          Instance
----------------------------------------------------------------------------------------------------------
vol-85cb3210    My shiny data 	 	us-west-2c    100GB   mike_oregon     available       --
</pre>

### Boot up instance with EBS volume attached

At this point, we recommend p2 instances for particle picking / ctf estimation (p2.xlarge), 2D classification (p2.16xlarge), and 3D classification/refinement (p2.8xlarge). To start one of these instances with your EBS volume attached: 

<pre>$ awslaunch --instance=p2.8xlarge --availZone=us-west-2c --volume=vol-85cb3210 </pre>

**Note:** If you want to upload data, use p2.8xlarge ($0.90/hr) or c4.xlarge ($0.199/hr).

After running this command, the log in information will be shown on the command line or you can always see it using: 

<pre>$ awsls 
AWS EC2 information for user mike_oregon in region us-west-2

----------------------------------------------------------------------------------------------------
InstanceType	Avail. Zone	InstanceID		Status		User		Login info
----------------------------------------------------------------------------------------------------
p2.8xlarge	us-west-2c	i-98sdkfksdf9ssd9	running		mike_oregon	ssh -X -i /home/michaelc/.aws/mike_oregon.pem ubuntu@35.322.393
</pre> 

To log on, just copy the ssh information into your command line: 

<pre>$ ssh -X -i /home/michaelc/.aws/mike_oregon.pem ubuntu@35.322.393</pre>

And it will log you onto your machine. 

**Your data will be located in the folder /data/**:

To access: 
<pre>$ cd /data</pre>

### Terminate instance
When you are finished analyzing your data, you can terminate your instance using: 

<pre>$ awskill 

Usage: awskill [instance ID or cluster name]</pre>

To kill your instance: 

<pre>$ awskill i-98sdkfksdf9ssd9 </pre>

**Note:** If there are any running processes such as a relion GUI or an open terminal, it cannot terminate the instance. Kill / stop these processes and then try again. 

## Getting started

###Software dependencies 
You'll need to install *pip*, *aws cli* **starcluster**, **cryptography**, and *fabric*: 
* **pip**: 
	* https://pypi.python.org/pypi/pip
* **aws cli**:
	* http://docs.aws.amazon.com/cli/latest/userguide/installing.html#install-with-pip
	* <pre>$ sudo pip install awscli</pre>
* **starcluster**: 
	* http://star.mit.edu/cluster/docs/latest/index.html
	* <pre>$ sudo pip install starcluster </pre>
	* Only required if you want to boot up clusters of CPU instances
* **fabric**: 
	* http://www.fabfile.org/installing.html
	* <pre>$ sudo pip install fabric</pre>
* **rclone**: 
	* http://rclone.org/
	* <pre>$ wget http://downloads.rclone.org/rclone-v1.35-linux-amd64.zip</pre>
	* <pre>$ unzip rclone-v1.35-linux-amd64.zip</pre>

###Environment setup
For each user, you will create a hidden directory in their home directory into which you will add the aws/aws_init.sh file and their keypair.  

* Create hidden folder: 
<pre>$ mkdir /home/[user]/.aws</pre>

* Copy aws/aws_init.sh file & edit to include credentials

* Copy keypair into directory, making sure to modify permissions of file using <pre>chmod 600</pre>

* Add the following line to .bashrc file: 
<pre>source /home/[user]/.aws/aws_init.sh</pre>

## Usage
The underlying code is written in python and aliased to simple commands: awsls, awslaunch, awskill. 

* **awsls**
	* Lists all instances & volumes assigned to user, where user instances are assigned based upon being tagged with key pair name as the instance Owner. 
	* Include **-i** to only show instances that are running
	* Include **-v** to only show EBS volumes
	* Include **-c** to include listings of any clusters created with STARcluster 
	* Example usage: 
		<pre>$ awsls
		---------------------------------------------------------------------------------------
		ReservedInstanceType	Avail. Zone	InstanceID	Status	IP Address	User
		---------------------------------------------------------------------------------------
		No instances found
		----------------------------------------------------------------------------------------------------------------------------------------
		SpotInstanceType	Avail. Zone	SpotInstanceID	SpotStatus	InstanceID	Status		IP Address	Price	User	
		----------------------------------------------------------------------------------------------------------------------------------------
		m3.medium		us-west-2a	sir-b6dg94hn	closed		---		---		---		$0.040	mike_oregon
		----------------------------------------------------------------------------------------
		Volume ID	Avail. Zone	Size	User		Status		Instance
		----------------------------------------------------------------------------------------
		vol-169efda2	us-west-2a	400GB	mike_oregon	available	--</pre>
	
* **awslaunch**
	* Command to launch instance, configuring security group into VPC automatically to only allow users IP address for incoming SSH traffic.
	* Example usage: 
		<pre>$ awslaunch
		Usage: awslaunch --instance=<instanceType>
		Options:
  		  -h, --help          show this help message and exit
 	 	  --instance=STRING   Specify instance type to launch
  		  --availZone=STRING  Specify availability zone
		  --spotPrice=FLOAT   Optional: Specify spot price (if spot instance
               		              requested)
  		  --volume=STRING     Optional: Specifiy volume ID to be mounted onto instance
                 		      (Must be same avail. zone)
  		  --relion2           Optional: Flag to use relion2 environment on non-GPU
                      		      machines (By default, relion2 software is only loaded
                      		      onto GPU (p2) instances)
		  --rosetta           Optional: Flag to use rosetta environment (Rosetta runs
             		              on CPUs)
		  --instanceList      Flag to list available instances
		  -d                  debug</pre>
		<pre>$ awslaunch --instance=t2.micro
		Launching AWS instance t2.micro for user keyName_virginia
		Configuring security settings ...
		Booting up instance ...
		Waiting for instance to pass system checks ...
		Instance is ready! To log in:
		ssh -i /home/[user]/.aws/keyName_virginia.pem ubuntu@54.209.133.219</pre>

* **awskill**
	* Command to terminate running instance or STARcluster
	* Example usage: 
		<pre>$ awskill
		Usage: awskill [instance ID]
		Specify instance ID that will be terminated, which can be found using "awsls"</pre>
		<pre>$ awskill i-112k43e
		Terminate instance i-112k43e? [Y/n] Y
		Removing instance ...</pre>
		<pre>$ awskill cluster-m3.medium</pre>

* **aws_ebs_create**
	* Command to create new EBS volume
	* Example usage: 
		<pre>$ aws_ebs_create 
		Usage: aws_ebs_create [size in GB] [zone]
		Specify size of EBS volume to be created (in GB) along with availability zone (e.g. us-east-1b)</pre>
		<pre>$ aws_ebs_create 100 us-east-1b</pre>

* **aws_ebs_delete**
	* Command to delete EBS volume. Be careful! This cannot be undone.
	* Example usage: 
		<pre>$ aws_ebs_delete 
		Usage: aws_ebs_delete [volume ID]
		Specify EBS volume to delete. Warning: Cannot be undone!!</pre>
		<pre>$ aws_ebs_delete vol-id559699</pre>

* **aws_ebs_attach**
	* Command to attach EBS volume to running instance.
	* Example usage: 
		<pre>$ aws_ebs_attach 
		Usage: aws_ebs_attach [instance ID] [volume ID]
		Attach EBS volume to instance.</pre>
		<pre>$ aws_ebs_attach i-112k43e vol-id559699</pre>

* **aws_ebs_detach**
	* Command to attach EBS volume to running instance.
        * Example usage:
                <pre>$ aws_ebs_detach
                Usage: aws_ebs_detach [volume ID]
                Detach EBS volume from instance.</pre>
                <pre>$ aws_ebs_detach vol-id559699</pre>

* **awslaunch_cluster**
        * Command to launch a cluster of instances using STARcluster. Note: Must have starcluster already installed (see above for install info).
        * Example usage:
                <pre>$ awslaunch_cluster
Usage: awslaunch_cluster --instance=<instanceType>
Options:
  -h, --help          show this help message and exit
  --instance=STRING   Specify instance type to launch into cluster
  --num=INTEGER       Number of instances in cluster
  --availZone=STRING  Specify availability zone
  --volume=STRING     Optional: Volume ID for volume that will be mounted onto
                      cluster
  --spotPrice=FLOAT   Optional: Specify spot price (if spot instance
                      requested)
  --instanceList      Flag to list available instances
  -d                  debug</pre>
                <pre>$ awslaunch_cluster --instance=c3.xlarge --num=4 --availZone=us-west-2a --spotPrice=0.2</pre>

* **aws_spot_price_history**
        * Command to list spot prices over the past 24 hours
        * Example usage:
        <pre>$ aws_spot_price_history
Usage: aws_spot_price_history [instance type] [avail. zone]
Specify instance type over which spot price history will be listed based upon availability zone</pre>
        <pre>$ aws_spot_price_history m3.meduim us-west-2a</pre>

## Removing temporary data storage on AWS

During the course of processing, this workflow will temporarily place data onto S3 and EBS volumes to speed up data processing. This data can be removed by running the command: 

<pre>$ delete_temp_s3_ebs </pre>

This will remove all S3 buckets and EBS volumes that have the naming pattern: rln-aws-tmp. 

All buckets and volumes will be removed based upon the number of days specified in your aws_init.sh file, where any bucket or volume older than the specified time will be removed. 
