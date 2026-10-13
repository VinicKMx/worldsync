# Desenvolvimento paralelo: Core Keeper

Preparação documental de 2026-10-13, baseada em master
`0d882e2132970f0e5c8fe8407c9e7e66cbbcc337`. Nenhuma feature de hospedagem dedicada
está implementada neste ponto. As duas branches partem do mesmo commit de
preparação e recebem todos os documentos abaixo.

| Leitura | Finalidade |
| --- | --- |
| [AUDIT.md](AUDIT.md) | Código encontrado, reutilização, lacunas e baseline de 264 testes. |
| [SHARED_CONTRACT.md](SHARED_CONTRACT.md) | Contrato revisão 1, operações/erros, estados, safety/liveness e falhas. |
| [SERVER_REFERENCES.md](SERVER_REFERENCES.md) | Fontes oficiais e o que ainda exige confirmação no pacote/na máquina. |
| [DISTRIBUTED_COORDINATION.md](DISTRIBUTED_COORDINATION.md) | Plano, matriz de testes e prompt de retomada de A. |
| [CORE_KEEPER_HANDOFF.md](CORE_KEEPER_HANDOFF.md) | Contexto autocontido, implementação incremental e prompt de retomada de B. |

A trabalha em `feature/distributed-coordination`; B em
`feature/core-keeper-server`. Preparação local:
`prep/core-keeper-parallel-20261013`. Não fazer merge na master, PR ou force push
como parte da preparação. Antes de retomar, inspecionar status/histórico e
preservar mudanças. Contratos são documentação inicialmente; não há framework
novo nem dependência adicional.

Separação essencial: A decide autoridade/publicação e orquestra; B controla
somente instalação/processo/dados do jogo e UI mínima. A publica estado canônico
após stop limpo de B. Expiração com parada incerta bloqueia nova hospedagem.
O modo antigo de pasta cloud permanece disponível; nunca dá acesso direto ao
canônico dedicado. Para começar uma sessão independente, usar o prompt no final
do documento da respectiva frente.

Para verificar o ponto compartilhado imediatamente após esta preparação:

```powershell
git fetch origin
git rev-parse origin/feature/distributed-coordination
git rev-parse origin/feature/core-keeper-server
git diff origin/master...origin/feature/core-keeper-server --stat
```

Os dois primeiros SHAs devem ser iguais nesse ponto inicial. Depois que cada
colaborador fizer commits, a divergência será normal; usar `git merge-base` para
encontrar a origem comum. O delta inicial contém somente estes seis documentos.
