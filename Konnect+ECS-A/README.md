# Kong Konnect Enterprise & Elastic Container Service Anywhere

Overcoming Challenges of Running Microservices in Amazon ECS-A with Kong

One of the most powerful capabilities provided by Kong Konnect Enterprise is the support for Hybrid deployments. In other words, it implements distributed API Gateway Clusters with multiple instances running on several environments at the same time.

Moreover, Konnect Enterprise provides a new topology option, named Hybrid Mode, with a total separation of the Control Plane (CP) and Data Plane (DP). That is, while Control Plane is responsible for administration tasks, the Data Plane is exclusively used by API Consumers.

Please, refer to the following link to read more about the Hybrid deployment: https://docs.konghq.com/enterprise/2.4.x/deployment/hybrid-mode/


Reference Architecture
Here's a Reference Architecture implemented in AWS:
The Control Plane runs as a Docker container on an EC2 instance. Notice the PostgreSQL RDS Database is located behind the CP.
The first Data Plane runs as another Docker container on a different EC2 instance.
The second Data Plane runs on an Elastic Kubernetes Service (EKS) Cluster.



Considering the capabilities provided by the Kubernetes platform, running Data Planes on this platform delivers a powerful environment. Here are some capabilities leveraged by the Data Plane on Kubernetes:
High Availability: One of the main Kubernetes' capabilities is "Self-Healing". If a "pod" crashes, Kubernetes takes care of it, reinitializing the "pod".
Scalability/Elasticity: HPA ("Horizontal Pod Autoscaler") is the capability to initialize and terminate "pod" replicas based on previously defined policies. The policies define "thresholds" to tell Kubernetes the conditions where it should initiate a brand new "pod" replica or terminate a running one.
Load Balancing: The Kubernetes Service notion defines an abstraction level on top of the "pod" replicas that might have been up or down (due HPA policies, for instance). Kubernetes keeps all the "pod" replicas hidden from the "callers" through Services.

Important remark #1: this tutorial is intended to be used for labs and PoC only. There are many aspects and processes, typically implemented in production sites, not described here. For example: Digital Certificate issuing, Cluster monitoring, etc.

Important remark #2: the deployment is based on Kong Enterprise 2.1. Please contact Kong to get a Kong Enterprise trial license to run this lab.
Kong for Kubernetes supports the following capabilities:
Scalability: Based on the Kong API gateway, it's responsible for managing the ingresses
Security: Leverages Kubernetes namespace-based RBAC model to ensure consistent access controls
Extensibility: An extensive plugin ecosystem offers a variety of options to protect your service mesh, such as OpenID Connect and mutual TLS authentication and authorization, rate-limiting, IP restrictions, and self-service credential registration through the Kong Enterprise Developer Portal.
Observability: It can be fully integrated with monitoring and tracing tools like Prometheus/Grafana, Jaeger, AWS Elasticsearch Service and AWS CloudWatch.




# Running microservices in Amazon EKS with AWS App Mesh and Kong
by Mikhail Shapirov | on 03 DEC 2020 | in Amazon Elastic Kubernetes Service, AWS App Mesh, Containers | Permalink |  Share
This post was created in collaboration with Claudio Acquaviva, Solution Engineer, Kong, and Morgan Davies, Kong Alliances.

A service mesh is transparent infrastructure layer that has become a common architectural pattern for intra-service communication. By combining Amazon EKS and AWS App Mesh, you form a powerful platform for your microservices, addressing technical requirements that occur in service-to-service communication, including load balancing, service discovery, observability, access control, tracing, health checks, and circuit breakers.

A modern enterprise solution requires clear management controls for the following categories:

API Management covering external traffic ingress to the API endpoints.
Service management capabilities focusing on operational controls and service health.
While service meshes primarily address the second category, the ingress is no less important and can benefit from a solution that supports cluster-wide policies such as throttling, application and user authentication, request logging and tracing, and data caching. In addition to these polices, the ingress is the layer that enables you to monetize your APIs by capturing usage, attaching billing systems, and generating alerts that go beyond operational concerns.

While it is possible to achieve this by stitching tools outside of the cluster perimeter, the Kong for Kubernetes Ingress Controller provides a solution that will protect your service mesh running side by side with your application services, leveraging Kubernetes capabilities like HPA, self-healing, RBAC, and cert-manager, among others.

This post will explore how to use Amazon EKS, AWS App Mesh, and Kong for Kubernetes to implement and protect a service mesh. The problem space that we will address is not just about managing your APIs and external traffic; we will also cover deeper integration scenarios. Not only will we handle the ingress in a Kubernetes-native way, we will also make it part of the service mesh itself, improving your observability, security, and traffic control.

Enter AWS App Mesh and Kong for Kubernetes Ingress Controller
AWS App Mesh is a fully managed service that customers can use to implement a service mesh. This service makes it easy to manage internal service-to-service communication across multiple types of compute infrastructure. Kong for Kubernetes is responsible for controlling the traffic going through the ingresses that expose the service mesh to external consumers by defining, applying, and enforcing policies to the ingresses.

Kong for Kubernetes supports the following capabilities:

Scalability: Based on the Kong API gateway, it’s responsible for managing the ingresses. It is common for applications to experience significant fluctuations in volume of traffic, affecting your ingress as well. Kong for Kubernetes is taking advantage of standard Kubernetes scalability controls like Horizontal Pod Autoscaler (HPA) and will scale seamlessly with the demand.
Security: Leverages Kubernetes namespace-based RBAC model to ensure consistent access controls. These controls are essential to segregate responsibilities between platform, API, and application teams, which handle their part in the software delivery and operations. For example, application teams restricted to their individual namespaces must still be able to define ingress objects, while access to the ingress controller and API management components can be restricted to the dedicated team(s).
Extensibility: An extensive plugin ecosystem offers a variety of options to protect your service mesh, such as OpenID Connect and mutual TLS authentication and authorization, rate-limiting, IP restrictions, and self-service credential registration through the Kong Enterprise Developer Portal.
Observability: It can be fully integrated with monitoring, tracing and logging tools like Prometheus, Jaeger, and AWS CloudWatch.
Here’s the Kong for Kubernetes architecture diagram:



Kong for Kubernetes Architecture
The following diagram describes the Kong for Kubernetes architecture:

Kong architecture diagram

The Kong for Kubernetes pod contains two containers:

The Kong Gateway container represents the data plane responsible for processing API traffic and enforcement of policies defined by the ready-to-use plugins available in Kong for Kubernetes.
The controller container represents the control plane that translates Kubernetes manifests and CRDs into Kong configuration, removing the need for separate administration of proxy and Kubernetes configuration.
In front of Kong for Kubernetes, there is a Classic Load Balancer (CLB) or Network Load Balancer (NLB) exposing the Kong Gateway to the external consumers. Furthermore, Kong for Kubernetes is protecting all services behind it, including ClusterIP services running inside the Kubernetes cluster or external services exposed in your cluster.

Prerequisites
Before starting this process, ensure the following prerequisites are ready:

An EKS 1.15 or higher cluster is already deployed. For this exercise, eksctl was used
Kubectl 1.15 or higher installed locally
Helm V3
Curl or any other HTTP client
Solution deployment
The deployment is an evolution of the DJ Service Mesh Application, adding the ingress controller layer on top of it. Kong for Kubernetes provides an extensive list of plugins to implement numerous policies, such as authentication, log processing, caching, and more.

To get started, let’s implement an API key-based security layer and rate-limiting policies to control the ingress consumption.

Step 1: Deploy your DJ service mesh application
Follow the following steps described in the EKS workshop to deploy the DJ service mesh application:







































Pre-requisites
The tutorial assumes you have already installed the following products:
AWS CLI
httpie and curl
jq

Control Plane ECS-A Cluster
Creating IAM Role
Create a file named ssm-trust-policy.json with the following content:

{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"Service": [
      "ssm.amazonaws.com"
    ]},
    "Action": "sts:AssumeRole"
  }
}


Create the IAM Role with:
$ aws iam create-role --role-name ECSA_role --assume-role-policy-document file://ssm-trust-policy.json
{
    "Role": {
        "Path": "/",
        "RoleName": "ECSA_role",
        "RoleId": "AROASGVFEB7FGCBNJCR7X",
        "Arn": "arn:aws:iam::151743893450:role/ECSA_role",
        "CreateDate": "2021-05-07T19:24:56+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": {
                "Effect": "Allow",
                "Principal": {
                    "Service": [
                        "ssm.amazonaws.com"
                    ]
                },
                "Action": "sts:AssumeRole"
            }
        }
    }
}

Attach the Role to both SSM and ECS:
aws iam attach-role-policy --role-name ECSA_role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam attach-role-policy --role-name ECSA_role --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

Check the Attached Roles:
$ aws iam list-attached-role-policies --role-name ECSA_role
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonSSMManagedInstanceCore",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
        },
        {
            "PolicyName": "AmazonEC2ContainerServiceforEC2Role",
            "PolicyArn": "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
        }
    ]
}


Creating ECS Anywhere Cluster for Konnect Control Plane
$ aws ecs create-cluster --cluster-name kong-control-plane-ecsa
{
    "cluster": {
        "clusterArn": "arn:aws:ecs:eu-central-1:151743893450:cluster/kong-control-plane-ecsa",
        "clusterName": "kong-control-plane-ecsa",
        "status": "ACTIVE",
        "registeredContainerInstancesCount": 0,
        "runningTasksCount": 0,
        "pendingTasksCount": 0,
        "activeServicesCount": 0,
        "statistics": [],
        "tags": [],
        "settings": [
            {
                "name": "containerInsights",
                "value": "disabled"
            }
        ],
        "capacityProviders": [],
        "defaultCapacityProviderStrategy": []
    }
}


$ aws ecs list-clusters
{
    "clusterArns": [
        "arn:aws:ecs:eu-central-1:151743893450:cluster/kong-control-plane-ecsa"
    ]
}




If you want to delete the cluster run:
aws ecs delete-cluster --cluster kong-control-plane-ecsa



Control Plane Installation - EC2/Docker
Go to EC2 dashboard and click on "Launch Instance". Select "Amazon Linux 2" using a "t2.large" Instance Type.

Choose the default VPC, a Public Subnet and enable "Auto-assign Public IP".

Change the Security Group to enable access from any address.

Open a terminal with command like this:
ssh -i "acqua-frankfurt.pem" ec2-user@ec2-52-59-250-107.eu-central-1.compute.amazonaws.com

Install utilities
sudo vi /etc/environment

add these lines
LANG=en_US.utf-8
LC_ALL=en_US.utf-8

In your $HOME directory change the .bashrc file adding the following line in the end:
sudo vi .bashrc
ulimit -n 4096


login again
sudo yum update -y
sudo amazon-linux-extras install epel -y
sudo yum install httpie -y
sudo yum install jq -y


Installing the ECS-A Agent

Creating Activation Key
The ECS-A Agent needs an Activation ID and Code to connect to the Control Plane:

# aws ssm create-activation --iam-role ECSA_role --registration-limit 50 | tee ssm-activation.json
{
    "ActivationId": "386aa0ed-6cf0-41da-951b-ebf2e774b8a0",
    "ActivationCode": "MiGiqFbs8YisEMTea0hT"
}




Installing ECS-A Agent
As expected, right after the Cluster creation, we don't have any Container instance defined:
# aws ecs list-container-instances --cluster kong-control-plane-ecsa
{
    "containerInstanceArns": []
}


Download the ECS-A Agent installation script
# sudo bash

# curl -o "ecs-anywhere-install.sh" "https://amazon-ecs-agent-packages-preview.s3.us-east-1.amazonaws.com/ecs-anywhere-install.sh" && sudo chmod +x ecs-anywhere-install.sh

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 12816  100 12816    0     0   9375      0  0:00:01  0:00:01 --:--:--  9375



Using the previously issued Activation ID and Code, install the Agent
# ./ecs-anywhere-install.sh \
     --cluster kong-control-plane-ecsa \
     --activation-id 386aa0ed-6cf0-41da-951b-ebf2e774b8a0 \
     --activation-code MiGiqFbs8YisEMTea0hT \
     --region eu-central-1
Running ECS install script on amzn 2
###

Loaded plugins: extras_suggestions, langpacks, priorities, update-motd

amzn2-core                                                                                                                                                                                                                                                               | 3.7 kB  00:00:00     
219 packages excluded due to repository priority protections
Package epel-release-7-11.noarch already installed and latest version
Nothing to do
Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
219 packages excluded due to repository priority protections
Package jq-1.5-1.amzn2.0.2.x86_64 already installed and latest version
Nothing to do

##########################
# Trying to install ssm agent ... 

Failed to get unit file state for snap.amazon-ssm-agent.amazon-ssm-agent.service: No such file or directory
ssm agent is installed, checking if the instance is registered.

##########################
# Trying to checking if instance already has an SSM managed instance ID. ... 


# ok
##########################

SSM Agent is installed. The instance needs to be registered.

##########################
# Trying to checking if instance already has an SSM managed instance ID. ... 


# ok
##########################

Error occurred fetching the seelog config file path:  open /etc/amazon/ssm/seelog.xml: no such file or directory
Initializing new seelog logger
New Seelog Logger Creation Complete
2021-05-13 15:51:03 WARN Could not read InstanceFingerprint file: InstanceFingerprint does not exist.
2021-05-13 15:51:03 INFO No initial fingerprint detected, generating fingerprint file...
2021-05-13 15:51:04 INFO Successfully registered the instance with AWS SSM using Managed instance-id: mi-04f2e9619b99dcc13
Instance is registered.

##########################
# Trying to install docker from docker repos ... 

Docker install repos not supported for this distro, trying distro install

##########################
# Trying to install docker from distribution repos ... 

Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
219 packages excluded due to repository priority protections
Resolving Dependencies
--> Running transaction check
---> Package docker.x86_64 0:20.10.4-1.amzn2 will be installed
--> Processing Dependency: runc >= 1.0.0 for package: docker-20.10.4-1.amzn2.x86_64
--> Processing Dependency: libcgroup >= 0.40.rc1-5.15 for package: docker-20.10.4-1.amzn2.x86_64
--> Processing Dependency: containerd >= 1.3.2 for package: docker-20.10.4-1.amzn2.x86_64
--> Processing Dependency: pigz for package: docker-20.10.4-1.amzn2.x86_64
--> Running transaction check
---> Package containerd.x86_64 0:1.4.4-1.amzn2 will be installed
---> Package libcgroup.x86_64 0:0.41-21.amzn2 will be installed
---> Package pigz.x86_64 0:2.3.4-1.amzn2.0.1 will be installed
---> Package runc.x86_64 0:1.0.0-0.1.20210225.git12644e6.amzn2 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================================================================================================================================================================================================================================
 Package                                                        Arch                                                       Version                                                                                  Repository                                                             Size
================================================================================================================================================================================================================================================================================================
Installing:
 docker                                                         x86_64                                                     20.10.4-1.amzn2                                                                          amzn2extra-docker                                                      32 M
Installing for dependencies:
 containerd                                                     x86_64                                                     1.4.4-1.amzn2                                                                            amzn2extra-docker                                                      24 M
 libcgroup                                                      x86_64                                                     0.41-21.amzn2                                                                            amzn2-core                                                             66 k
 pigz                                                           x86_64                                                     2.3.4-1.amzn2.0.1                                                                        amzn2-core                                                             81 k
 runc                                                           x86_64                                                     1.0.0-0.1.20210225.git12644e6.amzn2                                                      amzn2extra-docker                                                     3.2 M

Transaction Summary
================================================================================================================================================================================================================================================================================================
Install  1 Package (+4 Dependent packages)

Total download size: 59 M
Installed size: 243 M
Downloading packages:
(1/5): libcgroup-0.41-21.amzn2.x86_64.rpm                                                                                                                                                                                                                                |  66 kB  00:00:00     
(2/5): pigz-2.3.4-1.amzn2.0.1.x86_64.rpm                                                                                                                                                                                                                                 |  81 kB  00:00:00     
(3/5): containerd-1.4.4-1.amzn2.x86_64.rpm                                                                                                                                                                                                                               |  24 MB  00:00:00     
(4/5): docker-20.10.4-1.amzn2.x86_64.rpm                                                                                                                                                                                                                                 |  32 MB  00:00:00     
(5/5): runc-1.0.0-0.1.20210225.git12644e6.amzn2.x86_64.rpm                                                                                                                                                                                                               | 3.2 MB  00:00:00     
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                                                                                                            66 MB/s |  59 MB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : runc-1.0.0-0.1.20210225.git12644e6.amzn2.x86_64                                                                                                                                                                                                                              1/5 
  Installing : containerd-1.4.4-1.amzn2.x86_64                                                                                                                                                                                                                                              2/5 
  Installing : libcgroup-0.41-21.amzn2.x86_64                                                                                                                                                                                                                                               3/5 
  Installing : pigz-2.3.4-1.amzn2.0.1.x86_64                                                                                                                                                                                                                                                4/5 
  Installing : docker-20.10.4-1.amzn2.x86_64                                                                                                                                                                                                                                                5/5 
  Verifying  : containerd-1.4.4-1.amzn2.x86_64                                                                                                                                                                                                                                              1/5 
  Verifying  : docker-20.10.4-1.amzn2.x86_64                                                                                                                                                                                                                                                2/5 
  Verifying  : pigz-2.3.4-1.amzn2.0.1.x86_64                                                                                                                                                                                                                                                3/5 
  Verifying  : runc-1.0.0-0.1.20210225.git12644e6.amzn2.x86_64                                                                                                                                                                                                                              4/5 
  Verifying  : libcgroup-0.41-21.amzn2.x86_64                                                                                                                                                                                                                                               5/5 

Installed:
  docker.x86_64 0:20.10.4-1.amzn2                                                                                                                                                                                                                                                               

Dependency Installed:
  containerd.x86_64 0:1.4.4-1.amzn2                                   libcgroup.x86_64 0:0.41-21.amzn2                                   pigz.x86_64 0:2.3.4-1.amzn2.0.1                                   runc.x86_64 0:1.0.0-0.1.20210225.git12644e6.amzn2                                  

Complete!

# ok
##########################


##########################
# Trying to install ecs agent ... 

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 18.7M  100 18.7M    0     0  5175k      0  0:00:03  0:00:03 --:--:-- 5174k
/tmp/tmp.GYwGqMaQzN /home/ec2-user
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   100  100   100    0     0    247      0 --:--:-- --:--:-- --:--:--   246
amazon-ecs-init-latest.x86_64.rpm: OK
/home/ec2-user
Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
Existing lock /var/run/yum.pid: another copy is running as pid 4119.
Another app is currently holding the yum lock; waiting for it to exit...
  The other application is: yum
    Memory : 306 M RSS (523 MB VSZ)
    Started: Thu May 13 15:51:17 2021 - 00:04 ago
    State  : Running, pid: 4119
Another app is currently holding the yum lock; waiting for it to exit...
  The other application is: yum
    Memory : 306 M RSS (523 MB VSZ)
    Started: Thu May 13 15:51:17 2021 - 00:06 ago
    State  : Running, pid: 4119
Examining /tmp/tmp.GYwGqMaQzN/amazon-ecs-init-latest.x86_64.rpm: amazon-ecs-init-1.48.1-1.x86_64
Marking /tmp/tmp.GYwGqMaQzN/amazon-ecs-init-latest.x86_64.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package amazon-ecs-init.x86_64 0:1.48.1-1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================================================================================================================================================================================================================================
 Package                                                               Arch                                                         Version                                                          Repository                                                                            Size
================================================================================================================================================================================================================================================================================================
Installing:
 amazon-ecs-init                                                       x86_64                                                       1.48.1-1                                                         /amazon-ecs-init-latest.x86_64                                                        69 M

Transaction Summary
================================================================================================================================================================================================================================================================================================
Install  1 Package

Total size: 69 M
Installed size: 69 M
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : amazon-ecs-init-1.48.1-1.x86_64                                                                                                                                                                                                                                              1/1 
  Verifying  : amazon-ecs-init-1.48.1-1.x86_64                                                                                                                                                                                                                                              1/1 

Installed:
  amazon-ecs-init.x86_64 0:1.48.1-1                                                                                                                                                                                                                                                             

Complete!
Created symlink from /etc/systemd/system/multi-user.target.wants/ecs.service to /usr/lib/systemd/system/ecs.service.

# ok
##########################



Checking the Installation
# docker container ls -a
CONTAINER ID   IMAGE                            COMMAND    CREATED              STATUS                        PORTS     NAMES
9d3c441a4b29   amazon/amazon-ecs-agent:latest   "/agent"   About a minute ago   Up About a minute (healthy)             ecs-agent



On another local terminal run
# aws ssm describe-instance-information
{
    "InstanceInformationList": [
        {
            "IsLatestVersion": false, 
            "IamRole": "ECSA_role", 
            "ComputerName": "ip-172-31-14-43.eu-central-1.compute.internal", 
            "PingStatus": "Online", 
            "InstanceId": "mi-04f2e9619b99dcc13", 
            "IPAddress": "172.31.14.43", 
            "ResourceType": "ManagedInstance", 
            "ActivationId": "58dc667c-ead7-43e8-88f0-e6f4b3ea4fca", 
            "AgentVersion": "3.0.529.0", 
            "PlatformVersion": "2", 
            "RegistrationDate": 1620921064.127, 
            "PlatformName": "Amazon Linux", 
            "PlatformType": "Linux", 
            "LastPingDateTime": 1620921134.541
        }
    ]
}


# aws ecs list-container-instances --cluster kong-control-plane-ecsa
{
    "containerInstanceArns": [
        "arn:aws:ecs:eu-central-1:151743893450:container-instance/kong-control-plane-ecsa/8a6a7af42a244214a23f01722f5893c1"
    ]
}





Kong Konnect Enterprise Installation
https://aws.amazon.com/premiumsupport/knowledge-center/ecs-data-security-container-task/
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html
https://aws.amazon.com/blogs/security/how-to-use-aws-secrets-manager-securely-store-rotate-ssh-key-pairs/
https://aws.amazon.com/blogs/aws/aws-secrets-manager-store-distribute-and-rotate-credentials-securely/
https://docs.docker.com/cloud/ecs-integration/


Generating Private Key and Digital Certificate
mkdir /kong

openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
  -keyout ./cluster.key -out ./cluster.crt \
  -days 1095 -subj "/CN=kong_clustering"

chmod 644 cluster.*

cp cluster.* /kong



Create AWS Secret for License
aws secretsmanager create-secret --name kong_license_ecsa \
    --description "konglicense" \
    --secret-string file://license.json

$ aws secretsmanager get-secret-value --secret-id kong_license_ecsa --version-stage AWSCURRENT


Create AWS Secret for Digital Certificate and Key
This docs describes two options for storing the Certificate/Key pair: AWS Secrets Manager or Kong Konnect Control Plane. If you plan to use the AWS Secret Manager, you issue the pair like this:

Key
aws secretsmanager create-secret --name kong_key_ecsa \
    --description "kong_ecsa_key" \
    --secret-string file://cluster.key


Digital Certificate
aws secretsmanager create-secret --name kong_cert_ecsa \
    --description "kong_ecsa_certificate" \
    --secret-string file://cluster.crt

# aws secretsmanager list-secrets
{
    "SecretList": [
        {
            "Name": "kong_license_ecsa", 
            "LastChangedDate": 1621700190.969, 
            "SecretVersionsToStages": {
                "00a9cc04-1df2-43d1-8163-10dc7348ffb6": [
                    "AWSCURRENT"
                ]
            }, 
            "CreatedDate": 1621700157.109, 
            "LastAccessedDate": 1621641600.0, 
            "ARN": "arn:aws:secretsmanager:eu-central-1:151743893450:secret:kong_license_ecsa-3QmZjB", 
            "Description": "konglicense"
        }, 
        {
            "Name": "kong_key_ecsa", 
            "LastChangedDate": 1621769699.214, 
            "SecretVersionsToStages": {
                "bd4bac73-6a13-4a46-b51d-4dc9c81229c6": [
                    "AWSCURRENT"
                ]
            }, 
            "CreatedDate": 1621769699.167, 
            "ARN": "arn:aws:secretsmanager:eu-central-1:151743893450:secret:kong_key_ecsa-GZGijy", 
            "Description": "kong_ecsa_key"
        }, 
        {
            "Name": "kong_cert_ecsa", 
            "LastChangedDate": 1621769711.909, 
            "SecretVersionsToStages": {
                "480cef99-b77a-4b2d-b856-f68611792cb7": [
                    "AWSCURRENT"
                ]
            }, 
            "CreatedDate": 1621769711.86, 
            "ARN": "arn:aws:secretsmanager:eu-central-1:151743893450:secret:kong_cert_ecsa-PceCaK", 
            "Description": "kong_ecsa_certificate"
        }
    ]
}








ECS Task Definition File
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html
https://docs.amazonaws.cn/en_us/AmazonECS/latest/APIReference/API_RegisterTaskDefinition.html
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/example_task_definitions.html
https://aws.amazon.com/premiumsupport/knowledge-center/ecs-data-security-container-task/
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data-tutorial.html


Create a directory for Postgres Database:
mkdir /pgdata

Create a "kong_cp_ecsa.json" file with the ECS task definition. Use the EC2's Public IP for "KONG_API_URI" and "KONG_ADMIN_GUI_URL" environment variables.

{
  "requiresCompatibilities": [
    "EXTERNAL"
  ],
  "executionRoleArn": "arn:aws:iam::151743893450:role/ecsTaskExecutionRole",
  "volumes": [
    {
      "name": "kong-postgres-volume",
      "dockerVolumeConfiguration": {
        "autoprovision": true,
        "scope": "shared",
        "driver": "local",
        "driverOpts": {
          "o": "bind",
          "type": "none",
          "device": "/pgdata"
        }
      }
    },
    {
      "name": "kong-cert-volume",
      "dockerVolumeConfiguration": {
        "autoprovision": true,
        "scope": "shared",
        "driver": "local",
        "driverOpts": {
          "o": "bind",
          "type": "none",
          "device": "/kong"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "postgres",
      "mountPoints": [
        {
          "containerPath": "/var/lib/postgresql/data",
          "sourceVolume": "kong-postgres-volume",
          "readOnly": false
        }
      ],
      "image": "postgres:latest",
      "environment" : [
        {
          "name": "POSTGRES_USER",
          "value": "kong"
        },
        {
          "name": "POSTGRES_PASSWORD",
          "value": "kong"
        },
        {
          "name": "POSTGRES_DB",
          "value": "kong"
        },
        {
          "name": "POSTGRES_HOST_AUTH_METHOD",
          "value": "trust"
        }
      ],
      "hostname": "postgres",
      "memory": 256,
      "cpu": 256,
      "essential": false,
      "portMappings": [
        {
          "containerPort": 5432,
          "hostPort": 5432,
          "protocol": "tcp"
        }
      ],
      "healthCheck": {
          "command": ["CMD-SHELL", "pg_isready -U postgres"],
          "interval": 10,
          "retries": 5,
          "startPeriod": 5,
          "timeout": 5
      }
    },
    {
      "name": "konnect-bootstrap",
      "image": "kong/kong-gateway:2.4.1.0-alpine",
      "links": ["postgres"],
      "dependsOn": [
        {
          "condition": "HEALTHY",
          "containerName": "postgres"
        }
      ],
      "command": [
        "kong",
        "migrations",
        "bootstrap"
      ],
      "environment": [
        {
          "name": "KONG_DATABASE",
          "value": "postgres"
        },
        {
          "name": "KONG_PG_HOST",
          "value": "postgres"
        },
        {
          "name": "KONG_PG_DATABASE",
          "value": "kong"
        },
        {
          "name": "KONG_PG_USER",
          "value": "kong"
        },
        {
          "name": "KONG_PASSWORD",
          "value": "kong"
        },
        {
          "name": "POSTGRES_PASSWORD",
          "value": "kong"
        }
      ],
      "secrets": [
        {
          "valueFrom": "arn:aws:secretsmanager:eu-central-1:151743893450:secret:kong_license_ecsa-3QmZjB",
          "name": "KONG_LICENSE_DATA"
        }
      ],
      "memory": 256,
      "cpu": 256,
      "essential": false
    },
    {
      "name": "konnect",
      "mountPoints": [
        {
          "containerPath": "/config",
          "sourceVolume": "kong-cert-volume",
          "readOnly": false
        }
      ],
      "image": "kong/kong-gateway:2.4.1.0-alpine",
      "links": ["postgres"],
      "dependsOn": [
        {
          "condition": "COMPLETE",
          "containerName": "konnect-bootstrap"
        },
        {
          "condition": "START",
          "containerName": "postgres"
        }
      ],
      "environment": [
        {
          "name": "KONG_ROLE",
          "value": "control_plane"
        },
        {
          "name": "KONG_DATABASE",
          "value": "postgres"
        },
        {
          "name": "KONG_PG_HOST",
          "value": "postgres"
        },
        {
          "name": "KONG_ENFORCE_RBAC",
          "value": "on"
        },
        {
          "name": "KONG_API_URI",
          "value": "http:\/\/52.59.250.107:8001"
        },
        {
          "name": "KONG_ADMIN_GUI_URL",
          "value": "http:\/\/52.59.250.107:8002"
        },
        {
          "name": "KONG_ADMIN_GUI_AUTH",
          "value": "basic-auth"
        },
        {
          "name": "KONG_ADMIN_GUI_AUTH_CONF",
          "value": "{\"hide_credentials\":true}"
        },
        {
          "name": "KONG_ADMIN_GUI_SESSION_CONF",
          "value": "{\"secret\":\"secret\",\"storage\":\"kong\",\"cookie_secure\":false}"
        },
        {
          "name": "KONG_ADMIN_LISTEN",
          "value": "0.0.0.0:8001, 0.0.0.0:8444 ssl"
        },
        {
          "name": "ADMIN_GUI_LISTEN",
          "value": "0.0.0.0:8002, 0.0.0.0:8445 ssl"
        },
        {
          "name": "KONG_CLUSTER_LISTEN",
          "value": "0.0.0.0:8005"
        },
        {
          "name": "KONG_CLUSTER_TELEMETRY_LISTEN",
          "value": "0.0.0.0:8005"
        },
        {
          "name": "KONG_CLUSTER_KEY",
          "value": "0.0.0.0:8005"
        },
        {
          "name": "KONG_CLUSTER_TELEMETRY_LISTEN",
          "value": "0.0.0.0:8005"
        },
        {
          "name": "KONG_CLUSTER_CERT",
          "value": "/config/cluster.crt"
        },
        {
          "name": "KONG_CLUSTER_CERT_KEY",
          "value": "/config/cluster.key"
        }
      ],
      "secrets": [
        {
          "valueFrom": "arn:aws:secretsmanager:eu-central-1:151743893450:secret:kong_license_ecsa-3QmZjB",
          "name": "KONG_LICENSE_DATA"
        }
      ],
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000,
          "protocol": "tcp"
        },
        {
          "containerPort": 8001,
          "hostPort": 8001,
          "protocol": "tcp"
        },
        {
          "containerPort": 8002,
          "hostPort": 8002,
          "protocol": "tcp"
        },
        {
          "containerPort": 8005,
          "hostPort": 8005,
          "protocol": "tcp"
        },
        {
          "containerPort": 8006,
          "hostPort": 8006,
          "protocol": "tcp"
        },
        {
          "containerPort": 8444,
          "hostPort": 8444,
          "protocol": "tcp"
        },
        {
          "containerPort": 8445,
          "hostPort": 8445,
          "protocol": "tcp"
        }
      ],
      "hostname": "konnect",
      "memory": 400,
      "cpu": 400,
      "essential": true
    }
  ],
  "networkMode": "bridge",
  "family": "kong-ecs-anywhere"
}




Registering the ECS Task Definition
aws ecs register-task-definition --cli-input-json file://kong_cp_ecsa.json

$ aws ecs list-task-definition-families
{
    "families": [
        "kong-ecs-anywhere"
    ]
}


$ aws ecs list-task-definitions --family-prefix kong-ecs-anywhere
{
    "taskDefinitionArns": [
        "arn:aws:ecs:eu-central-1:151743893450:task-definition/kong-ecs-anywhere:1"
    ]
}

Running the ECS Task

$ aws ecs run-task --cluster kong-control-plane-ecsa --launch-type EXTERNAL --task-definition kong-ecs-anywhere
{
    "tasks": [
        {
            "attachments": [],
            "clusterArn": "arn:aws:ecs:eu-central-1:151743893450:cluster/kong-control-plane-ecsa",
            "containerInstanceArn": "arn:aws:ecs:eu-central-1:151743893450:container-instance/kong-control-plane-ecsa/18e2078111e24370b2924b4de1584a31",
            "containers": [
                {
                    "containerArn": "arn:aws:ecs:eu-central-1:151743893450:container/kong-control-plane-ecsa/eb0a67636aeb4d7a83b46345781a60d4/f2625c8f-7a8d-4c0f-9431-7d2a3b5f1431",
                    "taskArn": "arn:aws:ecs:eu-central-1:151743893450:task/kong-control-plane-ecsa/eb0a67636aeb4d7a83b46345781a60d4",
                    "name": "postgres",
                    "image": "postgres:latest",
                    "lastStatus": "PENDING",
                    "networkInterfaces": [],
                    "cpu": "256",
                    "memory": "256"
                },
                {
                    "containerArn": "arn:aws:ecs:eu-central-1:151743893450:container/kong-control-plane-ecsa/eb0a67636aeb4d7a83b46345781a60d4/12fed4ad-f47b-4997-a4c2-d5558dbdaf37",
                    "taskArn": "arn:aws:ecs:eu-central-1:151743893450:task/kong-control-plane-ecsa/eb0a67636aeb4d7a83b46345781a60d4",
                    "name": "konnect-bootstrap",
                    "image": "kong/kong-gateway:2.3.3.2-alpine",
                    "lastStatus": "PENDING",
                    "networkInterfaces": [],
                    "cpu": "256",
                    "memory": "256"
                },
                {
                    "containerArn": "arn:aws:ecs:eu-central-1:151743893450:container/kong-control-plane-ecsa/eb0a67636aeb4d7a83b46345781a60d4/e8916865-8a99-4105-a3b5-3039bc3f7a22",
                    "taskArn": "arn:aws:ecs:eu-central-1:151743893450:task/kong-control-plane-ecsa/eb0a67636aeb4d7a83b46345781a60d4",
                    "name": "konnect",
                    "image": "kong/kong-gateway:2.3.3.2-alpine",
                    "lastStatus": "PENDING",
                    "networkInterfaces": [],
                    "cpu": "400",
                    "memory": "400"
                }
            ],
            "cpu": "912",
            "createdAt": "2021-05-15T17:42:41.379000-03:00",
            "desiredStatus": "RUNNING",
            "enableExecuteCommand": false,
            "group": "family:kong-ecs-anywhere",
            "lastStatus": "PENDING",
            "launchType": "EXTERNAL",
            "memory": "912",
            "overrides": {
                "containerOverrides": [
                    {
                        "name": "konnect-bootstrap"
                    },
                    {
                        "name": "konnect"
                    },
                    {
                        "name": "postgres"
                    }
                ],
                "inferenceAcceleratorOverrides": []
            },
            "tags": [],
            "taskArn": "arn:aws:ecs:eu-central-1:151743893450:task/kong-control-plane-ecsa/eb0a67636aeb4d7a83b46345781a60d4",
            "taskDefinitionArn": "arn:aws:ecs:eu-central-1:151743893450:task-definition/kong-ecs-anywhere:37",
            "version": 1
        }
    ],
    "failures": []
}






Checking the Deployment
# docker container ls -a
CONTAINER ID   IMAGE                              COMMAND                  CREATED          STATUS                     PORTS                                                                                                                                          NAMES
dc43c961ff95   kong/kong-gateway:2.4.1.0-alpine   "/docker-entrypoint.…"   2 minutes ago    Up 2 minutes               0.0.0.0:8000-8002->8000-8002/tcp, 8003-8004/tcp, 0.0.0.0:8005-8006->8005-8006/tcp, 8443/tcp, 0.0.0.0:8444-8445->8444-8445/tcp, 8446-8447/tcp   ecs-kong-ecs-anywhere-63-konnect-d487c2cbeeb5c3f11f00
9e3d08897e65   kong/kong-gateway:2.4.1.0-alpine   "/docker-entrypoint.…"   2 minutes ago    Exited (0) 2 minutes ago                                                                                                                                                  ecs-kong-ecs-anywhere-63-konnect-bootstrap-ecb8b9a9b195c2ac7900
ee394da4ad73   postgres:latest                    "docker-entrypoint.s…"   3 minutes ago    Up 3 minutes (healthy)     0.0.0.0:5432->5432/tcp                                                                                                                         ecs-kong-ecs-anywhere-63-postgres-daa4b7ccb5d3bbd95200
3d5fc1f86816   amazon/amazon-ecs-agent:latest     "/agent"                 10 minutes ago   Up 10 minutes (healthy)                                                                                                                                                   ecs-agent









Checking the Kong Konnect Enterprise Control Plane
# http :8001 kong-admin-token:kong | jq .version
"2.4.1.0-enterprise-edition"


Redirect your browser to the ECS'2 Public IP: http://52.59.250.107:8002. Use the "kong_admin/kong" credentials.


Injecting the Certificate/Key in Control Plane
As described earlier, this docs describes two options for storing the Certificate/Key pair: AWS Secrets Manager or Kong Konnect Control Plane. If you plan to use the Control Plane, we can inject the pair like this:

http post :8001/workspaces name=certificates kong-admin-token:kong

curl -sX POST http://localhost:8001/certificates/certificates \
    -F "cert=@/kong/cluster.crt" \
    -F "key=@/kong/cluster.key" \
    -H "kong-admin-token: kong"

# http :8001/certificates/certificates kong-admin-token:kong
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 1144
Content-Type: application/json; charset=utf-8
Date: Sat, 22 May 2021 14:58:05 GMT
Server: kong/2.4.1.0-enterprise-edition
X-Kong-Admin-Latency: 2
X-Kong-Admin-Request-ID: Ei5MPToJrBEU98BhunSnHeJGsf8KLKj2
vary: Origin

{
    "data": [
        {
            "cert": "-----BEGIN CERTIFICATE-----\nMIIBtzCCAT6gAwIBAgIJAKS5Ggba9A0gMAoGCCqGSM49BAMCMBoxGDAWBgNVBAMM\nD2tvbmdfY2x1c3RlcmluZzAeFw0yMTA1MjIxNDM0MzVaFw0yNDA1MjExNDM0MzVa\nMBoxGDAWBgNVBAMMD2tvbmdfY2x1c3RlcmluZzB2MBAGByqGSM49AgEGBSuBBAAi\nA2IABDd8zqiLc588LjlBvJ+FidGOFGfAXwFegydzFR7ifA831xNKFjYLTLNjaDPa\nrZjQQM1HFKUXnOX//A+bsawGKWWSUZBbgUX/iNR3H+f/+Pse+J5lK8ybngZnwpEg\nOyUrD6NQME4wHQYDVR0OBBYEFCudQvpUMBdtbETR11vZfbQXVyb6MB8GA1UdIwQY\nMBaAFCudQvpUMBdtbETR11vZfbQXVyb6MAwGA1UdEwQFMAMBAf8wCgYIKoZIzj0E\nAwIDZwAwZAIwbPi7tdsgOI0XQi6hCHAgpcE/3FLW8Cema+OSMLC2PppMMRTXUuf3\ntUxzQelL5fgTAjBc7umPkYXy6DF3lSRPDklJuddQdfsXnODADdbt06ky1CJ8bTZa\nrMikh0yGVavP+QA=\n-----END CERTIFICATE-----\n", 
            "cert_alt": null, 
            "created_at": 1621695447, 
            "id": "a1d9ec60-301a-415e-b359-cffbdee2d359", 
            "key": "-----BEGIN PRIVATE KEY-----\nMIG2AgEAMBAGByqGSM49AgEGBSuBBAAiBIGeMIGbAgEBBDDEBVIhjZI1K69FAj1H\nN3UNJvSyuZ3Z8rQ0O3nxgnNzY6eXA5sP1RJVyawzwzhJvfahZANiAAQ3fM6oi3Of\nPC45QbyfhYnRjhRnwF8BXoMncxUe4nwPN9cTShY2C0yzY2gz2q2Y0EDNRxSlF5zl\n//wPm7GsBillklGQW4FF/4jUdx/n//j7HvieZSvMm54GZ8KRIDslKw8=\n-----END PRIVATE KEY-----\n", 
            "key_alt": null, 
            "snis": [], 
            "tags": null
        }
    ], 
    "next": null
}






Data Plane ECS-A Cluster
Creating ECS Anywhere Cluster for Konnect Data Plane
$ aws ecs create-cluster --cluster-name kong-data-plane-ecsa

$ aws ecs list-clusters
{
    "clusterArns": [
        "arn:aws:ecs:eu-central-1:151743893450:cluster/kong-data-plane-ecsa",
        "arn:aws:ecs:eu-central-1:151743893450:cluster/kong-control-plane-ecsa"
    ]
}

Data Plane Installation - EC2/Docker
Create another "Amazon Linux 2" EC2 instance using a "t2.large" Instance Type.

Choose the default VPC, a Public Subnet and enable "Auto-assign Public IP". Add the same EFS File System we created before. Change the Security Group to enable access from any address.

Open a terminal:
ssh -i "acqua-frankfurt.pem" ec2-user@ec2-3-121-100-188.eu-central-1.compute.amazonaws.com


Install utilities
sudo vi /etc/environment

add these lines
LANG=en_US.utf-8
LC_ALL=en_US.utf-8

In your $HOME directory change the .bashrc file adding the following line in the end:
sudo vi .bashrc
ulimit -n 4096


login again
sudo yum update -y
sudo amazon-linux-extras install epel -y
sudo yum install httpie -y
sudo yum install jq -y

Installing the ECS-A Agent
sudo bash


Installing ECS-A Agent
Download the ECS-A Agent installation script
# curl -o "ecs-anywhere-install.sh" "https://amazon-ecs-agent-packages-preview.s3.us-east-1.amazonaws.com/ecs-anywhere-install.sh" && sudo chmod +x ecs-anywhere-install.sh

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 12816  100 12816    0     0   9375      0  0:00:01  0:00:01 --:--:--  9375



Using the previously issued Activation ID and Code, install the Agent
# ./ecs-anywhere-install.sh \
     --cluster kong-data-plane-ecsa \
     --activation-id 386aa0ed-6cf0-41da-951b-ebf2e774b8a0 \
     --activation-code MiGiqFbs8YisEMTea0hT \
     --region eu-central-1


Checking the Installation
# docker container ls -a
CONTAINER ID   IMAGE                            COMMAND    CREATED              STATUS                        PORTS     NAMES
9d3c441a4b29   amazon/amazon-ecs-agent:latest   "/agent"   About a minute ago   Up About a minute (healthy)             ecs-agent

# aws ecs list-container-instances --cluster kong-data-plane-ecsa
{
    "containerInstanceArns": [
        "arn:aws:ecs:eu-central-1:151743893450:container-instance/kong-data-plane-ecsa/8a6a7af42a244214a23f01722f5893c1"
    ]
}




Kong Konnect Enterprise Installation

Connect Control Plane from Data Plane
# http 52.59.250.107:8001 kong-admin-token:kong | jq .version
"2.4.1.0-enterprise-edition"


Getting Certificate/Key from Control Plane - Konnect Control Plane version
mkdir /kong

http --verify=no https://18.198.24.235:8444/certificates/certificates kong-admin-token:kong | jq -r .data[0].cert > /kong/cluster.crt

http --verify=no https://18.198.24.235:8444/certificates/certificates kong-admin-token:kong | jq -r .data[0].key > /kong/cluster.key


Getting the Certificate/Key in Control Plane - AWS Secrets Manager version
mkdir /kong

aws secretsmanager get-secret-value --secret-id kong_cert_ecsa --version-stage AWSCURRENT | jq -r .SecretString > /kong/cluster.crt

aws secretsmanager get-secret-value --secret-id kong_key_ecsa --version-stage AWSCURRENT | jq -r .SecretString > /kong/cluster.key


ECS Task Definition File
Create a "kong_dp_ecsa.json" file with the ECS task definition. Use the EC2's Public IP for "KONG_CLUSTER_CONTROL_PLANE" and "KONG_CLUSTER_TELEMETRY_ENDPOINT" environment variables.


{
  "requiresCompatibilities": [
    "EXTERNAL"
  ],
  "executionRoleArn": "arn:aws:iam::151743893450:role/ecsTaskExecutionRole",
  "volumes": [
    {
      "name": "kong-cert-volume",
      "dockerVolumeConfiguration": {
        "autoprovision": true,
        "scope": "shared",
        "driver": "local",
        "driverOpts": {
          "o": "bind",
          "type": "none",
          "device": "/kong"
        }
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "konnect",
      "mountPoints": [
        {
          "containerPath": "/config",
          "sourceVolume": "kong-cert-volume",
          "readOnly": false
        }
      ],
      "image": "kong/kong-gateway:2.4.1.0-alpine",
      "environment": [
        {
          "name": "KONG_ROLE",
          "value": "data_plane"
        },
        {
          "name": "KONG_DATABASE",
          "value": "off"
        },
        {
          "name": "KONG_PORTAL",
          "value": "off"
        },
        {
          "name": "KONG_CLUSTER_CONTROL_PLANE",
          "value": "52.59.250.107:8005"
        },
        {
          "name": "KONG_CLUSTER_TELEMETRY_ENDPOINT",
          "value": "52.59.250.107:8006"
        },
        {
          "name": "KONG_CLUSTER_CERT",
          "value": "/config/cluster.crt"
        },
        {
          "name": "KONG_CLUSTER_CERT_KEY",
          "value": "/config/cluster.key"
        },
        {
          "name": "KONG_LUA_SSL_TRUSTED_CERTIFICATE",
          "value": "/config/cluster.crt"
        }
      ],
      "secrets": [
        {
          "valueFrom": "arn:aws:secretsmanager:eu-central-1:151743893450:secret:kong_license_ecsa-3QmZjB",
          "name": "KONG_LICENSE_DATA"
        }
      ],
      "portMappings": [
        {
          "containerPort": 8000,
          "hostPort": 8000,
          "protocol": "tcp"
        },
        {
          "containerPort": 8001,
          "hostPort": 8001,
          "protocol": "tcp"
        }
      ],
      "hostname": "konnect",
      "memory": 400,
      "cpu": 400,
      "essential": true
    }
  ],
  "networkMode": "bridge",
  "family": "kong-dataplane-ecs-anywhere"
}



Registering the ECS Task Definition
aws ecs register-task-definition --cli-input-json file://kong_dp_ecsa.json

$ aws ecs list-task-definition-families
{
    "families": [
        "kong-dataplane-ecs-anywhere",
        "kong-ecs-anywhere"
    ]
}

Running the ECS Task
$ aws ecs run-task --cluster kong-data-plane-ecsa --launch-type EXTERNAL --task-definition kong-dataplane-ecs-anywhere



Checking the Deployment
# docker container ls
CONTAINER ID   IMAGE                              COMMAND                  CREATED          STATUS                    PORTS                                                            NAMES
391f65bafd71   kong/kong-gateway:2.3.3.2-alpine   "/docker-entrypoint.…"   16 seconds ago   Up 13 seconds             8002-8004/tcp, 0.0.0.0:8000-8001->8000-8001/tcp, 8443-8447/tcp   ecs-kong-dataplane-ecs-anywhere-1-konnect-82fbb6b0f89d9c8f3200
e16f5af87197   amazon/amazon-ecs-agent:latest     "/agent"                 12 minutes ago   Up 12 minutes (healthy)                                                                    ecs-agent


Checking the Kong Konnect Enterprise Data Plane
# http :8000
HTTP/1.1 404 Not Found
Connection: keep-alive
Content-Length: 48
Content-Type: application/json; charset=utf-8
Date: Sat, 15 May 2021 21:10:11 GMT
Server: kong/2.3.3.2-enterprise-edition
X-Kong-Response-Latency: 1

{
    "message": "no Route matched with those values"
}


Creating and consuming Kong Konnect Service and Route
From your laptop, use the Control Plane's Public IP to create the Service and Route

http 52.59.250.107:8001/services name=httpbinservice url='http://httpbin.org' kong-admin-token:kong

http 52.59.250.107:8001/services/httpbinservice/routes name='httpbinroute' paths:='["/httpbin"]' kong-admin-token:kong


Now, use the Data Plane's Public IP to consumer the Route

$ http 3.121.100.188:8000/httpbin/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 434
Content-Type: application/json
Date: Sat, 15 May 2021 21:13:51 GMT
Server: gunicorn/19.9.0
Via: kong/2.3.3.2-enterprise-edition
X-Kong-Proxy-Latency: 59
X-Kong-Upstream-Latency: 182

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/2.4.0",
        "X-Amzn-Trace-Id": "Root=1-60a0398f-3b2c25473810111760cd655b",
        "X-Forwarded-Host": "3.124.242.23",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "186.204.145.204, 3.124.242.23",
    "url": "http://3.124.242.23/get"
}




Stop and Delete the Task
# aws ecs list-tasks --cluster kong-data-plane-ecsa
{
    "taskArns": [
        "arn:aws:ecs:eu-central-1:151743893450:task/kong-data-plane-ecsa/5fb9bd22535a49da8836e1e10b3f0fb2"
    ]
}



aws ecs stop-task --cluster kong-data-plane-ecsa --task 5fb9bd22535a49da8836e1e10b3f0fb2


# aws ecs list-tasks --cluster kong-data-plane-ecsa
{
    "taskArns": []
}

# docker container ls -a
CONTAINER ID   IMAGE                              COMMAND                  CREATED        STATUS                      PORTS     NAMES
391f65bafd71   kong/kong-gateway:2.3.3.2-alpine   "/docker-entrypoint.…"   24 hours ago   Exited (0) 32 seconds ago             ecs-kong-dataplane-ecs-anywhere-1-konnect-82fbb6b0f89d9c8f3200
e16f5af87197   amazon/amazon-ecs-agent:latest     "/agent"                 24 hours ago   Up 24 hours (healthy)                 ecs-agent

ECS Anywhere Remote Service
Creating the ECS Service and Task
aws ecs create-service --cluster kong-data-plane-ecsa --service-name kong-data-plane-ecsa-service --launch-type EXTERNAL --task-definition kong-dataplane-ecs-anywhere --desired-count 1



# aws ecs describe-services --cluster kong-data-plane-ecsa --service kong-data-plane-ecsa-service
{
    "services": [
        {
            "status": "ACTIVE", 
            "serviceRegistries": [], 
            "pendingCount": 1, 
            "launchType": "EXTERNAL", 
            "enableECSManagedTags": false, 
            "schedulingStrategy": "REPLICA", 
            "loadBalancers": [], 
            "placementConstraints": [], 
            "createdAt": 1621198756.207, 
            "desiredCount": 1, 
            "serviceName": "kong-data-plane-ecsa-service", 
            "clusterArn": "arn:aws:ecs:eu-central-1:151743893450:cluster/kong-data-plane-ecsa", 
            "createdBy": "arn:aws:iam::151743893450:user/claudioacquaviva", 
            "taskDefinition": "arn:aws:ecs:eu-central-1:151743893450:task-definition/kong-dataplane-ecs-anywhere:1", 
            "serviceArn": "arn:aws:ecs:eu-central-1:151743893450:service/kong-data-plane-ecsa/kong-data-plane-ecsa-service", 
            "propagateTags": "NONE", 
            "deploymentConfiguration": {
                "maximumPercent": 200, 
                "minimumHealthyPercent": 100
            }, 
            "deployments": [
                {
                    "status": "PRIMARY", 
                    "pendingCount": 1, 
                    "launchType": "EXTERNAL", 
                    "createdAt": 1621198756.207, 
                    "desiredCount": 1, 
                    "taskDefinition": "arn:aws:ecs:eu-central-1:151743893450:task-definition/kong-dataplane-ecs-anywhere:1", 
                    "updatedAt": 1621198756.207, 
                    "id": "ecs-svc/8118469539970930987", 
                    "runningCount": 0
                }
            ], 
            "events": [
                {
                    "message": "(service kong-data-plane-ecsa-service) has started 1 tasks: (task 7acf30ec498e42b9a1bbf97e5be5f932).", 
                    "id": "2211f6ea-5cad-4166-a8b8-0eefd4b74b5e", 
                    "createdAt": 1621198766.009
                }
            ], 
            "runningCount": 0, 
            "placementStrategy": []
        }
    ], 
    "failures": []
}


# aws ecs list-tasks --cluster kong-data-plane-ecsa
{
    "taskArns": [
        "arn:aws:ecs:eu-central-1:151743893450:task/kong-data-plane-ecsa/7acf30ec498e42b9a1bbf97e5be5f932"
    ]
}




# sudo docker container ls -a
CONTAINER ID   IMAGE                              COMMAND                  CREATED              STATUS                     PORTS                                                            NAMES
c078f2c16402   kong/kong-gateway:2.3.3.2-alpine   "/docker-entrypoint.…"   About a minute ago   Up About a minute          8002-8004/tcp, 0.0.0.0:8000-8001->8000-8001/tcp, 8443-8447/tcp   ecs-kong-dataplane-ecs-anywhere-1-konnect-9af598ebade1eeae0200
391f65bafd71   kong/kong-gateway:2.3.3.2-alpine   "/docker-entrypoint.…"   24 hours ago         Exited (0) 4 minutes ago                                                                    ecs-kong-dataplane-ecs-anywhere-1-konnect-82fbb6b0f89d9c8f3200
e16f5af87197   amazon/amazon-ecs-agent:latest     "/agent"                 24 hours ago         Up 24 hours (healthy)                                                                       ecs-agent




Checking the Kong Konnect Enterprise and Data Plane
$ http 3.121.100.188:8000/httpbin/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 434
Content-Type: application/json
Date: Sun, 16 May 2021 21:02:37 GMT
Server: gunicorn/19.9.0
Via: kong/2.3.3.2-enterprise-edition
X-Kong-Proxy-Latency: 22
X-Kong-Upstream-Latency: 888

{
    "args": {},
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Host": "httpbin.org",
        "User-Agent": "HTTPie/2.4.0",
        "X-Amzn-Trace-Id": "Root=1-60a1886c-3be198c07cb68a3a1f65d38d",
        "X-Forwarded-Host": "3.124.242.23",
        "X-Forwarded-Path": "/httpbin/get",
        "X-Forwarded-Prefix": "/httpbin"
    },
    "origin": "186.204.145.204, 3.124.242.23",
    "url": "http://3.124.242.23/get"
}




