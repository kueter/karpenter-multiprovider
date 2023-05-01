# karpenter-multiprovider
karpenter-multiprovider

				**************    Installation    *************
1. Install the aws cli
	
2. Export the local variable

	```$ export KARPENTER_VERSION=v0.27.2
	$ export CLUSTER_NAME="<your cluster name>"
	$ export AWS_DEFAULT_REGION="<your region>"
	$ export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
	$ export TEMPOUT=$(mktemp)
	$ export OIDC_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text)"```
	
3. Create iam role

Create the role `"KarpenterNodeRole-${CLUSTER_NAME}"` with this policies
```
	AmazonEKSWorkerNodePolicy
	AmazonEKS_CNI_Policy
	AmazonEC2ContainerRegistryReadOnly
	AmazonSSMManagedInstanceCore
```
Type this commande
```
$ echo '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}' > node-trust-policy.json

aws iam create-role --role-name "KarpenterNodeRole-${CLUSTER_NAME}" \
    --assume-role-policy-document file://node-trust-policy.json
```	
Then attach the policies to the role

```
aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```		

Attach the IAM role to an EC2 instance profile.

```
aws iam create-instance-profile --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}"

aws iam add-role-to-instance-profile --instance-profile-name "KarpenterNodeInstanceProfile-${CLUSTER_NAME}" --role-name "KarpenterNodeRole-${CLUSTER_NAME}"
```

Create an IAM role that the Karpenter controller will use to provision new instances

```
$ cat << EOF > controller-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
                    "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:karpenter:karpenter"
                }
            }
        }
    ]
}
EOF

```
```
	$ aws iam create-role --role-name KarpenterControllerRole-${CLUSTER_NAME} --assume-role-policy-document file://controller-trust-policy.json
```

```
$ cat << EOF > controller-policy.json
{
    "Statement": [
        {
            "Action": [
                "ssm:GetParameter",
                "ec2:DescribeImages",
                "ec2:RunInstances",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInstanceTypeOfferings",
                "ec2:DescribeAvailabilityZones",
                "ec2:DeleteLaunchTemplate",
                "ec2:CreateTags",
                "ec2:CreateLaunchTemplate",
                "ec2:CreateFleet",
                "ec2:DescribeSpotPriceHistory",
                "pricing:GetProducts"
            ],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "Karpenter"
        },
        {
            "Action": "ec2:TerminateInstances",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/karpenter.sh/provisioner-name": "*"
                }
            },
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "ConditionalEC2Termination"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}",
            "Sid": "PassNodeIAMRole"
        },
        {
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "arn:${AWS_PARTITION}:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${CLUSTER_NAME}",
            "Sid": "EKSClusterEndpointLookup"
        }
    ],
    "Version": "2012-10-17"
}
EOF
```


```
	$ aws iam put-role-policy --role-name KarpenterControllerRole-${CLUSTER_NAME} --policy-name KarpenterControllerPolicy-${CLUSTER_NAME} --policy-document file://controller-policy.json
```

5. Active the likerd-role
	
```
	$ aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

6. Logout of docker to perform an unauthenticated pull against the public ECR
```
	$ docker logout public.ecr.aws
```

7. Add tags to subnets and security groups

- Subnets
```
	for NODEGROUP in $(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} \
	    --query 'nodegroups' --output text); do aws ec2 create-tags \
	    --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
	    --resources $(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
	    --nodegroup-name $NODEGROUP --query 'nodegroup.subnets' --output text )
	done
```	
- Security Groups

```
	NODEGROUP=$(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} \
	    --query 'nodegroups[0]' --output text)

	LAUNCH_TEMPLATE=$(aws eks describe-nodegroup --cluster-name ${CLUSTER_NAME} \
	    --nodegroup-name ${NODEGROUP} --query 'nodegroup.launchTemplate.{id:id,version:version}' \
	    --output text | tr -s "\t" ",")
```
	# If your EKS setup is configured to use only Cluster security group, then please execute -
```
	SECURITY_GROUPS=$(aws eks describe-cluster \
	    --name ${CLUSTER_NAME} --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

	# If your setup uses the security groups in the Launch template of a managed node group, then :

	SECURITY_GROUPS=$(aws ec2 describe-launch-template-versions \
	    --launch-template-id ${LAUNCH_TEMPLATE%,*} --versions ${LAUNCH_TEMPLATE#*,} \
	    --query 'LaunchTemplateVersions[0].LaunchTemplateData.[NetworkInterfaces[0].Groups||SecurityGroupIds]' \
	    --output text)
```

```
	$ aws ec2 create-tags --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" --resources ${SECURITY_GROUPS}
```	

8. Update aws-auth ConfigMap of the cluster
```
	$ kubectl edit configmap aws-auth -n kube-system
```

Add this part

```
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}
  username: system:node:{{EC2PrivateDNSName}}

```

9. Install karpenter with helm command

```
	$ helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
	  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
	  --set settings.aws.clusterName=${CLUSTER_NAME} \
	  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
	  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
	  --set controller.resources.requests.cpu=1 \
	  --set controller.resources.requests.memory=1Gi \
	  --set controller.resources.limits.cpu=1 \
	  --set controller.resources.limits.memory=1Gi \
	  --set hostNetwork=true \
	  --wait
```

10. Install the provisioner with this command
```
	$ kubectl apply -f prov-app.yaml
``` 

11. Deploy the app
```
	$ kubectl apply -f inflate-app.yaml
``` 

12- Check the traffic
```	 	
	$ kubectl logs -f -n karpenter -c controller -l app.kubernetes.io/name=karpenter
```
