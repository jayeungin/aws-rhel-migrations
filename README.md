# AWS RHEL Migrations
Migrating from an On-Demand RHEL EC2 instance to a Marketplace RHEL EC2 instance involves several steps. You'll need to create a new EC2 instance using the RHEL AMI from the AWS Marketplace, and then migrate your data, applications, and configurations to this new instance. Here’s a high-level overview of the steps you need to take:

## Manual Steps ##
### 1. **Backup Your Existing On-Demand EC2 Instance**
Before starting the migration, it's important to create a backup of your current EC2 instance to avoid data loss. You can back up the instance in the following ways:
- **Create an Amazon Machine Image (AMI):**  
   This will capture the entire state of your existing EC2 instance (including configurations, software, etc.) and allow you to launch a new instance with the same configuration later if needed.
   
   Steps:
   1. Go to the EC2 Dashboard.
   2. Select your running On-Demand instance.
   3. Click on **Actions > Image and templates > Create Image**.
   4. Give the image a name, and click **Create Image**.
- **Create EBS Snapshots (optional):**  
   If you're using EBS volumes for your data, create snapshots to back up these volumes.
- **Manual Backup (for critical data):**  
   For critical data, consider copying it to an S3 bucket or using an external backup tool to ensure it's secure.

### 2. **Launch a New EC2 Instance Using the RHEL Marketplace AMI**
Now, you need to launch a new EC2 instance using the RHEL AMI from the AWS Marketplace.
- **Go to the AWS Marketplace:**
   - Visit the [AWS Marketplace](https://aws.amazon.com/marketplace) and search for Red Hat Enterprise Linux (RHEL) AMIs.
- **Select the Appropriate RHEL AMI:**
   - Be sure to select the RHEL marketplace listing provided by [Red Hat](https://aws.amazon.com/marketplace/pp/prodview-aezl6to7jjkao?sr=0-1&ref_=beagle&applicationId=AWSMPContessa)
   - Choose the version of RHEL that you want to use for the new instance (e.g., RHEL 8.x or RHEL 9.x).
   - Make sure to select an AMI with the desired configuration (e.g., instance size, region).
- **Launch the Instance:**
   - Follow the usual process to launch an EC2 instance, including choosing the instance type, configuring security groups, selecting storage, and setting up key pairs.
   - Make sure that the instance is launched in the same VPC and subnet as your original instance, or that you have network connectivity if they are in different VPCs.

### 3. **Update DNS, Load Balancer, and Security Groups, etc.**
- If you're using DNS or a load balancer, update the DNS records or load balancer configuration to point to the new EC2 instance’s IP address.
- Ensure security groups, subnets, service accounts, secrets, etc. are configured similarly on the new instance to match the old instance, or modify as needed.

### 4. **Migrate Your Data, Applications, and Configurations**
Once your new RHEL EC2 instance is running, you need to transfer your data and application configurations from the old On-Demand instance to the new Marketplace instance. This can be done in several ways:
#### a) **Using Amazon Elastic Block Store (EBS)**
Snapshot, snapshot, snapshot before remounting the EBS. This should be the preferred method, but admins should still ensure the volumes attached without issues.
1. Mount the snapshot to the EBS volume from your old EC2 instance.
2. Mount the volume and any additional data over to the new instance as needed.
#### b) **Using Amazon S3**
1. Copy your data from the old EC2 instance to an S3 bucket:
   ```bash
   aws s3 cp /path/to/data s3://your-bucket-name/ --recursive
   ```
2. Download the data from S3 to your new EC2 instance:
   ```bash
   aws s3 cp s3://your-bucket-name/ /path/to/destination/ --recursive
   ```
#### c) **Using Rsync or SCP for File Transfer**
1. **Install `rsync` or use `scp`** on both the source and destination instances.
2. **Transfer data** from the old EC2 instance to the new one:
   ```bash
   rsync -avz -e ssh /path/to/data/ user@new-instance-ip:/path/to/destination/
   ```

### 5. **Test the New EC2 Instance**
Once the snapshot(s) are mounted and networking is set up, test the new EC2 instance to ensure it is functioning properly. Verify that all services are running, and that the applications are configured correctly.
- Check logs.
- Run any application-specific tests.
- Verify network access and firewall rules.

### 6. **Decommission the Old EC2 Instance**
Once you're confident that the new instance is running as expected, you can safely decommission the old On-Demand EC2 instance:
1. Terminate the old instance from the EC2 Console.
2. Optionally delete any old snapshots or AMIs if they are no longer needed.

### 7. **Cost Considerations**
- Keep in mind that moving to an EC2 Marketplace instance may change your billing model. The Marketplace instances may involve different pricing models for licensing, and you should be aware of those costs.
  
   **On-Demand vs. Marketplace:**  
   - **On-Demand** pricing: You pay hourly or per second for the compute usage, plus the cost of the RHEL subscription (which is often separate).
   - **Marketplace** pricing: Some marketplace AMIs bundle the cost of the RHEL license into the hourly or usage-based cost.

## Automated Summary##
This playbook duplicates an EC2 instance with EBS snapshot, remount, tag copy, and IP assignment.

### Pre-requisites
The playbook assumes that the Ansible user has the necessary AWS credentials and permissions to create EC2 instances, EBS snapshots, and assign IP addresses.

### Task Explanations
1. Gather info about a running EC2 instance using instance-id and store as variables: This task uses the ec2_instance_info module to gather information about the source EC2 instance, including its block device mappings, instance type, availability zone, security groups, and tags. The task stores this information as variables for use in subsequent tasks.
2. Snapshot the EBS volumes for the running EC2: This task uses the ec2_snapshot module to create EBS snapshots of the source EC2 instance's block devices. The task only runs when the device name matches the regular expression /dev/sd[b-z], which selects only the EBS volumes attached to the instance's root device.
3. Create a new EC2 instance using a different ami and matching instance info as the running EC2: This task uses the ec2_instance module to create a new EC2 instance using a different AMI and matching instance information as the source EC2 instance. The task assigns a new name, Environment, and a new tag to the new EC2 instance. The task also configures the new EC2 instance to use the same EBS volumes as the source EC2 instance.
4. Assign source networking to the new EC2 instance: This task uses the ec2_network_interface module to assign the source EC2 instance's private IP address and subnet to the new EC2 instance.
5. Turn off the original EC2: This task uses the ec2_instance module to stop the source EC2 instance.
6. Remount the EBS volumes to the new EC2: This task uses the ec2_vol module to remount the EBS volumes from the source EC2 instance to the new EC2 instance.
7. Turn on the new EC2: This task uses the ec2_instance module to start the new EC2 instance.
8. Output the original EC2 information and the new EC2 information: This task uses the debug module to output the original EC2 information and the new EC2 information to the Ansible learner's terminal. The debug module is useful for troubleshooting and understanding the playbook's behavior.

# Final Notes:
- If you're moving to a different version of RHEL in the marketplace, be sure to check compatibility with your applications and configurations.
- If you're using AWS services like Auto Scaling or Elastic Load Balancing, make sure the new instance is properly integrated into those services.

This approach will give you a smooth migration from an On-Demand RHEL EC2 instance to a Marketplace RHEL EC2 instance.