FULL DOCS :
https://kops.sigs.k8s.io/getting_started/aws/


create a user and access key in aws then login in terminal using command :

aws configure

check it using :

aws configure list


for ssh to vm using :
ssh -i "gonewaje-locals.pem" ubuntu@ec2-13-229-99-198.ap-southeast-1.compute.amazonaws.com




then run below command to create group, user and attach policy :


aws iam create-group --group-name kops-group


aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops-group
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops-group
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops-group
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops-group
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops-group
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess --group-name kops-group
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEventBridgeFullAccess --group-name kops-group

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops-group

aws iam create-access-key --user-name kops



output json :

{
    "User": {
        "Path": "/",
        "UserName": "kops",
        "UserId": "AIDAQLVQQ3HCGDWB2A3IS",
        "Arn": "arn:aws:iam::025066265028:user/kops",
        "CreateDate": "2024-08-10T06:59:13+00:00"
    }
}


for DNS we use scenario 3

Scenario 3: Subdomain for clusters in route53, leaving the domain at another registrar


ID=$(uuidgen) && aws route53 create-hosted-zone --name kops.gonewaje.tech --caller-reference $ID | jq .DelegationSet.NameServers



output :
[
  "ns-47.awsdns-05.com",
  "ns-928.awsdns-52.net",
  "ns-1431.awsdns-50.org",
  "ns-1853.awsdns-39.co.uk"
]


ns1.dns-parking.com

ns2.dns-parking.com



create a s3

aws s3api create-bucket \
    --bucket kops-visi \
    --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1

output :

{
    "Location": "http://kops-gone.s3.amazonaws.com/"
}



versioning a bucket


aws s3api put-bucket-versioning --bucket kops-visi  --versioning-configuration Status=Enabled


if during create a cluster or generate yaml got an error access to s3, can add bucketgetlocation policy using this docs :

https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-policy-language-overview.html?icmpid=docs_amazons3_console
https://awspolicygen.s3.amazonaws.com/policygen.html

{
    "Version": "2012-10-17",
    "Id": "Policy1723276652844",
    "Statement": [
        {
            "Sid": "Stmt1723276650801",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::536697269508:user/kops"
            },
            "Action": "s3:GetBucketLocation",
            "Resource": "arn:aws:s3:::kops-gone"
        }
    ]
}


{
  "Id": "Policy1723626240456",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1723626238016",
      "Action": [
        "s3:GetBucketLocation"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::kops-visi",
      "Principal": {
        "AWS": [
          "arn:aws:iam::536697269508:user/kops"
        ]
      }
    }
  ]
}



create bucket for OIDC :



aws s3api create-bucket \
    --bucket kops-visi-oidc \
    --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1 \
    --object-ownership BucketOwnerPreferred
aws s3api put-public-access-block \
    --bucket kops-visi-oidc \
    --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
aws s3api put-bucket-acl \
    --bucket kops-visi-oidc \
    --acl public-read


generate yaml file :



some option in above command is depreceated, below is modified especially increase kubernetes version [please note if its for private cluster]
we also can use gossip based cluster like this :

export NAME=myfirstcluster.k8s.local
export KOPS_STATE_STORE=s3://prefix-example-com-state-store

or maybe a cluster based using dns/route53

export NAME=kops.gonewaje.tech
export KOPS_STATE_STORE=s3://kops-visi
kops create cluster $NAME \
    --zones "ap-southeast-1a,ap-southeast-1b,ap-southeast-1c" \
    --control-plane-zones "ap-southeast-1a,ap-southeast-1b,ap-southeast-1c" \
    --networking calico \
    --topology private \
    --bastion \
    --cloud=aws \
    --node-count 3 \
    --node-size t2.micro \
    --kubernetes-version v1.24.17 \
    --control-plane-size t2.micro \
    --network-id vpc-039f844812a9d098c \
    --dry-run \
    -o yaml > $NAME.yaml


for public cluster :


export NAME=kops.gonewaje.tech
export KOPS_STATE_STORE=s3://kops-visi
kops create cluster $NAME \
    --zones "ap-southeast-1a,ap-southeast-1b,ap-southeast-1c" \
    --control-plane-zones "ap-southeast-1a,ap-southeast-1b,ap-southeast-1c" \
    --networking calico \
    --topology private \
    --bastion \
    --cloud=aws \
    --node-count 3 \
    --node-size t2.medium \
    --kubernetes-version v1.24.17 \
    --control-plane-size t2.medium \
    --network-id vpc-0d0cd0b6996025a19 \
    --dry-run \
    -o yaml > $NAME.yaml


before execute this command make sure to :
- create new vpc with CIDR 10.0.0.0/16
- create new internet gateway that attach to your new vpc id


kops create -f $NAME.yaml
kops create secret --name $NAME sshpublickey admin -i ~/.ssh/id_rsa.pub
kops update cluster $NAME --yes --admin
kops rolling-update cluster $NAME --yes
kops delete cluster --name $NAME --yes


kops create cluster \
    --name=${NAME} \
    --cloud=aws \
    --zones "ap-southeast-1a,ap-southeast-1b,ap-southeast-1c" \
    --node-size t2.micro \
    --node-count 3 \
    --control-plane-size t2.micro \
    --control-plane-zones "ap-southeast-1a,ap-southeast-1b,ap-southeast-1c"
    --ssh-public-key gonewaje-locals


if you want to ssh to your bastion, using ssh command like this :
ssh ubuntu@{{LOAD-BALANCER-DNS-NAME}}


to check the nodes better in iterm terminal instead of vscode terminal, it will gain an error


kubectl expose deployment ui-deployment --port=80 --type=LoadBalancer -n visi-ns




{
    "AccessKey": {
        "UserName": "gonewaje",
        "AccessKeyId": "AKIAQLVQQ3HCK5LRUCHK",
        "Status": "Active",
        "SecretAccessKey": "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",
        "CreateDate": "2024-08-12T09:28:47+00:00"
    }
}
