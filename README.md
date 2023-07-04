閲覧頂きありがとうございます！

Cloud Formationを使用して、EC2インスタンス構築のコード化をしようとしております。

対象のymlファイルをアップロードしスタックの作成を実施した所、うまく行かず

エラーが出てしまいます。
128〜131行目のセキュリティグループ指定箇所をコメントアウトすると、正常に作成完了。
EC2インスタンスは正常に起動し、セキュリティグループも問題なく作成できています。
セキュリティグループはデフォルトが指定されている状態です。
EC2インスタンスに紐付けようとするとエラーが出てしまうのは何故なのでしょうか。。。。


別途ymlファイルもアップロードしておりますが、下記にも記載いたします。

以下記述内容--------------------------------

AWSTemplateFormatVersion: "2010-09-09"
Description: "Create BaseNetwork"
#パラメータの指定
Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      -
        Parameters:
          - SystemName
          - EnvType
Parameters:
  SystemName:
    Description: Please type the SystemName.
    Type: String
    Default: Sample
  EnvType:
    Description: Select Environment Type.
    Type: String
    Default: prd
    AllowedValues:
      - stg
      - prd
  CidrBlock:
    Description: Please type the CiderBlock.
    Type: String
    Default: 192.168.0.0/16
  KeyName:
    Description: The EC2 Key Pair to allow SSH access to the instance
    Type: 'AWS::EC2::KeyPair::KeyName'
  Ec2InstanceType:
    Type: String
    Default: t2.micro
  Ec2AMI:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
#VPCの作成
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: !Sub ${CidrBlock}
      Tags:
        - Key: Name
          Value: !Sub ${SystemName}-${EnvType}-vpc

#Subnetの作成
  Sub:
    Type: 'AWS::EC2::Subnet'
    Properties:
      AvailabilityZone: ap-northeast-1a
      VpcId: !Ref VPC
      CidrBlock: !Select [ 0, !Cidr [ !GetAtt VPC.CidrBlock, 1, 8 ]]
      Tags:
        - Key: Name
          Value: !Sub ${SystemName}-${EnvType}-subnet
#IGWの作成
  IGW:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
        - Key: Name
          Value: igw-cf

#IGWをVPCにアタッチ
  AttachGateway:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

#ルートテーブルの作成
  SubRT:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: sub-rt-clf

#Sub-インターネット間のルーティング
  SubToInternet:
    Type: "AWS::EC2::Route"
    Properties:
      RouteTableId: !Ref SubRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW

#ルートテーブルをサブネットに関連付け
  AssSubRT:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref Sub
      RouteTableId: !Ref SubRT

#セキュリティグループの作成
  SG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: SSH and HTTP access any
      GroupDescription: SH and HTTP access any
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: test-clf-sg

#EC2の作成
  Ec2Instance:
    Type: 'AWS::EC2::Instance'
    DependsOn: SG
    Properties:
      ImageId: !Ref Ec2AMI
      InstanceType: !Ref Ec2InstanceType
      KeyName: !Ref KeyName
      NetworkInterfaces:
        - AssociatePublicIpAddress: "true"
          DeviceIndex: "0"
          SubnetId: !Ref Sub
#セキュリティグループを指定するとエラーが出るのでコメントアウト....何故？
#      SecurityGroupIds:
#        - default
#        - !Ref SG
      Tags:
        - Key: Name
          Value: Sample-EC2-cl

