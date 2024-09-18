1. Create EKS cluster.
```
eksctl create cluster --name <cluster-name> --region <region-code> --fargate
```
2. Create a fargate profile for the namespace.
```
eksctl create fargateprofile \
    --cluster <cluster-name> \
    --region <region> \
    --name <name of profile> \
    --namespace <name of the namespace>
```

3. Deploy using the helm chart.

Use this repo for helm chart.
```
helm install v0.1.0 ./1-tier-chart
```
4. Install AWS ALB Controller.
    1. Create an IAM OIDC provider.

    ```
    eksctl utils associate-iam-oidc-provider \
    --region <region-code> \
    --cluster <your-cluster-name> \
    --approve
    ```

    2. Install the IAM Policy.

    ``` 
    curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.2/docs/install/iam_policy.json 
    ```

    3. Create the IAM Policy
    ```
    aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
    ```


    4. Create an IAM role and Kubernetes ServiceAccount for the LBC.
    ```
    eksctl create iamserviceaccount \
    --cluster=<cluster-name> \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region <region-code> \
    --approve
    ```

> Ensure the IAM Policy is attached to the role

    5. Install Controller
    - Add the Helm repo
    ```
    helm repo add eks https://aws.github.io/eks-charts
    ```

    - Install
    ```
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=<region of cluster> --set vpcId=<vpc id of cluster>
    ```

    - Wait for the installation of controller
    ```
    kubectl get deploy -n kube-system -w
    ```

    - Check the ingress resource to view the loadbalancer DNS
    ```
    kubectl get ingress -n motd-ns
    ```

    - View logs of controller if alb is not created
    ```
    kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
    ```


