# OpenStack Learning Notes

Practical notes and command examples collected while studying OpenStack and preparing for the Certified OpenStack Administrator (COA) exam.

> [!IMPORTANT]
> Replace example usernames, passwords, project IDs, IP addresses, and resource names before running these commands. Do not commit real credentials to GitHub.

## Contents

- [Learning resources](#learning-resources)
- [OpenStack overview](#openstack-overview)
- [Environment setup and authentication](#environment-setup-and-authentication)
- [Multipass and DevStack](#multipass-and-devstack)
- [Keystone: identity](#keystone-identity)
- [Glance: images](#glance-images)
- [Nova: compute](#nova-compute)
- [Cinder: block storage](#cinder-block-storage)
- [Neutron: networking](#neutron-networking)
- [Swift: object storage](#swift-object-storage)
- [Heat: orchestration](#heat-orchestration)
- [Image conversion](#image-conversion)
- [Troubleshooting](#troubleshooting)
- [OpenStack experience summary](#openstack-experience-summary)

## Learning resources

- [OpenStackClient command list](https://docs.openstack.org/python-openstackclient/2026.1/cli/command-list.html)
- [OpenStackClient documentation](https://docs.openstack.org/python-openstackclient/latest/cli/command-list.html)
- [COA study repository](https://github.com/AJNOURI/COA)
- [Pike COA lab](https://github.com/kris-at-occ/pike-coa-lab)
- [Packt practice material](https://subscription.packtpub.com/book/cloud-and-networking/9781787288416/12/ch12lvl1sec70/before-you-begin)
- [Springer OpenStack book](https://link.springer.com/book/10.1007/978-1-4842-8804-7)
- [OpenStack installation guide](https://docs.openstack.org/install-guide/)
- [Ubuntu installation guide (Newton)](https://docs.openstack.org/newton/install-guide-ubuntu/)
- [Basic Heat template](https://docs.openstack.org/heat/queens/template_guide/hello_world.html#a-most-basic-template)

## OpenStack overview

### Core services

| Service | Purpose |
| --- | --- |
| Keystone | Identity, authentication, authorization, and the service catalogue |
| Nova | Compute-instance lifecycle management |
| Neutron | Virtual networking |
| Cinder | Persistent block storage |
| Swift | Object storage |
| Glance | Image management |
| Heat | Infrastructure orchestration |
| Horizon | Web-based dashboard |

### Common compute inputs

- **Image:** operating-system template used to build an instance.
- **Flavor:** virtual CPU, RAM, and disk allocation.
- **Network interface:** connects an instance to a network.
- **Key pair:** supports SSH public-key authentication.
- **Security group:** controls permitted ingress and egress traffic.

### Cloud-init

`cloud-init` performs the initial configuration of a cloud VM during its first boot. It can:

- Create users.
- Install SSH public keys.
- Set the hostname.
- Configure networking.
- Install packages.
- Write configuration files.
- Run scripts and commands.
- Resize the root filesystem.
- Configure passwords and security settings.

### Initial discovery commands

```bash
source openrc admin admin

openstack token issue
openstack service list
openstack endpoint list
openstack network list
openstack image list
openstack flavor list
openstack compute service list
```

## Environment setup and authentication

### Source an OpenRC file

An OpenRC file exports the authentication URL and the user, project, and domain context.

```bash
source user_x-openrc
source openrc admin admin
source /opt/stack/devstack/openrc developer1 developer-project
```

### Authenticate using command-line options

```bash
openstack \
  --os-username demo \
  --os-password 'USER_PASSWORD' \
  --os-project-name PROJECT_NAME \
  --os-user-domain-name Default \
  --os-project-domain-name Default \
  --os-auth-url http://10.0.0.11:5000/v3 \
  --os-identity-api-version 3 \
  token issue
```

### Use `clouds.yaml`

The default per-user configuration path is:

```text
~/.config/openstack/clouds.yaml
```

Select a named cloud configuration with `--os-cloud`:

```bash
openstack --os-cloud admin service list --long
```

Example OpenRC variables:

```bash
export OS_REGION_NAME=RegionOne
export OS_PROJECT_DOMAIN_ID=default
export OS_AUTH_URL=http://192.168.252.2/identity
export OS_USER_DOMAIN_ID=default
export OS_USERNAME=admin
export OS_AUTH_TYPE=password
export OS_PROJECT_NAME=admin
export OS_PASSWORD='REPLACE_ME'
```

## Multipass and DevStack

### Manage the VM

```bash
# Inspect Multipass and its instances
multipass version
multipass list
multipass info openstack
multipass info openstack --snapshots

# Start, stop, or restart the VM
multipass start openstack
multipass stop openstack
multipass restart openstack

# Connect to the VM
multipass shell openstack
```

### Create and restore snapshots

```bash
# Create snapshots
multipass snapshot openstack --name clean-install
multipass snapshot openstack --name devstack-clean

# Restore a snapshot
multipass restore openstack.clean-install
multipass restore openstack.devstack-clean
```

> Snapshot syntax can vary by Multipass version. Run `multipass help restore` if the dotted instance/snapshot form is not accepted.

### Enter the DevStack environment

After connecting to the VM:

```bash
sudo su - stack
cd ~/devstack
source openrc admin admin
openstack token issue
```

## Keystone: identity

### Resource hierarchy

```text
Domain
├── Projects
│   ├── Compute resources
│   ├── Networks
│   └── Storage resources
├── Users
└── Groups

Roles grant permissions to users or groups within a project or domain.
```

A domain may contain multiple projects. A typical setup flow is:

1. Create a project.
2. Create a user.
3. Create or select a role.
4. Assign the role to the user for the project.

### Projects, users, roles, and groups

```bash
# Create a project
openstack project create \
  --description "Testing project creation" \
  --enable \
  sales-crm

# Create a user in the project
openstack user create \
  --project sales-crm \
  --description "Amy sales CRM user" \
  --enable \
  --password 'REPLACE_ME' \
  --email 'amy@coa.lab' \
  amy

# List users
openstack user list

# Assign the dev role to Amy in sales-crm
openstack role add --user amy --project sales-crm dev

# Add a user to a group
openstack group add user developers john
```

### Domains

```bash
# Create a domain
openstack domain create \
  --description "German subsidiary" \
  german-sub

# Create a project in the domain
openstack project create \
  --description "German R&D project" \
  --domain german-sub \
  german-rnd

# Create a user in the domain and project
openstack user create \
  --project german-rnd \
  --domain german-sub \
  --enable \
  --password 'REPLACE_ME' \
  --email 'toby@coa.lab' \
  --description "Toby — German R&D" \
  toby
```

### Services and endpoints

After registering a service, define its endpoints.

```bash
# Register a telemetry service
openstack service create \
  --name ceilometer \
  --description "Telemetry" \
  metering

# Create its public endpoint
openstack endpoint create \
  --region RegionOne \
  ceilometer \
  public \
  http://controller:8777
```

## Glance: images

Glance stores image metadata and image data. Metadata includes the name, ID, disk format, container format, size, visibility, status, properties, tags, and checksums. Image data can reside on a filesystem, Swift, Ceph RBD, or another configured backend.

When Nova boots an instance, the compute service obtains the selected image from Glance. Compute nodes can cache images locally to reduce repeated transfers.

```bash
# Create a public QCOW2 image
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --file /path/to/image.qcow2 \
  IMAGE_NAME

# Add or update an image property
openstack image set \
  --property description="Proposed description" \
  IMAGE_NAME
```

## Nova: compute

### Instance lifecycle operations

- Create and delete.
- Start, stop, pause, suspend, and resume.
- Resize.
- Rescue and unrescue.
- Rebuild.
- Migrate.
- Snapshot.

### Create supporting resources

```bash
# Create a key pair and protect its private key
openstack keypair create amy-key > ~/.ssh/amy-key.pem
chmod 600 ~/.ssh/amy-key.pem

# Create a flavor
openstack flavor create \
  --id 20 \
  --vcpus 1 \
  --ram 600 \
  --disk 1 \
  --public \
  --property os_shutdown_timeout=45 \
  FLAVOR_NAME

# Create a security group
openstack security group create SECURITY_GROUP_NAME

# Allow ICMP ingress
openstack security group rule create \
  --ingress \
  --protocol icmp \
  SECURITY_GROUP_NAME

# Allow SSH ingress
openstack security group rule create \
  --ingress \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0 \
  SECURITY_GROUP_NAME
```

### Create and manage an instance

```bash
# Create an instance
openstack server create \
  --image IMAGE_NAME \
  --flavor FLAVOR_NAME \
  --key-name KEY_NAME \
  --security-group SECURITY_GROUP_NAME \
  --network NETWORK_NAME \
  SERVER_NAME

# Create a snapshot of an existing instance
openstack server image create \
  --name SNAPSHOT_NAME \
  SERVER_NAME

# List and stop instances
openstack server list
openstack server stop SERVER_NAME
```

### Connect through a network namespace

In environments where the VM is not directly reachable, identify the relevant namespace and run network commands inside it:

```bash
openstack server list
openstack network list
sudo ip netns list

# Test connectivity
sudo ip netns exec NAMESPACE_ID ping -c 3 SERVER_IP

# Connect with SSH
sudo ip netns exec NAMESPACE_ID \
  ssh -i ~/.ssh/amy-key.pem USERNAME@SERVER_IP
```

### Compute maintenance

```bash
# List compute services
openstack compute service list

# Show host statistics
openstack host show HOST_NAME

# Enable or disable a compute service
openstack compute service set --enable HOST_NAME BINARY_NAME
openstack compute service set \
  --disable \
  --disable-reason "Maintenance" \
  HOST_NAME BINARY_NAME

# View usage and instances
openstack usage list
openstack server list --all-projects

# Legacy Nova client: retrieve instance diagnostics
nova diagnostics INSTANCE_NAME
```

Disabling a compute service prevents the scheduler from placing new instances on that host. It does not stop existing VMs.

## Cinder: block storage

Cinder provides persistent block devices that can be attached to compute instances.

```bash
# Create an empty 1 GiB volume
openstack volume create \
  --size 1 \
  --description "Test volume" \
  VOLUME_NAME

# List volumes
openstack volume list

# Create a volume from an image
openstack volume create \
  --size 1 \
  --image IMAGE_NAME \
  --description "Volume created from image" \
  VOLUME_NAME

# Attach or detach a volume
openstack server add volume SERVER_NAME VOLUME_NAME
openstack server remove volume SERVER_NAME VOLUME_NAME

# Create a new volume from a snapshot
openstack volume create \
  --snapshot SNAPSHOT_NAME \
  --size 1 \
  RESTORED_VOLUME_NAME
```

To boot a server from a volume:

```bash
openstack server create \
  --volume VOLUME_NAME \
  --flavor m1.tiny \
  --security-group default \
  --network NETWORK_NAME \
  SERVER_NAME
```

After attaching a new volume, identify, format, and mount the device inside the guest OS. Do not format a device that already contains data.

## Neutron: networking

### Network types

- **Project network:** isolates project traffic and connects virtual instances.
- **Provider network:** maps a virtual network to the physical data-centre network and can provide external connectivity.
- **North–south routing:** traffic between project and external/provider networks.
- **East–west routing:** traffic between internal networks or workloads.

A typical setup sequence is:

1. Create a network.
2. Create a subnet.
3. Create a router.
4. set the router's external gateway.
5. Connect the router to the subnet.
6. Configure security groups.
7. Allocate and associate a floating IP.

### Create a network, subnet, and router

```bash
openstack network create NETWORK_NAME

openstack subnet create \
  --network NETWORK_NAME \
  --subnet-range 192.168.30.0/24 \
  --dns-nameserver 8.8.8.8 \
  SUBNET_NAME

openstack router create ROUTER_NAME

# Connect the router to the provider network
openstack router set \
  --external-gateway provider \
  ROUTER_NAME

# Connect the router to the project subnet
openstack router add subnet ROUTER_NAME SUBNET_NAME
```

Legacy Neutron client equivalents:

```bash
neutron router-gateway-set ROUTER_NAME provider
neutron router-interface-add ROUTER_NAME SUBNET_NAME
```

### Security groups and floating IPs

```bash
openstack security group create crm-security-group

openstack security group rule create \
  --ingress \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0 \
  crm-security-group

openstack server add security group INSTANCE_NAME crm-security-group

# Allocate and associate a floating IP
openstack floating ip create EXTERNAL_NETWORK_NAME
openstack server add floating ip SERVER_NAME FLOATING_IP_ADDRESS
```

### Traffic flow

**Inbound:**

```text
Client → Provider network → Floating IP → Neutron router → DNAT
       → Project network → Security group → VM fixed IP
```

**Outbound:**

```text
VM → Project network → Neutron router → SNAT
   → Provider network → Client
```

## Swift: object storage

Swift uses containers to hold objects. A Swift container is broadly comparable to an S3 bucket.

| Swift | Similar S3 concept |
| --- | --- |
| Container | Bucket |
| Object | Object |
| Container ACL | Bucket/object access control |

### Container and object commands

```bash
# Create and inspect a container
openstack container create crm-container
openstack container show crm-container

# List, upload, and download objects
openstack object list crm-container
openstack object create crm-container file.txt
openstack object save crm-container file.txt

# Show account usage
openstack object store account show
```

### Container access controls

```bash
# Grant project-level read access
openstack container set \
  --property "X-Container-Read=sales-crm:*" \
  crm-container

# Allow anonymous downloads
openstack container set \
  --property "X-Container-Read=.r:*" \
  public-container

# Allow anonymous downloads and container listings
openstack container set \
  --property "X-Container-Read=.r:*,.rlistings" \
  public-container
```

- `.r:*` allows unauthenticated users to read/download objects.
- `.rlistings` allows unauthenticated users to list the objects in the container.

### Object expiration

```bash
# Delete the object after 3,600 seconds
openstack object set \
  --property "X-Delete-After=3600" \
  my-container my-file.txt

# Delete the object at a Unix timestamp
openstack object set \
  --property "X-Delete-At=1788192000" \
  my-container my-file.txt
```

### Public-access test with `curl`

```bash
# Check the service and endpoint
openstack service list
openstack endpoint list --service swift

# Create the container
openstack container create public-container

# Set values for the current environment
export SWIFT_URL="http://192.168.252.2:8080/v1/AUTH_<PROJECT_ID>"
export TOKEN="$(openstack token issue -f value -c id)"

# Make the container publicly readable and listable
curl -i -X POST \
  -H "X-Auth-Token: ${TOKEN}" \
  -H "X-Container-Read: .r:*,.rlistings" \
  "${SWIFT_URL}/public-container"

# Create and upload a test object
echo "Hello from OpenStack Swift" > test.txt
openstack object create public-container test.txt
openstack object list public-container

# Test anonymous access
curl -i "${SWIFT_URL}/public-container/test.txt"
```

Upload an object that expires after one hour:

```bash
curl -X PUT \
  -H "X-Auth-Token: ${TOKEN}" \
  -H "X-Delete-After: 3600" \
  --data-binary @file.txt \
  "${SWIFT_URL}/public-container/file.txt"
```

### Storage policies

Swift storage policies are configured by administrators in `/etc/swift/swift.conf`. After a policy is configured and deployed, create a container using it:

```bash
openstack container create \
  --storage-policy POLICY_NAME \
  CONTAINER_NAME
```

## Heat: orchestration

- **Template:** YAML file describing a collection of resources.
- **Stack:** deployed instance of a template.
- **Parameter:** input variable supplied to a template.
- **Resource:** infrastructure item declared in the template.
- **Output:** information returned by the deployed stack.

The `OS::` prefix in a Heat template identifies an OpenStack resource type.

### Manage a stack

```bash
# Create a stack
openstack stack create \
  --timeout 60 \
  --enable-rollback \
  --parameter flavor=m1.tiny \
  --parameter image=system-3.5 \
  --parameter key_name=KEY_NAME \
  --parameter net_name=NETWORK_NAME \
  --parameter vol_size=1 \
  --template FILE_NAME.yaml \
  STACK_NAME

# Inspect a stack and its resources
openstack stack show STACK_NAME
openstack stack resource list STACK_NAME
openstack stack output list STACK_NAME

# Update an existing stack
openstack stack update \
  --existing \
  --parameter net_name=NEW_NETWORK_NAME \
  STACK_NAME
```

## Image conversion

Convert a RAW image to QCOW2:

```bash
qemu-img convert -f raw -O qcow2 input.raw output.qcow2
```

| Argument | Meaning |
| --- | --- |
| `-f raw` | Input image format is RAW |
| `-O qcow2` | Output image format is QCOW2 |
| `input.raw` | Source image |
| `output.qcow2` | Converted image |

## Troubleshooting

### OpenStack services

```bash
# Identity catalogue and endpoints
openstack service list --long
openstack endpoint list

# Compute services
openstack compute service list

# Network agents
openstack network agent list
openstack network agent show AGENT_UUID

# Block-storage services
openstack volume service list

# System services
service --status-all
```

Older deployments or client packages may expose equivalent legacy commands:

```bash
neutron agent-list
cinder service-list
```

### Database backup

```bash
mysqldump --opt --all-databases > openstack-backup.sql
```

## OpenStack experience summary

While working with Lloyds Bank, I gained experience in a hybrid cloud environment spanning private and public cloud platforms. I helped deploy and manage resources in AWS, Azure, and Google Cloud using Terraform, Helm, and Ansible. In the private-cloud environment, I used the OpenStack Horizon dashboard and OpenStack services to manage identity, compute, networking, storage, images, and infrastructure orchestration.

---

These notes are intended as a study aid. Verify syntax against the documentation for the OpenStack release and client version installed in your environment.