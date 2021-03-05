# GitOps with Tekton and ArgoCD
## Simple Node JS App Soup to Nuts: From DeskTop Docker to Kubernetes

이 레포는 [ibm-cloud-architecture/node-web-app](https://github.com/ibm-cloud-architecture/node-web-app) 레포지토리의 내용을 쿠버네티스에서 따라하기 쉽게 변경하였습니다.

간단한 Node JS 웹앱이 있으며 argocd와 tekton을 사용한 gitops를 따라해 볼 수 있습니다.

전체적인 workflow는 아래와 같습니다.

![alt argo-flow-1](images/TektonArgoOpenShift.png)

### 이 레포를 포크하여 사용하는 방법(필수아님) 


1. gitops 예시를 자신의 레포지토리에서 진행하고 싶을때 포크를 할 수 있습니다.

    ![alt fork-repo](images/fork-repo.png)

2. 자신의 유저아이디를 선택합니다.

    ![alt fork-repo-2](images/fork-repo-2.png)

3. 포크한 레포를 클론합니다.
```console
 git clone https://github.com/<your-user>/node_web_app.git

 cd node_web_app
```

### 로컬에서 Node APP을 도커를 사용하여 테스트해보기 

Node APP은 아래의 Node JS APP을 도커라이제이션하는 튜토리얼을 완료하면 나오는 결과물입니다.
[Dockerizing a Node.js web app](https://nodejs.org/fr/docs/guides/nodejs-docker-webapp/)

1. 노드앱을 로컬에서 실행시키기 위해선 도커가 필요합니다.
윈도우의 경우 도커데스크톱 [Install Docker Desktop](https://www.docker.com/products/docker-desktop) 
리눅스의 경우 그냥 도커를 설치하시면 됩니다. [Install Docker](https://docs.docker.com/get-docker/)

2. 로컬에 Node JS가 설치되어 있을경우 실행시킬 수 있습니다.(필수 아님)

```
npm install 
node server.js

```



3. 배포를 위한 도커이미지 생성(Dockerfile 명령 실행)


```
docker build -t <your username>/node-web-app .
```

4. 도커 이미지 컨테이너로 실행
```
docker run -p 49160:8080 -d <your username>/node-web-app

```

5. 도커 컨테이너가 실행중인지 확인
```
docker ps
```

6. 어플리케이션 테스트

```
curl -i localhost:49160

```
### Pipeline Resource 파일의 내용 변경

트리거 이벤트 없이 콘솔에서 빌드하기 위해 필요

![alt fork-repo](images/pipeline-resource-fork.png)

### argocd 설치
argocd는 gitops의 구현체로써 git이나 helm 레포의 내용을 읽어 쿠버네티스 클러스터와 동기화 시켜준다.

operator에 대해 알고 있다면 밑에 방법을 추천하며 만약 모른다면 위에 직접설치를 사용하여 설치한다.

#### 직접설치
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
[argocd 직접설치](https://argoproj.github.io/argo-cd/getting_started/#1-install-argo-cd)
#### 오퍼레이터
[argocd operator 설치](https://argocd-operator.readthedocs.io/en/latest/install/start/)


### tekton 설치
tekton은 젠킨스와 비슷한 툴로 클라우드네이티브 환경에서 동작하는 ci/cd 툴이다.
#### 직접설치
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```

#### 오퍼레이터
[operator](https://operatorhub.io/operator/tektoncd-operator)

### 배포를 위한 Tekton Task 설치
#### 이미지 빌드
쿠버네티스 클러스터에서 이미지 빌드를 위한 Task
```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/v1beta1/buildah/buildah.yaml
```
[링크](https://github.com/tektoncd/catalog/tree/v1beta1/buildah)
#### argocd cli로 배포
argocd cli를 통해 쿠버네티스 클러스터에 배포하는 Task
```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/v1beta1/argocd/argocd.yaml
```
[링크](https://github.com/tektoncd/catalog/tree/v1beta1/argocd)
### argocd 시크릿 업데이트

![alt argo-secret](images/argosecret.png)

node-web-app-argocdsecret.template 파일을 node-web-app-argocdsecret.yaml 파일로 복사합니다.

```
cd pipeline
cp node-web-app-argocdsecret.template  node-web-app-argocdsecret.yaml
```
*** node-web-app-argocdsecret.yaml는 인증정보를 담고 있기때문에 깃허브에 올라가선 안됩니다. ***
*** pipeline/.gitignore 파일에 node-web-app-argocdsecret.yaml가 있으므로 파일이름을 함부로 변경하면 안됩니다. ***

![alt git-ignore](images/gitignore.png)

ARGOCD_SERVER 값을 실제 서버주소로 변경합니다.
ARGOCD_AUTH_TOKEN 또는 (ARGOCD_USERNAME AND ARGOCD_PASSWORD)를 입력해줍니다.
기본 username은 admin password는 아래 명령어 입력 후 나오는 argocd-server-69678b4f65-f5kq5 입니다.
argocd-server- 이후에 문자열은 매번 다릅니다.
```
$ kubectl get pods -n argocd
NAME                                 READY   STATUS    RESTARTS   AGE
argocd-application-controller-0      1/1     Running   0          13m
argocd-dex-server-567b48df49-cn6vx   1/1     Running   0          13m
argocd-redis-6fb68d9df5-7lvvh        1/1     Running   0          13m
argocd-repo-server-6dcbd9cb-r46tc    1/1     Running   0          13m
argocd-server-69678b4f65-f5kq5       1/1     Running   0          13m
```

![alt argo-secret](images/argosecret.png)

### Pipeline YAML 설명

텍톤 리소스에 대한 간략한 요약은 [링크](https://openshift.github.io/pipelines-docs/docs/0.10.5/con_pipelines-concepts.html)를 클릭해 읽을 수 있습니다.
또한 제가 미약하지만 개인적으로 정리한 [글](https://lcc3108.github.io/articles/2021-01/tekton-exlain-and-exaple)도 있습니다.
[node-web-app-pipeline-resources.yaml](pipeline/node-web-app-pipeline-resources.yaml): [Pipeline Resources](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md)는 Pipeline에 대한 설정입니다. git repo와 container image 두가지 타입이 있으며 해당리소스를 참고하여 주로 git repo를 읽어 container image를 만들어냅니다.

[node-web-app-pipeline.yaml](pipeline/node-web-app-pipeline.yaml): [Pipeline](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md)은 Node APP을 빌드, 배포합니다. 우리의 Pipeline에는 2개의 Task가 있는데 하나는 배포할 이미지를 만드며 다른하나는 ArgoCD cli를 사용해 쿠버네티스 클러스터에 배포합니다.


[node-web-app-triggertemplate.yaml](pipeline/node-web-app-triggertemplate.yaml):  외부소스(여기선 github 웹훅)을 어떤 방식으로 처리할지 정의한 리소스입니다.
- TriggerTemplate은 Pipeline Resource를 동적으로 생성합니다. 또한 PipelineRun 템플릿도 동적으로 생성합니다.

- TriggerBining은 이벤트 데이터(여기선 웹훅)를 TriggerTemplate에서 사용할 수 있도록 매핑시켜줍니다.

- EventListener는 웹훅을 받는 팟을 생성하여 이벤트 데이터를 파싱 혹은 검증하여 TriggerBining과 TriggerTemplate에 전달해 줍니다.


텍톤 리소스에 대한 예시는 [링크](https://github.com/tektoncd/pipeline/tree/master/docs#learn-more)를 참고하세요.


### ArgoCD 앱 생성 및 Tekton 리소스 설정

우리는 ArgoCD를 사용하여 Tekton을 배포 할 수 있습니다. 실제 프로젝트라면 pipline과 같은 리소스들은 분리된 레포지토리에 두는게 좋습니다. 

``` bash
kubectl apply -f - <<EOF
apiVersion: v1
items:
- apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: node-web-app
    namespace: argocd
  spec:
    destination:
      namespace: node-web-project
      server: https://kubernetes.default.svc
    project: default
    source:
      path: pipeline
      repoURL: https://github.com/lcc3108/node-web-app.git # 자신의 repoURL로 변경
      targetRevision: HEAD
    syncPolicy:
      syncOptions:
      - CreateNamespace=true
kind: List
EOF
```

위의 명령어를 실행시키면 아래와같이 ArgoCD APP이 만들어집니다.

![alt argo-pipeline](images/argo-node-web-pipeline.png) 

- Project: default
- cluster: URL
- namespace: namespace
- Targer Revision: Head
- PATH: pipeline
- repoURL: own repoURL

sync 버튼을 클릭하게되면 pipeline들이 배포됩니다.

![alt argo-pipeline](images/argocd-tekton-pipeline.png)

### OKD-console 설치(GUI 기능 제공)
아래 글에서 --base-address의 값을 http://localhost:9000 로 주면 okd콘솔이 설치된다.
[링크](https://lcc3108.github.io/articles/2021-01/operator-OLM-OKD-console#okd-console)
### Run a Build

At this point you can run a build.  The Build Should succeed, but the deploy should fail.  If you configure the deployment first however, deloyment will fail to start because the image has not been published.  

1. You can go into the Pipelines section of the OpenShift Console, right click the pipeline and click Start.

![alt kickoff](images/kickoffbuild.png)

2. You will see that the values are prepopulated with default PipelineResources as shown below.  


![alt default-resources](images/PipelineDefaultResouces.png)

3. The pipeline should run, the build should pass (Creates the Container Image and publishes it to the Container Registry).  The argo-cd sync should fail because we have not configured the argod app for deploying the node-web-project.  


![alt first-run](images/FirstRun.png)


### Examine Application 

Let's look at the OpenShift Configuration for our node application.

![alt deployment](images/deployment.png)

- [node-web-app-deployment.yaml](deployment/node-web-app-deployment.yaml) - This represents our Kubernetes Deployment. 

- [node-web-app-service.yaml](deployment/node-web-app-service.yaml):  This expose the node app to the cluster.

- [node-web-app-route.yaml](deployment/node-web-app-route.yaml): Exposes an OpenShift Route so you can access Node App from outside the cluster


### Create ArgoCD App for Web App Resources 

Just like we used argocd to deploy the tekton pipeline, you will create another argocd app that corresponds to the deployment.   [You can create an argo cd app via the GUI or commandline](https://argoproj.github.io/argo-cd/getting_started/).   

The screenshot below shows the parameters I entered.  You need to use your own forked git repo.  

![alt argo-pipeline](images/argo-node-web-app.png)

- Project: default
- cluster: (URL Of your OpenShift Cluster)
- namespace should be the name of your OpenShift Project
- repo url: should be your forked git repo.  
- Targer Revision: Head
- PATH: deployment
- AutoSync Disabled.  







### Sync Repo 
From here, you can trigger a sync manually by clicking sync.  Once your resources are deployed, your build from earlier is complete.  The screen should look like the figure below.  

![alt node-flow](images/argocd-node-flow.png)


### Run Pipeline 

1. You can go into the Pipelines section of the OpenShift Console, right click the pipeline and click Start.

![alt kickoff](images/kickoffbuild.png)

2. You will see that the values are prepopulated with default PipelineResources as shown below.  


![alt default-resources](images/PipelineDefaultResouces.png)

3. The pipeline should run, should now complete.  

![alt build-success](images/BuildSuccess.png)


### Configure Webhooks 

You will now need to configure 2 WebHooks.  

![alt webhooks](images/webhooks.png)

1. One WebHook will be configured to our argocd pipeline app.  This will enabled you to push changes to your pipeline plus for argocd to detect changes for your app (though autosync is not on)

    ![alt webhooks](images/argowebhook.png)


2. One webhook will go to your Tekton Event Listener to start a tekton build from git push

    ![alt webhooks](images/tekton-webhook.png)

### Make a code change and commit, look at build.   



1. Make a change to the deployment YAML.

    ![alt change-deployment](images/deployment-change.png)

2. Make a change to the Node JS Code.  


    ![alt webhooks](images/changecode.png)

3. Push the changes to your repo

```

git add .
git commit -m "First Deployment"
git push

```

    In a real deployment, you might have many webhooks.  git push can be build to dev while a git tag can be a build for test.  


3. Go to the OpenShift Console and you should see a pipleine run kickoff.  Wait till it is complete.  Notice the name of the pipleine run matches that in the tigger template.

    ![alt webhooks](images/gittrigger.png)

4. While waiting for the build, go to pipeline resources section and look to see new pipeline resources created for the webhook build.  It will dynamically create a resource for build so you know what parameters were used to run the build.  

    ![alt resourcebuild](images/pipelineresourcebuild.png)


5. Go back to the Pipeline Runs and check that the build is complete.  


    ![alt success](images/pipelinesuccess.png)


6. Go to the Topology view on the Developer Side and launch the app as shown.


    ![alt success](images/LaunchApp.png)


7.  If it all works out, your app should look like this




This completes loading the solution.


#### Troubleshooting

- argocd runs in an argocd namespace.  Yourtekton pipleine runs in your app namespace.  

- using tkn log <resource> intance to see tekton activity

- oc logs <resource> for various pods

- Redeploying TriggerTemplate does not cause the Event App to restart.  Delete pod to run new instance.  



