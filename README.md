# Todo Application on Kubernetes Cluster

This is my simple application, and I've intentionally implemented it with `over-engineering` in order to have hands-on experience with various technologies..

What is `over-engineering` in my opinion?

- A simple application split into microservices for the `frontend`, `backend`, and `database`.
- The application is deployed in a `Kubernetes Cluster` on the `Google Cloud Platform`.
- Provisioning of the GKE Cluster and other GCP services using `Terraform`.
- Deployment of metrics monitoring with `Grafana` and `Prometheus`.
- Deployment of logging monitoring with `ECK (Elastic Cloud Kubernetes)` using `Elasticsearch`, `Filebeat`, and `Kibana`.
- Automate build and deploy with `Jenkins` and `ArgoCD`

## Source Code

- [Terraform](https://github.com/DarNattp/gke-tf.git)
- [GitOps K8s manifest](https://github.com/DarNattp/todo-k8s-prod.git)
- [Backend with Go](https://github.com/DarNattp/todo-server.git)
- [Frontend with ReactJS](https://github.com/DarNattp/todo-client.git)

## Architecture Diagram

![System_Diagram](https://github.com/DarNattp/todo-devops/blob/master/images/System_Diagram.png?raw=true)

## Todo Application

**Frontend:** ReactJS

**Backend:** Go

**Database:** MySQL

**Overview website:** https://todo.natapat.me/

#### **Note: The project uses the Free Trial credit available on GCP. The website became unavailable when the credit was exhausted.**

![web-todo](https://github.com/DarNattp/todo-devops/blob/master/images/todo-app1.png?raw=true)

```sh {"id":"01HTAWBW8DPAE7NYZ007YMSD9A"}
NAME                                 READY   STATUS    RESTARTS   AGE
pod/backend-depl-7968dcff7d-r46h6    1/1     Running   0          20h
pod/frontend-depl-855c766b86-ss4s2   1/1     Running   0          20h
pod/maria-db-depl-8585d678c8-8h4wb   1/1     Running   0          20h

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/backend-srv    ClusterIP   10.52.15.231   <none>        5050/TCP   20h
service/frontend-srv   ClusterIP   10.52.2.226    <none>        80/TCP     20h
service/mariadb-srv    ClusterIP   10.52.4.107    <none>        3306/TCP   20h

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-depl    1/1     1            1           20h
deployment.apps/frontend-depl   1/1     1            1           20h
deployment.apps/maria-db-depl   1/1     1            1           20h

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-depl-7968dcff7d    1         1         1       20h
replicaset.apps/frontend-depl-855c766b86   1         1         1       20h
replicaset.apps/maria-db-depl-8585d678c8   1         1         1       20h
```

### frontend-deployment.yml

```yaml {"id":"01HTAWBW8DPAE7NYZ009S0CP0V"}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-frontend
  template:
    metadata:
      labels:
        app: todo-frontend
    spec:
      restartPolicy: Always
      containers:
      - name: frontend-depl
        image: darnattp/basic-k8s-todo-client:latest
        resources:
          limits:
            memory: "32Mi"
            cpu: "10m"
        ports:
        - containerPort: 80
      tolerations:
      - key: "kubernetes.io/arch"
        operator: "Equal"
        value: "arm64"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/gke-nodepool
                operator: In
                values:
                - arm64-pool          

```

### backend-deployment.yml

```yaml {"id":"01HTAWBW8DPAE7NYZ00CAGPH74"}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-depl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-backend
  template:
    metadata:
      labels:
        app: todo-backend
    spec:
      restartPolicy: Always
      containers:
      - name: backend-depl
        image: darnattp/basic-k8s-todo-server:latest
        resources:
          limits:
            memory: "32Mi"
            cpu: "20m" 
        ports:
        - containerPort: 5050
        envFrom:
          - configMapRef: 
              name: todo-app-configmap
          - secretRef:
              name: todo-app-secret
      tolerations:
      - key: "kubernetes.io/arch"
        operator: "Equal"
        value: "arm64"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/gke-nodepool
                operator: In
                values:
                - arm64-pool    
```

## CI/CD Flow

![System_Diagram](https://github.com/DarNattp/todo-devops/blob/master/images/CICD.png?raw=true)

![jenkins1](https://github.com/DarNattp/todo-devops/blob/master/images/jenkins1.png?raw=true)

### Jenkinsfile

#### [Frontend] 1st job - build/push image and then trigger next job

```groovy {"id":"01HTAWBW8DPAE7NYZ00FY7MTZM"}
node {
    def app
    stage('Clone repository') {
        checkout scm
    }

    stage('Build image') {
       app = docker.build("darnattp/basic-k8s-todo-client")
    }

    stage('Test image') {
        app.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push image') {
        docker.withRegistry('https://registry.hub.docker.com', 'darnattp-dockerhub') {
            app.push("${env.BUILD_NUMBER}")
        }
    }
    
    stage('Trigger ManifestUpdate') {
                echo "trigger updatemanifestjob"
                build job: 'frontend-updatemanifest', parameters: [string(name: 'DOCKERTAG', value: env.BUILD_NUMBER)]
        }
}
```

#### [Frontend] 2nd job - update tag on git deployment manifest file

```groovy {"id":"01HTAWBW8DPAE7NYZ00JH97CM5"}
node {
    def app

    stage('Clone repository') {
        checkout scm
    }

    stage('Update GIT') {
            script {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withCredentials([usernamePassword(credentialsId: 'darnattp-github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        //def encodedPassword = URLEncoder.encode("$GIT_PASSWORD",'UTF-8')
                        sh "git config user.email $GIT_EMAIL"
                        sh "git config user.name $GIT_USERNAME"
                        //sh "git switch master"
                        sh "cat frontend/frontend-deployment.yaml"
                        sh "sed -i 's+darnattp/basic-k8s-todo-client.*+darnattp/basic-k8s-todo-client:${DOCKERTAG}+g' frontend/frontend-deployment.yaml"
                        sh "cat frontend/frontend-deployment.yaml"
                        sh "git add ."
                        sh "git commit -m 'Done by Jenkins Job frontend-updatemanifest: ${env.BUILD_NUMBER}'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/todo-k8s-prod.git HEAD:master"
      }
    }
  }
}
}
```

#### [Backend] 1st job - build/push image and then trigger next job

```groovy {"id":"01HTAWBW8DPAE7NYZ00KXNZT57"}
node {
    def app
    stage('Clone repository') {
        checkout scm
    }

    stage('Build image') {
       app = docker.build("darnattp/basic-k8s-todo-server")
    }

    stage('Test image') {
        app.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push image') {
        docker.withRegistry('https://registry.hub.docker.com', 'darnattp-dockerhub') {
            app.push("${env.BUILD_NUMBER}")
        }
    }
    
    stage('Trigger ManifestUpdate') {
                echo "trigger updatemanifestjob"
                build job: 'backend-updatemanifest', parameters: [string(name: 'DOCKERTAG', value: env.BUILD_NUMBER)]
        }
}
```

#### [Backend] 2nd job - update tag on git deployment manifest file

```groovy {"id":"01HTAWBW8DPAE7NYZ00N6NQC35"}
node {
    def app

    stage('Clone repository') {
        checkout scm
    }

    stage('Update GIT') {
            script {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withCredentials([usernamePassword(credentialsId: 'darnattp-github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        //def encodedPassword = URLEncoder.encode("$GIT_PASSWORD",'UTF-8')
                        sh "git config user.email $GIT_EMAIL"
                        sh "git config user.name $GIT_USERNAME"
                        //sh "git switch master"
                        sh "cat backend/backend-deployment.yaml"
                        sh "sed -i 's+darnattp/basic-k8s-todo-server.*+darnattp/basic-k8s-todo-server:${DOCKERTAG}+g' backend/backend-deployment.yaml"
                        sh "cat backend/backend-deployment.yaml"
                        sh "git add ."
                        sh "git commit -m 'Done by Jenkins Job backend-updatemanifest: ${env.BUILD_NUMBER}'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/todo-k8s-prod.git HEAD:master"
      }
    }
  }
}
}
```

## ArgoCD

Install by `kubectl`

```sh {"id":"01HTAWBW8DPAE7NYZ00S6GP153"}
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```sh {"id":"01HTAWBW8DPAE7NYZ00TG7HJ8W"}
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                    1/1     Running   0          34h
pod/argocd-applicationset-controller-6ccb885cc-fkhjx   1/1     Running   0          34h
pod/argocd-dex-server-547dfc6dc9-4btmq                 1/1     Running   0          34h
pod/argocd-notifications-controller-77bffb68cc-gv548   1/1     Running   0          34h
pod/argocd-redis-76dff756d7-99h6v                      1/1     Running   0          34h
pod/argocd-repo-server-69c577765c-kqsn9                1/1     Running   0          34h
pod/argocd-server-6dfb99877b-zmt8z                     1/1     Running   0          34h

NAME                                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.52.15.23    <none>        7000/TCP,8080/TCP            7d
service/argocd-dex-server                         ClusterIP   10.52.2.62     <none>        5556/TCP,5557/TCP,5558/TCP   7d
service/argocd-metrics                            ClusterIP   10.52.7.189    <none>        8082/TCP                     7d
service/argocd-notifications-controller-metrics   ClusterIP   10.52.13.223   <none>        9001/TCP                     7d
service/argocd-redis                              ClusterIP   10.52.13.36    <none>        6379/TCP                     7d
service/argocd-repo-server                        ClusterIP   10.52.14.32    <none>        8081/TCP,8084/TCP            7d
service/argocd-server                             ClusterIP   10.52.4.1      <none>        80/TCP,443/TCP               7d
service/argocd-server-metrics                     ClusterIP   10.52.5.24     <none>        8083/TCP                     7d

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           7d
deployment.apps/argocd-dex-server                  1/1     1            1           7d
deployment.apps/argocd-notifications-controller    1/1     1            1           7d
deployment.apps/argocd-redis                       1/1     1            1           7d
deployment.apps/argocd-repo-server                 1/1     1            1           7d
deployment.apps/argocd-server                      1/1     1            1           7d

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-6ccb885cc   1         1         1       7d
replicaset.apps/argocd-dex-server-547dfc6dc9                 1         1         1       7d
replicaset.apps/argocd-notifications-controller-77bffb68cc   1         1         1       7d
replicaset.apps/argocd-redis-76dff756d7                      1         1         1       7d
replicaset.apps/argocd-repo-server-69c577765c                1         1         1       7d
replicaset.apps/argocd-server-5c67dbfcbb                     0         0         0       7d
replicaset.apps/argocd-server-6dfb99877b                     1         1         1       7d

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     7d
```

![argocd1](https://github.com/DarNattp/todo-devops/blob/master/images/argocd1.png?raw=true)

## Logging stack with Elasticsearch, Filebeat and Kibana

![elastic1](https://github.com/DarNattp/todo-devops/blob/master/images/elastic1.png?raw=true)

```sh {"id":"01HTAWBW8DPAE7NYZ00TW1GA12"}
NAME                             READY   STATUS    RESTARTS      AGE
pod/elasticsearch-es-default-0   1/1     Running   0             33h
pod/elasticsearch-es-default-1   1/1     Running   0             33h
pod/filebeat-hsszp               1/1     Running   9 (27h ago)   33h
pod/filebeat-lqbd4               1/1     Running   0             33h
pod/kibana-kb-67b4bcb8bb-bvnpf   1/1     Running   0             32h

NAME                                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/elasticsearch-es-default         ClusterIP   None          <none>        9200/TCP   2d21h
service/elasticsearch-es-http            ClusterIP   10.52.15.45   <none>        9200/TCP   2d21h
service/elasticsearch-es-internal-http   ClusterIP   10.52.6.251   <none>        9200/TCP   2d21h
service/elasticsearch-es-transport       ClusterIP   None          <none>        9300/TCP   2d21h
service/kibana-kb-http                   ClusterIP   10.52.1.26    <none>        5601/TCP   32h

NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/filebeat   2         2         2       2            2           <none>          2d20h

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kibana-kb   1/1     1            1           32h

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/kibana-kb-67b4bcb8bb   1         1         1       32h

NAME                                        READY   AGE
statefulset.apps/elasticsearch-es-default   2/2     2d21h
```

## Metrics monitoring stack with Grafana and Prometheus

![node1](https://github.com/DarNattp/todo-devops/blob/master/images/node1.png?raw=true)

![pod1](https://github.com/DarNattp/todo-devops/blob/master/images/pod1.png?raw=true)

```sh {"id":"01HTAWBW8DPAE7NYZ00XN5S4W8"}
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/grafana-576bdb85c-vk52m                             1/1     Running   0          33h
pod/prometheus-alertmanager-0                           1/1     Running   0          33h
pod/prometheus-kube-state-metrics-6fcf5978bf-tqn6g      1/1     Running   0          33h
pod/prometheus-prometheus-node-exporter-hglrf           1/1     Running   0          33h
pod/prometheus-prometheus-node-exporter-nhnxj           1/1     Running   0          33h
pod/prometheus-prometheus-pushgateway-fdb75d75f-rnjkr   1/1     Running   0          33h
pod/prometheus-server-5fcdf7c6f8-h97zl                  2/2     Running   0          33h

NAME                                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/grafana                               ClusterIP   10.52.11.87    <none>        80/TCP     5d20h
service/prometheus-alertmanager               ClusterIP   10.52.12.173   <none>        9093/TCP   5d20h
service/prometheus-alertmanager-headless      ClusterIP   None           <none>        9093/TCP   5d20h
service/prometheus-kube-state-metrics         ClusterIP   10.52.11.19    <none>        8080/TCP   5d20h
service/prometheus-prometheus-node-exporter   ClusterIP   10.52.14.111   <none>        9100/TCP   5d20h
service/prometheus-prometheus-pushgateway     ClusterIP   10.52.10.38    <none>        9091/TCP   5d20h
service/prometheus-server                     ClusterIP   10.52.11.255   <none>        80/TCP     5d20h

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-prometheus-node-exporter   2         2         2       2            2           <none>          5d20h

NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana                             1/1     1            1           5d20h
deployment.apps/prometheus-kube-state-metrics       1/1     1            1           5d20h
deployment.apps/prometheus-prometheus-pushgateway   1/1     1            1           5d20h
deployment.apps/prometheus-server                   1/1     1            1           5d20h

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-576bdb85c                             1         1         1       5d20h
replicaset.apps/prometheus-kube-state-metrics-6fcf5978bf      1         1         1       5d20h
replicaset.apps/prometheus-prometheus-pushgateway-fdb75d75f   1         1         1       5d20h
replicaset.apps/prometheus-server-5fcdf7c6f8                  1         1         1       5d20h

NAME                                       READY   AGE
statefulset.apps/prometheus-alertmanager   1/1     5d20h
```

## Ingress Controller with Traefik

I use `Traefik` in my Kubernetes Cluster because it allows me to utilize the `Let's Encrypt` plugin for the DNS01 challenge without the need for `cert-manager`.

```ini {"id":"01HTAWBW8DPAE7NYZ00XQV1565"}
- --entrypoints.websecure.http.tls=true
- --entrypoints.websecure.http.tls.certResolver=cloudflare
- --certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare
- --certificatesresolvers.cloudflare.acme.email=XXXXXX@gmail.com
- --certificatesresolvers.cloudflare.acme.dnschallenge.resolvers=1.1.1.1
- --certificatesresolvers.cloudflare.acme.storage=/ssl-certs/acme-cloudflare.json
- --providers.kubernetesingress.ingressendpoint.ip=X.X.X.X
```
