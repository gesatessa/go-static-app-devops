# Go Web Application

This is a simple website written in Golang. It uses the `net/http` package to serve HTTP requests.

## Running the server

To run the server, execute the following command:

```bash
go run main.go
```

The server will start on port 8080. You can access it by navigating to `http://localhost:8080/courses` in your web browser.

## run with Docker

```sh
docker build -t go-static-app:v1 

docker run -p 8080:8080 -it go-static-app:v1


```

## ECR

```sh
export APP_NAME=go-static-app
```

```sh
aws ecr get-login-password --region $AWS_REGION | \
docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

aws ecr create-repository --repository-name $APP_NAME --region $AWS_REGION

docker build -t ${APP_NAME}:latest .

IMG_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$APP_NAME:v1

docker tag ${APP_NAME}:latest $IMG_URI

docker push $IMG_URI

```

## EKS

```sh
export CLUSTER_NAME=static-app-go
export CLUSTER_NS=default
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export NODE_TYPE=t3.small
export NODE_CNT=1
export NODE_MAX=3

eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --node-type $NODE_TYPE \
  --nodes $NODE_CNT \
  --nodes-max $NODE_MAX \
  --managed \
  --spot

# eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME

# verify
k get nodes

eksctl get nodegroup --cluster $CLUSTER_NAME

## K8s

```sh
k apply -f k8s/

eksctl scale nodegroup \
  --cluster $CLUSTER_NAME \
  --name ng-2d85f854 \
  --nodes 2 \
  --nodes-max 2
```


## GW

Intall Gateway API CRDs and ENVOY Gateway:
```sh
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.6.1 \
  -n envoy-gateway-system \
  --create-namespace

# verify
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```


```sh
kubectl describe httproute backend

k get svc -n envoy-gateway-system

H_=ad12e90dff4794ea68a64ac9ab6081d8-1861919842.us-east-1.elb.amazonaws.com
curl -v --headers "Host: www.myapp.io" $H_
curl -i -H "Host: www.myapp.io" $H_/courses

#/etc/hosts (ip h)
```
To delete the resources

```sh
kubectl delete httproute backend
kubectl delete gateway eg

# If Helm will also manage the GatewayClass:
kubectl delete gatewayclass eg
```
## Helm

```sh
mkdir -p helm/go-web-app/templates

# ---

helm install web helm/go-web-app

# verify
kubectl get pods,svc

kubectl get gateway,gatewayclass,httproute

kubectl get httproute backend -o yaml
```

## GHA

```sh
mkdir -p .github/workflows
touch .github/workflows/ci.yml

```

📢
We'll need an IAM role named something like `arn:aws:iam::865274826587:role/github-actions-ecr-role` with permission to push to ECR.

We'll set this up using `GitHub OIDC`, so GitHub Actions can assume the IAM role without long-lived AWS keys.

```sh
export AWS_ACCOUNT_ID=802838070254
export AWS_REGION=us-east-1

export GH_USER=gesatessa
export GH_REPO_NAME=go-static-app-devops
export ROLE_NAME=gha-ecr-role
export ECR_REPO_NAME=go-static-app


```

```sh
cat > /tmp/trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GH_USER}/${GH_REPO_NAME}:ref:refs/heads/main"
        }
      }
    }
  ]
}
EOF

# create role
aws iam create-role \
  --role-name "$ROLE_NAME" \
  --assume-role-policy-document file:///tmp/trust-policy.json


aws iam get-role \
  --role-name "$ROLE_NAME" \
  --query 'Role.AssumeRolePolicyDocument' \
  --output json

aws iam update-assume-role-policy \
  --role-name $ROLE_NAME \
  --policy-document file:///tmp/trust-policy.json
```


ECR policies
```sh

```