# Aula 05

Para esta aula, por causa do Prometheus, vamos precisar rodar o Minikube com configurações especificas:

```bash
minikube delete
minikube start --driver=kvm2 \
--kubernetes-version=v1.27.0 \
--memory=6g \
--bootstrapper=kubeadm \
--extra-config=kubelet.authentication-token-webhook=true \
--extra-config=kubelet.authorization-mode=Webhook \
--extra-config=scheduler.bind-address=0.0.0.0 \
--extra-config=controller-manager.bind-address=0.0.0.0
minikube addons enable ingress
```

> **Atenção** ao parâmetro --driver pode ser hyperv para Windows, qemu ou virtualbox para Mac, etc.

## Prometheus Manual

Comandos para criar o `Service`, `Ingress`, `ConfigMap` e a `ClusterRoleBinding`:

```bash
# namespace
kubectl create ns monitoring

# prometheus
kubectl create -f manual/prometheus-deploy.yml -f manual/prometheus-clusterrole.yml
kubectl -n monitoring expose deploy/prometheus
kubectl -n monitoring create cm prometheus-conf --from-file manual/prometheus.yml 
kubectl -n monitoring create clusterrolebinding --clusterrole=prometheus --serviceaccount=monitoring:default prometheus
# grafana
kubectl -n monitoring create deploy grafana --image docker.io/grafana/grafana --port 3000
kubectl -n monitoring expose deploy/grafana

# ingress
MINIKUBE_IP=$(minikube ip | tr . -)
kubectl -n monitoring create ingress prometheus --rule="prometheus.$MINIKUBE_IP.nip.io/*=prometheus:9090"
kubectl -n monitoring create ingress grafana --rule="grafana.$MINIKUBE_IP.nip.io/*=grafana:3000"
```

Utilizar a dashboard [https://grafana.com/grafana/dashboards/15759-kubernetes-views-nodes/](https://grafana.com/grafana/dashboards/15759-kubernetes-views-nodes/).

### Rules

Criaremos o arquivo de rules e modificaremos o `deploy` do prometheus para montá-lo:

```bash
kubectl -n monitoring create cm prometheus-rules --from-file manual/prometheus-rules.yml
```

```yaml
      containers:
        volumeMounts:
        ...
        - mountPath: /etc/prometheus/prometheus-rules.yml
          name: prometheus-rules
          subPath: prometheus-rules.yml
      volumes:
      ...
      - configMap:
          name: prometheus-rules
        name: prometheus-rules
```

Agora precisaremos alterar o `configMap` já existente para carregar o arquivo de rules:

```yaml
# adicionar ao final do arquivo
rule_files:
  - /etc/prometheus/prometheus-rules.yml
```

```bash
kubectl -n monitoring create cm prometheus-conf --from-file manual/prometheus.yml
kubectl delete pods --all -n monitoring
```

## Prometheus Operator

[https://github.com/prometheus-operator/kube-prometheus/](https://github.com/prometheus-operator/kube-prometheus/)

```bash
git clone https://github.com/prometheus-operator/kube-prometheus/
cd kube-prometheus
git checkout release-0.13

kubectl apply --server-side -f manifests/setup
kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
kubectl apply -f manifests/
```

## Aplicação

```bash
kubectl create cm nginx-conf --from-file=lua-app/nginx.conf
kubectl create secret generic mysql-env --from-env-file lua-app/mysql.env

kubectl create -f lua-app/lua-deploy.yml 
kubectl create -f lua-app/mysql-deploy.yml 

MINIKUBE_IP=$(minikube ip | tr . -)
kubectl create ingress lua --rule="lua.$MINIKUBE_IP.nip.io/*=lua:8080"
```
