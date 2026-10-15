# NGINX no OpenShift — deploy, PVC e HPA

Exemplo simples para o trabalho de Cloud Builders: NGINX serve uma página HTML gravada em um volume persistente por um initContainer. Não é necessário construir uma imagem nem instalar Docker.

## Arquivos

| Arquivo | Objetivo |
| --- | --- |
| `1-pvc.yaml` | Solicitar 1 GiB de storage (bônus 2) |
| `2-deployment.yaml` | Executar NGINX e inicializar a página no PVC |
| `3-expose.yaml` | Service e Route HTTPS para acessar a página |
| `4-hpa.yaml` | Escalar de 1 a 4 réplicas por CPU (bônus 1) |
| [docs/relatorio.md](docs/relatorio.md) | Modelo separado para entrega com prints |

## Pré-requisitos

Cluster OpenShift, CLI `oc`, acesso a um projeto, download de imagens do Docker Hub e uma StorageClass padrão com provisionamento dinâmico. Para HPA, o cluster precisa fornecer métricas em `metrics.k8s.io`. A SCC padrão deve permitir o volume e atribuir o grupo de acesso ao PVC; não fixamos UID nem fsGroup, pois os intervalos permitidos variam por projeto.

## Passo 1 — projeto e PVC

Execute na raiz do repositório:

```bash
oc login https://API-DO-SEU-CLUSTER:6443
oc new-project trabalho-cloudbuilders
# Se já tem um projeto, use: oc project NOME-DO-PROJETO
oc apply -f 1-pvc.yaml
oc get pvc web-pvc
```

**Print:** criação do PVC. Ele pode ficar `Pending` até criar o Deployment se a StorageClass usar `WaitForFirstConsumer`. Após o deploy, registre o status `Bound` e o nome do PV:

```bash
oc get pvc web-pvc -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,PV:.spec.volumeName
```

O PVC solicita armazenamento; o provisionador do cluster cria o PersistentVolume. Não é necessário um `pv.yaml` estático.

## Passo 2 — Deployment

```bash
oc apply -f 2-deployment.yaml
oc rollout status deployment/web-app --timeout=180s
oc get pods -l app=web-app -o wide
```

**Print:** pod `Running` e trecho do Deployment com initContainer e volumeMounts.

Usamos `nginxinc/nginx-unprivileged:stable-alpine`, preparado para execução sem root na porta 8080. O initContainer cria `index.html` somente se ele não existir, preservando alterações após reiniciar pods. NGINX monta o PVC apenas para leitura; `/tmp` é um volume temporário para PID e arquivos internos. A tag `stable-alpine` acompanha atualizações; para reprodução exata, fixe o digest da imagem utilizada na execução.

O PVC é `ReadWriteOnce`: pode ser usado por vários pods no **mesmo nó**, não por vários nós simultaneamente. A afinidade obrigatória mantém as réplicas juntas para evitar conflito de montagem ao escalar. Se esse nó não tiver recursos, réplicas podem ficar `Pending`. Este exemplo demonstra HPA, mas não oferece distribuição entre nós; para isso é necessário storage com `ReadWriteMany` e remover a afinidade.

## Passo 3 — Service e Route

```bash
oc apply -f 3-expose.yaml
oc get svc web-service
oc get route web-route
oc get route web-route -o jsonpath='https://{.spec.host}{"\n"}'
```

Abra a URL gerada. O Service recebe na porta 80 e encaminha para a porta 8080 dos pods; a Route oferece HTTPS com terminação TLS edge.

**Print:** página no navegador e/ou Topology no console do OpenShift.

## Passo 4 — HPA

```bash
oc apply -f 4-hpa.yaml
oc get hpa web-hpa
oc describe hpa web-hpa
oc adm top pods
```

**Print:** arquivo `4-hpa.yaml`, resources do Deployment e colunas TARGETS, MINPODS e MAXPODS.

O HPA usa **50% do request de CPU**, e não do limit: com request de 100m, o alvo é uma média de 50m por pod. O request de CPU e as métricas são necessários para esse cálculo; limits delimitam recursos, mas não são a base da porcentagem. Usamos Deployment (`apps/v1`), cujo nome está em `scaleTargetRef`.

### Carga opcional

Em um terminal, gere tráfego para o Service:

```bash
oc run load-generator --image=busybox:1.37.0 --restart=Never -- /bin/sh -c 'for i in 1 2 3 4 5 6 7 8; do (while true; do wget -q -O /dev/null http://web-service/ || sleep 1; done) & done; sleep 180; kill $(jobs -p); wait'
```

Em outro terminal, acompanhe:

```bash
oc get hpa,pods -w
# Após aproximadamente 3 minutos, Ctrl+C encerra a observação.
oc delete pod load-generator
```

NGINX servindo uma página pequena pode consumir pouca CPU: tráfego não garante atingir o alvo nem criar mais réplicas. Registre o resultado real. A configuração do HPA já pode ser evidenciada sem afirmar que houve expansão. Se o pod de carga continuar ativo, sua exclusão interrompe a carga.

## Testar persistência

O initContainer grava uma marca com data e nome do pod original em um comentário HTML, somente quando cria o arquivo. Compare essa marca antes e depois da recriação:

```bash
oc exec deployment/web-app -- cat /usr/share/nginx/html/index.html
oc get pods -l app=web-app
oc delete pod -l app=web-app
oc rollout status deployment/web-app --timeout=180s
oc get pods -l app=web-app
oc exec deployment/web-app -- cat /usr/share/nginx/html/index.html
```

Execute preferencialmente antes do HPA, com uma réplica. O pod deve mudar e o comentário deve manter a data e o nome do pod original. Isso demonstra que o arquivo foi preservado no PVC, em vez de recriado pelo initContainer. **Print:** conteúdo antes/depois, nomes dos pods e PVC `Bound`.

## Diagnóstico

- PVC `Pending`: consulte `oc describe pvc web-pvc`. Se não houver StorageClass padrão, informe `storageClassName: NOME-DA-CLASSE` no spec do PVC antes de criá-lo. Confirme provisionamento com o administrador.
- HPA `<unknown>`: consulte `oc describe hpa web-hpa` e `oc adm top pods`; as métricas podem levar alguns minutos. Confirme que o cluster oferece métricas de CPU.
- Pods `Pending`: consulte `oc describe pod NOME`. As réplicas precisam caber no mesmo nó por causa do PVC ReadWriteOnce.
- Falhas de inicialização: consulte `oc logs NOME -c init-html` e `oc logs NOME -c nginx`, além de `oc describe pod NOME`, para verificar permissões, montagem, imagem e quotas.

## Referências

- [NGINX sem root](https://github.com/nginx/docker-nginx-unprivileged).
- [Kubernetes: Persistent Volumes e modos de acesso](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
- [Kubernetes: HPA e requests de CPU](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/).
