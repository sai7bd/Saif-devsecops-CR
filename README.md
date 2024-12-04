# Secure Pipelines Demo

Sample spring application with Jenkins pipeline script to demonstrate secure pipelines


![DevSecOps Pipeline](https://wac-cdn.atlassian.com/dam/jcr:5f26d67b-bed6-4be1-912b-4032de4d06b0/devsecops-diagram.png?cdnVersion=2318)


## Pre Requesites

1. K8s cluster 
2. eksctl, kubectl, helm cli

## Setup Setps

eksctl cli installation for Ubuntu
```
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

Create EKS cluster

```
eksctl create cluster -f setup/eks-cluster-setup.yaml
```

Node group scale
```
eksctl update nodegroup \
  --cluster=dev-secops-cluster \
  --name=ng-1 \
  --desired=3 \
  --min-size=2 \
  --max-size=3
```

OIDC is the replacement for using IAM roles or keys to create/modify/update/delete AWS service
```
$ eksctl utils associate-iam-oidc-provider --cluster dev-secops-cluster --approve --region=us-east-1

$ aws eks describe-cluster --name dev-secops-cluster --query "cluster.identity.oidc.issuer" --output text --region us-east-1

cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::747031328127:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/A9AE7E1083947032F5CE5B577FF74DF2"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/A9AE7E1083947032F5CE5B577FF74DF2:aud": "sts.amazonaws.com",
          "oidc.eks.us-east-1.amazonaws.com/id/A9AE7E1083947032F5CE5B577FF74DF2:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
EOF

$ aws iam create-role  --role-name AmazonEKS_EBS_CSI_DriverRole  --assume-role-policy-document file://"setup/trust-policy.json"

$ aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy --role-name AmazonEKS_EBS_CSI_DriverRole
```
RefL https://repost.aws/knowledge-center/eks-persistent-storage

ß
$ aws eks create-addon --cluster-name dev-secops-cluster --addon-name aws-ebs-csi-driver --service-account-role-arn arn:aws:iam::747031328127:role/AmazonEKS_EBS_CSI_DriverRole



Create CI/CD namespace
```
k create namespace ci
namespace/ci created
```

Helm installation
```
$ curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

Downloading https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm

$ helm version
version.BuildInfo{Version:"v3.16.2", GitCommit:"13654a52f7c70a143b1dd51416d633e1071faffb", GitTreeState:"clean", GoVersion:"go1.22.7"}

$ helm repo add jenkins https://charts.jenkins.io

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jenkins" chart repository
Update Complete. ⎈Happy Helming!⎈

```


Jenkins Installation
```
$ helm install --namespace ci --values setup/jenkins-values.yaml jenkins jenkins/jenkins
NAME: jenkins
LAST DEPLOYED: Mon Oct 21 18:41:04 2024
NAMESPACE: ci
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace ci -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export NODE_PORT=$(kubectl get --namespace ci -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
  export NODE_IP=$(kubectl get nodes --namespace ci -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://$NODE_IP:$NODE_PORT/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```

- Allow port of jenkins service in nodegroup SG

- Access Jenkins 

```
$ kubectl exec --namespace ci -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
r0QrC73ql4BqIdMQqUdxQM
```

- Modify admin password - It will prompt when restart jenkins after installing plugins


Jenkins configuration:
=====================

- Add additonal plugins to Jeninks server (Manage Jenkins -> Manage plugins)

  - BlueOcean
  - Configuration as Code




### New Jenkins Pipeline

Create a new Jenkins pipeline with this repo and trigger build

- Login to Jenkins -> New Item -> Enter name and choose Pipeline -> Choose GitHub project and set project URL
- Under pipeline section, Choose Pipeline script from SCM
- Choose git as SCM and provide repo details
- Save

Create secret of dockerHub

kubectl create secret -n ci docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=chandikas --docker-password=xxxxx --docker-email=xxxxxx@gmail.com




1. Talisman Installation
====================
https://thoughtworks.github.io/talisman/docs

# Download the talisman binary
curl https://thoughtworks.github.io/talisman/install.sh > ~/install-talisman.sh
chmod +x ~/install-talisman.sh

# Install to a single project (as pre-push hook)
cd my-git-project
~/install-talisman.sh //pre-push hook
~/install-talisman.sh pre-commit //pre-commit hook

mkdir sec_files && cd sec_files
echo "username:sidd-harth" > file1
echo "secure-password1234" > password.txt
echo "apiKey=Ajbehjruhr82832jcshh3435" > file2
echo "base64encodedsecret=cDSgwjekrnekrjitut" > file3

git commit 
git push 

If you want to ignore any file, add it under .talismanrc
fileignoreconfig:
- filename:
  checksum:


2. Images build using Kaniko
====================
kubectl create secret -n ci docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=chandikas --docker-password=xxxxxx --docker-email=erchandika@gmail.com


Configure github repo with jenkins by creating a new pipeline item

Run the build

Configure the pipeline to poll every minute

Add Kaniko stage in Jenkinsfile

```
        stage('OCI image build') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f "$(pwd)/Dockerfile" -c "$(pwd)" --insecure --skip-tls-verify --cache=true --destination=docker.io/chandikas/dso-demo --verbosity=debug' 
              }
            }
          }
```

3. SCA - OWASP dependency checker, OSS license checker, SBOM using cyclonedx

stage(‘Static Analysis’) {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
              // dependencyCheckPublisher pattern: 'report.xml'
                }
              }
            }
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              sh 'ls -al'
              sh '''#!/bin/bash --login
                    /bin/bash --login
                    rvm use default
                    gem install license_finder
                    license_finder
                    '''
                        }
                      }
                  }
          }
      }

4. Lint and Scan Docker image
    Lint using dockle - Check Dockerfiles/images for best practices, Image efficiency, and maintainability
      Best practises:
        Base image minimal
        Use official images
        Layer optimization
        Caching Efficiently
        Multistage Dockerfile
        

    Scan using trivy - Scan Docker images for security vulnerabilities and risks
      Running as non-root user
      Limit Ports and Privileges
      Avoid latest Tags

    
    Docker run after multistage pipeline and nonroot user
      docker run --rm -v $(pwd):/app bitnami/trivy image --exit-code 1 xxxxxx/dsodemo:multistage
      docker run --rm goodwithtech/dockle:v${DOCKLE_LATEST} docker.io/xxxxxx/dsodemo:multistage
    

5. Deploy using ArgoCD

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


kubectl -n argocd patch secret argocd-secret -p '{"stringData": {"admin.password": "$2a$12$qurzRi7xk0Q6nvGfve6BlOUW68NcbLo3vvUGYs/q9AzHjwgAnzcTS","admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'

kubectl get all -n argocd

kubectl patch svc argocd-server -n argocd --patch '{"spec": { "type": "NodePort", "ports": [ { "nodePort": 32100, "port": 443, "protocol": "TCP", "targetPort": 8080 } ] } }'


kubectl get svc -n argocd

kubectl get nodes -o wide

Browser to  https://NODEIP:32100

mkdir deploy

kubectl create deployment dso-demo --image=chandikas/dso-demo --replicas=1 --port=8080 --dry-run=client -o yaml | tee deploy/dso-demo-deploy.yaml

kubectl create service nodeport dso-demo --tcp=8080 --node-port=30080 --dry-run -o yaml | tee deploy/dso-demo-svc.yaml

git add deploy/dso-demo-deploy.yaml deploy/dso-demo-svc.yaml
git commit -am "add k8s manifests to deploy dso-demo app"
git push origin main

kubectl create ns dev
kubectl get ns

argocli installation
=======
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

kubectl patch cm -n argocd argocd-cm --patch-file setup/argocd_create_user-patch.yaml

kubectl describe cm -n argocd argocd-cm

argocd admin settings rbac validate --policy-file setup/jenkins.argorbacpolicy.csv

kubectl patch cm -n argocd argocd-rbac-cm --patch-file setup/argocd_user_rbac-patch.yaml
kubectl describe cm -n argocd argocd-rbac-cm

argocd login  34.205.144.35:31296
argocd cluster list
argocd admin settings rbac validate --policy-file setup/jenkins.argorbacpolicy.csv

argocd admin settings rbac can jenkins get applications devsecops/dso-demo --policy-file setup/jenkins.argorbacpolicy.csv 
Yes
argocd admin settings rbac can jenkins delete applications devsecops/dso-demo --policy-file setup/jenkins.argorbacpolicy.csv 
No
argocd admin settings rbac can jenkins sync applications devsecops/dso-demo --policy-file setup/jenkins.argorbacpolicy.csv 
argocd admin settings rbac can jenkins get projects project6-devsecops --policy-file setup/jenkins.argorbacpolicy.csv 


kubectl patch cm -n argocd argocd-rbac-cm --patch-file setup/argocd_user_rbac-patch.yaml

kubectl describe cm -n argocd argocd-rbac-cm

argocd account generate-token --account jenkins

argocd app sync devsecops --insecure --server 44.212.93.103:32100 --auth-token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJhcmdvY2QiLCJzdWIiOiJqZW5raW5zOmFwaUtleSIsIm5iZiI6MTcyOTgzNTA2MSwiaWF0IjoxNzI5ODM1MDYxLCJqdGkiOiJkMDhmMTBhZC1kNzc5LTQ0YWQtYjFmNC1jMjYxMmJkZDhkM2IifQ.Wpc1JpHbnH0OuD-2rOQ-vdExG6eaVnIIfHlU3uI9YvM

SAST - slscan or sonarqube scan
DAST - OWASP zap


Cleanup
=======
kubectl get pdb -n kube-system
kubectl delete pdb -n kube-system coredns
kubectl delete pdb -n kube-system ebs-csi-controller

eksctl delete cluster --name=dev-secops-cluster --region=us-east-1


# Pipeline

Refer the below screenshot for the stages in the pipeline

##### Pipeline View

![Pipeline View](imgs/Secure_Pipeline_1.png)

##### Stage View

![Stage View](imgs/Secure_Pipeline_2.png)

##### Dependency Track

![Dependency Track View](imgs/Dependency_Track.png)

## Tools

| Stage               | Tool                                                                      |
| ------------------- | ------------------------------------------------------------------------- |
| Secrets Scanner     | [truffleHog](https://github.com/dxa4481/truffleHog)                       |
| Dependency Checker  | [OWASP Dependency checker](https://jeremylong.github.io/DependencyCheck/) |
| SAST                | [OWASP Find Security Bugs](https://find-sec-bugs.github.io/)              |
| OSS License Checker | [LicenseFinder](https://github.com/pivotal/LicenseFinder)                 |
| SCA                 | [Dependency Track](https://dependencytrack.org/)                          |
| Image Scanner       | [Trivy](https://github.com/aquasecurity/trivy)                            |
| Image Hardening     | [Dockle](https://github.com/goodwithtech/dockle)                          |
| K8s Hardening       | [KubeSec](https://kubesec.io/)                                            |
| IaC Hardening       | [checkov](https://www.checkov.io/)                                        |
| DAST                | [OWASP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/) |

---

### TODO

Image Malware scanning - [ClamAV](https://github.com/openbridge/clamav)