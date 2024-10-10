# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# Update helm repository
helm repo update

# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
1.
  helm show values kubernetes-dashboard/kubernetes-dashboard >> values.yaml
  helm install kubernetes-dashboard --namespace kubernetes-dashboard - f values.yaml
2.
  helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard