# FIAP Store no Amazon EKS - Global Solution

Cloud Developer II - Kubernetes e Serverless

**Nickolas Corazza Alves - RM562265**

A aplicacao (API, front-end, worker e jobs) veio pronta do professor. Este repositorio contem a **plataforma**: cluster EKS, manifestos Kubernetes, rede, armazenamento, seguranca, escalonamento e entrega por GitOps (Argo CD).

O README original da aplicacao esta em `docs/README-aplicacao.md`.

## Integrantes

| Nome | RM |
|---|---|
| Nickolas Corazza Alves | RM 562265 |

**Link de Apresentação:** `https://drive.google.com/file/d/1K72tSfzghaUMfQHApWgKqmaAOWtKQZjM/view?usp=sharing` 

**Application do Argo CD:** `https://github.com/ncorazza12/Docker_GS/blob/main/argocd/application.yaml` (nome `store`, namespace `argocd`, repositorio `https://github.com/ncorazza12/Docker_GS.git`, pasta `k8s/`, branch `main`, `prune` e `selfHeal` ligados).

## Arquitetura

```
Internet (HTTPS) -> [Traefik / LoadBalancer AWS]            namespace traefik
                         |
                  [Ingress store-ingress]                   namespace store-prod (restricted)
                   /api -> store-api (3-6, HPA)
                   /    -> store-front (2)
                         |
        store-api --> fila Redis "pedidos" --> store-worker (0-5, KEDA)
        db-migrate (Job, hook Argo)    store-report (CronJob)
                         |
          ======== NetworkPolicy: nega toda entrada ========
        store-db (StatefulSet + EBS)   store-redis          namespace store-data (baseline)

        node-logger (DaemonSet, 1 Pod por no)               namespace monitoring
        Argo CD sincroniza a pasta k8s/                     namespace argocd
```

## Estrutura

| Caminho | Conteudo |
|---|---|
| `cluster.yaml` | cluster EKS (eksctl) |
| `argocd/application.yaml` | Application do Argo CD (aplicada a mao; fica fora de `k8s/`) |
| `k8s/` | manifestos sincronizados pelo Argo CD |
| `avaliacao/trivy/` | relatorios do scan e justificativas |
| `avaliacao/verifica-gs.sh` | script de correcao do professor |
| `store-api/`, `store-worker/`, `store-front/` | codigo da aplicacao (so o Dockerfile do store-front foi ajustado, ver secao 2) |

## Manifestos (k8s/)

| Arquivo | O que cria | Itens |
|---|---|---|
| `00-namespaces.yaml` | store-prod (restricted), store-data (baseline), monitoring | C1, F4 |
| `01-storageclass.yaml` | StorageClass `ebs-gp3` (ebs.csi.aws.com) | D2 |
| `02-quota-limits.yaml` | ResourceQuota e LimitRange em store-prod | C7 |
| `03-rbac.yaml` | SA da API + ClusterRole `get nodes`; Role somente leitura para `fiap:devs` | F1, F3 |
| `04-networkpolicy.yaml` | nega toda entrada em store-data; libera API, worker, jobs e KEDA | C5, C6 |
| `05-configmap.yaml` | ConfigMap `store-api-config` | D3 |
| `10-store-db.yaml` | StatefulSet + Service do PostgreSQL com volume EBS | D1, C2 |
| `11-store-redis.yaml` | Deployment + Service do Redis | D6, C2 |
| `12-db-migrate.yaml` | Job `db-migrate` (hook Sync do Argo CD) | E1 |
| `20-store-api.yaml` | Deployment, Service, HPA (3-6, CPU 50%) e PDB (minAvailable 2) | B1-B7, D4, E3, F1, F5 |
| `21-store-worker.yaml` | Deployment do worker (sem `replicas`, sem token) | F2, F5 |
| `22-store-front.yaml` | Deployment e Service do front (sem token, emptyDir em /tmp) | F2, F5 |
| `23-ingress.yaml` | Ingress `store-ingress` com TLS | C3, C4 |
| `24-keda-scaledobject.yaml` | ScaledObject `store-worker` (Redis, 0-5) | G1 |
| `25-store-report-cronjob.yaml` | CronJob `store-report` | E2 |
| `30-node-logger-daemonset.yaml` | DaemonSet em todos os nos | D5 |

## Pre-requisitos

aws CLI v2, eksctl, kubectl, helm, docker (buildx), trivy, git. Acesso ao AWS Academy Learner Lab (role `LabRole`, sem OIDC/IRSA). O script de correcao e bash: rode no **Git Bash**.

## 1. Credenciais (toda sessao)

Start Lab > AWS Details > copiar o bloco `[default]` para `%USERPROFILE%\.aws\credentials` (fora do repositorio). Nunca versionar credenciais.

```powershell
aws sts get-caller-identity
```

## 2. Imagens no ECR (uma vez por conta)

Repositorios `store-api`, `store-worker` e `store-front` ja criados. Build e push, sempre com `--platform linux/amd64` e tag fixa (nunca `latest`):

```powershell
$REGISTRY = "367776931485.dkr.ecr.us-east-1.amazonaws.com"
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $REGISTRY
docker buildx build --platform linux/amd64 --build-arg APP_VERSION=1.0 -t "$REGISTRY/store-api:1.0" --push store-api
docker buildx build --platform linux/amd64 --build-arg APP_VERSION=2.0 -t "$REGISTRY/store-api:2.0" --push store-api
docker buildx build --platform linux/amd64 -t "$REGISTRY/store-worker:1.0" --push store-worker
docker buildx build --platform linux/amd64 -t "$REGISTRY/store-front:1.3" --push store-front
```

Se a conta do lab mudar, trocar o numero da conta em `cluster.yaml` e nos `image:` de `k8s/`.

### Scan de vulnerabilidades (item M3)

```powershell
New-Item -ItemType Directory -Force avaliacao\trivy | Out-Null
foreach ($i in "store-api:1.0","store-worker:1.0","store-front:1.3") {
  docker pull "$REGISTRY/$i"
  trivy image --format table -o "avaliacao\trivy\$($i.Split(':')[0]).txt" "$REGISTRY/$i"
}
```

Para cada CRITICAL, registrar correcao ou justificativa em `avaliacao/trivy/JUSTIFICATIVA.md`.

#### Correcao dos CRITICAL do store-front

O scan da `store-front:1.0` encontrou 4 CRITICAL (brotli-libs, libcrypto3, libssl3 e nginx, todos com versao corrigida). O `store-front/Dockerfile` passou a remover os pacotes `nginx-module-*`, rodar `apk upgrade` e instalar `nginx=1.28.3-r7`, e a imagem foi publicada como `store-front:1.3`. O novo scan nao mostra vulnerabilidades. Detalhes em `avaliacao/trivy/JUSTIFICATIVA.md`. Um rebuild futuro depende de esse pacote continuar no repositorio Alpine; a imagem 1.3 ja esta no ECR.

## 3. Criar o ambiente do zero (toda sessao)

### 3.1 Cluster (15 a 20 min)

```powershell
eksctl create cluster -f cluster.yaml
kubectl get nodes -L topology.kubernetes.io/zone     # 4 nos Ready em 3 zonas
```

### 3.2 Componentes de plataforma

```powershell
helm repo add traefik https://traefik.github.io/charts
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm upgrade --install traefik traefik/traefik -n traefik --create-namespace `
  --set deployment.replicas=2 --set providers.kubernetesIngress.publishedService.enabled=true --wait
helm upgrade --install keda kedacore/keda -n keda --create-namespace --wait
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

`publishedService.enabled=true` faz o Traefik publicar o endereco no status do Ingress; sem isso o Argo CD nao marca a Application como Healthy.

### 3.3 Namespaces e Secrets (fora do Git)

```powershell
kubectl apply -f k8s\00-namespaces.yaml

$b = New-Object byte[] 18
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
$pw = [Convert]::ToBase64String($b) -replace '[+/=]','x'
kubectl -n store-prod create secret generic store-db-credentials --from-literal=password=$pw
kubectl -n store-data create secret generic store-db-credentials --from-literal=password=$pw
Remove-Variable pw
```

A mesma senha nos dois namespaces. O Postgres so le a senha na criacao do banco: nao recriar o Secret com outro valor depois.

Certificado TLS autoassinado. O CN tem limite de 64 caracteres e o hostname do balanceador e maior, entao o CN e curto e o hostname vai no subjectAltName:

```powershell
$LB = kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
$env:MSYS_NO_PATHCONV = 1
& "C:\Program Files\Git\usr\bin\openssl.exe" req -x509 -nodes -newkey rsa:2048 -days 30 `
  -keyout "$env:TEMP\tls.key" -out "$env:TEMP\tls.crt" -subj "/CN=fiap-store" -addext "subjectAltName=DNS:$LB"
kubectl -n store-prod create secret tls store-tls --cert="$env:TEMP\tls.crt" --key="$env:TEMP\tls.key"
Remove-Item "$env:TEMP\tls.key","$env:TEMP\tls.crt"
```

### 3.4 GitOps

```powershell
kubectl apply -f argocd\application.yaml
kubectl -n argocd get application store -w          # Synced / Healthy
```

### 3.5 Rollout 1.0 para 2.0 (obrigatorio no cluster da demonstracao)

O repositorio deve estar com `store-api:1.0` em `k8s/20-store-api.yaml` quando o cluster novo subir. Depois que o Argo CD estiver Healthy:

```powershell
(Get-Content k8s\20-store-api.yaml) -replace 'store-api:1.0','store-api:2.0' | Set-Content k8s\20-store-api.yaml -Encoding ascii
git commit -am "Rollout store-api 1.0 -> 2.0"; git push
kubectl -n argocd annotate application store argocd.argoproj.io/refresh=normal --overwrite
kubectl -n store-prod get rs -l app=store-api       # 2 ReplicaSets
```

Para a proxima sessao, voltar a tag para `1.0` e fazer commit antes de recriar o cluster.

### 3.6 Verificacao

```Powershell
# Git Powershell, na raiz do repositorio
$env:MSYS_NO_PATHCONV=1; & "C:\Program Files\Git\bin\bash.exe" avaliacao/verifica-gs.sh https://<endereco-do-balanceador>
```

## 4. Destruir todo o ambiente (nesta ordem)

```powershell
kubectl -n argocd delete application store            # senao o Argo recria tudo
kubectl delete namespace store-prod store-data monitoring
kubectl get pv                                         # repetir ate "No resources found"
helm uninstall traefik -n traefik                      # remove o balanceador
eksctl delete cluster -f cluster.yaml --disable-nodegroup-eviction
```

Conferir no console AWS (us-east-1): EC2 > Volumes e EC2 > Load Balancers sem recursos do projeto; CloudFormation sem stacks `eksctl-fiap-store-*`.

## 5. Desafios ao vivo - comandos

| # | Desafio | Comando |
|---|---|---|
| 1 | Self-healing | `kubectl -n store-prod delete (kubectl -n store-prod get pod -l app=store-api -o name \| Select-Object -First 1)` e `kubectl -n store-prod get pods -l app=store-api -w` |
| 2 | Manutencao de no | `kubectl -n store-data get pod store-db-0 -o wide`; `kubectl drain <outro-no> --ignore-daemonsets --delete-emptydir-data`; depois `kubectl uncordon <outro-no>` |
| 3 | Rollback por GitOps | trocar a tag em `k8s/20-store-api.yaml`, commit e push; o selo da pagina mostra a versao |
| 4 | Escala a zero | 50 pedidos no painel; `kubectl -n store-prod get pods -l app=store-worker -w` |
| 5 | Persistencia | `kubectl -n store-data delete pod store-db-0` |
| 6 | Pod Security | `kubectl run priv -n store-prod --image=busybox:1.36 --privileged -- sleep 3600` (recusado) |
| 7 | RBAC | `kubectl auth can-i list pods -n store-prod --as=dev --as-group=fiap:devs` (yes); `delete pods`, `get secrets`, `create pods --subresource=exec` (no) |
| 8 | Drift | `kubectl -n store-prod delete service store-front` (o Argo CD recria) |
| 9 | Rede | `kubectl -n store-prod exec deploy/store-front -- nc -z -w 3 store-db.store-data.svc.cluster.local 5432` (falha); a store-api conecta |
| 10 | Relatorio | `kubectl -n store-prod create job --from=cronjob/store-report relatorio-manual-1` |

## 6. Seguranca

- Nenhum segredo no Git: senha do banco e certificado TLS sao criados com `kubectl create secret`.
- `store-prod` com Pod Security `restricted` (enforce); todos os conteineres com `runAsNonRoot`, `seccompProfile: RuntimeDefault`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false` e `capabilities.drop: ["ALL"]`.
- Token da API do Kubernetes montado apenas na `store-api`, cuja ServiceAccount so pode `get nodes`.
- `store-data` sem entrada por padrao; so API, worker, jobs e o operador do KEDA alcancam banco e Redis.

## 7. Uso de IA

| Ferramenta | Para que |
|---|---|
| Claude (Anthropic) | apoio na leitura do enunciado, no planejamento, na escrita e revisao dos manifestos, do README e na resolucao de erros do ambiente Windows |

Todos os manifestos foram revisados pelo grupo, e cada integrante deve conseguir explicar cada arquivo de `k8s/`.
