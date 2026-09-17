# CI/CD Grupo 6

![CI](https://github.com/AlanGarci4/cicd-grupo-6/actions/workflows/ci.yml/badge.svg)

## Membros

- @AlanGarci4 (Owner)
- @tuliocoimbra
- @danielarrais
- José
- Tiago

## Visão geral

Este repositório reúne as Atividades 1 e 2 da disciplina. O CI valida a aplicação
e publica uma imagem versionada no Docker Hub. O CD leva exatamente essa imagem
até um cluster Kubernetes, demonstrando Rolling Update e Blue/Green com switch e
rollback.

```text
Commit na main
      |
      v
GitHub Actions CI -- push por access token --> Docker Hub
      |
      | workflow_dispatch + SSH/SCP
      v
EC2 Amazon Linux 2023
      |
      v
kind Kubernetes --> ingress-nginx --> Service ClusterIP --> Pod
```

O GitHub Actions acessa a EC2 por SSH. Os manifests são renderizados no runner,
copiados por SCP e aplicados com `kubectl` na máquina remota. O repositório não
precisa ser clonado na EC2.

## Evolução da Atividade 1

Na Atividade 1, o CI passou a executar Ruff, pytest, `pip-audit` e Trivy antes de
publicar a imagem. Em cada push na `main`, o job de publicação autentica no Docker
Hub com access token e envia duas tags:

```text
<DOCKERHUB_USERNAME>/todolist:<SHA-curto>
<DOCKERHUB_USERNAME>/todolist:latest
```

Na Atividade 2, os deploys aceitam somente o SHA curto. Assim, a mesma imagem que
passou pelos gates do CI é promovida até o cluster sem rebuild. O build também
injeta suas tags em `IMAGE_TAGS`, exibidas no rodapé da aplicação.

## Estratégias de CD

| Estratégia | Workflow | Namespace | Host |
|---|---|---|---|
| Rolling Update | `cd.yml` | `todolist` | `todolist.local` |
| Blue/Green deploy | `cd-blue-green.yml` | `todolist-bg` | Host fixo de cada cor |
| Blue/Green switch | `cd-blue-green-switch.yml` | `todolist-bg` | `todolist-bg.local` |

O Rolling atualiza um único Deployment e aguarda o novo pod ficar `Ready`. No
Blue/Green, os Deployments blue e green permanecem ativos. O deploy altera apenas
o slot inativo; o switch muda somente `Service/todolist.spec.selector.color`.

Deploy e switch são operações separadas para permitir validação do candidato
antes do cutover. O switch usa o environment `production`, com aprovação, e os
dois workflows compartilham um grupo de concorrência para não operarem ao mesmo
tempo.

## Configuração do GitHub

Cadastre em **Settings > Secrets and variables > Actions**, sem gravar os valores
no repositório:

| Nome | Tipo | Uso |
|---|---|---|
| `DOCKERHUB_USERNAME` | Secret | Compõe o nome da imagem |
| `DOCKERHUB_TOKEN` | Secret | Publicação autenticada no Docker Hub |
| `EC2_HOST` | Secret | IPv4 público atual da EC2 |
| `EC2_USER` | Secret | Usuário SSH, normalmente `ec2-user` |
| `EC2_SSH_KEY` | Secret | Chave privada autorizada na EC2 |
| `KIND_CLUSTER` | Variable | Nome do cluster, `devops-labs` |

O repositório de imagens no Docker Hub é público para que o Kubernetes faça pull
sem `imagePullSecret`. O IP público da EC2 pode mudar depois de um stop/start;
nesse caso, atualize `EC2_HOST` antes de executar os workflows.

O environment `production` deve ter required reviewers e aceitar deploy apenas da
branch protegida.

## Hosts do laboratório

Mapeie o IPv4 público atual da EC2 no `/etc/hosts` da máquina de demonstração:

```text
<EC2_PUBLIC_IP> todolist.local todolist-bg.local blue.todolist-bg.local green.todolist-bg.local
```

| Host | Função |
|---|---|
| `todolist.local` | Aplicação Rolling |
| `todolist-bg.local` | Produção Blue/Green |
| `blue.todolist-bg.local` | Teste direto do slot blue |
| `green.todolist-bg.local` | Teste direto do slot green |

## Como executar os workflows

Todos os deploys são manuais e devem usar a branch `main`.

### Validar o canal SSH

```bash
gh workflow run validate-ssh.yml --ref main
```

O workflow entra na EC2, seleciona `kind-devops-labs` e lista os namespaces. Ele
deve ser executado antes dos deploys para separar problemas de infraestrutura de
problemas da aplicação.

### Rolling Update

```bash
gh workflow run cd.yml --ref main -f image_tag=<sha-curto>
```

O workflow confirma a existência da imagem, renderiza o manifest, copia por SCP,
executa `kubectl apply`, aguarda o rollout e testa `/healthz` pelo Ingress.

Rollback Rolling pela pipeline:

```bash
gh workflow run cd.yml --ref main -f image_tag=<sha-anterior>
```

Alternativa operacional diretamente na EC2:

```bash
kubectl rollout undo deployment/todolist -n todolist
```

### Primeiro deploy Blue/Green

Na primeira execução, `baseline_tag` inicializa os dois slots com uma versão
estável. Como produção começa em blue, a candidata deve ser publicada em green:

```bash
gh workflow run cd-blue-green.yml --ref main \
  -f color=green \
  -f baseline_tag=<sha-estavel> \
  -f image_tag=<sha-candidato>
```

Nas execuções seguintes, informe somente a cor inativa e a nova imagem:

```bash
gh workflow run cd-blue-green.yml --ref main \
  -f color=<blue-ou-green> \
  -f image_tag=<sha-candidato>
```

O workflow consulta o selector do Service de produção e falha se a cor solicitada
estiver ativa. O smoke test usa o host fixo da cor, sem alterar o tráfego.

### Switch e rollback Blue/Green

Switch para green:

```bash
gh workflow run cd-blue-green-switch.yml --ref main -f color=green
```

Rollback real para blue:

```bash
gh workflow run cd-blue-green-switch.yml --ref main -f color=blue
```

Antes do patch, o workflow valida o slot alvo. Depois, testa produção pelo Ingress.
Se esse teste falhar, o selector anterior é restaurado automaticamente.

## Como rodar os checks localmente

```bash
python -m pip install -r requirements.txt -r requirements-dev.txt
pytest -v
ruff check .
pip-audit -r requirements.txt -r requirements-dev.txt
```

Esses checks correspondem aos gates executados pelo CI. Uma falha interrompe o
fluxo antes da publicação da imagem.

## Decisões e trade-offs

| Decisão | Motivo e consequência |
|---|---|
| GitHub Actions via SSH | Caminho curto para o laboratório; o runner conhece a credencial da EC2 |
| Deploy manual por SHA | Facilita a demonstração e garante rastreabilidade; não há promoção automática |
| Ingress + ClusterIP | Mantém um ponto de entrada estável e evita NodePort por aplicação |
| Blue/Green com dois slots | Rollback rápido por selector; consome recursos duplicados |
| SQLite em `emptyDir` | Mantém o laboratório simples; dados são perdidos quando o pod é substituído |
| Registry público | Evita `imagePullSecret`; as imagens podem ser baixadas publicamente |

Rolling e Blue/Green coexistem em namespaces e hosts diferentes. Helm, GitOps,
PVC, banco externo e service mesh ficaram fora do escopo para manter o foco no
fluxo do commit até o pod.

## Segurança e higiene

- Secrets ficam no cofre do GitHub e nunca são impressos ou versionados.
- Actions de terceiros são fixadas por SHA.
- Os workflows declaram `permissions: contents: read`.
- A configuração SSH repetida foi extraída para uma action local.
- Tags, nome do cluster e endereço da EC2 são inputs, variables ou secrets.
- O switch exige aprovação no environment `production`.
- Nenhuma chave, token, state do Terraform ou credencial deve entrar no Git.

## Evidências executadas

| Evidência | Resultado | Run |
|---|---|---|
| CI publicou `todolist:76ac744` | Sucesso | [GitHub Actions](https://github.com/AlanGarci4/cicd-grupo-6/actions/runs/35162303102) |
| Canal GitHub Actions -> EC2 -> kind | Sucesso | [GitHub Actions](https://github.com/AlanGarci4/cicd-grupo-6/actions/runs/35163544928) |
| Rolling Update + smoke test | Sucesso | [GitHub Actions](https://github.com/AlanGarci4/cicd-grupo-6/actions/runs/35163584284) |
| Deploy `76ac744` no slot green inativo | Sucesso, produção permaneceu blue | [GitHub Actions](https://github.com/AlanGarci4/cicd-grupo-6/actions/runs/35163640505) |
| Switch de blue para green | Sucesso após aprovação | [GitHub Actions](https://github.com/AlanGarci4/cicd-grupo-6/actions/runs/35164527620) |
| Rollback de green para blue | Sucesso | [GitHub Actions](https://github.com/AlanGarci4/cicd-grupo-6/actions/runs/35165021820) |

## Limitações conhecidas

- O rollback Rolling não é automático; é feito com a tag anterior ou `rollout undo`.
- O acesso ao cluster usa uma chave SSH armazenada como secret.
- O Security Group aberto para runners hospedados é uma simplificação do laboratório.
- O IP público muda quando a EC2 é parada e iniciada.
- O SQLite em `emptyDir` não preserva dados entre pods ou cores.

## Checklist de entrega

- [ ] Repositório privado
- [ ] Professor `HardSource` com permissão Read
- [x] Branch `main` protegida com PR e checks obrigatórios
- [x] Imagem publicada com tag do commit usando access token
- [x] Canal SSH validado pelo GitHub Actions
- [x] Manifests com Deployment, Service `ClusterIP`, Ingress e probes
- [x] Rolling Update com rollout e smoke test pelo Ingress
- [x] Blue/Green com deploy no slot inativo e switch separado
- [x] Rollback Blue/Green executado e registrado
- [x] Environment `production` com required reviewers
- [x] README com comandos, arquitetura, rollback e decisões técnicas
- [ ] Prints e vídeo curto de contingência organizados

## Teardown

Ao fim de cada sessão, pare a EC2 sem terminá-la. Ao iniciar novamente, atualize
`EC2_HOST` e a entrada do `/etc/hosts`.

Ao fim do curso:

- Termine a EC2 e confirme que não restaram volumes EBS órfãos.
- Revogue o access token do Docker Hub.
- Remova secrets e webhooks que não serão mais utilizados.
