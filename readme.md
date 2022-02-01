## Example to set up MetalLB on local cluster with Nginx demo

# Install metallb

helm install metallb metallb/metallb -f values.yaml --namespace=metallb-system
