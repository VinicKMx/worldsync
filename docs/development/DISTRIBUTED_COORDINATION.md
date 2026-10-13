# Plano A: Distributed Coordination

Branch `feature/distributed-coordination`, fork
<https://github.com/VinicKMx/worldsync>. Base original master
`0d882e2132970f0e5c8fe8407c9e7e66cbbcc337`; preparação em 2026-10-13.
Leia [AUDIT](AUDIT.md), [SHARED_CONTRACT revisão 1](SHARED_CONTRACT.md),
[fontes](SERVER_REFERENCES.md) e [handoff B](CORE_KEEPER_HANDOFF.md).
Esta branch começa documental, no mesmo commit de B. Não há backend implantado.

## Arquitetura mínima escolhida

Preservar modo pasta cloud e dez perfis. Acrescentar apenas modo dedicado
Core Keeper opt-in, cliente de coordenação/snapshots e pequena orquestração de
sessão chamada pelo controller existente. Não encapsular o sync inteiro em
framework de storage/providers e não mover a UI de Qt para outra stack.

Um Durable Object por identidade canônica do mundo coordena lease/fence/head;
R2 privado armazena pacotes imutáveis. Uma API autenticada encaminha operações e
transferência de bytes. A transação é apenas dentro do DO; put R2 é anterior e
pode deixar órfão. Após qualquer await externo, revalidar antes de trocar head.
Consultar documentação oficial no documento de fontes antes de codificar runtime.
Nenhum SDK é necessário no desktop para falar HTTP; começar com urllib da stdlib.
Linguagem/toolchain do backend será escolhida na implementação, sem impor Python
ao runtime cloud nem adicionar Node/wrangler nesta preparação.

## Reutilização e mudanças pontuais

| Reutilizar | Como |
| --- | --- |
| sync.world_files / world_fingerprint / sha256_file | Selecionar e verificar arquivos com patterns confiáveis de B; obter lista ordenada e confinada. Não usar fingerprint legacy do primeiro arquivo. |
| sync._backup_files / backups.create_checkpoint | Cópias locais antes de substituição; retenção de recuperação em checkpoint não podado. Não copiar mundo em execução. |
| storage.write_json / config.world_state | Cache e estado local serializados; journal crítico com tratamento explícito de corrupção e persistência suficiente para impedir replay/spawn. |
| config.effective_cfg / game_settings | Defaults legados preservados e campos opt-in; ID canônico adicional. Não usar ID local ou slot como identidade global. |
| Controller._busy / workers / Qt signals | Lock local para impedir chamadas duplicadas e alterações de config durante sessão; coordenação remota separada. |
| UI e callbacks existentes | Status dedicado mapeado explicitamente, sem cair no branch genérico "up to date" para erro desconhecido. |

Novos arquivos esperados, criados só quando implementados: `app/core/coordination.py`
(cliente e tipos consumidos), `app/core/host_session.py` (ordem da sessão, snapshots
e journal), testes correspondentes. Manter empacotamento junto da orquestração até
o tamanho real justificar separar. B entrega `core_keeper_server.py`; não duplicar
SteamCMD/process supervisor em A. Pequenas alterações compartilhadas em
controller/config são de A; B trabalha em seu módulo e UI conforme contrato.

Entradas que precisam de gates: start_play, push_now, refresh/status da biblioteca,
_prime_seen_versions/_scan_all_worlds, _after_push, checkpoint, restore no window,
troca/remoção do mundo, settings, quit e atualização do executável. `sync_dir=""`
vira Path('.') no código atual: mundo dedicado jamais pode atravessar esses
caminhos legados. Ocultar/desabilitar auxiliares de pasta (saga, nudges, convites
WS1, publicação de update/status.html) inicialmente para o modo dedicado evita
reimplementar extras. Não chamar _after_push do legado após commit R2.

## Modelo persistente e invariantes

Estado mínimo por mundo: identidade/grupo/jogo/slot/layout, estado,
`last_fence`, owner/session/hash do segredo/expiry/base_version (se há lease),
head canônico, recibos/tombstones terminais e motivo de recuperação. A definição
de entradas, saídas, erros e transições está no contrato, sem redefinições aqui.

IDLE -> acquire atômico -> STARTING -> activate -> ACTIVE -> begin_finalize ->
FINALIZING -> commit atômico -> IDLE. Expiração em qualquer fase com lease leva a
RECOVERY_REQUIRED, mesmo se o host ainda estava baixando. É o preço deliberado
para não precisar de uma operação extra de autorização de spawn. `release` só
cancela STARTING comprovadamente anterior a qualquer tentativa de start; depois
de spawn, finalizar ou recuperar. `commit` encerra lease; não requer release.

1. No máximo um lease válido por mundo; aquisição e incremento de fence na mesma
   transação, persistindo antes de responder. Fence nunca reinicia em deploy,
   eviction, recuperação ou restauração administrativa.
2. Commit requer lease não expirado, owner/session/segredo/fence atuais,
   FINALIZING, base=head, snapshot íntegro da sessão e stop limpo. Não basta
   aceitar tokens maiores do que o último commit: comparar com holder atual.
3. Head referencia bytes imutáveis completos. Só serviço pode escrever/promover;
   clientes antigos e acessos à pasta cloud não têm caminho de escrita canônica.
4. Troca de head, nova versão, recibo e liberação são uma transação. Pedido
   duplicado não cria nova versão. Timeout não equivale a falha confirmada.
5. Rejeitar expiração ainda que nenhum sucessor exista. Clock remoto é autoridade;
   local monotônico usa margem; alarmes não substituem validação em cada operação.
6. Journals e fakes não são prova contra host malicioso. Fencing protege canônico,
   não desconecta jogadores de processo isolado; incerteza de parada bloqueia
   takeover. Nenhum pedido de force overwrite ignora esses requisitos.
7. Restore/rollback produz nova versão crescente via autoridade válida, nunca
   decrementa versão/fence ou edita snapshot antigo. Snapshot ausente/corrompido
   bloqueia, não inicializa mundo novo.

Backend autenticado inicial: credencial de membro/grupo provisionada fora do
convite, world_id autorizado explicitamente; o lease_secret é distinto e vinculado
à sessão. Segredos R2 só no serviço. Sem usuários globais, OAuth, pub/sub, filas,
WebSockets, event sourcing, delta sync ou GC automático neste incremento.
Qual mecanismo seguro de guardar credencial local será usado é um gate de
produção; nunca publicá-la no repo ou distribuir uma chave R2 no exe.

## Persistência dos bytes e recuperação

A cria pacote stdlib a partir de raiz parada validada com layout de B. Registra
inventário/hash e base. Serviço grava candidato exclusivo no R2, valida bytes
e estrutura, e só então promove referência por transação DO. Erros antes disso
mantêm head anterior; objetos órfãos são aceitáveis inicialmente. Não confiar em
ETag como checksum do pacote, nem em hash autodeclarado sem verificação.

Recuperação inicialmente manual e auditável: consultar recibos e journals,
confirmar antiga instância e qualquer spawn pendente definitivamente parados,
preservar dados, escolher snapshot recuperado íntegro ou último head, registrar
eventual perda de progresso, invalidar sessão antiga e manter fence monotônico.
Se não é possível confirmar parada/inibição, permanecer bloqueado. Projetar
operação administrativa restrita com CAS sobre sessão/head/fence esperados,
separada do protocolo de commits expirados. Reativar um lease antigo é proibido.
O primeiro incremento pode não implementar resolução administrativa, desde que
RECOVERY_REQUIRED seja terminal seguro e não haja rollout para saves reais.

Bootstrap de um mundo é explícito e anterior ao primeiro acquire. Para testes,
seed sintético de versão inicial e layout conhecido; para produto, import
validado de cópia parada/backupeada numa sessão futura. Sem import automático do
Drive, sem sincronização bidirecional entre R2 e a antiga pasta compartilhada.
Restaurar backup do próprio DO não pode reemitir fences antigos: antes de voltar
a servir, reconciliar high-water mark/epoch com registros preservados ou manter
o mundo bloqueado e reprovisionar identidade/credenciais de forma deliberada.

## Contrato com B e integração

B fornece prepare -> PreparedServer(build_id, layout_id, patterns),
start(world_path, session_id) e health -> ServerHealth, stop -> StopResult.
Game ID vem de health; clean=false/unknown bloqueia commit normal. A é dono de
retry/heartbeat/rede e do journal de operação; B é dono da identificação e
terminalidade do processo. A nunca envia sinais do SO por conta própria.

Preparar binários antes de adquirir; adquirir antes de baixar; validar antes de
iniciar; anunciar só depois de ready + activate confirmado; finalizar antes de
parar; publicar só após stop limpo. Heartbeat não pode ser bloqueado por download,
startup, shutdown ou upload. Stop local deve funcionar mesmo sem backend.
Snapshot/local root por session_id nunca aponta para a pasta pessoal do cliente.
Durante desenvolvimento, A usa um fake de servidor; B usa testes de processo.

## Plano incremental

1. Testes de modelo determinístico e fakes (tempo/storage/servidor), contratos de
   registros e erros. Primeiro provar corrida de acquire e bloqueio por expiração.
   Fake de storage não deve ser confundido com mutex distribuído de produção.
2. Implementar journal/orquestração Python com servidor fake, staging sintético,
   hashing/backups existentes, erros e retries. Exercitar timeout em cada fronteira.
3. Implementar posteriormente backend mínimo DO/R2 em ambiente local de teste,
   com transações reais do runtime e autenticação; testar interleaving de await.
   Nada de deploy/provisionamento sem sessão específica para isso.
4. Conectar via opt-in no controller/config, adicionar gates de todos os caminhos
   legados, integrar UI de B e testes offscreen. Contrato antes de integração real.
5. Integrar adapter B, testar sucesso entre dois hosts simulados e falhas. Revisar
   versão/updates do fork para impedir instalação de upstream sem suporte dedicado.
6. Antes de saves reais: teste autorizado em hardware com mundo descartável,
   validação de layout/build/stop, revisão de credenciais, limites e procedimento
   de recuperação. Só então decidir deploy e bootstrap/migração em outra sessão.

## Matriz mínima de testes

| Cenário | Evidência obrigatória |
| --- | --- |
| Acquire simultâneo A/B e repetição do vencedor | Um holder; fence incrementado uma vez para retry; perdedor não inicia processo. |
| Relógio remoto no limite expires_at, local adiantado/atrasado, alarmes atrasados | Expiração rejeita mutações; sem ressurreição por heartbeat tardio. |
| Restart/eviction do DO e perda do estado em memória | Owner/head/recibo/fence persistem. |
| Heartbeat durante start/stop/upload lentos | Renovação independente e margem local aplicada; sem iniciar após prazo vencido. |
| Crash antes/depois de start e antes de activate | Nenhuma aquisição nova após TTL; recovery obrigatório mesmo sem Game ID publicado. |
| Partição com servidor antigo ativo | Novo host bloqueado; tentativa de commit antigo rejeitada; snapshot preservado localmente. |
| Dois commits concorrentes / base antiga / outro mundo / segredo errado | Só operação válida publica; rejeição sem mudar head. |
| Upload parcial, checksum falso, inventário incompleto, path traversal/ADS/link/case collision | Zero start/promote; arquivos fora da raiz intocados. |
| Expiração ou recuperação entre put R2 e transação final | Revalidação rejeita; head continua anterior. |
| R2 put concluído seguido de crash; commit concluído seguido de resposta perdida | Órfão recuperável no primeiro; recibo confirma sucesso único no segundo. |
| Retry tardio de acquire/release/commit após novo host | Não ganha nova sessão, não limpa novo owner, não avança versão novamente. |
| stop incerto/kill, journal corrompido, disco cheio, backup falha | Bloqueio, sem publicação ou perda da única cópia local. |
| Restore/bootstrap recuperado, snapshot inválido ou build incompatível | Operação explícita e válida ou bloqueio; nenhum fallback a mundo vazio. |
| Cliente legado / gravador direto / acesso sem autorização | Nenhuma permissão canônica; flags em UI não são a proteção. |
| UI manual push/restore/checkpoint/quit/update e mudança de mundo durante sessão | Todos usam gates, sem tocar sync_dir vazio ou save servido. |
| Config antigo e roundtrip dos dez jogos | Sem mudança no comportamento legado e suite existente verde. |

Injetar falha antes/depois de cada persistência relevante e cada resposta de rede;
usar barreiras controladas para corridas, não sleeps aleatórios. Somar testes do
runtime DO quando existir implementação: um fake serializado sozinho não comprova
as garantias da plataforma. Sem ferramenta nova de property testing por padrão.

## Questões abertas e riscos antes do rollout

- TTL/margem iniciais 90/20/15s são valores de desenvolvimento, não prova de que
  stop termina em 15s. Validar com B; segurança continua pelo bloqueio remoto.
- Modelo exato de autenticação, revogação, armazenamento local de segredo e
  recuperação administrativa precisam ser implementados antes de produção.
- B deve confirmar arquivos de mundo, servermaps, argumentos, saída Game ID e
  parada/flush por build. Enquanto não houver evidência, feature permanece desligada.
- Tamanho de snapshots e limites de request/CPU/memória Cloudflare precisam ser
  medidos; se ultrapassar limites, upload multipart restrito, sem alterar o commit.
- Sem takeover automático em partição há perda deliberada de disponibilidade.
  Não prometer zero perda do progresso ainda não publicado após disco/host perdido.
- Atualizador aponta ao upstream e CI não roda em push de feature branch. Tratar
  distribuição/CI posteriormente com mudança pequena e explícita; não nesta sessão.
- Testes locais foram no Python 3.14.4/Windows; CI declara 3.12/Windows+Ubuntu.
  Testes reais de servidor e backend ainda não existem.

## Ambiente e retomada

Clone novo; em checkout existente conferir alterações antes de switch:

```powershell
git clone --branch feature/distributed-coordination https://github.com/VinicKMx/worldsync.git
cd worldsync
git status --short --branch
py -3.12 -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt -r requirements-dev.txt
$env:QT_QPA_PLATFORM = 'offscreen'
.\.venv\Scripts\python -m pytest tests -q
.\.venv\Scripts\python -m pytest tests/test_sync.py tests/test_games.py tests/test_every_game.py tests/test_presence.py tests/test_config.py -q
```

Se só houver Python 3.14, `python -m venv .venv` reproduz o ambiente local desta
auditoria. Baseline completo: 264 passed em 15,39s. Futuros testes:
`tests/test_coordination.py` e `tests/test_host_session.py`, depois de criados.
Usar dados sintéticos; não rodar app contra config real ou iniciar servidor para
desenvolver o modelo. Não adicionar dependências antes de demonstrar necessidade.

> Retome o fork https://github.com/VinicKMx/worldsync na branch
> feature/distributed-coordination como desenvolvedor A. Leia README, DESIGN e
> docs/development/AUDIT.md, SHARED_CONTRACT.md, DISTRIBUTED_COORDINATION.md e
> CORE_KEEPER_HANDOFF.md. Implemente incrementalmente coordenação e orquestração,
> começando pelo modelo/testes determinísticos e servidor fake. Preserve o sync
> legado e reutilize hashing/backups/config. DO por mundo + R2 privado é a direção,
> sem assumir transação entre eles. Lease expirado não pode publicar ou liberar
> takeover incerto. B cuida de SteamCMD e processos; não implemente seus internos.
> Preserve alterações Git, não use saves reais, não inicie jogo, não provisione
> cloud, não faça deploy, merge na master, PR ou force push. Execute testes
> offscreen e documente riscos e gates antes de qualquer rollout.
