# Creating a private RHCOS AMI for your own use

If you wish to use a private AMI for RHCOS installs, you will need to create this AMI image. We will follow the steps below to create an AMI image.

## Prerequisites

You will need the following tools installed on your machine to follow these instructions:

    * awscli - see the following documentation for install process [Installing, updating, and uninstalling the AWS CLI version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
    * curl
    * gunzip

## Importing a vmdk for use as an AMI

To start, you will need to download the following base image: https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.6/latest/rhcos-4.6.8-x86_64-aws.x86_64.vmdk.gz and then un-compress it.

```
$ curl https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.6/latest/rhcos-4.6.8-x86_64-aws.x86_64.vmdk.gz -O
$ gunzip rhcos-4.6.8-x86_64-aws.x86_64.vmdk.gz
```

We now need to store this image in an S3 bucket to import in as an AMI.

```
$ aws s3 mb s3://rhcosscanimage
$ aws s3 cp rhcos-4.6.8-x86_64-aws.x86_64.vmdk s3://rhcosscanimage/rhcos-4.6.8-x86_64-aws.x86_64.vmdk
# list out the contents of the bucket to make sure the copy worked
$ aws s3 ls s3://rhcosscanimage
```

Now that the image is in a bucket, we need to define a container file to create the AMI. Create a file called "containers.json" with the following contents.

```
{
   "Description": "rhcos-4.6.8-x86_64-aws.x86_64",
   "Format": "vmdk",
   "UserBucket": {
      "S3Bucket": "rhcosscanimage",
      "S3Key": "rhcos-4.6.8-x86_64-aws.x86_64.vmdk"
   }
}
```

Using the aws command we will now import the image into ec2. (NOTE: if you are not using us-east-2 be sure to update the region name)

```
aws ec2 import-snapshot --region us-east-2 \
     --description "red hat core os image for scanning" \
     --disk-container file://containers.json
```

**NOTE:**  If you get an error about not having a vmimport policy you will need to create a vmimport policy. See the section in the Appendix below for instructions on how to do this.

The import process will take some time. Use the following command to watch for the import to complete.

```
watch -n 5 aws ec2 describe-import-snapshot-tasks --region us-east-2
```

Wait until you see that the snapshot import is "complete". Record the snapshot id (eg snap-0dd5fd5cca7adb6a0) this will be used in the next command.

Be sure to update the command below with the \<snapshotID\> from the watch command above.

```
aws ec2 register-image --region us-east-2 --architecture x86_64 --description "rhcos-4.6.8-x86_64-aws.x86_64" --ena-support --name "rhcos-4.6.8-x86_64-aws.x86_64" --virtualization-type hvm --root-device-name '/dev/xvda' --block-device-mappings 'DeviceName=/dev/xvda,Ebs={DeleteOnTermination=true,SnapshotId=<snapshot_ID>}' 
```

This command will output an AMI such as "ami-0c63084594da7adf4". Record this AMI id, we will use it in the next section.

You now have a private ami available.

# Running a security scan on a RHCOS ec2 instance

The Red Hat CoreOS (RHCOS) image is available as a Community AMI or can be accessed from a Private AMI created following the process earlier in this document. The following instructions will create a base RHCOS machine in AWS which can then be remotely accessed and run in a stand alone mode to give you an opportunity to run AV/vulnerability scans against the image before using the image in your clusters.

## Before we begin

RHCOS is an immutable OS. It does not make use of tools such as yum or dnf to install applications, but instead relies on the use of containers and container images to run software on the machine. Any scanning tools will need to be run from within a container in order to run your scans. This document gives you two examples of open source scanners that could be used. You will need to work with your security vendor, or security team to see how to run your specific scan tool on this running instance. 

The following instructions will create a single ec2 instance running in the us-east-2 region. It will create one user on the machine called "core" and the only remote access will be via the "core" user over ssh with an SSH key. There is no password set on the core user, you must specify a valid SSH key for remote access.

For additional information on Red Hat Core OS please see the following link [RED HAT ENTERPRISE LINUX COREOS (RHCOS)](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.6/html/architecture/architecture-rhcos)

## Creating an RHCOS EC2 Instance

We will start by creating a small ignition file which will allow us to SSH to the host. You will need to update the "sshAuthorizedKeys" field below with a valid ssh public key.

workerpayload.ign
```
{
	"ignition": {
		"version": "3.0.0"
	},
	"passwd": {
		"users": [
			{
				"name": "core",
				"sshAuthorizedKeys": [
					"ssh-rsa AAAAB3NzaC1...+Jr7xKAoaDbJxgbcj+IIsEQID5u0lX0bBHfhYSFDCGPB4Bynh2ExP1WBJqCumCV0bLEgbgyYQ== user@example.com"
				]
			}
		]
	}
}
```

Using the aws cli command we will now start up a instance. You will need a subnet and an AMI instance to start from.  Be sure to update the following fields before running the command:

* \<subnetID\>
* \<amiID\>
* \<machineName\>

```
$ aws ec2 run-instances --subnet-id <subnetID> \
  --instance-type m4.large --image-id <amiID> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=<machineName>}]' \
  --user-data file://workerpayload.ign
```

Wait for the new EC2 instance to start, and then ssh to the machine using the ssh key you supplied in the ignition file. Be sure to use "core" as the user name.

ssh core@\<ip address of new machine\>

At this point you are connected to the machine as the "core" user. This user has full "sudo" rights. Keep in mind that the RHCOS machine does not have tools like yum or dnf to install applications, you will need to run your scan tools from container images. Examples of running two scan tools are listed below.

## Running a scan tool on the running RHCOS machine

Because RHCOS is an immutable OS, any scan tools must be run from within a container. The below instructions show how to run two different vulnerability scanning tools against the running RHCOS image. If you wish to use a different tool, you will need to run that scan tool from within a container. Below are two example scan tools to run against your running RHCOS image. The first uses the [OpenSCAP](https://www.open-scap.org/tools/openscap-base/) scanning tool to check the AMI image against the Red Hat CVE database.  The second runs the open source AntiVirus tool [ClamAV](https://www.clamav.net/).

**These are given as example tools to run, if you want to use a different tool, you will need to come up with the list of steps to run your scan tool on the system**

### OpenSCAP:

The following commands will do the following:

1. start a container based on the Red Hat UBI7 image, with privileged OS access and mount the host's root file system to "/host" inside the container
2. add the epel repository so we can install openscap into the container
3. install the openscap application
4. download the latest Red Hat CVE database
5. scan the base system against the latest CVE database
   
```
ssh core@\<ip address of new machine\>
sudo podman run --mount type=bind,source=/,destination=/host --privileged -it registry.access.redhat.com/ubi7/ubi /bin/bash
curl -L \
http://copr.fedoraproject.org/coprs/openscapmaint/openscap-latest/repo/epel-7/openscapmaint-openscap-latest-epel-7.repo -o \
 /etc/yum.repos.d/openscapmaint-openscap-latest-epel-7.repo
yum install -y openscap openscap-utils scap-security-guide bzip2 --skip-broken
# pull down the latest definitions
curl https://www.redhat.com/security/data/metrics/com.redhat.rhsa-all.xccdf.xml -O
curl https://www.redhat.com/security/data/oval/com.redhat.rhsa-all.xml -O
oscap-chroot /host xccdf eval --results results.xml --report report.html com.redhat.rhsa-all.xccdf.xml
```

### ClamAV:

The following commands will do the following:

1. start a container based on the latest centos base image, with privileged OS access and mount the host's root file system to "/host" inside the container
2. add the epel repository so we can install clamav and the clamav update tool into the container
3. install the clamav and the clamav update tool applications
4. download the latest clamav vulnerability database
5. scan the base system against the latest clamav vulnerability database

```
ssh core@\<ip address of new machine\>
sudo podman run --mount type=bind,source=/,destination=/host --privileged -it quay.io/centos/centos:latest /bin/bash
yum -y install epel-release
yum -y install clamav clamav-update
freshclam
clamscan -r /host
```

# Appendix

## Creating a VMimport policy

Depending on your environment setup, you may need to create a VMImport policy. The following steps are based on the official [AWS documentation](https://docs.aws.amazon.com/vm-import/latest/userguide/vmie_prereqs.html#vmimport-role)

create a role file with with the following information:

trust-policy.json
```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
```

apply the role by running:
```
$ aws iam create-role --role-name vmimport --assume-role-policy-document file://trust-policy.json
```

We now need to attach that role to the bucket that you will be using to create the AMI image from. Start by creating a policy file to attach to the bucket.

role-policy.json
```
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect": "Allow",
         "Action": [
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket"
         ],
         "Resource": [
            "arn:aws:s3:::rhcosscanimage",
            "arn:aws:s3:::rhcosscanimage/*"
         ]
      },
      {
         "Effect": "Allow",
         "Action": [
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource": "*"
      }
   ]
}
```

Finally apply that role to the bucket you are using:

```
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://role-policy.json
```

You should now be able to re-run the command to import the snapshot.