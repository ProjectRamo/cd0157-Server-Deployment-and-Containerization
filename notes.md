# Up and running

git cloned from repo
used conda env "contain"
updated requirements.txt to flask 2.1.0 because the old one used too many unsupported packages
pip installed requirements.txt

# Just used their example defaults:

 ## A secret text string to be used to creating a JWT
 > export JWT_SECRET='myjwtsecret'
 > export LOG_LEVEL=DEBUG
 ## Verify
 > echo $JWT_SECRET
 > echo $LOG_LEVEL


 # Testing

## To run server
> python main.py
## New terminal
> curl --request GET http://localhost:8080/
"Healthy"

## check token
tried "brew install jq" but it was already installed
> export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
> echo $TOKEN

## decode jwt
> curl --request GET 'http://localhost:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .

# Dockerize

## Dockerfile

"
# Use the `python:3.7` as a source image from the Amazon ECR Public Gallery
# We are not using `python:3.7.2-slim` from Dockerhub because it has put a  pull rate limit. 
FROM public.ecr.aws/sam/build-python3.7:latest
# Set up an app directory for your code
COPY . /app
WORKDIR /app
# Install `pip` and needed Python packages from `requirements.txt`
RUN pip install --upgrade pip
RUN pip install -r requirements.txt
# Define an entrypoint which will run the main app using the Gunicorn WSGI server.
ENTRYPOINT ["gunicorn", "-b", ":8080", "main:APP"]
"

## .env_file
JWT_SECRET='myjwtsecret'
LOG_LEVEL=DEBUG

## build the image
docker build -t udfsndimage .

## check image was built
docker image ls

## run the image
docker run --name udfsnd --env-file=.env_file -p 80:8080 udfsndimage

## test container
curl --request GET 'http://localhost:80/'
### Calls the endpoint 'localhost:80/auth' with the email/password as the message body. 
### The return JWT token assigned to the environment variable 'TOKEN' 
export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"WindowsPwd"}' --header "Content-Type: application/json" -X POST localhost:80/auth  | jq -r '.token'`
echo $TOKEN
# Decrypt the token and returns its content
curl --request GET 'http://localhost:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .

# AWS Cluster
installed AWS CLI

## aws configure
> aws configure
Prompt	Value
AWS Access Key ID	[Copy from the classroom]
AWS Secret Access Key	[Copy from the classroom]
Default region name	"us-east-2"
Default output format	"json"

## kubectl
> kubectly version
seems to work though I don't recall installing it

## install eksctl
DOing this on my mac
### Check Homebrew 
brew --version
### Install eksctl
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
### check install
> kubectl version  
### create cluster
eksctl create cluster --name simple-jwt-api --nodes=2 --version=1.22 --instance-types=t2.medium --region=us-east-2
## aws configure with an admin user
This took a while to get right, but you can't just do
> aws configure
and type in the keys from the panel in Udacity.
Create a new IAM user, and attach an admin polocy specifically
"AdministratorAccess"
Other policies like EKSCluster etc, isn't enough by itself.
Use the credentials of the admin user in aws configure.

### Create this trust,json file in the same folder
{
"Version": "2012-10-17",
"Statement": [
 {
     "Effect": "Allow",
     "Principal": {
         "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
     },
     "Action": "sts:AssumeRole"
 }
]
}

but replace account id with the id from
> aws sts get-caller-identity --query Account --output text

### create this role
aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'

### create this policy file iam-role-policy.json
{
"Version": "2012-10-17",
"Statement": [
  {
      "Effect": "Allow",
      "Action": [
          "eks:Describe*",
             "ssm:GetParameters"
      ],
      "Resource": "*"
  }
]
}
### attach role to policy
aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json

### There are a whole bunch of steps
### 1. Generate github token
### 2. modify ci-cd-codepipeline.cfn.yml 
EksClusterName	simple-jwt-api
Name of the EKS cluster you created
GitSourceRepo	cd0157-Server-Deployment-and-Containerization
Github repo name
GitBranch	master
Or any other you want to link to the Pipeline
GitHubUser	Your Github username
KubectlRoleName	UdacityFlaskDeployCBKubectlRole
### 3. Create Stack in cloud formation services
Have to upload the yml file
### 4. Save secret parameter
> aws ssm put-parameter --name JWT_SECRET --overwrite --value "myjwtsecret" --type SecureString
### 5. Update kubectly verion in buildspec to reflect
> kubectl version --short --client

# Verify

### 1. 
git push

to launch the build step


policy: ufsnd-kub