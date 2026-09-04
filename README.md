# CI/CD Grupo <N>

![CI](https://github.com/AlanGarci4/cicd-grupo-6/actions/workflows/ci.yml/badge.svg)

## Membros
- @AlanGarci4 (Owner) · @tuliocoimbra · @danielarrais · José · Tiago

## Acesso para avaliação
- Repositório da equipe em modo privado
- Professor HardSource adicionado como collaborator com permissão Read
- Branch main protegida com required status checks e revisão de Code Owners

## Pipeline de CI
O pipeline do laboratório usa GitHub Actions como quality gate de verdade.

- Gatilhos em `pull_request` e `push` na branch `main`
- Matrix em Python 3.10, 3.11 e 3.12
- Cache de dependências do pip com chave baseada em `requirements*.txt`
- Reusable workflow para centralizar os testes e manter o YAML enxuto
- Lint com Ruff, testes com pytest e auditoria de dependências com pip-audit
- Scan de vulnerabilidades com Trivy antes do build da imagem
- Environment `staging` com required reviewer
- Permissions explícitas, mínimo privilégio e actions pinadas por SHA
- Notificação por webhook com status final do run
- Push da imagem para Docker Hub somente quando os gates passam

## Pipeline de CD
A parte de CD do projeto usa o mesmo repositório como base e segue a arquitetura aprendida no laboratório para deploy em staging, com ambientes protegidos e publicação de artefatos versionados. O fluxo foi montado para ser reutilizado na entrega posterior com rolling e blue/green.

## Como rodar localmente
```bash
python -m pip install -r requirements.txt -r requirements-dev.txt
pytest -v
ruff check .
pip-audit -r requirements.txt -r requirements-dev.txt
```

## Shift-left demonstrado
A segurança foi tratada cedo no ciclo de vida:
1. o Trivy escaneia o filesystem antes do build;
2. o `pip-audit` valida dependências conhecidas por CVE;
3. qualquer falha bloqueia o pipeline antes da publicação da imagem.

Esse padrão reduz o risco de defeitos de segurança chegarem à `main`.

## Checklist de entrega
- [ ] Professor (HardSource) adicionado como collaborator com permissão Read
- [ ] Branch `main` protegida com PR obrigatório e required checks
- [x] CODEOWNERS definido para proteger workflows e manifests
- [x] `ci.yml` disparando em `pull_request` e `push` na `main`
- [x] `pytest` e `pip-audit` como gates de qualidade
- [x] Matrix de Python com cache de dependências
- [x] Reusable workflow `workflow_call`
- [ ] Environment com required reviewer
- [x] `permissions` explícitas e mínimo privilégio
- [x] Actions com pin por SHA
- [x] Trivy como gate de segurança
- [ ] Notificação por webhook validada no canal da equipe
- [ ] Badge do CI apontando para o repositório final
- [x] README documentando o pipeline e comandos locais
- [ ] Falha de segurança demonstrada em PR, bloqueio visível e correção validada

- [ ] Imagem publicada no Docker Hub com tag do commit, via access token
- [ ] EC2 com kind + ingress-nginx, chave SSH gerada dentro da VM
- [ ] Manifestos em `k8s/`: Deployment + Service `ClusterIP` + Ingress
- [ ] `cd.yml`: `scp` + `kubectl apply` + `rollout status` + smoke test `/healthz`
- [ ] `cd-blue-green.yml` + `cd-blue-green-switch.yml`, com `run-name` dinâmico
- [ ] Rollback do Blue/Green demonstrado (re-switch de tráfego)
- [ ] README atualizado com a arquitetura de deploy

---

## Teardown

**Ao fim de toda sessão prática:**

- [ ] EC2 em **Stop instance** — não Terminate. Preserva o disco e o cluster kind
- [ ] Confirmar no Console que o estado é `stopped`
- [ ] Lembrar que o `EC2_HOST` precisará ser atualizado na próxima sessão

**Ao fim do curso:**

- [ ] **Terminate instance** — apaga a EC2 e o volume EBS
- [ ] Conferir em **EC2 → Volumes** se não sobrou volume órfão
- [ ] Revogar o access token do Docker Hub
- [ ] Remover o webhook de notificação do canal

> `t3.small` custa ~US$ 0,023/h, então o crédito do Learner Lab tem folga enorme. O
> motivo do teardown é higiene e disciplina de FinOps, não risco de estourar o
> crédito. Instância esquecida ligada é o clássico da vida real.
