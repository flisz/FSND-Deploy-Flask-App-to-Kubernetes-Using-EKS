## Prerequisite: 

1) install AWSCLI on your system: `pip install awscli --upgrade`
2) generate an AWS Access Key ID and Secret access key for an AWS IAM user. 
  1) Use the AWS website to set up new user.
  2) Ensure new user has 'AdministratorAccess' permission set.
3) Set up your environment with keys generated above. 
  * to view default/current settings: `aws configure list`
  * To change default/current settings `aws configure --profile default`
4) Install the `eksctl` tool (which allows eks cluster interaction from command line).
  * Linux: 
  	* `curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp`
	* `sudo mv /tmp/eksctl /usr/local/bin`
  * In Windows: instructions found here: https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html
 
## Create a Kubernetes (EKS) Cluster

Following command sets up a cluser with provided name in the default region with one nodegroup containing two Following command sets up a cluser with provided name in the default region with one nodegroup containing two m5.large nodes

`eksctl create cluster --name <NAME-HERE>`

to view current progress go to AWS 'CloudFormation' site or use the following command:

`kubectl get nodes`

Always remember to delete any unused deployed production nodes as soon as they are not needed with:

`eksctl delete cluster <NAME-HERE>`

## Set Up an IAM Role for the Cluster

### Get your AWS account id:

`aws sts get-caller-identity --query Account --output text`

### Create a role trust policy using a trust.json

```
{ 
  "Version": "2012-10-17", 
  "Statement":[{    
    "Effect": "Allow",    
    "Principal": {"AWS":"arn:aws:iam::<ACCOUNT_ID>:root"},
    "Action": "sts:AssumeRole" 
  }] 
}
```

### Create a role named 'UdacityFlaskDeployCBKubectlRole' using the trust.json above:

`aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'`

### Create a role policy document that allows the actions "eks:Describe*" and "ssm:GetParameters". 

`echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": [ "eks:Describe*", "ssm:GetParameters" ], "Resource": "*" } ] }' > /tmp/iam-role-policy.json`

### Attach the policy to the 'UdacityFlaskDeployCBKubectlRole' using the iam-role-policy.json

`aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json`

## Ensure that the current user has been given the arn-role:

`aws --region us-east-2 eks update-kubeconfig --name simple-jwt-api --role-arn arn:aws:iam::aws_account_id:role/UdacityFlaskDeployCBKubectlRole`

## Grant the Role Access to the Cluster

"aws-auth ConfigMap" is use dto grant role-based access control to your cluster. When your cluster is created only the creating user can administer it, you need to add the role you just created so that CodeBuild can also administer the cluster. 

### Get the current aws-auth ConfigMap

`kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml`

### edit the aws-auth ConfigMap to add new permissions: 

`kubectl edit -n kube-system configmap/aws-auth`

### add the following to the yml directly below 'mapRoles: |'

```
apiVersion: v1 
data:   
  mapRoles: |     
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/<EXISTENT_ROLE_INFORMATION>
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:masters
      rolearn: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
      username: build   
  mapUsers: |
    []
kind: ConfigMap 
metadata:   
  creationTimestamp: "<AUTO_GENERATED_VALUE>"
  name: aws-auth
  namespace: kube-system
  resourceVersion: <AUTO_GENERATED_VALUE>
  selfLink: /api/v1/namespaces/kube-system/configmaps/aws-auth
  uid: <AUTO_GENERATED_VALUE>
```

If successful, console will return: `configmap/aws-auth edited`

## Deploy to Kubernetes using CodePipeline and CodeBuild

### Edit ci-cd-codepipeline.cfn.yml and add EksClusterName, GitSourceRepo, GitHubUser, KubectlRoleName

#### Example:
* EksClusterName: Default: simple-jwt-api
* GitSourceRepo: Default: FSND-Deploy-Flask-App-to-Kubernetes-Using-EKS
* GitHubUser: Default: flisz
* KubectlRoleName: Default: UdacityFlaskDeployCBKubectlRole

### Under 'new stack' , 'new resources' select upload template and choose the ci-cd-codepipeline.cfn.yml above. 

### Hit 'Next' and add stack name and GitHub token. (generated GitHub token must have full private repo permissions

## Securely Set a Secret using the AWS Parameter Store

Edit the `buildspec.yml` file and add the following:

```
env:      
  parameter-store:
    JWT_SECRET: JWT_SECRET
```

then, add the secret into your AWS Parameter Store:

`aws ssm put-parameter --name JWT_SECRET --value "YourJWTSecret" --type SecureString`

Once you are done with this project, you should delete the variable from the parameter store using the command:

`aws ssm delete-parameter --name JWT_SECRET`


