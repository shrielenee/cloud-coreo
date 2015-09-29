##Test Heading##

## Cloud Services
Defined by the `config.rb` file.

The `config.rb` file resides in the `<repository-dir>/services` directory and exists to define interactions within the cloud provider in which you are operating. Our [API Documentation](http://docs.cloudcoreo.com/docs/frames/index) provides more specifics.

We define a ***resource*** as the actual cloud service you are interacting with, such as a [*Route53*](https://aws.amazon.com/route53/) entry or [*Autoscaling Group*](https://aws.amazon.com/autoscaling/). Resources are defined by the desired configuration within your cloud provider account. In general, the structure is:
```
<resource> "<name>" do
    <property1> <value>
    <property2> <value>
    <property3> <value>
    <property4> <value>
end
```
For instance:
```
coreo_aws_vpc_vpc "my-vpc" do
  action :sustain
  cidr "12.0.0.0/16"
  internet_gateway true
end
```
Based on the above snippet, CloudCoreo would create a VPC with the CIDR address "12.0.0.0/16" and attach an *Internet gateway* to the VPC to allow traffic to the public subnets (which haven't been defined in this specific example).

This is all very inflexible in terms of cloning, which is why ***templating*** has been built into the system.

### Templating System
Templating allows variables to be inserted into the system at runtime, allowing a single template to be used in many different environments. The substitution comes from either:

1. The value provided by the user in the CloudCoreo UI or...
2. The values set during the run of the CLI tool or...
3. The default set in the config.yaml file

This is the order in which they are applied, so if there is no UI or CLI variable set, CloudCoreo will apply the default.

The look of the variables is similar to *bash* but many of the "bash-isms" do not apply. To use a variable use `$[]` to contain it, like `${VARIABLE_NAME}`. The above example of creating a VPC would be more useful with the following templating:
```
coreo_aws_vpc_vpc "${VPC_NAME}" do
  action :sustain
  cidr "${VPC_CIDR}"
  internet_gateway true
end
```
Within your `config.yaml` file, you can now call out a `VPC_NAME` variable and apply defaults. This variable name will show up in the web UI for any users to modify.

### Overriding Resources
The `services/` directory honors the hierarchy we've already gone over. The difference, though, is that in the `config.rb` file each resource is override-able on an individual basis.

For instance, say we have the following directory structure and files:

* `<repository-dir>/extends/services/config.rb`and
* `<repository-dir>/services/config.rb`

Based on that, we know that anything in the `extends/services` directory will be run ***before*** the root services directory. While this is true, it also means that the resources are compiled before running. Simply, ***any resource in the root level `services/config.rb` file will override any resource located in the `extends/services/config.rb` file, where the key is the `<resource> "<name>" do` line***.

**Note:** Within CloudCoreo, the key is defined as the name of the resource. There can only be one key/name of resource within an individual stack. For instance, if you have a *coreo_aws_ec2_instance "my-instance"* block somewhere and again have a *coreo_aws_ec2_instance "my-instance"* block close to the root, the root will win! It will override the first block. Simply put, the resource name is the key and vice versa.

This follows the rule regarding overrides we talk about above. Since the resource names are effectively keys, as we get closer and closer to the root, they are added to the compilation stack in the defined order and overwrite (override) the previously added resources. 

Let's illlustrate this with an example utilizing the directory and file structure we have already described:

* `<repository-dir>/extends/services/config.rb`
* `<repository-dir>/services/config.rb`

Assume we have a **VPC** resource in the `extends/services/config.rb` file as follows:
```
coreo_aws_vpc_vpc "${VPC_NAME}" do
  action :sustain
  cidr "12.0.0.0/16"
  internet_gateway true
end
```
This doesn't do us any good because we can't clone this to different environments. We need to have templating and variables, and we want to add some tags to the VPC. We can override this resource by keying off the `<resource> "<name>" do` line and placing the new block in the root `services/config.rb` file. So, within the `<repository-dir>/services/config.rb` file we call out a new VPC as follows:
```
coreo_aws_vpc_vpc "${VPC_NAME}" do
  action :sustain
  cidr "${VPC_CIDR}"
  internet_gateway true
  tags [
     "Name=${VPC_NAME_TAG}",
     "Team=${VPC_TEAM}"
  ]
end
```
See the *key* as the first line? In this case it is `coreo_aws_vpc_vpc "${VPC_NAME}" do`. This means that as CloudCoreo is compiling the resources, it will see the VPC resource in the root last and will replace any existing blocks with this same resource key. The important part is that CloudCoreo will honor the positioning of the resources that it's overriding and simply override the block. We call this ***anchoring***.

### Anchoring
Anchoring is what CloudCoreo does with overridden resources. The key (`<resource> "<name>" do`) that is being overridden becomes an anchor for all pre-requisite resources found in an overriding file. That's a mouthful! Here's an abstract example (pay no attention to the blocks as they are irrelevant and shortened for clarity) using our standard directory structure:

* <repository-dir>/extends/services/config.rb
* <repository-dir>/services/config.rb

In our extends/services/config.rb file we will have a ***subnet definition*** and an ***elastic load balancer***:
```
coreo_aws_vpc_subnet 'public-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch true
end

coreo_aws_ec2_elb 'my-elb' do
   type 'public'
   subnet 'public-subnet'
end
```
This is not adequate! First of all we want the ELB to contain a security group and we want it to be an `internal` load balancer, not public. To achieve this, move to your root `services/config.rb` file and add your key resource. This will become an anchor, and we can modify anything other than the key itself.
```
coreo_aws_ec2_elb 'my-elb' do
   type 'internal'
   subnet 'private-subnet'
   security_groups ['my-elb-sg']
end
```
This one is modified correctly, but we are missing some pieces: the private subnet and the security groups. Re-edit your file with the knowledge that your anchor brings along all the pieces that it requires. Our new root-level `services/config.rb` file looks as follows:
```
coreo_aws_vpc_subnet 'private-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch false
end

coreo_aws_ec2_securityGroup 'my-elb-sg' do
   vpc 'my-vpc'
   allows [ 
            { 
              :direction => :ingress,
              :protocol => :tcp,
              :ports => ${INGRESS_PORTS},
              :cidrs => ${INGRESS_CIDRS}
            }
          ]
end

coreo_aws_ec2_elb 'my-elb' do
   type 'internal'
   subnet 'private-subnet'
   security_groups ['my-elb-sg']
end
```
The ***anchor*** in this is the ELB resource. Anything above that has not already overridden something will be considered a pre-requisite of the resource doing the overriding (the key). The root level `services/config.rb` file's coreo_aws_ec2_elb resource block will override the one found in the `extends/services/config.rb` file. Not only that, but all the resources above the key in the root level will maintain order and override with the key.

The final compilation will be a merge of the two files with the *anchor* located in the same place as the original, with any other non-overriding resources preceding it from the overriding file. The compiled version becomes:
```
coreo_aws_vpc_subnet 'public-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch true
end

coreo_aws_vpc_subnet 'private-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch false
end

coreo_aws_ec2_securityGroup 'my-elb-sg' do
   vpc 'my-vpc'
   allows [ 
            { 
              :direction => :ingress,
              :protocol => :tcp,
              :ports => ${INGRESS_PORTS},
              :cidrs => ${INGRESS_CIDRS}
            }
          ]
end

coreo_aws_ec2_elb 'my-elb' do
   type 'internal'
   subnet 'private-subnet'
   security_groups ['my-elb-sg']
end
```
The *'public-subnet'* resource has been included from the `extends/services/config.rb` file, while the *'my-elb'* from the     extends/services/config.rb   essentially becomes the *'private-subnet'* + *'my-elb-sg'* + *'my-elb'*.

### API Docs
If you are working on services within CloudCoreo, you need to know what options, parameters and more you can use. Head over to our [API Documentation](http://docs.cloudcoreo.com/docs/frames/index) to get more info on those.
