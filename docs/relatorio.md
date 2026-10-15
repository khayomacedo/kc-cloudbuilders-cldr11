# Deploy de NGINX no OpenShift

**Grupo / RMs:** [preencher]
**Disciplina:** [preencher]
**Data e projeto:** [preencher]

> Complete após executar o README, com resultados reais. Salve os prints em `docs/prints/`, insira neste documento e exporte para PDF. Use aproximadamente 2 a 3 páginas.

## 1. Volume persistente — bônus 2

Aplicamos `1-pvc.yaml` para solicitar 1 GiB com acesso ReadWriteOnce. O provisionador da StorageClass vinculou o PVC `web-pvc` a um PersistentVolume. O arquivo HTML fica nesse volume.

**Print 1:** [criação do PVC e `oc get pvc`, com status Bound e nome do PV; pode ocorrer após criar o Deployment]

**Comentário do grupo:** [resultado observado]

## 2. Deploy da aplicação

Aplicamos `2-deployment.yaml`. O Deployment `web-app` executa NGINX sem root na porta 8080. Um initContainer cria a página no PVC apenas na primeira execução. O NGINX monta o conteúdo para leitura e possui requests/limits de CPU e memória, além de probes HTTP.

Como ReadWriteOnce permite montagem em um único nó, configuramos afinidade para manter as réplicas nesse nó. Essa escolha simplifica a demonstração, mas limita a expansão à capacidade dele.

**Print 2:** [pod Running e trecho do Deployment com initContainer e volume]

**Comentário do grupo:** [resultado observado]

## 3. Acesso à aplicação

Aplicamos `3-expose.yaml`: o Service recebe na porta 80 e encaminha para a porta 8080. A Route expõe a aplicação por HTTPS com terminação TLS edge.

**Print 3:** [página no navegador com URL e/ou Topology]

**Comentário do grupo:** [URL e resultado observado]

## 4. HPA — bônus 1

Aplicamos `4-hpa.yaml`. O HPA monitora o Deployment `web-app`, com mínimo de 1 e máximo de 4 pods. O alvo de CPU é 50% do request de 100m, equivalente a uma média de 50m por pod, e depende das métricas fornecidas pelo cluster.

**Print 4:** [YAML do HPA, resources do Deployment e `oc get hpa web-hpa`]

**Carga opcional:** [registrar somente se executada; informar métricas e réplicas observadas. Servir HTML pode consumir pouca CPU e não provocar expansão]

## 5. Evidência de persistência e resultado

Comparamos o HTML antes e depois de excluir o pod. O comentário com data e nome do pod que inicializou o volume permite verificar que o arquivo foi preservado, pois o initContainer não o sobrescreve.

**Print 5:** [HTML antes/depois, nome do pod antigo/novo e PVC Bound]

**Resultado observado:** [informar se a marca permaneceu e se a página voltou a responder]

**Conclusão:** [resumir o deploy e os bônus comprovados, incluindo eventuais limitações encontradas]
