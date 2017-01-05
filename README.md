# Containerless #

Warning: may contain containers. Also, docker.

Deploys applications into an AWS ECS Cluster.



## Installation ##

The plugin is currently not published to npm, so you will need to copy and install it locally in your serverless application.

`cp containerless  .serverless_plugins`

Add the following to serverless.yml

```
plugins:
  - containerless
```



<!--
```
yarn add containerless
```
-->


## Commands ##

`sls cls-build`

- builds a set of docker images
- push images to repository


## Configuration ##

Configuration lives in a `custom.containerless` namespace in your `serverless.yml`

Containerless can use an existing ECS Cluster or can be configured to create a new cluster.

Minimal Configuration Example

```
custom:
  containerless:    
    cluster:
      vpcId: vpc-000000000001
      subnets:
        - sg-000000000001
        - sg-000000000002
    repository: 000000000000.dkr.ecr.ap-southeast-2.amazonaws.com/containerless
    applications:
      hello-world
```

### Cluster ###

Required fields:

 - vpcId
 - subnets

 A vpcId and one or more subnets in that VPC are required.
 If using an existing cluster, provide the VPC and subnets that the cluster has been created in.

### Use Existing Cluster ###

Required Fields:

- id (as aws arn)
- security_group

If an existing cluster id is provided, a security group is required in order to configure a load balancer for mapping URLs to containers. Please see below for more details.

Existing Cluster Configuration Example

```
custom:
  containerless:    
    cluster:
      id: arn:aws:ecs:ap-southeast-2:000000000000:cluster/ECSCluster
      vpcId: vpc-000000000001      
      subnets:
        - sg-000000000001
        - sg-000000000002
    repository: 000000000000.dkr.ecr.ap-southeast-2.amazonaws.com/containerless
```

### Create New Cluster ###

Optional fields:

 - capacity (defaults to 1)
 - instance_type (defaults to 't2.micro')
 - region (defaults to 'ap-southeast-2')
 - size (defaults to 1)

A new cluster will be created in an AutoScalingGroup. Capacity is the DesiredCapacity of the ASG. Please refer to AWS docs for details.

New Cluster Configuration Example

```
custom:
  containerless:    
    cluster:
      region: us-west-2
      capacity: 3
      vpcId: vpc-000000000001      
      subnets:
        - sg-000000000001
        - sg-000000000002
    repository: 000000000000.dkr.ecr.ap-southeast-2.amazonaws.com/containerless
```


### Applications ###

Each application requires a `Dockerfile` with the relevant docker setup.
Please refer to the samples provided for more details.

Fields:
  - src (if not provided will default to the application name)  
  - memory (defaults to 128)
  - url
  - port

In order to map to the load balancer applications need to provide both a URL and Port.

If url and port are omitted, the application will not be routed, and will be run as a task in the container.

Ports do not have to be unique, the system will dyanmically map ports to the docker container.

You can use any valid AWS Application Load Balancer path pattern as the URL.

Only one application can be mounted to the root url '/', because of the way in which the load balancer routes paths.
Other applications will be routed based on a pattern, and you will need to remember to mount your application on that route as the load balancer forwards the whole url.

For example, requests to the url `/hello` will be forwarded to the application retaining the `/hello` path, so your application needs to handle this.



```
custom:
  containerless:
    applications:
      hello-1:
        url: /
        port: 3000
        memory: 256
      hello-2:
        src: src-2
        url: /hello
        port: 3000
      hello-3
```


### Security Group ###

If you are using an existing ECS Cluster, the load balancer needs to be part of a security group that can route traffic to the Cluster Instances on the ephermeral port range used to map to docker containers.

Sample Security Group Configuration

```
{
  'Type': 'AWS::EC2::SecurityGroupIngress',
  'Properties': {
    'IpProtocol': 'tcp',
    'FromPort': 31000,
    'ToPort': 61000,
    'GroupId': {
      'Ref': 'ContainerlessSecurityGroup'
    },
    'SourceSecurityGroupId': {
      'Ref': 'ContainerlessSecurityGroup'
    }
  }
```
