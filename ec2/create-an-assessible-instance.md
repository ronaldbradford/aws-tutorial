# Create an Internet Accessible EC2 Instance

In this example we will create an EC2 Instance accessible from your current Internet IPV4 address that can be used to create and manage RDS resources. This tutorial requires the creation of the <a href="create-rds-security-group.md">RDS Security Group</a> that enables access from your local IP.

## Required configurable parameters

    # Input Parameters
    KEYPAIR="rds"
    INSTANCE_TYPE="t2.micro"
    SG_NAME="ec2-rds-sg"    # See create-rds-security-group.md

## Setup

    export AWS_PROFILE=administrator
    aws sts get-caller-identity
    REGION=$(aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]')

    LATEST_AMI=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --region ${REGION} --query 'Parameters[0].[Value]' --output text)
    SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=${SG_NAME} --query '*[].GroupId' --output text)

    KEYPAIR_PRIVATE_KEY="${HOME}/.ssh/id_aws_${KEYPAIR}"
    if ! aws ec2 describe-key-pairs --key-name ${KEYPAIR} 2>/dev/null; then
      aws ec2 create-key-pair --key-name ${KEYPAIR} | tee ${HOME}/${KEYPAIR}.json
      mkdir -p ${HOME}/.ssh && chmod 700 ${HOME}/.ssh
      rm -f ${KEYPAIR_PRIVATE_KEY}
      jq -r .KeyMaterial ~/${KEYPAIR}.json > ${KEYPAIR_PRIVATE_KEY}
      chmod 400 ${KEYPAIR_PRIVATE_KEY}
      ssh-keygen -f ${KEYPAIR_PRIVATE_KEY} -l
    fi
    ls -l ${HOME}/.ssh/id_aws_${KEYPAIR}*

    # This assumes a single VPC setup with single subnets per AZ
    VPC_ID=$(aws ec2 describe-vpcs --query '*[0].VpcId' --output text)
    SUBNET_ID=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=${VPC_ID} --query '*[0].SubnetId' --output text)
    echo "REGION=${REGION},VPC_ID=${VPC_ID},SUBNET_ID=${SUBNET_ID},SG_ID=${SG_ID},SG_NAME=${SG_NAME},KEYPAIR=${KEYPAIR},LATEST_AMI=${LATEST_AMI}"

    OUTPUT=$(aws ec2 run-instances --image-id ${LATEST_AMI} --instance-type ${INSTANCE_TYPE} --key-name ${KEYPAIR} --security-group-ids ${SG_ID} --subnet-id ${SUBNET_ID})
    jq . <<< ${OUTPUT}
    # Get Instance Id
    INSTANCE_ID=$(jq -r '.Instances[].InstanceId' <<< ${OUTPUT})

    # Wait for the resources to be ready (this is a synchronous call)
    time aws ec2 wait instance-running --instance-ids ${INSTANCE_ID}

    # Test access to new instance
    IP=$(aws ec2 describe-instances --instance-id ${INSTANCE_ID} --query '*[].Instances[0].NetworkInterfaces[].Association.PublicIp' --output text)
    nc -vz ${IP} 22
    ssh -i ${KEYPAIR_PRIVATE_KEY} -o "StrictHostKeyChecking=no" ec2-user@${IP} uptime

## Automate Local Access to ec2 instance

Using your local ssh configuration you can create a shortcut host with all the requirements defined within the config file. This is one example approach.

    SSH_CONFIG=${HOME}/.ssh/config
    touch ${SSH_CONFIG}
    AWS_SSH_DIR=${HOME}/.ssh/aws
    mkdir -p ${AWS_SSH_DIR}
    [[ $(grep -c "Include aws" ${SSH_CONFIG}) -eq 0 ]] && echo "Include aws/*" >> ${SSH_CONFIG}

    echo "Host ec2
      Hostname ${IP}
      User ec2-user
      Port 22
      UserKnownHostsFile /dev/null
      StrictHostKeyChecking no
      PasswordAuthentication no
      IdentityFile ${KEYPAIR_PRIVATE_KEY}
      IdentitiesOnly yes
      LogLevel ERROR" > ${AWS_SSH_DIR}/ec2

    ssh ec2 uptime

## Additional Instance Setup

The following is additional setup performed on the EC2 instance that is used for later tutorials.

    sudo yum update -y
    sudo yum install -y jq nc pigz
    sudo amazon-linux-extras install -y docker
    sudo service docker start
    sudo usermod -a -G docker ec2-user
    newgrp docker
    id

    mkdir -p ${HOME}/.aws
    # NOTE: Add your specific credentials here
    echo "[rdsdemo] ..." > .aws/credentials
    export AWS_PROFILE=rdsdemo
    aws rds describe-db-clusters


# Teardown

It is always a good practice to keep a clean infrastructure especially for testing purposes.

    aws ec2 terminate-instances --instance-ids ${INSTANCE_ID}
    aws ec2 wait instance-terminated --instance-ids ${INSTANCE_ID}

    # Or interative waiting alternative to `ec2 wait`
    EXPECTED_STATUS="terminated"
    while : ; do
      STATUS=$(aws ec2 describe-instances --instance-ids ${INSTANCE_ID} --query '*[].Instances[].State.Name' --output text)
      [ "${STATUS}" = "${EXPECTED_STATUS}" ] && break
      echo "$(date) ${STATUS}"
      sleep 5
    done

    aws ec2 delete-key-pair --key-name ${KEYPAIR}
    rm -f ${KEYPAIR_PRIVATE_KEY} ~/${KEYPAIR}.json

# References

## awscli
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/create-key-pair.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/run-instances.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-instances.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/terminate-instances.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/delete-key-pair.html

## User Guide
- https://docs.aws.amazon.com/efs/latest/ug/gs-step-one-create-ec2-resources.html
- https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-instances.html

## IaAS
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance

## Errors
- https://the.error.expert/amazon-web-services/awscli/ec2/
