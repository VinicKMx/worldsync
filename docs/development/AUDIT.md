# Auditoria do fork WorldSync

Data: 2026-10-13. Baseline: `origin/master` em
`0d882e2132970f0e5c8fe8407c9e7e66cbbcc337` (WorldSync 2.1.2).
Esta preparação contém somente documentação; descreve trabalho futuro, não
garantias já implementadas. Leia também [o contrato](SHARED_CONTRACT.md).

## Inspeção e preservação

O remoto `https://github.com/VinicKMx/worldsync.git` tinha somente `master`.
Não havia checkout WorldSync nas pastas locais de desenvolvimento inspecionadas.
Foi criado um clone novo em `C:\Users\vinic\Documents\dev\worldsync`, com árvore
limpa. Nenhum trabalho local de terceiros foi reutilizado ou sobrescrito.
`master` permanece no SHA acima; a preparação ocorre em
`prep/core-keeper-parallel-20261013`. As duas branches de trabalho receberão o
mesmo tip da preparação, incluindo os dois handoffs. Sem merge, PR ou force push.

Foram lidos integralmente README, DESIGN e docs/GAMES. Não existem
CONTRIBUTING.md nem AGENTS.md versionados; também não foram encontradas instruções
AGENTS nos diretórios ancestrais inspecionados. As regras de contribuição estão
na seção Contributing do README: preservar compatibilidade do manifesto e testar
alterações no sync. `docs/GAMES.md` é gerado por `tools/games_doc.py`; editar sua
fonte em `games.py` se houver mudança futura, não o arquivo gerado diretamente.

## Implementação encontrada e reaproveitamento

| Componente real | Comportamento e limite observado | Decisão para o modo dedicado |
| --- | --- | --- |
| `app/core/sync.py`: `world_files`, `world_fingerprint`, `sha256_file` | Seleção por glob; lista ordenada; fingerprint inclui caminho relativo + SHA-256 de todos os arquivos quando `patterns` não é None. Dragonwilds usa apenas o primeiro arquivo. | Reutilizar enumeração/hash com layout dedicado validado e caminhos confinados; sempre usar patterns explícitos. |
| `do_pull`, `do_push`, `get_shared_manifest`, `get_status` | Pasta local sincronizada por software de terceiros; `version.json`; comparação com `last_applied_version`; confirmação permite sobrescrever conflito. | Preservar para o modo atual. Não usar como cliente R2, mutex ou commit canônico. |
| `storage.write_json` | JSON temporário + `os.replace`; nome temporário fixo; sem transação entre arquivos, sem lock distribuído e sem fsync explícito. `read_json` trata corrupção como ausência. | Reutilizar para config/cache local com escritor serializado. Diário crítico exige detecção conservadora de ausência/corrupção; não equivale a persistência distribuída. |
| `sync._backup_files`, `backups.create_checkpoint`, `restore_backup` | Backups locais; dez automáticos, checkpoints fixos; restore copia localmente e permite mirroring. | Reutilizar para staging parado e cópias de segurança. Falha bloqueia substituição; recuperação pendente deve ser checkpoint fora da poda automática. |
| `worldhistory.archive_version` | Copia após push, fail-soft, `_history/vN`, mantém três versões; não participa do commit. | Não usar como histórico canônico transacional. Histórico R2 será por objetos imutáveis. |
| `presence.py` | `playing.json`, `next.json`, `nudge.json`; advisory, fail-soft; presença expira em quatro horas, sem heartbeat renovador no fluxo atual. Identifica pessoa pelo nome. | Pode continuar no fluxo legado; não concede autoridade nem descobre sessão dedicada. |
| `config.py` | Schema 3; migrations v1/v2 com cópias; `effective_cfg` achata config por mundo. `make_world` gera ID local. | Acrescentar campos opt-in no mesmo dicionário; ID canônico compartilhado separado do ID local e do slot. A é dono dessa alteração. |
| `games.py`: `GameProfile`, `CORE_KEEPER`, `discover_worlds` | Dez jogos como dados; Core Keeper beta, app cliente 1621690, `CoreKeeper.exe`, conta Steam mais recente, mundo por slot. | Manter perfil cliente. B define apenas layout e especificação do Dedicated Server, sem framework de plugins. |
| `game.py` | `launch_game` lança exe/URI Steam; procura por nome de processo, aguarda reinício por 20s e espera 3s após saída. | Pode abrir o cliente para ingressar. Não fornece ownership do processo, readiness, stop ou garantia de flush do Dedicated Server. |
| `steam.py` | Registro Windows, libraryfolders.vdf, appmanifest, descoberta de exe por perfil; sem SteamCMD. | Reutilizar parsing e descoberta, com adaptação pequena para app servidor. B implementa instalação/supervisão. |
| `clouds.py`, `invite.py` | Descoberta de pastas cloud; códigos WS1/DWS1 carregam jogo, slot/nome, pasta/link, não identidade global autenticada. | Continuar legado; acesso dedicado por endpoint configurado + ID canônico + credencial de grupo. Não inventar novo sistema social/OAuth. |
| `health.py`, `preflight.py` | Heurísticas de arquivo maior >= 1024 bytes, extensões parciais, probe de escrita na pasta cloud. | Auxiliares diagnósticos, não validadores de snapshot. Não executar preflight de pasta cloud em mundo dedicado sem adaptação. |
| `controller.py` | QObject, workers Python daemon, sinais Qt, `_busy` é threading.Lock local; polling a cada 20s; `_play_flow` faz pull/play/watch/push. | Reutilizar worker/sinais e seleção de mundo. A acrescenta orquestração dedicada estreita; B produz observações do servidor. |
| `ui/main_screen.py`, `window.py`, `settings.py` | Fases checking/launching/waiting/ingame/pushing; botões de save/restore; saída confirmável no meio da sessão. | B adapta apresentação mínima; A garante gates no controller e callbacks seguros para parar/finalizar. Sem I/O bloqueante na thread Qt. |
| `github_update.py`, `channel.py` | Atualizações de releases apontam para `AndorManu/worldsync`, reports ativados por padrão no canal GitHub. | Antes de distribuir exe do fork, A precisa isolar atualização do upstream para não remover suporte dedicado. Nunca incluir lease/credencial/Game ID em reports. |

## Lacunas relevantes às garantias exigidas

1. Leitura de versão seguida de cópia e incremento não é compare-and-swap. Dois
   escritores podem ler N e produzir N+1, intercalar arquivos e perder uma sessão.
2. O manifesto não carrega inventário/hash remoto para validar todos os bytes.
   Pull verifica se há algum arquivo, não se o conjunto está completo. O controller
   permite lançar cópia local quando há download parcial, arquivos ausentes ou
   conflito recusado. Isso não é admissível no novo modo.
3. Renomear o manifesto atomicamente não publica atomicamente todos os arquivos
   nem transforma a replicação de Drive/Dropbox/OneDrive em transação.
4. O README diz que toda sobrescrita tem backup; em `do_push`, backup da cópia
   compartilhada ocorre somente no ramo de conflito aceito. Não há backup prévio
   de toda sobrescrita normal. O arquivo de histórico posterior também pode falhar.
5. `amend_history_entry` regrava o manifesto inteiro sem CAS; em concorrência pode
   reintroduzir metadados antigos. Nunca apontar esse caminho para o canônico.
6. O aviso de schema novo é principalmente UI; não é controle de acesso contra
   clientes antigos. Um marcador em `version.json` não protege armazenamento.
7. O perfil cliente Core Keeper inclui `worlds/{slot}.world.gzip*`,
   `maps/*/{slot}.mapparts.gzip*`, `worldinfos/{slot}.*` e
   `worldgenparams/{slot}.*`. Não enumera `servermaps`. A lista dedicada precisa
   de evidência do pacote/versão, sem assumir equivalência ao cliente.
8. `MainWindow._restore_backup` chama restore diretamente em worker, fora do
   `_busy` do controller. Close/update, checkpoints, restore, save manual e
   mudança de mundo/config precisam de gates no modo dedicado, além do botão Play.
9. Os testes existentes são valiosos para compatibilidade, mas não provam mutex,
   fencing, concorrência entre hosts, crash consistency nem lifecycle dedicado.
   `test_integration.py` cobre personagens Dragonwilds, não coordenação distribuída.

Esses limites são do protocolo cooperativo atual. Não os corrigir com refatoração
geral nesta preparação. A menor mudança segura é uma via dedicada opt-in com
armazenamento separado, aproveitando utilitários, config e UI existentes.

## Baseline reproduzido

Executado em 2026-10-13, antes de qualquer alteração versionada:

```powershell
python -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt -r requirements-dev.txt
.\.venv\Scripts\python -m pip check
$env:QT_QPA_PLATFORM = 'offscreen'
.\.venv\Scripts\python -m pytest tests -q
```

Resultado: **264 passed in 15.39s**, exit code 0; pip check sem requisitos
quebrados. Windows, Python 3.14.4, pytest 9.1.1, PySide6/shiboken6 6.12.0,
psutil 7.2.2, Pillow 12.3.0, PyInstaller 6.22.3. Somente dependências dos dois
requirements existentes foram instaladas, em `.venv` ignorado pelo Git.
CI usa Python 3.12 em Windows e Ubuntu; essas combinações não foram executadas
localmente. A CI atual dispara push apenas em master e em PR: publicar estas
branches sem PR não dispara a suite automaticamente.

Leitura integral dos testes relevantes: test_sync, test_games, test_every_game,
test_presence, test_backups, test_config, test_game_watch, test_features,
test_saga_history, test_ui_smoke, test_clouds, test_invite, test_integration e
test_github_update, além de conftest. A suite usa arquivos sintéticos/tmp_path,
processos fake nos testes de watch e UI offscreen com paths substituídos.
`conftest.py` desabilita envio de telemetria. Nenhum jogo, SteamCMD ou servidor
foi iniciado; nenhum save real foi usado ou migrado. Nenhuma infraestrutura foi
criada. O status Git após os testes continuou limpo.

## Referências e próximos trabalhos

- [Evidência externa e limites de verificação](SERVER_REFERENCES.md).
- [Contrato normativo e cenários de falha](SHARED_CONTRACT.md).
- [Plano de A](DISTRIBUTED_COORDINATION.md).
- [Handoff autocontido de B](CORE_KEEPER_HANDOFF.md).
