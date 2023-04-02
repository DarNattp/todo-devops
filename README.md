
# Todo Application on Kubernetes Cluster

This is my simple application, and I've intentionally implemented it with `over-engineering` in order to have hands-on experience with various technologies.

What is `over-engineering` in my opinion?

- A simple application split into microservices for the `frontend`, `backend`, and `database`.
- The application is deployed in a `Kubernetes Cluster` on the `Google Cloud Platform`.
- Provisioning of the GKE Cluster and other GCP services using `Terraform`.
- Deployment of metrics monitoring with `Grafana` and `Prometheus`.
- Deployment of logging monitoring using `Elasticsearch`, `Filebeat`, and `Kibana`.
## Architecture Diagram

![System_Diagram](https://github.com/DarNattp/todo-devops/blob/master/images/System_Diagram.png?raw=true)


## Todo Application
## CI/CD Flow

![System_Diagram](https://github.com/DarNattp/todo-devops/blob/master/images/CICD.png?raw=true)
## Source Code

- [Terraform](https://linktodocumentation)


## Horizontal Pod Autoscaling (HPA) 


## Running Tests

To run tests, run the following command

```bash
npm run test
```
```zsh
npm run test
```
