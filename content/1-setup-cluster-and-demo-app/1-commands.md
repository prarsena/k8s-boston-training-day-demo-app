## 1. Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

## Install Helm

```bash {"terminalRows":"32"}
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## 2. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/stable.txt"
KUBECTL_VERSION=$(cat stable.txt)
curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
rm stable.txt
kubectl version --client
```

## 3. Create a kind cluster with 2 nodes, a control plane and a worker node

```bash
kind create cluster --name training --config /workspaces/k8s-boston-training-day-demo-app/misc/kind-config.yaml
```

## 4. Check the cluster

```bash
kubectl get nodes
kubectl cluster-info
```

## 5. Install bash auto completion

```bash
sudo apt-get install -y bash-completion
```

## 6. Add kubectl auto completion and alias

```bash
#Add completion and alias
cat <<'EOF' >> ~/.bashrc
alias k=kubectl
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
alias kpa='kubectl get pods -A'
EOF
source ~/.bashrc

```

## 7. Try it!

```bash
k get pods -A
k get svc -A
```

## 8. Let's install a demo application with a eCommerce product listing page

```bash
kubectl create ns demo-app
kubectl apply -f https://raw.githubusercontent.com/odigos-io/simple-demo/main/kubernetes/deployment.yaml -n demo-app
```

## 9. Check the pods

```bash
k get pods -A
```

## 10. Here are the services

```bash
k get svc -A
```

## 11. We forward the service to port 8080

```bash
k port-forward --address 0.0.0.0 svc/frontend -n demo-app 8080:8080 &
```

