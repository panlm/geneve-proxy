## AWS Gateway Load Balancer Centralized Architecture Validation

### 概述

* 本章节提供步骤验证 GWLB 集中架构
* 建议部署一台 cloud9 进行本章节验证步骤
  * 将访问 Instance 的私钥上传到 cloud9 中

### 验证从 Spoke1 VPC 中运行的应用程序访问 Internet

* 通过以下脚本从cloudformation堆栈中获取各vpc的堡垒主机的公共 IP 地址和相关的私有 IP 地址。方便登录验证

![](images/appliance_vpc_stack_outputs.jpg)

```sh
key_file=awskey # 输入 private key 路径
region_name=us-east-2 # 选择你的区域
stack_name=GwlbCentralizedDemo

sudo yum install -y jq
export AWS_DEFAULT_REGION=$region_name
sub_stack_name=$(aws cloudformation describe-stack-resources \
  --stack-name $stack_name |\
  jq -r '.StackResources[].PhysicalResourceId' |\
  awk -F'/' '{print $2}')
tgw_stack_name=$(echo $sub_stack_name |xargs -n 1 |grep 'TgwStack')
appliance_stack_name=$(echo $sub_stack_name |xargs -n 1 |grep 'ApplianceVpcStack')
spoke1_stack_name=$(echo $sub_stack_name |xargs -n 1 |grep 'Spoke1VpcStack')
spoke2_stack_name=$(echo $sub_stack_name |xargs -n 1 |grep 'Spoke2VpcStack')

echo "export key_file=$key_file" >> ~/.bash_profile
echo "export stack_name=$stack_name" >> ~/.bash_profile
echo "export region_name=$region_name" >> ~/.bash_profile
echo "export tgw_stack_name=$tgw_stack_name" >> ~/.bash_profile
echo "export appliance_stack_name=$appliance_stack_name" >> ~/.bash_profile
echo "export spoke1_stack_name=$spoke1_stack_name" >> ~/.bash_profile
echo "export spoke2_stack_name=$spoke2_stack_name" >> ~/.bash_profile
echo "alias ssh='ssh -o StrictHostKeyChecking=no'" >> ~/.bash_profile
source ~/.bash_profile

# get output from tgw
tgw_attachment_id=$(aws cloudformation describe-stacks \
  --stack-name $tgw_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='TgwApplianceVpcAttachmendId'].OutputValue" \
  --output text)

echo "export tgw_attachment_id=$tgw_attachment_id" >> ~/.bash_profile

# get addresses for appliance
appliance_bastion_pub_ip=$(aws cloudformation describe-stacks \
  --stack-name $appliance_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='ApplianceBastionHostPublicIp'].OutputValue" \
  --output text)
appliance1_priv_ip=$(aws cloudformation describe-stacks \
  --stack-name $appliance_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='Appliance1PrivateIp'].OutputValue" \
  --output text)
appliance2_priv_ip=$(aws cloudformation describe-stacks \
  --stack-name $appliance_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='Appliance2PrivateIp'].OutputValue" \
  --output text)

echo "export appliance_bastion_pub_ip=$appliance_bastion_pub_ip" >> ~/.bash_profile
echo "export appliance1_priv_ip=$appliance1_priv_ip" >> ~/.bash_profile
echo "export appliance2_priv_ip=$appliance2_priv_ip" >> ~/.bash_profile

# put connection to appliance
cat $key_file |\
  ssh ec2-user@$appliance_bastion_pub_ip -i $key_file \
    'cat - > /tmp/key ; chmod 600 /tmp/key'
echo "ssh -o StrictHostKeyChecking=no ec2-user@$appliance1_priv_ip -i /tmp/key" |\
  ssh ec2-user@$appliance_bastion_pub_ip -i $key_file \
    'cat - > ~/connect-to-appliance1.sh ; chmod a+x ~/connect-to-appliance1.sh'
echo "ssh -o StrictHostKeyChecking=no ec2-user@$appliance2_priv_ip -i /tmp/key" |\
  ssh ec2-user@$appliance_bastion_pub_ip -i $key_file \
    'cat - > ~/connect-to-appliance2.sh ; chmod a+x ~/connect-to-appliance2.sh'

# get addresses for spoke1
spoke1_bastion_pub_ip=$(aws cloudformation describe-stacks \
  --stack-name $spoke1_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='SpokeBastionHostPublicIp'].OutputValue" \
  --output text)
spoke1_application1_priv_ip=$(aws cloudformation describe-stacks \
  --stack-name $spoke1_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='SpokeApplication1PrivateIp'].OutputValue" \
  --output text)
spoke1_application2_priv_ip=$(aws cloudformation describe-stacks \
  --stack-name $spoke1_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='SpokeApplication2PrivateIp'].OutputValue" \
  --output text)

echo "export spoke1_bastion_pub_ip=$spoke1_bastion_pub_ip" >> ~/.bash_profile
echo "export spoke1_application1_priv_ip=$spoke1_application1_priv_ip" >> ~/.bash_profile
echo "export spoke1_application2_priv_ip=$spoke1_application2_priv_ip" >> ~/.bash_profile

# put connection to spoke1's instance
cat $key_file |\
  ssh ec2-user@$spoke1_bastion_pub_ip -i $key_file \
    'cat - > /tmp/key ; chmod 600 /tmp/key'
echo "ssh -o StrictHostKeyChecking=no ec2-user@$spoke1_application1_priv_ip -i /tmp/key" |\
  ssh ec2-user@$spoke1_bastion_pub_ip -i $key_file \
    'cat - > ~/connect-to-application1.sh ; chmod a+x ~/connect-to-application1.sh'
echo "ssh -o StrictHostKeyChecking=no ec2-user@$spoke1_application2_priv_ip -i /tmp/key" |\
  ssh ec2-user@$spoke1_bastion_pub_ip -i $key_file \
    'cat - > ~/connect-to-application2.sh ; chmod a+x ~/connect-to-application2.sh'

# get addresses for spoke2
spoke2_bastion_pub_ip=$(aws cloudformation describe-stacks \
  --stack-name $spoke2_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='SpokeBastionHostPublicIp'].OutputValue" \
  --output text)
spoke2_application1_priv_ip=$(aws cloudformation describe-stacks \
  --stack-name $spoke2_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='SpokeApplication1PrivateIp'].OutputValue" \
  --output text)
spoke2_application2_priv_ip=$(aws cloudformation describe-stacks \
  --stack-name $spoke2_stack_name \
  --query  "Stacks[0].Outputs[?OutputKey=='SpokeApplication2PrivateIp'].OutputValue" \
  --output text)

echo "export spoke2_bastion_pub_ip=$spoke2_bastion_pub_ip" >> ~/.bash_profile
echo "export spoke2_application1_priv_ip=$spoke2_application1_priv_ip" >> ~/.bash_profile
echo "export spoke2_application2_priv_ip=$spoke2_application2_priv_ip" >> ~/.bash_profile

# put connection to spoke2's instance
cat $key_file |\
  ssh ec2-user@$spoke2_bastion_pub_ip -i $key_file \
    'cat - > /tmp/key ; chmod 600 /tmp/key'
echo "ssh -o StrictHostKeyChecking=no ec2-user@$spoke2_application1_priv_ip -i /tmp/key" |\
  ssh ec2-user@$spoke2_bastion_pub_ip -i $key_file \
    'cat - > ~/connect-to-application1.sh ; chmod a+x ~/connect-to-application1.sh'
echo "ssh -o StrictHostKeyChecking=no ec2-user@$spoke2_application2_priv_ip -i /tmp/key" |\
  ssh ec2-user@$spoke2_bastion_pub_ip -i $key_file \
    'cat - > ~/connect-to-application2.sh ; chmod a+x ~/connect-to-application2.sh'

source ~/.bash_profile

```

* Access appliances 1/2 through bastion host:

![](images/access_appliances.jpg)

```sh
source ~/.bash_profile

# connect to appliance1
ssh ec2-user@$appliance_bastion_pub_ip -i $key_file
./connect-to-appliance1.sh

sudo tcpdump -ni eth0 port 6081 |grep 8.8.8.8

```

```sh
source ~/.bash_profile

# connect to appliance2
ssh ec2-user@$appliance_bastion_pub_ip -i $key_file
./connect-to-appliance2.sh

sudo tcpdump -ni eth0 port 6081 |grep 8.8.8.8

```

* From Spoke1 VPC stack Outputs tab, get the public IP address of the bastion host and private IP addresses of the application instance:

![](images/spoke1_vpc_stack_outputs.jpg)

* Access application instance through bastion host:

![](images/access_application.jpg)

```sh
source ~/.bash_profile

ssh ec2-user@$spoke1_bastion_pub_ip -i $key_file
./connect-to-application1.sh

ping 8.8.8.8

```

#### Ping:

* On both the appliances capture GENEVE traffic using tcpdump.
* From application instance running in Spoke1 VPC, ping a resource on the internet.
* Ping is successful and ICMP traffic is sent to appliance using GENEVE.

![](images/ping_access.jpg)

#### HTTP:

* On both the appliances capture GENEVE traffic using tcpdump.
* From application instance running in Spoke1 VPC, access a resource on the Internet over HTTP.
* Example below uses a curl command to access simple webserver running on an EC2 instance. Command is successfull. HTTP traffic is sent to appliance using GENEVE.

![](images/http_access.jpg)

### 验证从 Spoke1 VPC 到 Spoke2 VPC (East-West/VPC-to-VPC):

* As explained in the [ Centralized inspection architecture with AWS Gateway Load Balancer and AWS Transit Gateway blog](https://aws.amazon.com/blogs/networking-and-content-delivery/centralized-inspection-architecture-with-aws-gateway-load-balancer-and-aws-transit-gateway/), to ensure flow symmetry, Transit Gateway appliance mode should be enabled on the Appliance VPC attachment. In example below, application instance in Availability Zone (AZ) A of Spoke1 VPC tries to SSH into application instances in AZ A and AZ C of Spoke2 VPC.

* [reference blog](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-aws-gateway-load-balancer-supported-architecture-patterns/#:~:text=With%20the%20AWS%20Transit%20Gateway%20appliance,Transit%20Gateway%20appliance%20mode%C2%A0here.)

* From Spoke2 VPC stack Outputs tab, get the private IP addresses of the application instances:

![](images/spoke2_vpc_stack_outputs.jpg)

#### Transit Gateway appliance mode enabled:

* From Transit Gateway stack Outputs tab, get the Appliance VPC attahcment ID:

![](images/tgw_stack_outputs.jpg)

```sh
source ~/.bash_profile

aws ec2 describe-transit-gateway-vpc-attachments \
  --transit-gateway-attachment-ids $tgw_attachment_id \
  --region $region_name

```

* 当 Transit Gateway appliance mode 启用, 在 Spoke1 VPC 中的应用实例可以访问 Spoke2 VPC 中的任何应用实例。以下 ping 操作都可以正常返回。

```sh
source ~/.bash_profile
echo "spoke2_application1_priv_ip:" $spoke2_application1_priv_ip
echo "spoke2_application2_priv_ip:" $spoke2_application2_priv_ip

ssh ec2-user@$spoke1_bastion_pub_ip -i $key_file
./connect-to-application1.sh
ping -c 2 <spoke2_application1_priv_ip>
ping -c 2 <spoke2_application2_priv_ip>

ssh ec2-user@$spoke1_bastion_pub_ip -i $key_file
./connect-to-application2.sh
ping -c 2 <spoke2_application1_priv_ip>
ping -c 2 <spoke2_application2_priv_ip>

```

#### 禁用 Transit Gateway appliance mode:

* 禁用 Transit Gateway appliance mode

```sh
source ~/.bash_profile
aws ec2 modify-transit-gateway-vpc-attachment \
  --transit-gateway-attachment-id $tgw_attachment_id \
  --options ApplianceModeSupport=disable \
  --region $region_name

```

* 等待 1 到 2 分钟

* 当 Transit Gateway appliance mode 禁用,  在 Spoke1 VPC AZ A 中的应用实例只能访问在 Spoke2 VPC AZ A 中的应用实例，但是不能访问在 Spoke2 VPC AZ B 中的应用实例
