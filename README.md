# AWS EKS TERRAFORM

We build a private EKS cluster using Terraform and Cloud9 as a bastion host.

The diagram depicts the end state:

![](/images/master-scenario.png)

## 1. Prerequisites

### 1. Add an IAM user for the workshop.

![](/images/iam-1-create-user.png)

### Attach the AdministratorAccess IAM Policy

![](/images/iam-2-attach-policy.png)

### Create new user

![](/images/iam-3-create-user.png)

### Save URL

![](/images/iam-4-save-url.png)

## 2. Create Cloud9 Workspace

- Launch Cloud9 in your closest region 

- Click <!--Create environment-->
- On the next page, give Name: **eks-terraform**, then click Next

![](/images/c9-create1.png)

On the next screen choose these options:

- “Create a new no-ingress EC2 instance for the environment (access via Systems Manager)"
- “t3.small (2GiB RAM + 2CPU)"
- “Amazon Linux 2 (recommended)"
- Set the Cost-saving setting to “After one hour”
- And click Next Step

![](/images/c9-create2.png)

On the Review page double-check the Name is set to “eks-terraform” and then click Create environment
When it comes up, customize the environment by closing the welcome tab and lower work area, and opening a new terminal tab in the main work area:

![](/images/c9before.png)

Your workspace should now look like this:

![](/images/c9after.png)

## 2. Create IAM Role for Workspace

1. Go to the IAM Console.
2. Confirm that AWS service and EC2 are selected, then click Next to view permissions.
3. Confirm that AdministratorAccess is checked, then click Next: Tags to assign tags.
4. Take the defaults, and click Next: Review to review.
5. Enter eksworkshop-admin for the Name, and click Create role.

![](/images/createrole.png)

## 3. Attach IAM Role to Workspace

Go to your newly created EC2 instance

Select the instance, then choose **Actions / Security / Modify IAM Role**

![](/images/ModifyIAMRole.jpg)

Choose **eksworkshop-admin** from the **IAM Role** drop down, and select **Save**

![](/images/SaveIAMRole.jpg)

## 4. Update settings for your Workspace

- Return to your workspace and click the gear icon (in top right corner), or click to open a new tab and choose “Open Preferences”
- Select AWS Settings
- Turn off AWS managed temporary credentials
- Close the Preferences tab

![](/images/c9disableiam.png)

## 5. Install Kubernetes Tools

Clone the workshop repo and use a helper script to set up the workshop tools:

```bash
cd ~/environment
```

```bash
git clone https://github.com/ai-systems-today/code.git
```

```bash
cd ~/environment/code
```

```bash
source setup-tools.sh
```

```bash
./check.sh
```

 Provisioned a Cloud9 IDE in the default VPC

![](/images/cloud9-ide.jpg)

# 2. Terraform State

Create the S3 bucket and DynamoDB tables for Terraform state files & locks.

```bash
cd ~/environment/code/tf-setup 

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```



- Creates a unique bucket name based on your hostname. (see gen-bucket-name.sh)
- Initializes Terraform in the tf-setup directory.
- Runs Terraform (plan and apply) which:
  - Creates a s3 bucket
  - Creates the DynamoDB tables for terraform locks
  - Runs the the gen-backend.sh script from a Terraform “null resource”

![](/images/tf-state-aws.jpg)

# 3. Set up the VPC, Subnets, Security Groups and VPC Endpoints

Diagram shows the EKS VPC and CI/CD VPC we will build in this section:

![](/images/net-1.jpg)

```bash
cd ~/environment/code/net 

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

You can see from the plan the following resources will be created - open the corresponding files to see the Terraform HCL code that details the configuration:

- A VPC (**vpc-cluster.tf**).
- A secondary VPC CIDR block (**aws_vpc_ipv4_cidr_block_association__vpc-cidr-assoc.tf**).
- Various VPE Endpoints (**vpce.tf**).
- Subnets (**subnets-eks.tf**).
- Route Tables (**aws_route_table__rtb-\*.tf**).
- Route Table Associations (**aws_route_table_association__rtbassoc\*.tf**).
- Security Groups (**aws_security_group__allnodes-sg.tf** & **aws_security_group__cluster-sg.tf**).
- NAT Gateway (**aws_nat_gateway__eks-cicd.tf**).

There are also Terraform file to setup the VPC and subnets used by CodeBuild part of the CICD pipeline

- The VPC (**aws_vpc__eks-cicd.tf**) and it’s associated:
- Subnets (**aws_subnet__eks-cicd\*.tf**).
- Security groups (**aws_security_group__sg-eks-cicd.tf**).
- Route tables (**aws_route_table__private1.tf** & **aws_route_table__public1.tf**).
- Route Table Associations (**aws_route_table_association__private1.tf** & **aws_route_table_association__public1.tf** ).
- Internet Gateway (**aws_internet_gateway__eks-cicd.tf**).
- A NAT Gateway (**aws_eip__eipalloc-cicd-natgw.tf**).

# 4 Set up the IAM Roles and Policies for EKS

```bash
cd ~/environment/code/iam

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

From the plan the following resources will be created - open the corresponding files to see the Terraform HCL code that details the configuration

- A Cluster Service Role (**aws_iam_role__cluster-ServiceRole.tf**)
- A Node Group Service Role (**aws_iam_role__nodegroup-NodeInstanceRole.tf**)
- Various policy definitions that EKS needs eg: (**aws_iam_role_policy__nodegroup-NodeInstanceRole-PolicyAutoScaling.tf**)
- Policy attachments to the cluster and node group roles eg: (**aws_iam_role_policy_attachment__cluster-ServiceRole-AmazonEKSClusterPolicy.tf**)

# 5. Link the Cloud9 IDE and CI/CD to the EKS using VPC Peering

```bash
cd ~/environment/code/c9net

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

# 6. Create EKS Cluster 

```bash
cd ~/environment/code/cluster

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

You can see from the plan the following resources will be created

- An EKS Cluster
- Configure the cluster with an OIDC provider and add support for ISRA (IAM Roles for Service Accounts)
- A null resource which runs a small script to test connectivity to EKS with nmap and write the local .kubeconfig file

# 7. Node group

Deploy a customized Managed Node Group using an AMI we specify and a SSM agent as a demonstration of deploying custom software to the worker nodes.

![](/images/nodeg-build.jpg)

```bash
cd ~/environment/code/nodeg

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

# 8. Deploy the CICD Infrastructure

```bash
cd ~/environment/code/cicd

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

Check CodeBuild is authorized to access the EKS cluster ok

```bash
kubectl get -n kube-system configmap/aws-auth -o yaml | grep -i codebuild
```

if have issues, run this command:

```bash
./auth-cicd.sh
```

# 9. Use secondary CIDR with EKS

```bash
cd ~/environment/code/eks-cidr

terraform init

terraform validate

terraform plan -out tfplan

kubectl set env ds aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

terraform apply tfplan

kubectl get pods -A -o wide
```

Test networking:

```bash
kubectl create deployment nginx --image=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/nginx 

kubectl scale --replicas=3 deployments/nginx 

kubectl expose deployment/nginx --type=NodePort --port 80 

kubectl get pods -o wide
```

If after 10-20 seconds have elapsed you see the containers are not Running - but in status ContainerCreating instead run this script to re-annotate the worker nodes:

```bash
./reannotate-nodes.sh

kubectl get pods -o wide
```

You can use busybox pod and ping pods within same host or across hosts using IP address:

```bash
kubectl run -i --rm --tty debug --image=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/busybox -- sh
```

```bash
If you don't see a command prompt, try pressing enter.
/ # 
```

Test access to internet and to nginx service

Internet connectivity - should fail (hang) as we build our EKS cluster in a private VPC !

```bash
wget google.com -O -
```

Type ctrl-c to escape:

```bash
wget nginx -O -
```

you should see:

> Connecting to nginx (10.100.170.156:80)
> <!DOCTYPE html>
> <html>
> <head>
> <title>Welcome to nginx!</title>
>
> ***TRUNCATED**
>
> <p><em>Thank you for using nginx.</em></p>
> </body>
> </html>
> -                    100% |**************************************************************************************|   612  0:00:00 ETA
> written to stdout

Finally exit from the busybox container:

```bash
exit
```

# 10. Deploy the AWS Load Balancer Controller

```bash
cd ~/environment/code/lb2

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

You can see from the plan the following resources will be created

- A Load Balancer policy.
- A null provider “policy” to create the policy.
- A null provider “post_policy” to implement the CRD.
- Terraform installs the helm chart for the Load Balancer Controller.

The above has:

- Downloaded the policy definition file.
- Created a Load Balancer policy using the file.
- Started the post-policy.sh shell script which:
- Downloads and creates the Custom Resource Definition extension.
- Installs the aws-load-balancer-controller helm chart.

Confirm the controller is operational with the command below and look for “Running” in the output:

```bash
kubectl get pods -A | grep aws-load-balancer-controller 
```

you can also look at the helm output with:

```bash
helm ls -n kube-system
```

# 11. Deploy sample application to EKS using CI/CD

```bash
cd ~/environment/code/sampleapp
```

Create a service credential to use with our CodeCommit git repo:

```bash
usercred=$(aws iam create-service-specific-credential --user-name git-user --service-name codecommit.amazonaws.com)
GIT_USERNAME=$(echo $usercred | jq -r '.ServiceSpecificCredential.ServiceUserName')
GIT_PASSWORD=$(echo $usercred | jq -r '.ServiceSpecificCredential.ServicePassword')
CREDENTIAL_ID=$(echo $usercred| jq -r '.ServiceSpecificCredential.ServiceSpecificCredentialId')
test -n "$GIT_USERNAME" && echo GIT_USERNAME is "$GIT_USERNAME" || "echo GIT_USERNAME is not set"
```

Clone the (empty) repo:

```bash
test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || "echo AWS_REGION is not set"
git clone codecommit::$AWS_REGION://eksworkshop-app
```

```
Cloning into 'eksworkshop-app'...

'Namespace' object has no attribute 'cli_binary_format'
warning: You appear to have cloned an empty repository.
```

------

Populate with our source files - including the special file **buildspec.yaml** which has the steps CodeBuild will follow.

```bash
cd eksworkshop-app
cp ../buildspec.yml .
cp ../*.tf .
```

Add files, commit and push

```bash
git add --all
git commit -m "Initial commit."
git push
```

------

This should now trigger a few activities

Check you can see your code in CodeCommit - navigate to your repository in the console and confirm you can see the files:

[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/codecommit-1.png)](https://tf-eks-workshop.workshop.aws/images/andyt/codecommit-1.png)

Next check if the CodePipeline is running - navigate to it in the console

[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/pipeline-1.png)](https://tf-eks-workshop.workshop.aws/images/andyt/pipeline-1.png)

You can also link through to the CodeBuild project:

[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/codebuild-1.png)](https://tf-eks-workshop.workshop.aws/images/andyt/codebuild-1.png)

And tail to logs of the build job (scroll the window or use the `Tail Logs` button):

[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/codebuild-2.png)](https://tf-eks-workshop.workshop.aws/images/andyt/codebuild-2.png)

------

Check everything is running ?

```bash
# kubectl get pods,svc,deployment -n game-2048 -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP              NODE                                       NOMINATED NODE   READINESS GATES
pod/deployment-2048-76d4bff958-5w94k   1/1     Running   0          55s   100.64.143.56   ip-10-0-3-166.eu-west-1.compute.internal   <none>           <none>
pod/deployment-2048-76d4bff958-r4jhb   1/1     Running   0          55s   100.64.24.3     ip-10-0-1-228.eu-west-1.compute.internal   <none>           <none>

NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/service-2048   NodePort   172.20.162.86   <none>        80:32624/TCP   20s   app.kubernetes.io/name=app-2048

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                                    SELECTOR
deployment.apps/deployment-2048   2/2     2            2           55s   app-2048     123456789012.dkr.ecr.eu-west-1.amazonaws.com/sample-app   app.kubernetes.io/name=app-2048
```

**Note that**:

- The pods are deployed to a 100.64.x.x. address.
- The service is exposing port 80.
- The deployment is referencing a private ECR repository belonging to your account. (see the IMAGES section of thr deployment output)

------

Enable port forwarding so we can see the application in out Cloud9 IDE

```bash
kubectl port-forward service/service-2048 8080:80 -n game-2048
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
```

Preview the running (port-forwarded service) application from the cloud 9 IDE

`Preview` -> `Preview Running Application`[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-0.jpg)](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-0.jpg)

You should then see the app running in the browser

[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-1.jpg)](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-1.jpg)

------

## Finding the Internal Load Balancer 

As before with the CLI the CI/CD pipeline has also deployed a Load Balancer.

The load balancer will take about 8 minutes to provision and come online

Check how long it has bene provisioning by using the command:

```bash
kubectl get ingress -n game-2048
NAME           CLASS    HOSTS   ADDRESS   PORTS   AGE
ingress-2048   <none>   *                 80      5m27s
```

Watching the aws-load-balancer-controller - open another terminal and use this command to watch the logs:

```bash
kubectl logs `kubectl get pods -n kube-system | grep aws-load-balancer-controller | awk '{print $1}'` -n kube-system --follow
```

**After 8 minutes have elapsed**

Check the `targetbindings` have populated. This is the new CRD type that was created as part of the load balancer controller installation.

```bash
kubectl get targetgroupbindings -A
NAMESPACE   NAME                               SERVICE-NAME   SERVICE-PORT   TARGET-TYPE   AGE
game-2048   k8s-game2048-service2-11af83fe8f   service-2048   80             ip            82s
```

Then obtain the internal DNS name of the load balancer using and check valid HTML is returned with curl

```bash
ALB=$(aws elbv2 describe-load-balancers --query 'LoadBalancers[*].DNSName' | jq -r .[])
curl $ALB:8080
<!DOCTYPE html>`
<html>
<head>
  <meta charset="utf-8">
  <title>2048</title>

** Output truncated for brevity **

  <script src="js/application.js"></script>
</body>
</html>
```

------

## Cleanup

Interrupt the port forwarding with ctrl-c if necessary.

**The following destroy operation make take up to 12 minutes as it deletes the ingress and ALB**

```bash
terraform destroy -auto-approve
null_resource.cleanup: Destroying... [id=9012327125218962041]
null_resource.cleanup: Provisioning with 'local-exec'...
null_resource.cleanup (local-exec): Executing: ["/bin/bash" "-c" "        echo \"remote git credentials &\" sample app\n        ./cleanup.sh\n        echo \"************************************************************************************\"\n"]
null_resource.cleanup (local-exec): remote git credentials & sample app
kubernetes_namespace.game-2048: Destroying... [id=game-2048]
kubernetes_service.game-2048__service-2048: Destroying... [id=game-2048/service-2048]
kubernetes_ingress.game-2048__ingress-2048: Destroying... [id=game-2048/ingress-2048]
kubernetes_deployment.game-2048__deployment-2048: Destroying... [id=game-2048/deployment-2048]
kubernetes_ingress.game-2048__ingress-2048: Destruction complete after 2s
kubernetes_service.game-2048__service-2048: Destruction complete after 2s
kubernetes_deployment.game-2048__deployment-2048: Destruction complete after 2s
null_resource.cleanup (local-exec): ************************************************************************************
null_resource.cleanup: Destruction complete after 3s
kubernetes_namespace.game-2048: Still destroying... [id=game-2048, 10s elapsed]
kubernetes_namespace.game-2048: Still destroying... [id=game-2048, 20s elapsed]

...

kubernetes_namespace.game-2048: Still destroying... [id=game-2048, 12m10s elapsed]
kubernetes_namespace.game-2048: Destruction complete after 12m15s

Destroy complete! Resources: 5 destroyed.
```

------

Note: it’s only possible to delete the application from the command line like this because we are using S3 for the Terraform backend state files. The CI/CD pipeline also used the same backend state files, so both this command line and the CI/CD pipeline are viewing the same state.

# 12. Build a second node group that uses SPOT instances

```bash
cd ~/environment/code/nodeg2

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

You can see from the plan the following resources will be created

- A Launch template
- A NodeGroup using the launch template above

We should now have 4 kubernetes worker nodes

```bash
# kubectl get nodes 
NAME                                       STATUS   ROLES    AGE     VERSION
ip-10-0-1-231.eu-west-1.compute.internal   Ready    <none>   3m      v1.18.9-eks-d1db3c
ip-10-0-1-25.eu-west-1.compute.internal    Ready    <none>   3h58m   v1.18.9-eks-d1db3c
ip-10-0-2-179.eu-west-1.compute.internal   Ready    <none>   3h56m   v1.18.9-eks-d1db3c
ip-10-0-2-71.eu-west-1.compute.internal    Ready    <none>   2m57s   v1.18.9-eks-d1db3c
```

Annotate the nodes in the second group such that they only use 10.x addresses

```bash
cd ~/environment/code/extra/eks-cidr2

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

Deploy two sample applications to separate node groups:

```bash
cd ~/environment/code/extra/sampleapp2

terraform init

terraform validate

terraform plan -out tfplan

terraform apply tfplan
```

Check everything is running ?

```bash
# kubectl get pods,svc,deployment -A -o wide | grep game
game1-2048    pod/deployment-2048-ng1-788c7f7874-mccgw            1/1     Running   0          2m8s    100.64.100.188   ip-10-0-2-179.eu-west-1.compute.internal   <none>           <none>
game1-2048    pod/deployment-2048-ng1-788c7f7874-nvlqq            1/1     Running   0          2m8s    100.64.42.14     ip-10-0-1-25.eu-west-1.compute.internal    <none>           <none>
game2-2048    pod/deployment-2048-ng2-74bbf67dc5-w9sbh            1/1     Running   0          2m7s    10.0.2.166       ip-10-0-2-71.eu-west-1.compute.internal    <none>           <none>
game2-2048    pod/deployment-2048-ng2-74bbf67dc5-zq7p6            1/1     Running   0          2m7s    10.0.1.180       ip-10-0-1-231.eu-west-1.compute.internal   <none>           <none>
game1-2048    service/service1-2048                       NodePort    172.20.87.238    <none>        80:32481/TCP    2m6s    app.kubernetes.io/name=app1-2048
game1-2048    service/service2-2048                       NodePort    172.20.206.182   <none>        80:30243/TCP    2m5s    app.kubernetes.io/name=app2-2048
game1-2048    deployment.apps/deployment-2048-ng1            2/2     2            2           2m8s    app1-2048                      136434655158.dkr.ecr.eu-west-1.amazonaws.com/sample-app                                   app.kubernetes.io/name=app1-2048
game2-2048    deployment.apps/deployment-2048-ng2            2/2     2            2           2m7s    app2-2048                      136434655158.dkr.ecr.eu-west-1.amazonaws.com/sample-app                                   app.kubernetes.io/name=app2-2048
```

Note from the output that:

- The pods are deployed to 110.64.x.x (node group 1) and 10.0.x.x addresses (node group 2).
- The services are exposing port 80.
- The deployment is referencing a private ECR repository belonging to your account.

**If you see pods apparently stuck in “ContainerCreating” mode for a minute or more try the following:**

 Expand here to see the fix

------

Enable port forwarding so we can see the application in out Cloud9 IDE:

```bash
# kubectl port-forward service/service2-2048 8080:80 -n game2-2048
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
```

Preview the running (port-forwarded service) application from the cloud 9 IDE”

`Preview` -> `Preview Running Application`[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-0.jpg)](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-0.jpg)

You should then see the app running in the browser

[![tf-state](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-1.jpg)](https://tf-eks-workshop.workshop.aws/images/andyt/game-2048-1.jpg)

As the Terraform files are similar to the previous section they are not explained here.

------

## Cleanup

Interrupt the port forwarding with **ctrl-C**

Then use Terraform to delete the Kubernetes resources:

```bash
terraform destroy -auto-approve
```
