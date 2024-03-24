#!/bin/bash -ex

# Update YUM packages
yum update -y

# Install Docker
yum install -y docker
systemctl enable --now docker

# Install Docker Compose
DOCKER_COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | jq -r .name)
curl -L "https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# Install Git, AWS CLI v2, and jq
yum install -y git jq
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install

# Clone the desired repository
git clone https://github.com/drmayu7/argilla.git
cd argilla

# Assume the EC2 instance role has the necessary permissions for Secrets Manager

# Retrieve and set secrets
SECRET_NAME="doccano-admin-login"
REGION="ap-southeast-1"
SECRET=$(aws secretsmanager get-secret-value --secret-id $SECRET_NAME --region $REGION | jq -r .SecretString)

export ARGILLA_SECRET=$(echo $SECRET | jq -r .argilla_secret)
export DEFAULT_PASSWORD=$(echo $SECRET | jq -r .password)
export DEFAULT_APIKEY=$(echo $SECRET | jq -r .api_key)

# Setup for EFS Mount
# Install necessary utilities for Amazon EFS
yum install -y amazon-efs-utils nfs-utils

file_system_id_1="fs-05938a0db346416a4"
efs_mount_point_1="/mnt/efs/fs1"
mkdir -p "${efs_mount_point_1}"

# Create directories for Argilla and Elasticsearch on EFS with appropriate permissions
mkdir -p /mnt/efs/fs1/argilla/data
mkdir -p /mnt/efs/fs1/argilla/config
mkdir -p /mnt/efs/fs1/elasticsearch

# Setting permissions with go+rw
chmod -R go+rw /mnt/efs/fs1/argilla
chmod -R go+rw /mnt/efs/fs1/elasticsearch

# Setup /etc/fstab for auto-mount
if [ -f "/sbin/mount.efs" ]; then
  echo "${file_system_id_1}:/ ${efs_mount_point_1} efs tls,_netdev" >> /etc/fstab
else
  echo "${file_system_id_1}.efs.${REGION}.amazonaws.com:/ ${efs_mount_point_1} nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0" >> /etc/fstab
fi

# Mount now
mount -a

# Start the Docker containers
docker-compose -f docker/docker-compose.yaml up -d
