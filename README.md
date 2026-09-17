# HabitApp — Java DevSecOps CI/CD and GitOps Project

A complete hands-on DevOps/DevSecOps implementation for a Spring Boot Habit Tracker application. The project demonstrates the complete application delivery lifecycle from GitHub source control through Jenkins CI, Maven testing, SonarQube quality analysis, Docker image creation, Amazon ECR, Helm, Kubernetes/K3s, Argo CD GitOps deployment, rolling updates, rollback, restoration, security verification, monitoring, and troubleshooting.

## 1. Project Objective

The objective is to implement a production-style CI/CD and GitOps workflow without allowing Jenkins to directly deploy the application to Kubernetes.

The implemented flow is:

```text
Developer
   |
   v
GitHub
   |
   v
Jenkins CI
   |
   +--> Checkout
   +--> Maven Build & Unit Tests
   +--> SonarQube Analysis
   +--> Quality Gate
   +--> Docker Build
   +--> Image Versioning
   +--> Amazon ECR
   +--> Update Helm values.yaml
   +--> Commit Deployment Change
   |
   v
GitHub
   |
   v
Argo CD
   |
   v
Kubernetes / K3s
   |
   v
HabitApp
```

Jenkins performs CI and changes the desired deployment state in Git. Argo CD monitors GitHub and performs the Continuous Deployment to Kubernetes.

---

## 2. Technology Stack

| Area | Technology |
|---|---|
| Application | Java / Spring Boot 3.3.4 |
| Java Runtime | Java 21 |
| Build | Maven |
| Source Control | Git / GitHub |
| CI | Jenkins |
| Code Quality | SonarQube |
| Containerization | Docker |
| Registry | Amazon ECR |
| Packaging | Helm 4.3.0 |
| Orchestration | Kubernetes / K3s |
| CD / GitOps | Argo CD |
| Ingress | Traefik |
| Cloud | AWS |
| AWS Region | `ap-south-1` (Mumbai) |
| Application Port | `8080` |
| Kubernetes Namespace | `habitapp` |

---

## 3. Repository

GitHub repository:

```text
https://github.com/rokidexter/Habit-App-Java-Application-CI-CD.git
```

Branch:

```text
main
```

Local project directory used on Windows:

```text
C:\DevOps\project-5-habitapp
```

EC2 project directory:

```text
~/HabitApp
```

---

## 4. Project Structure

```text
HabitApp/
├── src/
│   └── main/
│       ├── java/
│       │   └── ...
│       └── resources/
│           └── application.properties
├── helm/
│   └── habitapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── configmap.yaml
│           └── ingress.yaml
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── Dockerfile
├── Jenkinsfile
├── pom.xml
├── sonar-project.properties
├── README.md
└── .gitignore
```

The repository also contains Maven-generated `target/` content locally, while build artifacts and Terraform-style/generated state are excluded from Git as appropriate through `.gitignore`.

---

# 5. Java Application

HabitApp is a Spring Boot Habit Tracker application.

The application is packaged as:

```text
habit-tracker.jar
```

The application listens on:

```text
8080
```

Application configuration:

```properties
spring.application.name=habit-tracker
server.port=8080
spring.mvc.problemdetails.enabled=true
```

The application does not require an external database. Habit data is held in memory for the lifetime of the process.

---

# 6. Maven Build and Testing

Maven is used for dependency management, compilation, packaging, and unit testing.

Local verification:

```bash
mvn clean verify
```

Jenkins executes the same build/test flow through the `Maven Build & Test` stage.

Successful Jenkins evidence:

```text
BUILD SUCCESS
Tests run: 1
Failures: 0
Errors: 0
Skipped: 0
```

---

# 7. SonarQube

SonarQube is integrated into Jenkins for static code quality analysis.

Project:

```text
HabitApp
```

Jenkins runs:

```bash
mvn sonar:sonar \
  -Dsonar.projectKey=HabitApp \
  -Dsonar.projectName=HabitApp
```

The pipeline waits for the SonarQube Quality Gate using the configured Jenkins webhook.

### SonarQube implementation

- SonarQube project created for `HabitApp`.
- SonarQube token stored in Jenkins Credentials.
- Jenkins SonarQube server configuration created.
- SonarQube webhook configured for Jenkins.
- Webhook secret stored as a Jenkins credential.
- Jenkins waits for the Quality Gate result.
- Pipeline aborts if the Quality Gate fails.

### Verified result

```text
SonarQube task status: SUCCESS
Quality Gate: OK
```

The Jenkins log also confirmed that the incoming webhook matched the configured webhook secret.

---

# 8. Docker

The project uses a multi-stage Dockerfile.

## Build stage

```dockerfile
FROM maven:3.9.12-eclipse-temurin-21 AS build
```

The application is compiled and packaged with Maven.

## Runtime stage

```dockerfile
FROM eclipse-temurin:21-jre
```

The final container runs the packaged JAR using Java 21.

## Non-root container

A dedicated Linux user is created:

```dockerfile
RUN useradd --system --create-home --shell /usr/sbin/nologin appuser
USER appuser
```

This was verified inside Kubernetes:

```text
uid=999(appuser) gid=999(appuser) groups=999(appuser)
```

Therefore the application container does not run as root.

---

# 9. Amazon ECR

Amazon Elastic Container Registry is used as the private container registry.

Repository:

```text
habitapp
```

Registry:

```text
382170164329.dkr.ecr.ap-south-1.amazonaws.com
```

Full image format:

```text
382170164329.dkr.ecr.ap-south-1.amazonaws.com/habitapp:<git-sha>
```

Image tags are generated from the first seven characters of the Git commit SHA.

Example:

```text
ab5a554
```

This provides traceability:

```text
Git Commit -> Docker Image -> Helm Version -> Kubernetes Deployment
```

The successful Jenkins pipeline pushed image `ab5a554` to ECR with digest:

```text
sha256:7124e9e3162d928fa0205489109ec2ff17bd681f7f038792d6881500b07b266c
```

The previous working image `388d376` was also verified to remain available in ECR before rollback.

---

# 10. Jenkins

Jenkins is running in Docker on the AWS EC2 instance.

Jenkins container configuration includes:

- Jenkins LTS with JDK 21
- Jenkins external port `8081`
- Docker socket access for image builds
- Maven configuration `Maven3`
- Docker CLI
- AWS CLI
- SonarQube integration

Jenkins obtains AWS permissions through the EC2 IAM role rather than static AWS access keys.

The EC2 role used by Jenkins is:

```text
ECR
```

AWS identity verification from the Jenkins container returned the EC2 assumed role identity.

---

# 11. Jenkins CI Pipeline

The Declarative Jenkins pipeline contains these stages:

1. Checkout
2. Maven Build & Test
3. SonarQube Analysis
4. Quality Gate
5. Docker Build
6. Push Image
7. Update Helm Configuration
8. Commit Deployment Change

The project requirement originally describes the CI flow as:

```text
Checkout
Maven Build
Unit Test
SonarQube Analysis
Quality Gate
Docker Build
Image Versioning
Push Image
Update Helm Configuration
Commit Deployment Change
```

The build and unit-test work is implemented together in the `Maven Build & Test` stage.

## Image versioning

The Jenkinsfile uses:

```groovy
IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
```

For the successful deployment this generated:

```text
ab5a554
```

## Helm update

Jenkins updates:

```text
helm/habitapp/values.yaml
```

and changes the image tag automatically.

## Git commit

Jenkins creates a deployment commit such as:

```text
63d097f Update HabitApp image to ab5a554
```

and pushes it to `main`.

## Successful CI evidence

The successful Jenkins pipeline demonstrated:

- Maven build successful.
- Unit test passed.
- SonarQube analysis successful.
- Quality Gate `OK`.
- Docker image built successfully.
- ECR login successful.
- Image pushed successfully.
- Helm image tag updated.
- Git deployment commit created.
- GitHub push successful.

Final Jenkins result:

```text
Finished: SUCCESS
```

---

# 12. Jenkins Credentials and AWS Authentication

Jenkins credentials were configured using Jenkins Credentials rather than hardcoding secret values in the repository.

Configured credentials include:

```text
sonarqube-token
sonarqube-webhook-secret
github-push
```

The GitHub credential is used by Jenkins to push deployment configuration changes.

AWS authentication uses the EC2 IAM role through instance metadata instead of storing AWS access keys in Jenkins.

The Jenkins container successfully executed:

```bash
aws sts get-caller-identity
```

and received the EC2 role identity.

ECR login was also successfully verified from Jenkins.

---

# 13. Helm Chart

HabitApp is packaged as a reusable Helm chart:

```text
helm/habitapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    └── ingress.yaml
```

Chart metadata:

```yaml
apiVersion: v2
name: habitapp
type: application
version: 0.1.0
appVersion: "1.0.0"
```

## Helm features implemented

- Deployment
- 2 replicas
- Service
- Ingress
- ConfigMap
- ECR image pull Secret
- CPU requests and limits
- Memory requests and limits
- Readiness probe
- Liveness probe
- Rolling update configuration
- Configurable image repository
- Configurable image tag
- Configurable application settings

## Validation

Lint:

```bash
helm lint helm/habitapp
```

Result:

```text
1 chart(s) linted, 0 chart(s) failed
```

Render templates:

```bash
helm template habitapp helm/habitapp
```

Install/upgrade:

```bash
helm upgrade --install habitapp helm/habitapp -n habitapp
```

Release status:

```bash
helm status habitapp -n habitapp
```

---

# 14. Kubernetes / K3s

The application runs on a single-node K3s cluster.

Cluster information:

```text
Kubernetes: v1.36.4+k3s1
K3s node: ip-172-31-37-72
Node role: control-plane
Node status: Ready
```

Namespace:

```text
habitapp
```

Deployment:

```text
habitapp
```

Replicas:

```text
2
```

Container port:

```text
8080
```

## Resource configuration

Requests:

```text
CPU: 100m
Memory: 256Mi
```

Limits:

```text
CPU: 500m
Memory: 512Mi
```

## Health probes

Readiness probe:

```text
/api/habits
```

Liveness probe:

```text
/api/habits
```

## Rolling update

```yaml
maxUnavailable: 0
maxSurge: 1
```

This configuration supports zero unavailable replicas during a normal rolling update while Kubernetes creates replacement Pods.

---

# 15. Kubernetes Service

The application Service is:

```text
habitapp
```

Service type:

```text
NodePort
```

Port:

```text
80
```

Target port:

```text
8080
```

NodePort:

```text
30711
```

The Service provides stable internal access to the application Pods.

---

# 16. Traefik Ingress

Traefik is the Ingress controller for the K3s cluster.

HabitApp Ingress:

```text
Host: habitapp.local
Path: /
Backend: habitapp:80
```

Traefik LoadBalancer service uses the K3s node address.

The application was verified locally on the EC2 instance using:

```bash
curl -I -H "Host: habitapp.local" http://127.0.0.1/
```

Expected result:

```text
HTTP/1.1 200 OK
```

It was also verified through the public EC2 address:

```bash
curl -I -H "Host: habitapp.local" http://13.201.25.113/
```

The application returned HTTP `200` when the correct Host header was supplied.

The public address without the `habitapp.local` Host header returned `404`, which demonstrated that Traefik was reachable but the hostname-based Ingress route was not selected.

For browser access, the Windows hosts file was configured with:

```text
13.201.25.113 habitapp.local
```

After DNS cache refresh, the application was accessible through:

```text
http://habitapp.local
```

---

# 17. Argo CD

Argo CD was installed in the `argocd` namespace using the official stable installation manifest.

The Argo CD server was configured for HTTP access behind the Traefik Ingress.

Argo CD application:

```text
habitapp
```

Source repository:

```text
https://github.com/rokidexter/Habit-App-Java-Application-CI-CD.git
```

Branch:

```text
main
```

Path:

```text
helm/habitapp
```

Destination:

```text
https://kubernetes.default.svc
```

Namespace:

```text
habitapp
```

Automated synchronization is enabled with:

```text
automated sync
prune
selfHeal
```

## Verified GitOps behavior

The application reached:

```text
Synced
Healthy
```

Argo CD detected the Jenkins-generated Git commit and deployed the new ECR image.

The Kubernetes Deployment image changed to:

```text
382170164329.dkr.ecr.ap-south-1.amazonaws.com/habitapp:ab5a554
```

This proved:

```text
Jenkins -> GitHub -> Argo CD -> Kubernetes
```

without Jenkins directly applying Kubernetes manifests.

---

# 18. GitOps Deployment Evidence

The successful deployment generated:

```text
63d097f Update HabitApp image to ab5a554
```

Argo CD detected revision:

```text
63d097f9f760c531a8d8e221685b81abd1089172
```

Argo CD synchronized the Helm change and Kubernetes reached a healthy state.

The deployed image was:

```text
382170164329.dkr.ecr.ap-south-1.amazonaws.com/habitapp:ab5a554
```

Two healthy Pods were running.

---

# 19. Rolling Deployment and Rollback

The project explicitly demonstrates:

- New application version deployment.
- Application health verification.
- Rollback to a previous working version.
- Restoration of the newer version.

## New version

New image:

```text
ab5a554
```

Deployment commit:

```text
63d097f Update HabitApp image to ab5a554
```

## Rollback

Before rollback, the previous image was verified in ECR:

```bash
aws ecr describe-images \
  --repository-name habitapp \
  --image-ids imageTag=388d376 \
  --region ap-south-1 \
  --query 'imageDetails[0].imageTags' \
  --output json
```

The previous image was present:

```text
["388d376"]
```

Rollback commit:

```text
4c690cf Rollback HabitApp image to 388d376
```

Argo CD detected the rollback commit and synchronized it.

Kubernetes rollout verification:

```text
deployment "habitapp" successfully rolled out
```

The actual Deployment image was verified as:

```text
382170164329.dkr.ecr.ap-south-1.amazonaws.com/habitapp:388d376
```

Pods after rollback:

```text
habitapp-7f7644b489-6nb6n   1/1   Running   0
habitapp-7f7644b489-fcgcx   1/1   Running   0
```

This demonstrates recovery to a known working version.

## Restoration

After rollback evidence was captured, the newer version was restored through GitOps.

Restore commit:

```text
a7492b5 Restore HabitApp image to ab5a554
```

Final Git history:

```text
a7492b5 Restore HabitApp image to ab5a554
4c690cf Rollback HabitApp image to 388d376
63d097f Update HabitApp image to ab5a554
ab5a554 Add deployment change commit stage
```

Final Kubernetes image:

```text
382170164329.dkr.ecr.ap-south-1.amazonaws.com/habitapp:ab5a554
```

The restored deployment completed successfully and the application returned to the newer working version.

---

# 20. Security

Security practices implemented in the project include:

## Jenkins credentials

Sensitive values are stored in Jenkins Credentials rather than committed directly to source control.

Credentials include:

```text
sonarqube-token
sonarqube-webhook-secret
github-push
```

## AWS authentication

Jenkins uses the EC2 IAM role for AWS access instead of static AWS access keys.

The Jenkins container successfully accessed AWS through the EC2 instance role and authenticated to ECR.

## Kubernetes ECR Secret

Private ECR authentication is stored in:

```text
ecr-secret
```

Secret type:

```text
kubernetes.io/dockerconfigjson
```

## Non-root container

The container runs as:

```text
appuser
```

and was verified using `kubectl exec`.

## Kubernetes RBAC

The application does not need Kubernetes API access.

Verification showed only the default ServiceAccount in the application namespace and no custom application Roles or RoleBindings.

## Credential search

Repository checks found no actual AWS keys, passwords, tokens, or other secret values committed to the repository.

The occurrences of words such as `token`, `secret`, and `password` are configuration references, Jenkins credential variables, commands, placeholders, or Kubernetes secret references rather than exposed secret values.

---

# 21. Monitoring and Troubleshooting

Monitoring and troubleshooting were demonstrated using Kubernetes, Metrics Server, application logs, and Argo CD.

## Pod status

Command:

```bash
kubectl get pods -n habitapp -o wide
```

Verified state:

```text
2 replicas
1/1 Running
0 restarts
```

## CPU and memory

Command:

```bash
kubectl top pods -n habitapp
```

Observed during verification:

```text
habitapp-7f7644b489-6nb6n   2m   121Mi
habitapp-7f7644b489-fcgcx   3m   116Mi
```

This confirms that Kubernetes metrics were available through Metrics Server.

## Restart monitoring

Command:

```bash
kubectl get pods -n habitapp \
  -o custom-columns="POD:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount"
```

Verified:

```text
Running   0
Running   0
```

## Application logs

Command:

```bash
kubectl logs -n habitapp deploy/habitapp --tail=50
```

The captured logs showed:

- Spring Boot 3.3.4 startup.
- Java 21.0.12.
- Tomcat initialized on port 8080.
- Spring WebApplicationContext initialization.
- Welcome page mapping.
- Tomcat started successfully.
- HabitTrackerApplication started successfully.
- DispatcherServlet initialization.
- No startup errors in the captured output.

## Argo CD health

Command:

```bash
kubectl get application habitapp -n argocd -o wide
```

Healthy deployment state:

```text
SYNC STATUS: Synced
HEALTH STATUS: Healthy
```

These checks demonstrate the ability to identify application and deployment issues using Pod status, resource metrics, restart counts, application logs, rollout status, and GitOps health.

---

# 22. Troubleshooting Commands

## Check Pods

```bash
kubectl get pods -n habitapp -o wide
```

## Describe a Pod

```bash
kubectl describe pod <pod-name> -n habitapp
```

## Check Deployment

```bash
kubectl get deployment habitapp -n habitapp
```

## Describe Deployment

```bash
kubectl describe deployment habitapp -n habitapp
```

## Check events

```bash
kubectl get events -n habitapp --sort-by=.lastTimestamp
```

## Check logs

```bash
kubectl logs -n habitapp deploy/habitapp --tail=50
```

## Check resources

```bash
kubectl top pods -n habitapp
```

## Check rollout

```bash
kubectl rollout status deployment/habitapp -n habitapp
```

## Check image

```bash
kubectl get deployment habitapp -n habitapp \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## Check Argo CD

```bash
kubectl get application habitapp -n argocd -o wide
```

## Check Helm

```bash
helm status habitapp -n habitapp
```

---

# 23. Application Verification

## Local cluster verification

```bash
curl -I -H "Host: habitapp.local" http://127.0.0.1/
```

Expected:

```text
HTTP/1.1 200 OK
```

## Public EC2 verification

```bash
curl -I -H "Host: habitapp.local" http://13.201.25.113/
```

Expected:

```text
HTTP/1.1 200 OK
```

## Browser URL

```text
http://habitapp.local
```

---

# 24. Submission Requirements

| Requirement | Status | Evidence / Location |
|---|---|---|
| GitHub Repository | Complete | GitHub repository listed above |
| Java Application | Complete | `src/` |
| `pom.xml` | Complete | Repository root |
| Dockerfile | Complete | Repository root |
| Jenkinsfile | Complete | Repository root |
| Helm Chart | Complete | `helm/habitapp/` |
| Argo CD configuration | Complete | `habitapp` Argo CD Application |
| Kubernetes configuration | Complete | `k8s/` and Helm templates |
| Container Registry | Complete | Amazon ECR `habitapp` |
| SonarQube Project | Complete | `HabitApp` |
| README.md | Complete | Repository root |
| Application URL | Complete | `http://habitapp.local` |

Additional demonstrated requirements:

- Maven build and unit tests
- SonarQube Quality Gate
- Docker image versioning
- ECR image publishing
- GitOps deployment
- Helm deployment
- Kubernetes rolling deployment
- Kubernetes rollback
- Restoration of newer version
- Container non-root execution
- Jenkins credential management
- EC2 IAM role based AWS authentication
- Kubernetes ECR Secret
- Resource requests and limits
- Readiness and liveness probes
- Pod monitoring
- CPU/memory monitoring
- Restart monitoring
- Application logs
- Argo CD health monitoring

---

# 25. Final Project Outcome

The project demonstrates an end-to-end DevOps/DevSecOps lifecycle:

```text
Source Code
    |
    v
GitHub
    |
    v
Jenkins CI
    |
    +--> Maven Build
    +--> Unit Tests
    +--> SonarQube
    +--> Quality Gate
    +--> Docker Build
    +--> Image Versioning
    +--> Amazon ECR
    +--> Helm values update
    +--> Git commit
    |
    v
GitHub GitOps Repository
    |
    v
Argo CD
    |
    v
K3s / Kubernetes
    |
    +--> Deployment
    +--> Service
    +--> ConfigMap
    +--> Ingress
    +--> Health Probes
    +--> Resource Controls
    |
    v
HabitApp
    |
    +--> Monitoring
    +--> Logs
    +--> Rolling Update
    +--> Rollback
    +--> Restoration
```

The project therefore covers the requested CI/CD, DevSecOps, GitOps, Kubernetes deployment, rollback, security, monitoring, and troubleshooting workflow in a single end-to-end implementation.
