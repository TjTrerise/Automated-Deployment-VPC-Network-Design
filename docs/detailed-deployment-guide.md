# Automated Deployment of AWS Multi-Region VPC Network Architecture with Transit Gateway.

## Detailed Deployment Guide

### Step 1: Create S3 Buckets for Both regions.

Head over to the S3 Console
  - Create EU bucket.
   - Select eu-west-2. 
   - Click *Create bucket*.
     - Name: <identifier of your choosing>-multi-region-vpc-eu
     - Block all public access.
     - Disable bucket versioning.
     - Default encryption.
     - Create bucket.

  - Create US bucket.
    - select us-east-1.
    - Click *Create bucket*
     - Name: <identifier of your choosing>-multi-region-vpc-us
     - Block all public access.
     - Disable bucket versioning.
     - Default encryption.
     - Create bucket.
![Created S3 Buckets](/visual-guides/1.create-s3-buckets.png)

### Step 2: Create The CloudFormation Service Role and IAM Role

Create a YAML file called `cloudformation-service-role.yaml` that you will store in the EU bucket. 
  
  - Once the YAML file is created paste the below into the file.
```YAML 
AWSTemplateFormatVersion: '2010-09-09'
Description: IAM role for CloudFormation to assume when managing stacks.

Resources:
  CloudFormationServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: CloudFormationServiceRole-MultiRegion
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: cloudformation.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      Policies:
        - PolicyName: CloudFormationStackManagementPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'CloudFormation:*'
                  - 'iam:*'
                  - 'ec2:*'         
                  - 'iam:*'          
                  - 's3:*'           
                  - 'logs:*'       
                Resource: '*'


Outputs:
  CloudFormationServiceRoleArn:
    Description: The ARN of the CloudFormation Service Role.
    Value: !GetAtt CloudFormationServiceRole.Arn
    Export:
      Name: CloudFormationServiceRoleArn
```

  - Create another YAML file  called `iam-role-test-instances.yaml`, copy and paste the following code into that file and upload to a bucket of your choosing.

```YAML
AWSTemplateFormatVersion: "2010-09-09"
Description: Creating the instance role for EC2

Resources:

  IamInstanceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ec2-allow-ssm-cloudwatch-role
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action:
              - 'sts:AssumeRole'
      Path: /
      ManagedPolicyArns:
       - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
       - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
      
      Tags:
        - Key: Name
          Value: ec2-allow-ssm-cloudwatch
  
  IamInstanceRoleParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /app/iam/ec2-allow-ssm-cloudwatch-role-name 
      Type: String
      Value: !Ref IamInstanceRole 
      Description: IAM role name for EC2 instances allowing SSM and CloudWatch access.
```

  - Upload both of the files into a bucket of your choosing. 

![Create Roles](/visual-guides/2.create-roles.png)


### Step 3: Launch The CloudFormation & IAM Role

Head over to CloudFormation
  - Click on *Stacks*
  - Click *Create stack*
    - Choose an existing template
    - Enter the S3 URL for the `cloudformation-service-role.yaml` file. 
    - Select "next".
    - Name: `multi-region-vpc-cloudformation-service-role`
    - Leave all other settings as default.
    - Review and submit.
![CloudFormation Service Role Stack](/visual-guides/3.cloudformation-service-role-stack.png)

  - Next launch the IAM role stack.
  - Click on *Stacks*
  - Click *Create stack*
    - Choose an existing template
    - Enter the S3 URL for the `iam-role-test-instances.yaml` file. 
    - Select "next".
    - Name: `multi-region-iam-role-ec2`
    - Leave all other settings as default.
    - Review and submit.
![IAM Role Stack](/visual-guides/3.iam-role-ec2.png)

### Step 4: Create the VPC YAML Files.

Create a YAML file called `eu-west-2.yaml` that you will store in the EU Bucket. 
  
  - Once the YAML file is created paste the below into the file.

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the VPC for region 1

Resources:

  EuVpc:
    Type: "AWS::EC2::VPC"
    Properties:
      CidrBlock: 10.16.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true 
      Tags:
        - Key: Name
          Value: my-vpc-eu-west-2

  EuInternetGateway:
    Type: AWS::EC2::InternetGateway
    DependsOn: EuVpc
    Properties:
      Tags:
        - Key: Name
          Value: my-igw-eu-west-2

  AttachEuGateway: 
    Type: AWS::EC2::VPCGatewayAttachment
    DependsOn: 
        - EuInternetGateway
    Properties:
      InternetGatewayId: !Ref EuInternetGateway
      VpcId: !Ref EuVpc

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref EuVpc 
      AvailabilityZone: !Select [ 0, !GetAZs "" ]
      CidrBlock: 10.16.0.0/20
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: my-public-subnet-az1-eu-west-2
  
  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref EuVpc
      AvailabilityZone: !Select [1, !GetAZs "" ]
      CidrBlock: 10.16.16.0/20
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: my-public-subnet-az2-eu-west-2
    
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref EuVpc
      AvailabilityZone: !Select [0, !GetAZs "" ]
      MapPublicIpOnLaunch: false
      CidrBlock: 10.16.32.0/20
      Tags:
        - Key: Name
          Value: my-private-subnet-az1-eu-west-2

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties: 
      VpcId: !Ref EuVpc
      AvailabilityZone: !Select [1, !GetAZs "" ]
      MapPublicIpOnLaunch: false
      CidrBlock: 10.16.48.0/20
      Tags: 
        - Key: Name
          Value: my-private-subnet-az2-eu-west-2

  ElasticIp1:
    Type: AWS::EC2::EIP
    Properties: 
      Domain: vpc
      Tags:
        - Key: Name
          Value: eip1-Nat-Gateway-Subnet-1

  ElasticIp2:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      Tags: 
        - Key: Name
          Value: eip2-Nat-Gateway-Subnet-2
  
  NatGatewayAz1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIp1.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: eu-nat-gw-az1

  NatGatewayAz2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIp2.AllocationId
      SubnetId: !Ref PublicSubnet2
      Tags:
        - Key: Name
          Value: eu-nat-gw-az2

  PublicRouteTable:
    Type: AWS::EC2::RouteTable 
    Properties:
      VpcId: !Ref EuVpc 
      Tags:
        - Key: Name
          Value: my-public-rt-eu-west-2
  
  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref EuVpc
      Tags: 
        - Key: Name
          Value: eu-private-rt-az1-eu-west-2

  PrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref EuVpc
      Tags:
        - Key: Name
          Value: eu-private-rt-az2-eu-west-2
  
  PublicRtAssociations1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1
  
  PublicRtAssociations2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  PrivateRtAssociations1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet1

  PrivateRtAssociations2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      SubnetId: !Ref PrivateSubnet2

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachEuGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref EuInternetGateway
  
  PrivateRoute1External:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayAz1

  PrivateRoute2External:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayAz2
  
Outputs: 

  EuVpcId:
    Description: The VPC for the EU Region
    Value: !Ref EuVpc
      
  PublicSubnet1Id:
    Description: The first public subnet for the EU VPC
    Value: !Ref PublicSubnet1
    
  
  PublicSubnet2Id:
    Description: The second public subnet for the EU VPC
    Value: !Ref PublicSubnet2
    
    
  PrivateSubnet1Id:
    Description: The first private subnet for the EU VPC
    Value: !Ref PrivateSubnet1
   
    
  PrivateSubnet2Id:
    Description: The second private subnet for the EU VPC
    Value: !Ref PrivateSubnet2
    
  PublicRouteTableId:
    Description: The public route table for the eu VPC
    Value: !Ref PublicRouteTable
    
  
  PrivateRouteTable1Id:
    Description: The first private route table for the EU VPC
    Value: !Ref PrivateRouteTable1
    
    
  PrivateRouteTable2Id:
    Description: The second private route table for the EU VPC
    Value: !Ref PrivateRouteTable2
    
```

  - Create another YAML file  called `us-vpc.yaml`, copy and paste the following code into that file and upload to the US bucket. 

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the VPC for region 2

Resources:

  UsVpc:
    Type: "AWS::EC2::VPC"
    Properties:
      CidrBlock: 10.32.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true 
      Tags:
        - Key: Name
          Value: my-vpc-us-east-1 

  UsInternetGateway:
    Type: AWS::EC2::InternetGateway
    DependsOn: UsVpc
    Properties:
      Tags:
        - Key: Name
          Value: my-igw-us-east-1

  AttachUsGateway: 
    Type: AWS::EC2::VPCGatewayAttachment
    DependsOn: 
        - UsInternetGateway
    Properties:
      InternetGatewayId: !Ref UsInternetGateway
      VpcId: !Ref UsVpc

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref UsVpc 
      AvailabilityZone: !Select [ 0, !GetAZs "" ]
      CidrBlock: 10.32.0.0/20
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: my-public-subnet-az1-us-east-1
  
  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref UsVpc
      AvailabilityZone: !Select [1, !GetAZs "" ]
      CidrBlock: 10.32.16.0/20
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: my-public-subnet-az2-us-east-1
    
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref UsVpc
      AvailabilityZone: !Select [0, !GetAZs "" ]
      MapPublicIpOnLaunch: false
      CidrBlock: 10.32.32.0/20
      Tags:
        - Key: Name
          Value: my-private-subnet-az1-us-east-1

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties: 
      VpcId: !Ref UsVpc
      AvailabilityZone: !Select [1, !GetAZs "" ]
      MapPublicIpOnLaunch: false
      CidrBlock: 10.32.48.0/20
      Tags: 
        - Key: Name
          Value: my-private-subnet-az2-us-east-1

  ElasticIp1:
    Type: AWS::EC2::EIP
    Properties: 
      Domain: vpc
      Tags:
        - Key: Name
          Value: eip1-Nat-Gateway-Subnet-1

  ElasticIp2:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      Tags: 
        - Key: Name
          Value: eip2-Nat-Gateway-Subnet-2
  
  NatGatewayAz1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIp1.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: us-nat-gw-az1

  NatGatewayAz2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt ElasticIp2.AllocationId
      SubnetId: !Ref PublicSubnet2
      Tags:
        - Key: Name
          Value: us-nat-gw-az2

  PublicRouteTable:
    Type: AWS::EC2::RouteTable 
    Properties:
      VpcId: !Ref UsVpc 
      Tags:
        - Key: Name
          Value: my-public-rt-us-east-1
  
  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref UsVpc
      Tags: 
        - Key: Name
          Value: us-private-rt-az1-us-east-1

  PrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref UsVpc
      Tags:
        - Key: Name
          Value: us-private-rt-az2-us-east-1
  
  PublicRtAssociations1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1
  
  PublicRtAssociations2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  PrivateRtAssociations1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet1

  PrivateRtAssociations2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      SubnetId: !Ref PrivateSubnet2

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachUsGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref UsInternetGateway
  
  PrivateRoute1External:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayAz1

  PrivateRoute2External:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayAz2
  
Outputs: 

  UsVpcId:
    Description: The VPC for the US Region
    Value: !Ref UsVpc
    Export:
      Name: us-east-1-vpc
      
  PublicSubnet1Id:
    Description: The first public subnet for the US VPC
    Value: !Ref PublicSubnet1
    Export:
      Name: us-east-1-vpc-public-subnet-1
  
  PublicSubnet2Id:
    Description: The second public subnet for the US VPC
    Value: !Ref PublicSubnet2
    Export:
      Name: us-east-1-vpc-public-subnet-2
    
  PrivateSubnet1Id:
    Description: The first private subnet for the US VPC
    Value: !Ref PrivateSubnet1
    Export:
      Name: us-east-1-vpc-private-subnet-1
    
  PrivateSubnet2Id:
    Description: The second private subnet for the US VPC
    Value: !Ref PrivateSubnet2
    Export:
      Name: us-east-1-vpc-private-subnet-2
    
  PublicRouteTableId:
    Description: The public route table for the US VPC
    Value: !Ref PublicRouteTable
    Export:
      Name: us-east-1-vpc-public-rt
  
  PrivateRouteTable1Id:
    Description: The first private route table for the US VPC
    Value: !Ref PrivateRouteTable1
    
  PrivateRouteTable2Id:
    Description: The second private route table for the US VPC
    Value: !Ref PrivateRouteTable2
```

### Step 4: Create the YAML Files For The Transit Gateway 


Create a YAML file called `eu-tgw.yaml` that you will store in the EU bucket. 
  
  - Once the YAML file is created paste the below into the file.
```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the tgw for region 1

Parameters:
  
  EuVpcId:
    Type: String
    Description: The EU VPC ID
  PrivateSubnet1Id:
    Type: String
    Description: The first private subnet ID
  PrivateSubnet2Id:
    Type: String 
    Description: The second private subnet ID
  PrivateRouteTable1Id:
    Type: String
    Description: The route table for the first private subnet
  PrivateRouteTable2Id:
    Type: String
    Description: The Route table for the second private subnet

Resources:

  EuTgw:
    Type: AWS::EC2::TransitGateway
    Properties:
      DefaultRouteTableAssociation: disable
      DefaultRouteTablePropagation: disable
      Tags:
      - Key: Name
        Value: eu-tgw

  EuTgwRt:
    Type: AWS::EC2::TransitGatewayRouteTable
    Properties:
      Tags: 
      - Key: Name
        Value: eu-tgw-rt
      TransitGatewayId: !Ref EuTgw

  EuTgwAttVpc:
    Type: AWS::EC2::TransitGatewayAttachment
    Properties:
      SubnetIds: 
      - !Ref PrivateSubnet1Id
      - !Ref PrivateSubnet2Id
      TransitGatewayId: !Ref EuTgw 
      VpcId: !Ref EuVpcId
      Tags:
      - Key: Name
        Value: eu-tgw-att-vpc
  
  EuTgwVpcAttachmentAssociation:
    Type: AWS::EC2::TransitGatewayRouteTableAssociation
    DependsOn:
      - EuTgwAttVpc
      - EuTgwRt
    Properties:
      TransitGatewayAttachmentId: !Ref EuTgwAttVpc
      TransitGatewayRouteTableId: !Ref EuTgwRt

  EUTgwVpcRoute:
    Type: AWS::EC2::TransitGatewayRoute
    DependsOn: EuTgwVpcAttachmentAssociation
    Properties:
      DestinationCidrBlock: 10.16.0.0/16
      TransitGatewayAttachmentId: !Ref EuTgwAttVpc
      TransitGatewayRouteTableId: !Ref EuTgwRt

  PrivateSubnet1RouteToVpc:
    Type: AWS::EC2::Route
    DependsOn: EuTgwVpcAttachmentAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1Id
      DestinationCidrBlock: 10.32.0.0/16
      TransitGatewayId: !Ref EuTgw

  PrivateSubnet2RouteToVpc:
    Type: AWS::EC2::Route
    DependsOn: EuTgwVpcAttachmentAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable2Id
      DestinationCidrBlock: 10.32.0.0/16
      TransitGatewayId: !Ref EuTgw 
  
Outputs: 

  EuTgwId:
    Description: The transit gateway for EU region
    Value: !Ref EuTgw
    Export:
      Name: EuTgwId
  
  EuTgwRtId:
    Description: The transit gateway route table for EU region
    Value: !Ref EuTgwRt
    Export:
      Name: EuTgwRtId

  EuTgwAttVpcId:
    Description: Transit Gateway VPC attachment
    Value: !Ref EuTgwAttVpc
```

  - Create another YAML file  called `us-tgw.yaml`, copy and paste the following code into that file and upload to the US bucket. 

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the tgw for region 2

Parameters:
  
  UsVpcId:
    Type: String
    Description: The ID for the US VPC
  PrivateSubnet1Id:
    Type: String
    Description: The ID for the first private subnet
  PrivateSubnet2Id:
    Type: String
    Description: The ID for the second private subnet
  PrivateRouteTable1Id:
    Type: String
    Description: The route table for the first private subnet
  PrivateRouteTable2Id:
    Type: String
    Description: The Route table for the second private subnet

Resources:

  UsTgw:
    Type: AWS::EC2::TransitGateway
    Properties:
      DefaultRouteTableAssociation: disable
      DefaultRouteTablePropagation: disable
      Tags:
      - Key: Name
        Value: us-tgw

  UsTgwRt:
    Type: AWS::EC2::TransitGatewayRouteTable
    Properties:
      Tags: 
      - Key: Name
        Value: us-tgw-rt
      TransitGatewayId: !Ref UsTgw

  UsTgwAttVpc:
    Type: AWS::EC2::TransitGatewayAttachment
    Properties:
      SubnetIds: 
      - !Ref PrivateSubnet1Id
      - !Ref PrivateSubnet2Id
      TransitGatewayId: !Ref UsTgw 
      VpcId: !Ref UsVpcId
      Tags:
      - Key: Name
        Value: us-tgw-att-vpc
  
  UsTgwVpcAttachmentAssociation:
    Type: AWS::EC2::TransitGatewayRouteTableAssociation
    DependsOn: 
      - UsTgwAttVpc
      - UsTgwRt
    Properties:
      TransitGatewayAttachmentId: !Ref UsTgwAttVpc
      TransitGatewayRouteTableId: !Ref UsTgwRt

  UsTgwVpcRoute:
    Type: AWS::EC2::TransitGatewayRoute
    DependsOn: UsTgwVpcAttachmentAssociation
    Properties:
      DestinationCidrBlock: 10.32.0.0/16
      TransitGatewayAttachmentId: !Ref UsTgwAttVpc
      TransitGatewayRouteTableId: !Ref UsTgwRt
  
  PrivateSubnet1RouteToVpc:
    Type: AWS::EC2::Route
    DependsOn: UsTgwAttVpc
    Properties:
      RouteTableId: !Ref PrivateRouteTable1Id
      DestinationCidrBlock: 10.16.0.0/16
      TransitGatewayId: !Ref UsTgw

  PrivateSubnet2RouteToVpc:
    Type: AWS::EC2::Route
    DependsOn: UsTgwAttVpc
    Properties:
      RouteTableId: !Ref PrivateRouteTable2Id
      DestinationCidrBlock: 10.16.0.0/16
      TransitGatewayId: !Ref UsTgw
  
Outputs: 

  UsTgwId:
    Description: The transit gateway for US region
    Value: !Ref UsTgw
    Export:
      Name: UsTgwId
  
  UsTgwRtId:
    Description: The transit gateway route table for US region
    Value: !Ref UsTgwRt
    Export:
      Name: UsTgwRtId
```

### Step 5: Create The YAML Files For The Security Groups

Create a YAML file called `eu-security-groups.yaml` that you will store in the EU bucket. 
  
  - Once the YAML file is created paste the below into the file.

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the security group for region 1

Parameters:
  EuVpcId:
    Type: String
    Description: The EU VPC ID

Resources:

  EuEC2TestSg:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group that controls EC2 test instances
      VpcId: !Ref EuVpcId
      SecurityGroupIngress: 
        - IpProtocol: -1
          CidrIp: 10.32.0.0/16
        - IpProtocol: -1
          CidrIp: 10.16.0.0/16

      Tags:
        -  Key: Name
           Value: eu-west-2-ec2-test-sg

Outputs: 
  EuEC2TestSgId:
    Description: Security group for the EU test instances.
    Value: !Ref EuEC2TestSg
```

  - Create another YAML file  called `us-security-groups.yaml`, copy and paste the following code into that file and upload to the US bucket. 

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the security group for region 2

Parameters:
  UsVpcId:
    Type: String
    Description: The ID for the US VPC

Resources:

  UsEC2TestSg:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group that controls EC2 test instances
      VpcId: !Ref UsVpcId
      SecurityGroupIngress: 
        - IpProtocol: -1
          CidrIp: 10.16.0.0/16
        - IpProtocol: -1
          CidrIp: 10.32.0.0/16

Outputs: 
  UsEC2TestSgId:
    Description: Security group for the US test instances.
    Value: !Ref UsEC2TestSg
```

### Step 6: Create The YAML Files For the EC2 Instances

Create a YAML file called `eu-ec2.yaml` that you will store in the EU bucket. 
  
  - Once the YAML file is created paste the below into the file.

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the EC2 instances for region 1


Parameters:
  EuVpcId:
    Type: String
    Description: The EU VPC ID
  PublicSubnet1Id:
    Type: String
    Description: The public subnet 1 ID
  PublicSubnet2Id:
    Type: String
    Description: The public subnet 2 ID
  PrivateSubnet1Id:
    Type: String
    Description: The private subnet 1 ID
  PrivateSubnet2Id:
    Type: String
    Description: The private subnet 2 ID
  EuEC2TestSgId:
    Type: String
    Description: The security group ID

Resources:

  MyEc2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - ec2-allow-ssm-cloudwatch-role
      InstanceProfileName: !Sub 'ec2-allow-ssm-cloudwatch-role-${AWS::Region}'

  InstancePublic1:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-00fefe800bd08f8da
      InstanceType: t2.micro
      SubnetId: !Ref PublicSubnet1Id
      SecurityGroupIds:
        - !Ref EuEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile 
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: EU-Public-Test-Instance-AZ1

  InstancePublic2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-00fefe800bd08f8da
      InstanceType: t2.micro
      SubnetId: !Ref PublicSubnet2Id
      SecurityGroupIds:
        - !Ref EuEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: EU-Public-Test-Instance-AZ2
  
  InstancePrivate1:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-00fefe800bd08f8da
      InstanceType: t2.micro
      SubnetId: !Ref PrivateSubnet1Id
      SecurityGroupIds:
        - !Ref EuEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: EU-Private-Test-Instance-AZ1

  InstancePrivate2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-00fefe800bd08f8da
      InstanceType: t2.micro
      SubnetId: !Ref PrivateSubnet2Id
      SecurityGroupIds:
        - !Ref EuEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile 
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: EU-Private-Test-Instance-AZ2
```

  - Create another YAML file  called `us-ec2.yaml`, copy and paste the following code into that file and upload to the US bucket. 

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the EC2 instances for region 2


Parameters:
  UsVpcId:
    Type: String
    Description: The VPC ID for US 
  PublicSubnet1Id:
    Type: String
    Description: The ID for the first public subnet
  PublicSubnet2Id:
    Type: String
    Description: The ID for the second public subnet
  PrivateSubnet1Id:
    Type: String
    Description: The ID for the first private subnet
  PrivateSubnet2Id:
    Type: String
    Description: The ID for the second private subnet
  UsEC2TestSgId:
    Type: String
    Description: The ID for the security group

Resources:

  MyEc2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - ! ec2-allow-ssm-cloudwatch-role
      InstanceProfileName: !Sub 'ec2-allow-ssm-cloudwatch-role-${AWS::Region}'

  InstancePublic1:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-08a6efd148b1f7504
      InstanceType: t2.micro
      SubnetId: !Ref PublicSubnet1Id
      SecurityGroupIds:
        - !Ref UsEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: us-Public-Test-Instance-AZ1

  InstancePublic2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-08a6efd148b1f7504
      InstanceType: t2.micro
      SubnetId: !Ref PublicSubnet2Id
      SecurityGroupIds:
        - !Ref UsEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: us-Public-Test-Instance-AZ2
  
  InstancePrivate1:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-08a6efd148b1f7504
      InstanceType: t2.micro
      SubnetId: !Ref PrivateSubnet1Id
      SecurityGroupIds:
        - !Ref UsEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: us-Private-Test-Instance-AZ1

  InstancePrivate2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-08a6efd148b1f7504
      InstanceType: t2.micro
      SubnetId: !Ref PrivateSubnet2Id
      SecurityGroupIds:
        - !Ref UsEC2TestSgId
      IamInstanceProfile: !Ref MyEc2InstanceProfile
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
      Tags:
      - Key: Name
        Value: us-Private-Test-Instance-AZ2
```

### Step 7: Create The Main YAML Files & Upload To S3

Create a YAML file called `eu-main.yaml` that you will store in the EU bucket. 
  
  - Once the YAML file is created paste the below into the file.

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Main YAML template for the creation of a multi-region VPC eu-west-2

Resources:
 
  EuWest2VpcStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-eu.s3.eu-west-2.amazonaws.com/eu-vpc.yaml"
  
  EuWest2TgwStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: EuWest2VpcStack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-eu.s3.eu-west-2.amazonaws.com/eu.tgw.yaml"
      Parameters:
        EuVpcId: !GetAtt EuWest2VpcStack.Outputs.EuVpcId
        PrivateSubnet1Id: !GetAtt EuWest2VpcStack.Outputs.PrivateSubnet1Id
        PrivateSubnet2Id: !GetAtt EuWest2VpcStack.Outputs.PrivateSubnet2Id
        PrivateRouteTable1Id: !GetAtt EuWest2VpcStack.Outputs.PrivateRouteTable1Id 
        PrivateRouteTable2Id: !GetAtt EuWest2VpcStack.Outputs.PrivateRouteTable2Id

  EuWest2SecurityGroupStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: EuWest2VpcStack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-eu.s3.eu-west-2.amazonaws.com/eu-security-groups.yaml"
      Parameters:
        EuVpcId: !GetAtt EuWest2VpcStack.Outputs.EuVpcId
  
  EuWest2Ec2Stack:
    Type: AWS::CloudFormation::Stack
    DependsOn: 
      - EuWest2VpcStack 
      - EuWest2SecurityGroupStack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-eu.s3.eu-west-2.amazonaws.com/eu-ec2.yaml"
      Parameters: 
        EuVpcId: !GetAtt EuWest2VpcStack.Outputs.EuVpcId
        PublicSubnet1Id: !GetAtt EuWest2VpcStack.Outputs.PublicSubnet1Id
        PublicSubnet2Id: !GetAtt EuWest2VpcStack.Outputs.PublicSubnet2Id
        PrivateSubnet1Id: !GetAtt EuWest2VpcStack.Outputs.PrivateSubnet1Id
        PrivateSubnet2Id: !GetAtt EuWest2VpcStack.Outputs.PrivateSubnet2Id
        EuEC2TestSgId: !GetAtt EuWest2SecurityGroupStack.Outputs.EuEC2TestSgId
```

  - Create another YAML file  called `us-main.yaml`, copy and paste the following code into that file and upload to the US bucket. 

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Main YAML template for the creation of a multi-region VPC us-east-1

Resources:
 
  UsEast1VpcStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-us.s3.us-east-1.amazonaws.com/us-vpc.yaml"

  
  UsEast1TgwStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: UsEast1VpcStack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-us.s3.us-east-1.amazonaws.com/us-tgw.yaml"
      Parameters:
        UsVpcId: !GetAtt UsEast1VpcStack.Outputs.UsVpcId
        PrivateSubnet1Id: !GetAtt UsEast1VpcStack.Outputs.PrivateSubnet1Id
        PrivateSubnet2Id: !GetAtt UsEast1VpcStack.Outputs.PrivateSubnet2Id
        PrivateRouteTable1Id: !GetAtt UsEast1VpcStack.Outputs.PrivateRouteTable1Id
        PrivateRouteTable2Id: !GetAtt UsEast1VpcStack.Outputs.PrivateRouteTable2Id

  UsEast1SecurityGroupStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: UsEast1VpcStack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-us.s3.us-east-1.amazonaws.com/us-security-groups.yaml"
      Parameters:
        UsVpcId: !GetAtt UsEast1VpcStack.Outputs.UsVpcId
  
  UsEast1Ec2Stack:
    Type: AWS::CloudFormation::Stack
    DependsOn: 
      - UsEast1VpcStack 
      - UsEast1SecurityGroupStack
    Properties:
      TemplateURL: !Sub "https://tjt-multi-region-vpc-us.s3.us-east-1.amazonaws.com/us-ec2.yaml"
      Parameters: 
        UsVpcId: !GetAtt UsEast1VpcStack.Outputs.UsVpcId
        PublicSubnet1Id: !GetAtt UsEast1VpcStack.Outputs.PublicSubnet1Id
        PublicSubnet2Id: !GetAtt UsEast1VpcStack.Outputs.PublicSubnet2Id
        PrivateSubnet1Id: !GetAtt UsEast1VpcStack.Outputs.PrivateSubnet1Id
        PrivateSubnet2Id: !GetAtt UsEast1VpcStack.Outputs.PrivateSubnet2Id
        UsEC2TestSgId: !GetAtt UsEast1SecurityGroupStack.Outputs.UsEC2TestSgId
```

  - Once you have uploaded these.
  - Head over to CloudFormation on the us-east-1 console.
  - Click on *Stacks*
  - Click *Create stack*
    - Choose an existing template
    - Enter the S3 URL for the `us-main.yaml` file. 
    - Select "next".
    - Name: `us-multi-region-vpc-tgw`
    - Leave all other settings as default.
    - For the IAM role - Select CloudFormationServiceRole-MultiRegion
    - Acknowledge the warnings.
    - Review and submit.

![US Created Stack](/visual-guides/7.create-us-stack.png)


  - Once you have uploaded these.
  - Head over to CloudFormation on the eu-west-2 console.
  - Click on *Stacks*
  - Click *Create stack*
    - Choose an existing template
    - Enter the S3 URL for the `eu-main.yaml` file. 
    - Select "next".
    - Name: `eu-multi-region-vpc-tgw`
    - Leave all other settings as default.
    - For the IAM role - Select CloudFormationServiceRole-MultiRegion
    - Acknowledge the warnings.
    - Review and submit.

![EU Created Stack](/visual-guides/7.create-eu-stack.png)


### Step 8: Create The Peering Attachment Files, Upload and Accept The Attachment

Create a YAML file called `peering.yaml` that you will store in the EU bucket. 
  
  - Once the YAML file is created paste the below into the file. *Make sure you replace the transit gateway ID and account number with your own account values*

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Peering attachment for EU TransitGateway

Resources:

  TgwAttPeering:
    Type: AWS::EC2::TransitGatewayPeeringAttachment
    Properties:
      PeerTransitGatewayId: <enter US transit gateway ID>
      PeerAccountId: "<Enter AWS Account number>"
      PeerRegion: us-east-1
      TransitGatewayId: !ImportValue EuTgwId
      Tags:
        - Key: Name
          Value: eu-us-tgw-peering-attachment

Outputs:
  TgwPeeringId:
    Description: The tgw peering attachment ID
    Value: !Ref TgwAttPeering
    Export:
      Name: eu-us-tgw-peering
```

  - Head over to CloudFormation on the eu-west-2 console.
  - Click on *Stacks*
  - Click *Create stack*
    - Choose an existing template
    - Enter the S3 URL for the `peering.yaml` file. 
    - Select "next".
    - Name: `eu-us-multi-region-peering`
    - Leave all other settings as default.
    - For the IAM role - Select CloudFormationServiceRole-MultiRegion
    - Acknowledge the warnings.
    - Review and submit.

![Created Peering Stack](/visual-guides/8.created-peering-stack.png)

  - Once created go onto the us-east-1 VPC console
    - Select Transit gateway attachments.
    - Select the peering attachment that is pending acceptance.
    - Right click and Accept peering connection.
    - Make a note of the Transit gateway attachment ID

![Accept Peering Connection](/visual-guides/8.accept-peering.png)

### Step 9: Create the Peering Routes And Associations 

Create a YAML file called `eu-peering-routes.yaml` that you will store in the EU bucket. 
  
  - Once the YAML file is created paste the below into the file.

``` YAML

AWSTemplateFormatVersion: '2010-09-09'
Description: Create the EU peering route

Resources: 

  EuTgwVpcAttachmentAssociation:
    Type: AWS::EC2::TransitGatewayRouteTableAssociation
    Properties:
      TransitGatewayAttachmentId: !ImportValue eu-us-tgw-peering
      TransitGatewayRouteTableId: !ImportValue EuTgwRtId

  TgwPeeringRoute:
    Type: AWS::EC2::TransitGatewayRoute
    Properties:
      DestinationCidrBlock: 10.32.0.0/16
      TransitGatewayAttachmentId: !ImportValue eu-us-tgw-peering
      TransitGatewayRouteTableId: !ImportValue EuTgwRtId
```

  - Head over to CloudFormation on the eu-west-2 console.
  - Click on *Stacks*
  - Click *Create stack*
    - Choose an existing template
    - Enter the S3 URL for the `eu-peering-route.yaml` file. 
    - Select "next".
    - Name: `eu-us-multi-region-peering-route`
    - Leave all other settings as default.
    - For the IAM role - Select CloudFormationServiceRole-MultiRegion
    - Acknowledge the warnings.
    - Review and submit.

![EU Peering Route](/visual-guides/9.created-peering-routes-eu.png)

  - Create another YAML file  called `us-peering-routes.yaml`, copy and paste the following code into that file and upload to the US bucket. 

```YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Creating the peering route for US side. 

Resources: 

  EuTgwVpcAttachmentAssociation:
    Type: AWS::EC2::TransitGatewayRouteTableAssociation
    Properties:
      TransitGatewayAttachmentId: tgw-attach-0432eb5c60018cf59
      TransitGatewayRouteTableId: !ImportValue UsTgwRtId

  TgwPeeringRoute:
    Type: AWS::EC2::TransitGatewayRoute
    Properties:
      DestinationCidrBlock: 10.16.0.0/16
      TransitGatewayAttachmentId: tgw-attach-0432eb5c60018cf59
      TransitGatewayRouteTableId: !ImportValue UsTgwRtId
```
  - Head over to CloudFormation on the us-east-1 console.
  - Click on *Stacks*
  - Click *Create stack*
    - Choose an existing template
    - Enter the S3 URL for the `us-peering-routes.yaml` file. 
    - Select "next".
    - Name: `us-eu-multi-region-peering-route`
    - Leave all other settings as default.
    - For the IAM role - Select CloudFormationServiceRole-MultiRegion
    - Acknowledge the warnings.
    - Review and submit.

![US Peering Route](/visual-guides/9.created-peering-routes-us.png)


### Step 9: Test Connectivity 

Test connectivity to outside and across regions.

On either of the private instances from either region.

Head to the EC2 Console us-east-1.
  - Select Instances.
  - Select us-Private-Test-Instance-AZ1 and get it's private IP.
  - Select us-Private-Test-Instance-AZ2 and get it's private IP.

Open a new tab and head to EC2 Console eu-west-2
  - Select Instances.
  - Select eu-Private-Test-Instance-AZ1 and get it's private IP.
  - Select eu-Private-Test-Instance-AZ2 and get it's private IP.

Connect to eu-Private-Test-Instance-AZ1.
  - Select the instance and press connect.
  - Select session manager and connect.
  - Enter the below commands
    - `bash`.
    - `ping <ip of eu-Private-Test-Instance-AZ2>`
    - `ping <ip of us-Private-Test-Instance-AZ1>`
    - `ping <ip of us-Private-Test-Instance-AZ2>`
    - `ping 1.1.1.1`

 - If successful these will prove inter-region connectivity as well as connectivity to the outside world.

![Successful Pings EU](/visual-guides/9eu-west-2-test.png)


Connect to us-Private-Test-Instance-AZ1.
  - Select the instance and press connect.
  - Select session manager and connect.
  - Enter the below commands
    - `bash`.
    - `ping <ip of us-Private-Test-Instance-AZ2>`
    - `ping <ip of eu-Private-Test-Instance-AZ1>`
    - `ping <ip of eu-Private-Test-Instance-AZ2>`
    - `ping 1.1.1.1`

 - If successful these will prove inter-region connectivity as well as connectivity to the outside world.

![Successful Pings US](/visual-guides/9.us-east-1-test.png)