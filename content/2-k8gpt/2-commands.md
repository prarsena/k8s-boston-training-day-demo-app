Download K8sGPT

```bash
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/download/v0.4.31/k8sgpt_amd64.deb
```

Install K8sGPT

```bash
sudo dpkg -i k8sgpt_amd64.deb
```

See what k8sgpt finds about our cluster

```bash
k8sgpt analyzek8sgpt analyze
```

Setup Gemini with a Key

```bash
k8sgpt auth add --backend google --model gemini-2.5-flash --password $GEMINI_API_KEY
```

Google Gemini is the default model

```bash
k8sgpt auth list
```

Use AI to explain what it found

```bash
k8sgpt analyze --explain --backend google
```

Anonymize... Yes, AI is public unless you ask it not to be

```bash
k8sgpt analyse --explain --backend google --interactive  --anonymize
```

Let's create a cpu hogger pod

```bash
kubectl run pending-pod --image=nginx --overrides='{"spec":{"containers":[{"name":"cpu-hog","image":"nginx","resources":{"requests":{"cpu":"99999"}}}]}}'
```

Let's see what it found

```bash
k8sgpt analyse --explain --backend google --interactive  --anonymize
```

Delete the pb

```bash
kubectl delete pod pending-pod -n default
```

Let's create a broken app

```bash
kubctl apply -f broken-app.yaml
```

Unleash k8sgpt

```bash
k8sgpt analyse --explain --backend google --interactive  --anonymize
```

Delete the broken app

```bash
kubctl delete -f broken-app.yaml
```