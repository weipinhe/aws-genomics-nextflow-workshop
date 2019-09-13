# Nextflow on AWS

- [Nextflow on AWS](#nextflow-on-aws)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Module 0 - Cloud9 Environment Setup](#module-0---cloud9-environment-setup)
  - [Module 1 - AWS Resources](#module-1---aws-resources)
    - [S3 Bucket](#s3-bucket)
    - [IAM Roles](#iam-roles)
      - [Create a Batch Service Role](#create-a-batch-service-role)
      - [Create an EC2 Instance Role](#create-an-ec2-instance-role)
      - [Create an EC2 SpotFleet Role](#create-an-ec2-spotfleet-role)
    - [EC2 Launch Template](#ec2-launch-template)
    - [Batch Compute Environments](#batch-compute-environments)
      - [Create an "optimal" on-demand compute environment](#create-an-%22optimal%22-on-demand-compute-environment)
      - [Create an "optimal" spot compute environment](#create-an-%22optimal%22-spot-compute-environment)
    - [Batch Job Queues](#batch-job-queues)
      - [Create a "default" job queue](#create-a-%22default%22-job-queue)
      - [Create a "high-priority" job queue](#create-a-%22high-priority%22-job-queue)
  - [Module 2 - Running Nextflow](#module-2---running-nextflow)
    - [Local master and jobs](#local-master-and-jobs)
    - [Local master and AWS Batch jobs](#local-master-and-aws-batch-jobs)
    - [Batch-Squared](#batch-squared)

## Overview

The amount of genomic sequence data has exponentially increased year over year since the introduction of NextGen Sequencing nearly a decade ago.  While traditionally this data was processed using on-premise computing clusters, the scale of recent datasets, and the processing throughput needed for clinical sequencing applications can easily exceed the capabilities and availability of such hardware.  In contrast, the cloud offers unlimited computing resources that can be leveraged for highly efficient, performant, and cost effective genomics analysis workloads.

Nextflow is a highly scalable reactive open source workflow framework that runs on infrastructure ranging from personal laptops, on-premise HPC clusters, and in the cloud using services like AWS Batch - a fully managed batch processing service from Amazon Web Services.

This tutorial will walk you through how to setup AWS infrastructure and Nextflow to run genomics analysis pipelines in the cloud.  You will learn how to create AWS Batch Compute Environments and Job Queues that leverage Amazon EC2 Spot and On-Demand instances.  Using these resources, you will build architecture that runs Nextflow entirely on AWS Batch in a cost effective and scalable fashion.  You will also learn how to process large genomic datasets from Public and Private S3 Buckets. By the end of the session, you will have the resources you need to build a genomics workflow environment with Nextflow on AWS, both from scratch and using automated mechanisms like CloudFormation.

## Prerequisites

Attendees are expected to:

* have administrative access to an AWS Account
* be comfortable using the Linux command line (e.g. using the `bash` shell)
* be comfortable with Git version control at the command line
* have familiarity with the AWS CLI
* have familiarity with Docker based software containers
* have familiarity with genomics data formats such as FASTQ, BAM, and VCF
* have familiarity with genomics secondary analysis steps and tooling
* have familiarity with Nextflow and its domain specific workflow language

## Module 0 - Cloud9 Environment Setup

Start you Cloud9 IDE:

* Go to the AWS Cloud9 Console
* Go to **Your Environments**
* Select the **"genomics-workflows"** environment
* Click on the **Open IDE** button

This will launch AWS Cloud9 in a new tab of your web browser.  The IDE takes about 1-2min to spin up.

Associate an administrative role to the EC2 Instance used by Cloud9:

* Go to the AWS EC2 Console
* Select the instance named **"aws-cloud9-genomics-workflows-xxxxxx"** where "xxxxxx" is the unique id for your Cloud9 environment.
* In the **Actions** drop-down, select **Instance Settings > Attach/Replace IAM Role**
* Select the role named **"NextflowAdminInstanceRole"**
* Click **Apply**

Cloud9 normally manages IAM credentials dynamically. This isn’t currently compatible with the `nextflow` CLI, so we will disable it and rely on the IAM instance role instead.

* Goto to your Cloud9 IDE and in the Menu go to **AWS Cloud9 > Preferences**.  This will launch a new "Preferences" tab.
* Select **AWS SETTINGS**
* Turn off **AWS managed teporary credentials**
* Close the "Preference" tab
* In a bash terminal tab type the following to remove any existing credentials:

```bash
rm -vf ~/.aws/credentials
```

* Verify that credentials are based on the instance role:

```bash
aws sts get-caller-identity

# Output should look like this:
# {
#     "Account": "123456789012", 
#     "UserId": "AROA1SAMPLEAWSIAMROLE:i-01234567890abcdef", 
#     "Arn": "arn:aws:sts::123456789012:assumed-role/NextflowAdminInstanceRole/i-01234567890abcdef"
# }
```

If your output does not match the above **DO NOT PROCEED**.  Ask for assistance.

Configure the AWS CLI with the current region as default:

```bash
sudo yum install -y jq
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

echo "export ACCOUNT_ID=${ACCOUNT_ID}" >> ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

Install Nextflow:

```bash
# nextflow requires java 8
sudo yum install -y java-1.8.0-openjdk

# The Cloud9 AMI is Amazon Linux 1, which defaults to Java 7
# change the default to Java 8 and set JAVA_HOME accordingly
sudo alternatives --set java /usr/lib/jvm/jre-1.8.0-openjdk.x86_64/bin/java
export JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk.x86_64

# do this so that any new shells spawned use the correct
# java version
echo "export JAVA_HOME=$JAVA_HOME" >> ~/.bash_profile

# get and install nextflow
mkdir -p ~/bin
cd ~/bin
curl -s https://get.nextflow.io | bash

cd ~/environment
```

When the above is complete you should see something like the following:

```text
      N E X T F L O W
      version 19.07.0 build 5106
      created 27-07-2019 13:22 UTC 
      cite doi:10.1038/nbt.3820
      http://nextflow.io


Nextflow installation completed. Please note:
- the executable file `nextflow` has been created in the folder: /home/ec2-user/bin
```

## Module 1 - AWS Resources

This module will cover building the AWS resources you need to run Nextflow on AWS from scratch.

If you are attending an in person workshop, these resources have been created ahead of time in the AWS accounts you were provided.  If you are running this lab in your own account, use the CloudFormation templates available at the link below to setup your own environment.

[Genomics Workflows on AWS - Nextflow Full Stack](https://docs.opendata.aws/genomics-workflows/orchestration/nextflow/nextflow-overview/#full-stack-deployment)

### S3 Bucket

You'll need an S3 bucket to store both your input data and workflow results.
S3 is an ideal location to store datasets of the size encountered in genomics, 
which often equal or exceed 100GB per sample.

S3 also makes it easy to collaboratively work on such large datasets because buckets
and the data stored in them are globally available.

* Go to the S3 Console
* Click on the "Create Bucket" button

In the dialog that opens:

  * Provide a "Bucket Name".  This needs to be globally unique.  A pattern that usually works is

```text
<workshop-name>-<your-initials>-<date>
```

for example:

```text
nextflow-workshop-abc-20190101
```

  * Select the region for the bucket.  Buckets are globally accessible, but the data resides on physical hardware with in a specific region.  It is best to choose a region that is closest to where you are and where you will launch compute resources to reduce network latency and avoid inter-region transfer costs.

The default options for bucket configuration are sufficient for the purposes of this workshop.

* Click the "Create" button to accept defaults and create the bucket.


### IAM Roles

IAM is used to contol access to your AWS resources.  This includes access by users and groups in your account, as well as access by AWS services operating on your behalf.

Services use IAM Roles which provide temporary access to AWS resources when needed.

**IMPORTANT**:
> You need to have Administrative access to your AWS account to create IAM roles.
>
> The recommended way to do this is to create a user and add that user to a group
> with the `AdministratorAccess` managed policy attached.  This makes it easier to 
> revoke these privileges if necessary.

#### Create a Batch Service Role

This is a role used by AWS Batch to launch EC2 instances on your behalf.

This can be done in the AWS Console.

* Go to the IAM Console
* Click on "Roles"
* Click on "Create role"
* Select "AWS service" as the trusted entity
* Choose "Batch" as the service to use the role
* Click "Next: Permissions"

In Attached permissions policies, the "AWSBatchServiceRole" will already be attached

* Click "Next: Tags".  (adding tags is optional)
* Click "Next: Review"
* Set the Role Name to "AWSBatchServiceRole"
* Click "Create role"


#### Create an EC2 Instance Role

This is a role that controls what AWS Resources EC2 instances launched by AWS Batch have access to.
In this case, you will limit S3 access to just the bucket you created earlier.

This can be done in the AWS Console

* Go to the IAM Console
* Click on "Roles"
* Click on "Create role"
* Select "AWS service" as the trusted entity
* Choose EC2 from the larger services list
* Choose "EC2 - Allows EC2 instances to call AWS services on your behalf" as the use case.
* Click "Next: Permissions"
* Type "ContainerService" in the search field for policies
* Click the checkbox next to "AmazonEC2ContainerServiceforEC2Role" to attach the policy
* Click "Next: Tags".  (adding tags is optional)
* Click "Next: Review"
* Set the Role Name to "ecsInstanceRole"
* Click "Create role"


#### Create an EC2 SpotFleet Role

This is a role that allows creation and launch of Spot fleets - Spot instances with similar compute capabilities (i.e. vCPUs and RAM).  This is for using Spot instances when running jobs in AWS Batch.

This can be done in the AWS Console

* Go to the IAM Console
* Click on "Roles"
* Click on "Create role"
* Select "AWS service" as the trusted entity
* Choose EC2 from the larger services list
* Choose "EC2 - Spot Fleet Tagging" as the use case

In Attached permissions policies, the "AmazonEC2SpotFleetTaggingRole" will already be attached

* Click "Next: Tags".  (adding tags is optional)
* Click "Next: Review"
* Set the Role Name to "AWSSpotFleetTaggingRole"
* Click "Create role"

### EC2 Launch Template

An EC2 Launch Template is used to predefine EC2 instance configuration options such as Amazon Machine Image (AMI), Security Groups, and EBS volumes.  They can also be used to define User Data scripts to provision instances when they boot.  This is simpler than creating (and maintaining) a custom AMI in cases where the provisioning reequirements are simple (e.g. addition of small software utilities) but potentially changing often with new versions.

AWS Batch supports both custom AMIs and EC2 Launch Templates as methods to bootstrap EC2 instances launched for job execution.

In most cases, EC2 Launch Templates can be created using the AWS EC2 Console.
For this case, we need to use the AWS CLI.

Create a file named `launch-template-data.json` with the following contents:

```json
{
  "TagSpecifications": [
    {
      "ResourceType": "instance",
      "Tags": [
        {
          "Key": "architecture",
          "Value": "genomics-workflow"
        },
        {
          "Key": "solution",
          "Value": "nextflow"
        }
      ]
    }
  ],
  "BlockDeviceMappings": [
    {
      "Ebs": {
        "DeleteOnTermination": true,
        "VolumeSize": 50,
        "VolumeType": "gp2"
      },
      "DeviceName": "/dev/xvda"
    },
    {
      "Ebs": {
        "Encrypted": true,
        "DeleteOnTermination": true,
        "VolumeSize": 75,
        "VolumeType": "gp2"
      },
      "DeviceName": "/dev/xvdcz"
    },
    {
      "Ebs": {
        "Encrypted": true,
        "DeleteOnTermination": true,
        "VolumeSize": 20,
        "VolumeType": "gp2"
      },
      "DeviceName": "/dev/sdc"
    }
  ],
  "UserData": "TUlNRS1WZXJzaW9uOiAxLjAKQ29udGVudC1UeXBlOiBtdWx0aXBhcnQvbWl4ZWQ7IGJvdW5kYXJ5PSI9PUJPVU5EQVJZPT0iCgotLT09Qk9VTkRBUlk9PQpDb250ZW50LVR5cGU6IHRleHQvY2xvdWQtY29uZmlnOyBjaGFyc2V0PSJ1cy1hc2NpaSIKCnBhY2thZ2VzOgotIGpxCi0gYnRyZnMtcHJvZ3MKLSBweXRob24yNy1waXAKLSBzZWQKLSB3Z2V0CgpydW5jbWQ6Ci0gcGlwIGluc3RhbGwgLVUgYXdzY2xpIGJvdG8zCi0gc2NyYXRjaFBhdGg9Ii92YXIvbGliL2RvY2tlciIKLSBhcnRpZmFjdFJvb3RVcmw9Imh0dHBzOi8vczMuYW1hem9uYXdzLmNvbS9hd3MtZ2Vub21pY3Mtd29ya2Zsb3dzL2FydGlmYWN0cyIKLSBzZXJ2aWNlIGRvY2tlciBzdG9wCi0gY3AgLWF1IC92YXIvbGliL2RvY2tlciAvdmFyL2xpYi9kb2NrZXIuYmsgIAotIGNkIC9vcHQgJiYgd2dldCAkYXJ0aWZhY3RSb290VXJsL2F3cy1lYnMtYXV0b3NjYWxlLnRneiAmJiB0YXIgLXh6ZiBhd3MtZWJzLWF1dG9zY2FsZS50Z3oKLSBzaCAvb3B0L2Vicy1hdXRvc2NhbGUvYmluL2luaXQtZWJzLWF1dG9zY2FsZS5zaCAkc2NyYXRjaFBhdGggL2Rldi9zZGMgIDI+JjEgPiAvdmFyL2xvZy9pbml0LWVicy1hdXRvc2NhbGUubG9nCi0gY2QgL29wdCAmJiB3Z2V0ICRhcnRpZmFjdFJvb3RVcmwvYXdzLWVjcy1hZGRpdGlvbnMudGd6ICYmIHRhciAteHpmIGF3cy1lY3MtYWRkaXRpb25zLnRnegotIHNoIC9vcHQvZWNzLWFkZGl0aW9ucy9lY3MtYWRkaXRpb25zLW5leHRmbG93LnNoIAotIHNlZCAtaSAncytPUFRJT05TPS4qK09QVElPTlM9Ii0tc3RvcmFnZS1kcml2ZXIgYnRyZnMiK2cnIC9ldGMvc3lzY29uZmlnL2RvY2tlci1zdG9yYWdlCi0gc2VydmljZSBkb2NrZXIgc3RhcnQKLSBzdGFydCBlY3MKCi0tPT1CT1VOREFSWT09LS0="
}
```

The above template will create an instance with three attached EBS volumes.

* `/dev/xvda`: will be used for the root volume
* `/dev/xvdcz`: will be used for the docker metadata volume
* `/dev/sdc`: will be the initial volume use for scratch space (more on this below)

 The `UserData` value is the `base64` encoded version of the following:

```yaml
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="==BOUNDARY=="

--==BOUNDARY==
Content-Type: text/cloud-config; charset="us-ascii"

packages:
- jq
- btrfs-progs
- python27-pip
- sed
- wget

runcmd:
- pip install -U awscli boto3
- scratchPath="/var/lib/docker"
- artifactRootUrl="https://s3.amazonaws.com/aws-genomics-workflows/artifacts"
- service docker stop
- cp -au /var/lib/docker /var/lib/docker.bk  
- cd /opt && wget $artifactRootUrl/aws-ebs-autoscale.tgz && tar -xzf aws-ebs-autoscale.tgz
- sh /opt/ebs-autoscale/bin/init-ebs-autoscale.sh $scratchPath /dev/sdc  2>&1 > /var/log/init-ebs-autoscale.log
- cd /opt && wget $artifactRootUrl/aws-ecs-additions.tgz && tar -xzf aws-ecs-additions.tgz
- sh /opt/ecs-additions/ecs-additions-nextflow.sh 
- sed -i 's+OPTIONS=.*+OPTIONS="--storage-driver btrfs"+g' /etc/sysconfig/docker-storage
- service docker start
- start ecs

--==BOUNDARY==--
```

The above script installs a daemon called `aws-ebs-autoscale` which will create a BTRFS filesystem at a specified mountpoint, spread it across multiple EBS volumes, and add more volumes to ensure availbility of disk space.

In this case, the mount point for auto expanding EBS volumes is set to `/var/lib/docker` - the location of docker container data volumes.  This allows for containers used in the workflow to stage in as much data as they need without needing to bind mount a special location on the host.

The above is used to handle the unpredictable sizes of data files encountered in genomics workflows, which can range from 10s of MBs to 100s of GBs.

Use the command below to create the corresponding launch template:

```bash
aws ec2 \
    create-launch-template \
        --launch-template-name genomics-workflow-template \
        --launch-template-data file://launch-template-data.json
```

You should get something like the following as a response:

```json
{
    "LaunchTemplate": {
        "LatestVersionNumber": 1, 
        "LaunchTemplateId": "lt-0123456789abcdef0", 
        "LaunchTemplateName": "genomics-workflow-template", 
        "DefaultVersionNumber": 1, 
        "CreatedBy": "arn:aws:iam::123456789012:user/alice", 
        "CreateTime": "2019-01-01T00:00:00.000Z"
    }
}
```

Note the `LaunchTemplateId` value, you will need it later.

### Batch Compute Environments

AWS Batch compute environments are groupings of EC2 instance types that jobs are scheduled onto based on their individual compute resource needs.  From an HPC perspective, you can think of compute environments like a virtual cluster of compute nodes.  Compute environments can be based on either On-demand or Spot EC2 instances, where the latter enables significant savings.

You can create several compute environments to suit your needs.  Below we'll create the following:

* An "optimal" compute environment using on-demand instances
* An "optimal" compute environment using spot instances


#### Create an "optimal" on-demand compute environment

1. Go to the AWS Batch Console
2. Click on "Compute environments"
3. Click on "Create environment"
4. Select "Managed" as the "Compute environment type"
5. For "Compute environment name" type: "ondemand"
6. In the "Service role" drop down, select the `AWSBatchServiceRole` you created previously
7. In the "Instance role" drop down, select the `ecsInstanceRole` you created previously
8. For "Provisioning model" select "On-Demand"
9. "Allowed instance types" will be already populated with "optimal" - which is a mixture of M4, C4, and R4 instances.
10. In the "Launch template" drop down, select the `genomics-workflow-template` you created previously
11. Set Minimum and Desired vCPUs to 0.
 
!!! info
    Minimum vCPUs is the lowest number of active vCPUs (i.e. instances) your compute environment will keep running and available for placing jobs when there are no jobs queued.  Setting this to 0 means that AWS Batch will terminate all instances when all queued jobs are complete.

    Desired vCPUs is the number of active vCPUs (i.e. instances) that are currently needed in the compute environment to process queued
    jobs.  Setting this to 0 implies that there are currently no queued jobs.  AWS Batch will adjust this number based on the number of jobs queued and their resource requirements.

    Maximum vCPUs is the highest number of active vCPUs (i.e. instances) your compute environment will launch.  This places a limit on
    the number of jobs the compute environment can process in parallel.

For networking, the options are populated with your account's default VPC, public subnets, and security group.  This should be sufficient for the purposes of this workshop.  In a production setting, it is recommended to use a separate VPC, private subnets therein, and associated security groups.

Optional: (Recommended) Add EC tags.  These will help identify which EC2 instances were launched by AWS Batch.  At minimum:  

* Key: "Name"
* Value: "batch-ondemand-worker"
  
Click on "Create"

#### Create an "optimal" spot compute environment

1. Go to the AWS Batch Console
2. Click on "Compute environments"
3. Click on "Create environment"
4. Select "Managed" as the "Compute environment type"
5. For "Compute environment name" type: "spot"
6. In the "Service role" drop down, select the `AWSBatchServiceRole` you created previously
7. In the "Instance role" drop down, select the `ecsInstanceRole` you created previously
8. For "Provisioning model" select "Spot"
9. In the "Spot fleet role" drop down, select the `AWSSpotFleetTaggingRole` you created previously
10. "Allowed instance types" will be already populated with "optimal" - which is a mixture of M4, C4, and R4 instances.
11. In the "Launch template" drop down, select the `genomics-workflow-template` you created previously
12. Set Minimum and Desired vCPUs to 0.

For networking, the options are populated with your account's default VPC, public subnets, and security group.  This should be sufficient for the purposes of this workshop.  In a production setting, it is recommended to use a separate VPC, private subnets therein, and associated security groups.

Optional: (Recommended) Add EC tags.  These will help identify which EC2 instances were launched by AWS Batch.  At minimum:  

* Key: "Name"
* Value: "batch-spot-worker"
  
Click on "Create"

### Batch Job Queues

AWS Batch job queues, are where you submit and monitor the status of jobs.
Job queues can be associated with one or more compute environments in a preferred order.
Multiple job queues can be associated with the same compute environment.  Thus to handle scheduling, job queues also have a priority weight as well.

Below we'll create two job queues:

 * A "Default" job queue
 * A "High Priority" job queue

Both job queues will use both compute environments you created previously.

#### Create a "default" job queue

This queue is intended for jobs that do not require urgent completion, and can handle potential interruption.
Thus queue will schedule jobs to:

1. The "spot" compute environment
2. The "ondemand" compute environment

in that order.

Because it primarily leverages Spot instances, it will also be the most cost effective job queue.

* Go to the AWS Batch Console
* Click on "Job queues"
* Click on "Create queue"
* For "Queue name" use "default"
* Set "Priority" to 1
* Under "Connected compute environments for this queue", using the drop down menu:

    1. Select the "spot" compute environment you created previously, then
    2. Select the "ondemand" compute environment you created previously

* Click on "Create Job Queue"

#### Create a "high-priority" job queue

This queue is intended for jobs that are urgent and can handle potential interruption.
Thus queue will schedule jobs to:

1. The "ondemand" compute environment
2. The "spot" compute environment

in that order.

* Go to the AWS Batch Console
* Click on "Job queues"
* Click on "Create queue"
* For "Queue name" use "highpriority"
* Set "Priority" to 100 (higher values mean higher priority)
* Under "Connected compute environments for this queue", using the drop down menu:

    1. Select the "ondemand" compute environment you created previously, then
    2. Select the "spot" compute environment you created previously

* Click on "Create Job Queue"

## Module 2 - Running Nextflow

There are a couple key ways to run Nextflow:

*  locally for both the master process and jobs
*  locally for the master process with AWS Batch for jobs
*  (Advanced) Containerized with "Batch-Squared" Infrastructure - An AWS Batch job for the master process that creates additional AWS Batch jobs

### Local master and jobs

You can run Nextflow workflows entire on a single compute instance.  This can either be your local laptop, or a remote server like an EC2 instance.  in this workshop, your AWS Cloud9 Environment can simulate this scenario.

In a bash terminal, type the following:

```bash
nextflow run hello
```

This will run Nextflow's built-in "hello world" workflow.

### Local master and AWS Batch jobs

Genomics and life sciences workflows typically use a variety of tools that each have distinct computing resource requirements, such as high CPU or RAM utilization, or GPU acceleration.
Sometimes these requirements are beyond what a laptop or a single EC2 instance can provide.  Plus, provisioning a single large instance so that a couple of steps in a workflow can run would be a waste of computing resources.

A more cost effective method is to provision compute resources dynamically, as they are needed for each step of the workflow.
This is what AWS Batch is good at doing.

To configure your local Nextflow installation to use AWS Batch for workflow steps (aka jobs, or processes) you'll need to know the following:

* The "default" AWS Batch Job Queue workflows will be submitted to
* The S3 path that will be used as your nextflow working directory

These parameters need to go into a Nextflow config file.
To create this file, open a bash terminal and run the following:

```bash
cd ~/environment
mkdir -p work

cd ~/environment/work
python ~/environment/nextflow-workshop/create-config.py > nextflow.config
```  

This will create a file called `~/environment/work/nextflow.config` with contents like the following:

```groovy
workDir = "s3://genomics-workflows-cfa71800-c83f-11e9-8cd7-0ae846f1e916/_nextflow/runs"
process.executor = "awsbatch"
process.queue = "arn:aws:batch:us-west-2:402873085799:job-queue/default-45e553b0-c840-11e9-bb02-02c3ece5f9fa"
aws.batch.cliPath = "/home/ec2-user/miniconda/bin/aws"
```

Now when your run the `nextflow` "hello world" example from within this `work` folder you should see:

```bash
cd ~/environment/work
nextflow run hello
```

If everything is configured correctly, you should see the following output:

```
N E X T F L O W  ~  version 19.07.0
Launching `nextflow-io/hello` [angry_heisenberg] - revision: a9012339ce [master]
WARN: The use of `echo` method is deprecated
executor >  awsbatch (4)
[54/5481e0] process > sayHello (4) [100%] 4 of 4 ✔
Hello world!

Ciao world!

Bonjour world!

Hola world!

Completed at: 12-Sep-2019 00:46:56
Duration    : 3m 1s
CPU hours   : (a few seconds)
Succeeded   : 4
```

This will be similar to the output in the previous section with the only difference being:

```
executor >  awsbatch (4)
```

which indicates that workflow processes were run remotely as AWS Batch Jobs and not on the local instance.

### Batch-Squared

Since the master `nextflow` process needs to be connected to jobs to monitor progress, using a local laptop, or a dedicated EC2 instance for the master `nextflow` process is not ideal for long running workflows.
