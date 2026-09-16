<div align="center">

# MASS.
### Agent Governance Harness

_Deterministic Architecture & Runtime Governance for Autonomous Systems_

<p align="center">
  <a href="./README.md"><b>English</b></a> | <a href="./README.pt-BR.md"><b>Português</b></a>
</p>

---
</div>

**Worktrees herméticas, concorrência determinística e gates adversariais para geração autônoma de código.**

Uma arquitetura de referência para permitir que agentes de codificação por IA reivindiquem tarefas, escrevam código, rodem testes e abram pull requests **sem acesso de escrita irrestrito à sua branch principal** — e sem recair no padrão que destrói throughput de um humano aprovando cada edição de arquivo manualmente, uma por uma.

---

## O problema

Agentes autônomos de codificação, num time de engenharia de produção, tendem a falhar em uma de duas direções:

1. **Staging manual de candidatos.** O agente propõe uma edição, um humano aprova arquivo por arquivo antes de qualquer coisa ser escrita em disco. Seguro, mas não escala: cada tarefa vira uma fila de aprovações uma a uma, e a "autonomia" é, na prática, teatro.
2. **Acesso de escrita irrestrito.** O agente ganha uma branch real, direitos reais de commit, às vezes direitos reais de push em infraestrutura compartilhada. Isso escala — até o agente alucinar uma implementação plausível mas errada, que aterrissa numa branch compartilhada — ou pior, na `main` — antes que alguém a leia.

Nenhuma das duas é aceitável quando agentes fazem trabalho real, sem supervisão direta, contra um issue tracker real. O harness deste repositório é o caminho intermediário ao qual chegamos depois de conviver com os dois modos de falha.

## A ideia central

Dar ao agente **autoridade real para escrever, testar e commitar — mas só dentro de uma worktree isolada e descartável, e só até a borda de um pull request que um segundo processo independente precisa aprovar antes que um humano precise sequer olhar para ele.**

Nada do que o agente faz fica visível fora dessa worktree até que:
- a própria mudança passe na suíte de testes real do projeto, e
- uma revisão adversarial independente — um segundo modelo ou processo sem nenhum interesse em "parecer produtivo" — aprove o diff.

Só então um pull request **draft** é promovido para revisão humana. A autoridade de merge e a transição final para "concluído" permanecem exclusivamente humanas, sempre.

## Arquitetura

```
                     [ Inbound Task / Issue ]
                              │
                              │  strict WIP ceiling enforced
                              │  (global + per-category capacity)
                              ▼
                     ┌─────────────────┐
                     │   Orchestrator  │  claims the task, tracks liveness,
                     │                 │  never self-approves to "Done"
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │     Executor    │  ephemeral git worktree, created
                     │    (coding)     │  fresh from origin — never touches
                     │                 │  the shared local checkout
                     │                 │
                     │                 │  policy enforcement: denied_paths
                     │                 │  is fail-closed (deny wins, always)
                     │                 │
                     │                 │  runs the project's real test suite
                     │                 │  inside the worktree; non-zero exit
                     │                 │  aborts everything downstream
                     └────────┬────────┘
                              │  tests green
                              ▼
                     ┌─────────────────┐
                     │   Draft PR /    │  commit + push to a dedicated
                     │  Patch Emitted  │  branch (agent/<issue-id>)
                     │                 │  zero direct writes to main
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │   Adversarial   │  independent static/security review:
                     │      Gate       │  auth boundaries, tenant isolation,
                     │                 │  secret exposure, data-handling risk
                     │                 │
                     │  REJECT ─────── │  close the draft PR, no merge, no
                     │                 │  retry — the branch stays for audit
                     │                 │
                     │  APPROVE ────── │  promote the PR out of draft
                     └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │ Human Approval  │  exclusive merge authority,
                     │                 │  exclusive "Done" transition
                     └─────────────────┘
```

*(rótulos do diagrama mantidos em inglês de propósito — são os mesmos termos usados no código e na [versão em inglês](./README.md) deste documento.)*

Tradução livre de cada etapa: **Tarefa/Issue de Entrada** (teto rígido de WIP aplicado) → **Orquestrador** (reivindica a tarefa, monitora atividade, nunca se auto-aprova para "Concluído") → **Executor** (worktree Git efêmera criada a partir da origem, nunca toca o checkout local compartilhado; aplicação de política com `denied_paths` fail-closed; roda a suíte de testes real do projeto — código de saída não-zero aborta tudo a partir dali) → **PR Draft / Patch Emitido** (commit + push para uma branch dedicada, zero escrita direta na `main`) → **Gate Adversarial** (revisão estática/de segurança independente: limites de autorização, isolamento de tenant, exposição de segredo, risco no tratamento de dados — REJEITAR fecha o PR draft sem merge, sem retentativa; APROVAR promove o PR pra fora do modo draft) → **Aprovação Humana** (autoridade exclusiva de merge, transição exclusiva para "Concluído").

## Princípios norteadores

- **Sistema de arquivos hermético.** Todo trabalho do agente acontece numa worktree criada do zero a partir da `origin`, nunca de um checkout local compartilhado que outros processos possam estar usando. A worktree é descartável; uma que corrompe ou é abandonada não custa nada.
- **Aplicação de política fail-closed.** `denied_paths` são checados contra caminhos literais e resolvidos — sem sintaxe de glob pra errar sutilmente, sem padrão permissivo. Qualquer coisa que não esteja explicitamente dentro de `allowed_paths` é inalcançável por construção, não por convenção.
- **Concorrência determinística.** Um teto global de WIP, mais sub-limites por categoria (ex.: "no máximo uma sessão de codificação simultânea baseada em CLI", "no máximo N operações leves") — de forma que o throughput escala sem que um único recurso escasso trave sob carga concorrente.
- **Revisão adversarial antes da revisão humana.** O gate que decide se uma mudança sequer vale o tempo de um humano é um segundo processo independente — não o mesmo modelo corrigindo a própria prova, e não uma suíte de testes unitários, que prova comportamento, não intenção.
- **Regra do nunca-concluído.** Nenhum agente, nenhum adapter, nenhum orquestrador neste desenho transiciona uma tarefa para o estado terminal de "concluído". Essa é sempre uma ação humana, sem exceção.
- **Rejeição deixa evidência.** Uma mudança rejeitada nunca é descartada silenciosamente. O PR draft é fechado, não apagado — o diff, a execução de testes e o veredito adversarial continuam anexados a ele para auditoria.

## Estrutura do repositório

```
docs/
  adrs/                    registros de decisão arquitetural documentando como
                            este desenho foi alcançado (e o que ele substituiu)
  architecture/
    lifecycle-flow.md       um passo a passo mais detalhado do diagrama acima
templates/
  policies.canonical.yaml   uma política de runtime anotada, pronta para preencher
.github/
  workflows/
    pr-audit-template.yml   um hook de CI ilustrativo para o gate adversarial
```

## Status

Este repositório documenta o padrão, não uma ferramenta empacotada de instalar-e-rodar. A implementação de referência da qual este padrão foi extraído está integrada a um orquestrador interno, um issue tracker e um modelo de revisão local específico — nada disso é portável na forma atual. O que é publicado aqui é a arquitetura e o raciocínio de governança por trás dela, sanitizados de qualquer nome de sistema interno, credencial ou regra de negócio, para que possa ser implementado sobre qualquer orquestrador, tracker e ferramental de revisão que você já use.

## Licença

MIT — veja [`LICENSE`](./LICENSE).

## Sobre

Mantido pela [MASS](https://github.com/contatomasstech). Issues e discussões sobre a arquitetura são bem-vindas; isto é publicado como um desenho de referência, não como um produto suportado.
