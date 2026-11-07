# FIAP Store no Amazon EKS - Global Solution

**Integrantes:** Nickolas Corazza Alves (RM ____) | (demais integrantes e RM)
**Argo CD Application:** `argocd/application.yaml` (nome `store`, namespace `argocd`)
**Uso de IA:** Claude (Anthropic) para gerar e revisar os manifestos e este README. Todo o grupo revisou e consegue explicar cada arquivo.

Estrutura: `k8s/` (sincronizada pelo Argo CD), `argocd/application.yaml`, `cluster.yaml`, `avaliacao/`.

## 1. Criar o ambiente do zero (PowerShell)

Pre-requisitos: aws, kubectl, helm, eksctl, docker, git. Imagens ja publicadas no ECR (`store-api:1.0` e `:2.0`, `store-worker:1.0`, `store-front:1.0`).

```powershell
# 1. Credenciais do Learner Lab em %USERPROFILE%\.aws\credentials
aws sts get-caller-identity

# 2. Cluster (15 a 20 min)
eksctl create cluster -f cluster.yaml
aws eks update-kubeconfig --name fiap-store --region us-east-1
kubectl get nodes -L topology.kubernetes.io/zone

# 3. Componentes de plataforma
helm repo add traefik https://traefik.github.io/charts
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik -n traefik --create-namespace `
  --set deployment.replicas=2 --set providers.kubernetesIngress.publishedService.enabled=true --wait
helm upgrade --install keda kedacore/keda -n keda --create-namespace --wait
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 4. Namespaces (antes dos Secrets)
kubectl apply -f k8s/00-namespaces.yaml

# 5. Secrets (valores NUNCA vao para o Git) - ver secao 2

# 6. Entregar ao Argo CD
kubectl apply -f argocd/application.yaml
kubectl -n argocd get application store -w
```

## 2. Criar os Secrets (sem mostrar valores)

Senha do banco (a mesma nos dois namespaces; o Postgres so le a senha na criacao do banco):

```powershell
$b = New-Object byte[] 18
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
$pw = [Convert]::ToBase64String($b) -replace '[+/=]','x'
kubectl -n store-prod create secret generic store-db-credentials --from-literal=password=$pw
kubectl -n store-data create secret generic store-db-credentials --from-literal=password=$pw
Remove-Variable pw
```

Certificado TLS autoassinado com o hostname do balanceador no subjectAltName (o arquivo fica em %TEMP%, fora do repositorio):

```powershell
$LB = kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
$env:MSYS_NO_PATHCONV = 1
$ossl = "C:\Program Files\Git\usr\bin\openssl.exe"
& $ossl req -x509 -nodes -newkey rsa:2048 -days 30 -keyout "$env:TEMP\tls.key" -out "$env:TEMP\tls.crt" -subj "/CN=$LB" -addext "subjectAltName=DNS:$LB"
kubectl -n store-prod create secret tls store-tls --cert="$env:TEMP\tls.crt" --key="$env:TEMP\tls.key"
Remove-Item "$env:TEMP\tls.key","$env:TEMP\tls.crt"
```

## 3. Verificar

```powershell
# Git Bash (vem com o Git for Windows), na raiz do repositorio:
bash ./avaliacao/verifica-gs.sh https://$LB
```

## 4. Destruir tudo (nesta ordem)

```powershell
kubectl -n argocd delete application store          # senao o Argo recria tudo
kubectl delete namespace store-prod store-data monitoring
kubectl get pv                                       # esperar ficar vazio (volume EBS some em segundo plano)
helm uninstall traefik -n traefik                    # remove o balanceador
eksctl delete cluster -f cluster.yaml --disable-nodegroup-eviction
# conferir no console: sem volumes EBS e sem load balancers sobrando
```

## 5. Rollout 1.0 -> 2.0 e rollback (por commit)

Editar a tag em `k8s/20-store-api.yaml` (`store-api:1.0` -> `store-api:2.0`), commit e push. O Argo CD aplica sozinho. Para voltar, `git revert` do commit.
