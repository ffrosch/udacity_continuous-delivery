# Deploying a Flask API

This is the project starter repo for the fourth course in the [Udacity Full Stack Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004): Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'.
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token.

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.

## Project Steps

Completing the project involves several steps:

1. Write a Dockerfile for a simple Flask API
2. Build and test the container locally
3. Create an EKS cluster
4. Store a secret using AWS Parameter Store
5. Create a CodePipeline pipeline triggered by GitHub checkins
6. Create a CodeBuild stage which will build, test, and deploy your code

For more detail about each of these steps, see the project lesson [here](https://classroom.udacity.com/nanodegrees/nd004/parts/1d842ebf-5b10-4749-9e5e-ef28fe98f173/modules/ac13842f-c841-4c1a-b284-b47899f4613d/lessons/becb2dac-c108-4143-8f6c-11b30413e28d/concepts/092cdb35-28f7-4145-b6e6-6278b8dd7527).

## Initial setup
1. Fork this project to your Github account.
2. Locally clone your forked version to begin working on the project.

## Dependencies

- Docker Engine
    - Installation instructions for all OSes can be found [here](https://docs.docker.com/install/).
    - For Mac users, if you have no previous Docker Toolbox installation, you can install Docker Desktop for Mac. If you already have a Docker Toolbox installation, please read [this](https://docs.docker.com/docker-for-mac/docker-toolbox/) before installing.
 - AWS Account
     - You can create an AWS account by signing up [here](https://aws.amazon.com/#).

## Steps to run the API locally (to test endpoints)
1. Install python dependencies with `pip install -r requirements.txt`.
2. Set up the environment with `touch .env`, open the file and add these variables:
    ```bash
    JWT_SECRET='myjwtsecret'
    LOG_LEVEL=DEBUG
    ```
3. Run the app with the flask development server with `python main.py`

Open a new browser to run http://127.0.0.1:8080/ and it will give you a response as `"Healthy"`.

## Try the API endpoints on Command-Line
4. Open a new shell and install jq, which is a package that helps to read or manipulate JSON processors.
Use `sudo apt-get install jq` on Linux or `chocolatey install jq` on Windows.
5. To try the `/auth` endpoint, use the following command, replacing `<EMAIL>` and `<PASSWORD>` with any values:
    ```bash
    export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
    ```
6. This calls the endpoint 'localhost:8080/auth' with the `{"email":"<EMAIL>","password":"<PASSWORD>"}` as the message body. The return value is a JWT token based on the secret string you supplied. We are assigning that secret to the environment variable 'TOKEN'. To see the JWT token run `echo $TOKEN`.

7. To try the /contents endpoint which decrypts the token and returns its content, run:
    ```bash
    curl --request GET 'http://127.0.0.1:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
    ```

## Build and run a Docker Image
Build the Image:
```bash
docker build -t "jwt-api-test" .
docker image ls
```

If required, you can delete an image using `docker image rm -f <image_name>`

Run the Image locally:
```bash
docker run --env-file=.env -p 80:8080 jwt-api-test
docker container ls
```

If required, you can stop a container using `docker stop [OPTIONS] CONTAINER [CONTAINER...]`
or delete a container using `docker rm [OPTIONS] CONTAINER [CONTAINER...]`

Use the endpoints like before to test whether the container is running correctly.
This time make sure to use the port specified for the docker container (80) and
also adjust the IP address depending on your system. On Windows the default
machine IP should be `192.168.99.100`. It can be obtained by running `docker-machine ip default`.

```bash
export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST 192.168.99.100:80/auth  | jq -r
curl --request GET '192.168.99.100:80/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
```

## Create an EKS Cluster and IAM Role
1. Install the AWSCLI:
    ```bash
    pip install awscli --upgrade
    which aws
    aws --version
    ```
2. Generate AWS Access key ID and Secret access key for an existing IAM User on the
AWS website.
3. Setup your environment to use Access key ID and Secret access key.
    * On your terminal, run `aws configure list`. It will show you the values as not set.
    * Run `aws configure --profile default`. It will prompt you for Access key ID, Secret access key, and Default region name.
4. Install the 'eksctl' tool. The 'eksctl' tool allows interaction with an EKS cluster from the command line.

    Linux:
    ```bash
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin
    ```

    Windows:
    ```bash
    choco install -y eksctl # or if already installed: choco upgrade -y eksctl
    eksctl version
    ```
5. Create an EKS cluster named “simple-jwt-api” and set up all the associated services:
    ```bash
    eksctl create cluster --name simple-jwt-api
    ```
    The command above will create an EKS cluster in your default region, as specified by your AWS CLI configuration, with one nodegroup containing two m5.large nodes.
    You can go to the CloudFormation console to view progress. If you don’t see any progress, be sure that you are viewing clusters in the same region that they are being created. To confirm your region, you can run aws configure list command. Once the status is ‘CREATE_COMPLETE’ in your command line, check the health of your clusters nodes: `kubectl get nodes`

    > ```diff
    > - Remember to delete the cluster as soon as it is not needed anymore:
    > ```

```bash
eksctl delete cluster simple-jwt-api
```

## Set Up an IAM Role for the Cluster
The next steps are provided to quickly set up an IAM role for your cluster. This is the role that CodeBuild will assume to administer the EKS cluster.

1. Get your AWS account id:
    ```bash
    aws sts get-caller-identity --query Account --output text
    ```
2. Create a role trust policy. You can do this in either of the following two ways:
    * Set an environment variable `TRUST` with the role trust policy. In the command below, replace `<ACCOUNT_ID>` with your account id:
    ```bash
    TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::<ACCOUNT_ID>:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"
    ```
    * Create a `trust.json` file, as shown in the Deployment lesson, under AWSCLI concept. The `trust.json` file would contain the policy details as:
    ```json
    { "Version": "2012-10-17",
    "Statement":[{
    "Effect": "Allow",
    "Principal": {"AWS":"arn:aws:iam::<ACCOUNT_ID>:root"},
    "Action": "sts:AssumeRole"
    }]
    }
    ```
3. Create a role named 'UdacityFlaskDeployCBKubectlRole' using the role trust policy from the TRUST variable:
    ```bash
    aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'
    ```

    If you have used a `trust.json` file, the command will be:
    ```bash
    aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'
    ```
4. Create a role policy document that allows the actions "eks:Describe*" and "ssm:GetParameters". You can create the document in your tmp directory:
    ```bash
    echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": [ "eks:Describe*", "ssm:GetParameters" ], "Resource": "*" } ] }' > /tmp/iam-role-policy
    ```

    Alternatively, you can create `iam-role-policy.json` as:
    ```json
    {
    "Version": "2012-10-17",
    "Statement":[{
        "Effect": "Allow",
        "Action": ["eks:Describe*", "ssm:GetParameters"],
        "Resource":"*"
    }]
    }
    ```
5. Attach the policy to the 'UdacityFlaskDeployCBKubectlRole'. You can read the policy file from tmp directory created in step above, and attach the policy as:
    ```bash
    aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
    ```

    For `iam-role-policy.json`, the command will be:
    ```bash
    aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json
    ```

You have now created a role named 'UdacityFlaskDeployCBKubectlRole'.

## Grant the Role Access to the Cluster
The next steps are provided to quickly set up an IAM role for your cluster. This is the role that CodeBuild will assume to administer the EKS cluster.

1. The 'aws-auth ConfigMap' is used to grant role-based access control to your cluster. When your cluster is first created, the user who created it is given sole permission to administer it. You need to add the role you just created so that CodeBuild can administer it as well. Get the current configmap and save it to a file:

    ```bash
    kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml
    ```

    The file will be created at `/System/Volumes/Data/private/tmp/aws-auth-patch.yml` path on Linux or at `~\AppData\Local\Temp` on Windows.

2. Open the `aws-auth-patch.yml` file. In the “data/mapRoles” section of this document add the following, replacing `<ACCOUNT_ID>` with your AWS account id:

    ```yml
        - groups:
          - system:masters
          rolearn: arn:aws:iam::<ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
          username: build
    ```

3. Update your cluster's configmap:

    ```bash
    kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
    ```

## Deployment to Kubernetes using CodePipeline and CodeBuild
You should now be ready to deploy your application using CodePipeline and CodeBuild.

You will now create a pipeline that watches your Github. When changes are checked in, it will build a new image and deploy it to your cluster. When you created your cluster, eksctl automatically created a CloudFormation stack, now we will use a CloudFormation template to create a stack for a CodePipeline using the AWS console. Follow the steps below to complete your project.

1. Generate a GitHub Access Token

    A Github access token will allow CodePipeline to monitor when a repo is changed. A token can be generated [here](https://github.com/settings/tokens/). You should generate the token with full control of private repositories, as shown in the image below. Be sure to save the token somewhere that is secure.

2. Create the CodePipeline Pipeline using CloudFormation Template

    1. Modify CloudFormation template - There is a file named `ci-cd-codepipeline.cfn.yml` provided in your starter repo. **This is the template file that you will use to create your CodePipeline pipeline and "build project" in CodeBuild**. Open this file, and go to the 'Parameters' section. These are parameters that will accept values when you create a stack. Fill in the value for 'Default' field for the following:

        * **EksClusterName** : use the name of the EKS cluster you created above
        * **GitSourceRepo** : use the name of your project's github repo.
        * **GitHubUser** : use your github user name
        * **KubectlRoleName** : use the name of the role you created for kubectl above

        For example,

        ```yml
        EksClusterName:
        Default: simple-jwt-api
        ```

        ```yml
        KubectlRoleName:
        Default: UdacityFlaskDeployCBKubectlRole
        ```

        Do not use any quotes in your values. Save this `ci-cd-codepipeline.cfn.yml` file. An explanation about this file is provided in the "Addendum - CloudFormation" section at the bottom of the current page.

    2. Create a stack for CodePipeline using the CloudFormation template (`ci-cd-codepipeline.cfn.yml`) file. Go to the [CloudFormation service](https://eu-central-1.console.aws.amazon.com/cloudformation/) in the AWS console. Press the 'Create Stack' button. It will make you go through the following three steps -

        * Step 1 - Specify template - Choose the 'Upload a template file ' option and upload the template file 'ci-cd-codepipeline.cfn.yml'

        * Step 2 - Specify stack details - Press 'Next'. Give the stack a name, fill in your GitHub login, and the Github access token generated in the previous step. Make sure that the cluster name matches the one you have created, and the 'kubectl IAM role' matches the role you created above, and the repository matches the name of your forked repo.

        * Step 3 - Configure stack options - Leave default, and create the stack. You can check its status in the [CloudFormation console](https://eu-central-1.console.aws.amazon.com/cloudformation/).

3. Set a Secret using AWS Parameter Store

    We need a way to pass your JWT secret to the app in kubernetes securly. You will be using [AWS Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) to do this. First, add the following to your `buildspec.yml` file:

    ```yml
    env:
      parameter-store:
        JWT_SECRET: JWT_SECRET
    ```

    This lets CodeBuild know to set an environment variable based on a value in the parameter-store. Put secret into AWS Parameter Store:

    ```bash
    aws ssm put-parameter --name JWT_SECRET --value "YourJWTSecret" --type SecureString
    ```

    > ```diff
    > - Once you submit your project, and receive the reviews, you can consider deleting the variable from > parameter-store using:
    >```

    ```bash
    aws ssm delete-parameter --name JWT_SECRET
    ```

4. Make use of `buildspec.yml` for Build using CodeBuild

    The `buildspec.yml` file specifies the different phases of a build, such as an install, pre-build, build, and post-build. Each phase has a set of commands to be automatically executed by CodeBuild. When you trigger a build in the CodeBuild, you can see each command being executed in the CodeBuild log console.

    In the previous step, the CloudFormation template file `ci-cd-codepipeline.cfn.yml` would automatically create a "build project" in [CodeBuild](https://aws.amazon.com/codebuild/). You just have to start the build process by clicking on the “Start build” button. The build process will make use of `buildspec.yml` file automatically.

## Successfully Working Pipeline

1. Check the pipeline works. Once the stack is successfully created, commit a change to the master branch of your github repo, as:

    ```bash
    git remote -v
    git add . --all
    git commit -m “my comment“
    git push
    ```
    Then, in the aws console go to the [CodePipeline UI](https://eu-central-1.console.aws.amazon.com/codesuite/codepipeline). You should see that the build is running.

2. To test your api endpoints, get the external IP for your service:

    ```bash
    export URL="$(kubectl get services simple-jwt-api -o wide | grep -o "[0-9\.a-z-]*amazonaws.com")"
    ```

    Now use the external IP url to test the app:

    ```bash
    export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST ${URL}/auth  | jq -r '.token'`
    curl --request GET "${URL}/contents" -H "Authorization: Bearer ${TOKEN}" | jq
    curl -s GET "${URL}" # returns "Healthy"
    ```

3. **Save the external IP from above to provide to the reviewer when you will submit your project**. Go to the next page for details about the project submission process.

## Add Tests to the Build

Tests can be run locally using `pytest test_main.py`.

1. Add running tests as part of the build.

    To require the unit tests to pass before our build will deploy new code to your cluster, you will add the tests to the build stage. You will add the same commands to the buildspec.yml:

    ```yml
    pre_build:
    commands:
    - pip3 install -r requirements.txt
    - python -m pytest test_main.py
    ```

2. You can check the tests prevent a bad deployment by breaking the tests on purpose:

    * Open the test_main.py file
    * Add `assert False` to any of the tests
    * Commit your code and push it to Github
    * Check that the build fails in [CodePipeline](https://eu-central-1.console.aws.amazon.com/codesuite/codepipeline)
