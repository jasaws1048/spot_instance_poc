# Spot Instance POC 

- Create the EKS cluster

```
eksctl create cluster --version=1.25 --name=spotcluster-eksctl --node-private-networking --managed --nodes=3 --alb-ingress-access --region=us-west-2 --node-type t3.medium --node-labels="lifecycle=OnDemand" --asg-access
```

- Create EKS Managed Node group

```
kubectl apply -f spot_nodegroups.yaml`
```

- Install AWS Node Termination Handler 

https://artifacthub.io/packages/helm/aws/aws-node-termination-handler

- Install the Cluster Autoscaler

Deploy the Cluster autoScaler

For additional detail, see the EKS page here. Export the Cluster Autoscaler into a configuration file:

```
curl -LO https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

Open the file created and edit the cluster-autoscaler container command to replace <YOUR CLUSTER NAME> with your clusters name, and add the following options.

```
--balance-similar-node-groups
--skip-nodes-with-system-pods=false
```

You also need to change the expander configuration. Search for - --expander= and replace least-waste with random

Example:

```
    spec:
        containers:
        - command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=random
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
```
Save the file and then deploy the Cluster Autoscaler:

```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

```kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```


```kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=registry.k8s.io/autoscaling/cluster-autoscaler:v1.25.3
```

To view the Cluster Autoscaler logs, use the following command:

```kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

- Deploy the sample application

Create a new file web-app.yaml, paste the following specification into it and save the file:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stateless
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: web-stateless
        resources:
          limits:
            cpu: 1000m
            memory: 1024Mi
          requests:
            cpu: 1000m
            memory: 1024Mi
      nodeSelector:    
        lifecycle: Ec2Spot
--- 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stateful
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        service: redis
        app: redis
    spec:
      containers:
      - image: redis:3.2-alpine
        name: web-stateful
        resources:
          limits:
            cpu: 1000m
            memory: 1024Mi
          requests:
            cpu: 1000m
            memory: 1024Mi
      nodeSelector:
        lifecycle: OnDemand
```
This deploys three replicas, which land on one of the Spot Instance node groups due to the nodeSelector choosing lifecycle: Ec2Spot. The “web-stateful” nodes are not fault-tolerant and not appropriate to be deployed on Spot Instances. So, you use nodeSelector again, and instead choose lifecycle: OnDemand. By guiding fault-tolerant pods to Spot Instance nodes, and stateful pods to On-Demand nodes, you can even use this to support multi-tenant clusters.

To deploy the application:

```
kubectl apply -f web-app.yaml
```

Confirm that both deployments are running:

```
kubectl get deployment/web-stateless
```
```kubectl get deployment/web-stateful
```

Now, scale out the stateless application:

```kubectl scale --replicas=30 deployment/web-stateless
```

Check to see that there are pending pods. Wait approximately 5 minutes, then check again to confirm the pending pods have been scheduled:

```kubectl get pods
```

- Clean-Up

Remove the AWS Node Termination Handler:

```kubectl delete daemonset aws-node-termination-handler -n kube-system
```

Remove the two Spot node groups (EC2 Auto Scaling group) that you deployed in the tutorial.

```eksctl delete nodegroup ng-4vcpu-16gb-spot --cluster spotcluster-eksctl
```
```eksctl delete nodegroup ng-8vcpu-32gb-spot --cluster spotcluster-eksctl
```

If you used a new cluster and not your existing cluster, delete the EKS cluster.

eksctl confirms the deletion of the cluster’s CloudFormation stack immediately but the deletion could take up to 15 minutes. You can optionally track it in the CloudFormation Console.

```eksctl delete cluster --name spotcluster-eksctl
```