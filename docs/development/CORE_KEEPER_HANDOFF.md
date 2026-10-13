# Handoff B: Core Keeper Dedicated Server

Branch: `feature/core-keeper-server`. Repositório:
<https://github.com/VinicKMx/worldsync>. Preparação documental em 2026-10-13,
a partir da master `0d882e2132970f0e5c8fe8407c9e7e66cbbcc337`.
Esta página e seus links estão na sua branch; não é necessário ter a conversa
que originou o trabalho. Nenhum servidor, backend ou novo fluxo foi implementado.

## Produto e arquitetura existente

Queremos que o primeiro jogador adquira autoridade sobre um mundo compartilhado,
baixe o estado canônico e inicie Core Keeper Dedicated Server no próprio PC.
Todos, inclusive o host, entram como jogadores nesse servidor. Ao encerrar a
hospedagem, o mundo só fica disponível a outro host depois de parar o servidor
e confirmar publicação segura. Fechar apenas o cliente Core Keeper não deve ser
interpretado como confirmação de parada do servidor.

WorldSync hoje é um app Windows Python/PySide6, com dez perfis de jogos como
dados e cópia de saves por pasta de Drive/Dropbox/OneDrive. `version.json` e
fingerprints detectam conflitos cooperativos. `playing.json` é presença advisory,
não lock. A UI usa Controller, threads e sinais Qt. Existe suporte beta ao
**cliente** Core Keeper por slot; não há SteamCMD ou supervisor de Dedicated Server.
Preservaremos esse funcionamento com um novo modo dedicado opt-in.

A (Vinicius, branch `feature/distributed-coordination`) implementará coordenador,
leases/fencing, cliente de snapshots e orquestração da sessão. Escolha preliminar
confirmada: um Durable Object por mundo e R2 privado com snapshots imutáveis,
sem provisionamento nesta preparação. Você implementa somente detalhes locais
do servidor e apresentação mínima. Você pode trabalhar com fakes sem backend.

## Leitura inicial e mapa de código

Ler README e DESIGN integralmente, a seção Contributing do README (não existe
CONTRIBUTING separado), [AUDIT](AUDIT.md), [SHARED_CONTRACT](SHARED_CONTRACT.md) e
[SERVER_REFERENCES](SERVER_REFERENCES.md). O contrato revisão 1 é normativo.

| Arquivo/função | O que estudar/reutilizar |
| --- | --- |
| `app/core/games.py`: GameProfile, CORE_KEEPER, discover_worlds | Slot, patterns do cliente e descoberta. Perfil do servidor deve ser separado; o app cliente é 1621690. |
| `app/core/steam.py`: installed_app_ids, _library_folders, _install_dir, find_game_exe | Parsing de bibliotecas/appmanifest. Reutilizar com ajuste pequeno, sem duplicar scanner. |
| `app/core/game.py`: launch_game, find_game_process, wait_for_game_exit | Abertura do cliente e limites do watcher por nome. Não basta para supervisionar seu processo dedicado. |
| `app/core/config.py`: effective_cfg, game_settings, make_world, world_state | Extensões aditivas, IDs locais e separação por jogo. A é dono da persistência/gates. |
| `app/core/sync.py`: world_files, world_fingerprint, sha256_file, _backup_files | A usa esses auxiliares com os patterns dedicados fornecidos por você. Não criar outro motor de sync. |
| `app/core/backups.py`: create_checkpoint, restore_backup | Backups existentes; não aplicar restore sobre diretório servido. |
| `app/controller.py`: start_play, _play_flow, push_now, _do_push_guarded, _world_status, _set_phase | Fronteira UI/core, threads, status. A liga a orquestração aqui. _busy é apenas lock local. |
| `app/ui/main_screen.py`: set_phase, set_status, _rerender | Apresentação existente para hospedar, ingressar, parar e bloqueio de recuperação. |
| `app/ui/window.py`: _wire, closeEvent, _restore_backup, _apply_update | Saída/update/restore hoje permitem caminhos inadequados para sessão dedicada. Consumir gates de A, sem implementar política de lease na UI. |
| `app/ui/settings.py`: load, _submit | Configuração local mínima para SteamCMD/instalação/dados; sem nova tela extensa. |
| `tests/test_game_watch.py`, test_games.py, test_every_game.py | Padrão de fake de processo, descoberta e isolamento dos outros mundos. |
| `tests/test_ui_smoke.py`, conftest.py | Qt offscreen, sandbox por tmp_path, reports desabilitados. |

## Escopo exato e API para A

Criar inicialmente um módulo focado `app/core/core_keeper_server.py` e
`tests/test_core_keeper_server.py`. Usar stdlib (`subprocess`, `pathlib`, `json`,
`time`, eventualmente ctypes no Windows) e psutil já existente. Nenhuma biblioteca
de gerenciamento de servidores, Docker ou framework de adapters é necessária.

Contrato de uma instância `CoreKeeperServer(settings, log)`:

| Método | Resultado |
| --- | --- |
| prepare() | PreparedServer(build_id, layout_id, patterns); verifica/instala binários conforme opções, não inicia servidor nem mundo. |
| start(world_path, session_id) | ServerHealth; inicia uma única instância própria em raiz explicitamente entregue por A; espera readiness com timeout. |
| health() | ServerHealth(state, session_id, game_id, pid, detail); inclui Game ID somente quando fresco e ready. |
| stop() | StopResult(session_id, stopped, clean, detail); cancela start pendente, confirma processo/filhos encerrados e condição de save limpo. |

Estados locais: stopped, starting, ready, stopping, failed, unknown. Erros e
idempotência completos no contrato. Stop deve funcionar após start parcial e
poder cancelar start em progresso. Nunca reiniciar uma session_id terminal.
Não basta fechar wrapper, reconhecer PID pelo nome ou reutilizar Game ID de disco.
Não adicionar método game_id(): health já carrega essa informação.

A chama prepare -> acquire -> download/validação -> start -> activate ->
begin_finalize -> stop -> snapshot/commit. Heartbeat é independente, de A.
Em perda de autoridade A pede stop; você não decide validade de lease, publica
save, libera mundo nem recomeça o servidor sozinho. Na perda de backend, stop
local continua funcionando. Sua evidência stopped/clean serve a A; o backend não
consegue provar que uma máquina maliciosa deixou de executar o jogo.

Na sua branch, testes usam instância/fake com essa forma. A pode usar um fake de
servidor com os mesmos campos em sua branch. Dataclasses podem nascer no módulo
do adaptador quando necessárias; não criar pacote de contratos genérico, HTTP
falso de produção ou arquivos vazios só para facilitar imports.

## Regras de instalação, dados e execução

1. Windows + Steam primeiro. Identificar build oficial antes de suporte funcional.
   Ler README.txt/ARGUMENTS.txt/launchers do pacote e registrar evidência conforme
   SERVER_REFERENCES. O candidato a app servidor é 1963720, não substituir o app
   cliente em CORE_KEEPER. SteamCMD/servidor são programas externos, não novas
   dependências pip nem binários a commitar.
2. Instalação, logs/artefatos de sessão e dados de mundo têm diretórios distintos.
   Dados de execução são raiz privada por session_id, criada/materializada por A
   fora de cloud drives e saves pessoais. Exigir caminho absoluto e confinado;
   rejeitar raiz de disco, caminhos que escapam via links/junctions e overlap com
   instalação ou outra sessão. Não adotar default do jogo silenciosamente.
3. `prepare` não recebe save real e não dispara geração de mundo. Falha de rede,
   licença, download parcial ou versão desconhecida resulta em erro legível.
   Atualização só com processo parado e fora de sessão; conferir build e arquivos,
   não confiar apenas em exit code ou texto de sucesso do SteamCMD. Não atualizar
   nem aplicar validate sobre arquivos de configuração customizados em uso.
4. Construir argv por lista, `shell=False` sempre que possível; testar caminhos
   com espaços/Unicode. Se launcher oficial exige shell, usar wrapper fixo e
   argumentos validados, sem interpolar texto do usuário. Ocultar consoles
   auxiliares sem perder mecanismo de encerramento requerido pela build.
5. Antes de spawn, registrar session_id, intenção e informações de propriedade;
   depois registrar PID/creation time/paths e filhos. Janela de crash entre spawn
   e registro não pode resultar em "nenhum processo" presumido. Sem capacidade
   de identificar processo próprio, retornar unknown e impedir outra instância.
6. Confirmar `-datapath`, slot, argumentos e mecanismo de Game ID no pacote. ID
   antigo, timestamp sozinho, log incompleto ou arquivo vazio não prova readiness.
   Isolar/remover somente artefato gerado da tentativa anterior em pasta gerenciada,
   conferir processo vivo e evidência da tentativa atual. Nunca abrir mundo vazio
   quando a cópia canônica esperada não existe. Sem restart automático.
7. Definir inventário de mundo do Dedicated Server com fixtures por build;
   investigar servermaps e metadados necessários. Não reutilizar cegamente o
   padrão maps do cliente, nem copiar AppData inteiro. Config privada de host,
   credenciais, logs e Game ID ficam fora do snapshot. Declarar layout_id e
   incompatibilidades de build para A bloquear hospedagem com dados incompatíveis.
8. Encerramento gracioso é responsabilidade sua. A FAQ oficial orienta `q` no
   launcher; isso não prova que enviar `q` para stdin do exe seja suportado.
   Inspecionar launcher e confirmar comportamento. Espera fixa de 3s do watcher
   atual não é prova de flush. Timeout/kill/crash => clean=false; nunca alegar
   sucesso para permitir publicação. Não matar todos os processos pelo nome.
9. Game ID fica visível apenas aos membros pela sessão ACTIVE de A. UI de
   convidados mostra/copiar ID; ingresso manual em Join Game basta. Sem scraping
   de tela, teclado simulado, portas abertas automaticamente ou suposição de RCON.

## Testes automatizados sem jogo

Tudo deve usar tmp_path, bytes sintéticos, monkeypatch/unittest.mock e relógio
injetável. Substituir Popen/run, SteamCMD, psutil e toda sondagem do sistema antes
de testar. Nenhum teste padrão baixa binário, conecta Steam ou abre mundo.

| Caso | Asserção que importa |
| --- | --- |
| prepare já instalado / atualização / erro de licença / download truncado | Idempotência, erro claro, zero spawn de jogo e zero escrita em dados. |
| paths com espaços/Unicode, traversal, junction, instalação igual aos dados | argv preservado; caminhos inseguros rejeitados antes de qualquer subprocesso. |
| start normal / duplicado / reaproveitar session_id terminal | Um único spawn; estado e identidade corretos; sem restart posterior. |
| Game ID residual, parcial, errado, ausente, timeout, processo morreu cedo | Nunca ready/activate com artefato residual; stop ainda utilizável. |
| launcher termina mas filho continua; PID reutilizado/acesso negado | Não declarar stopped indevidamente; não atingir processo estranho. |
| stop normal / repetido / durante start / pendurado / kill | Nenhum spawn tardio depois do stop; clean só com término gracioso verificado. |
| WorldSync reinicia com journal ausente/corrompido | Unknown/bloqueio conservador; sem auto-respawn. |
| Layout com slots vizinhos e arquivos privados | Só arquivos de mundo corretos entram em patterns; roundtrip preserva bytes. |
| UI offscreen com sessão ativa de terceiro | Mostra Game ID, zero start local; stop/quit encaminhados a A. |

Use testes de comportamento, não asserts que só repetem strings internas. A
simula lease expirado e decide stop em seus testes; seu fake de processo confirma
que stop funciona independentemente da rede. Teste integrado posterior combina
coordenador fake e seu adapter com subprocesso fake, sem hardware.

## Plano incremental e critérios de aceitação

1. Especificar opções locais/records e fixtures; testes de paths, leitura de
   build e argv, sem instalação real. Entregar inventário preliminar e gates a A.
2. Implementar prepare/SteamCMD por subprocessos mockados: timeouts, exit code,
   arquivos/build esperados, repetição segura; nenhum acesso a saves.
3. Implementar start/health/stop e registro de processo; cobrir todos os cenários
   da matriz antes de UI. Resolver mecanismo Windows real de parada na build-alvo.
4. Adaptar minimamente main_screen/settings/window consumindo callbacks de A;
   em ausência da orquestração usar fakes somente em testes, manter feature
   desligada. Entregar sinais/dados para A integrar controller/config.
5. Teste conjunto com fakes e regressão dos dez jogos. Só depois, sessão
   explicitamente autorizada de hardware com mundo descartável; registrar build,
   start, ingresso por ID, gravação, parada, relançamento do snapshot e comparação.

Aceite automatizado: suite existente passa; nenhuma dependência nova sem motivo;
um processo próprio por sessão; install/update separado de dados; readiness não
aceita ID antigo; stop clean verificável; falhas não publicam/liberam; zero
mutex/lease/fencing/R2 no módulo B; UI não bloqueia thread principal; perfil
legado Core Keeper e outros nove jogos preservados. Aceite funcional real fica
pendente até testar pacote, mecanismo de parada e roundtrip em hardware.

Proibido nesta frente: refatorar sync/presence para lock, tratar playing.json como
autoridade, implementar serviço cloud, alterar formato WS1/DWS1, migrar saves
reais automaticamente, habilitar modo dedicado sem gates de A, fazer deploy,
merge na master, force push ou modificar arquivos gerados sem sua fonte.

## Ambiente e comandos

Em um clone novo (se já houver clone, primeiro inspecionar status e preservar
alterações; não executar reset/clean):

```powershell
git clone --branch feature/core-keeper-server https://github.com/VinicKMx/worldsync.git
cd worldsync
git status --short --branch
py -3.12 -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt -r requirements-dev.txt
$env:QT_QPA_PLATFORM = 'offscreen'
.\.venv\Scripts\python -m pytest tests -q
.\.venv\Scripts\python -m pytest tests/test_game_watch.py tests/test_games.py tests/test_every_game.py tests/test_ui_smoke.py -q
```

Python 3.12 é a versão da CI; se só houver 3.14, `python -m venv .venv` foi
validado nesta auditoria (3.14.4). Baseline: 264 passed em 15,39s; dependências
resolvidas em AUDIT. Após criar seus testes:
`.\.venv\Scripts\python -m pytest tests/test_core_keeper_server.py -q`.
Não executar `python -m app` ou `build.ps1` para validar este incremento: testes
offscreen bastam e não carregam configurações/saves pessoais.

## Integração posterior

Você e A começam no mesmo commit documental. Trabalhar em arquivos distintos,
commits pequenos; comunicar mudança de contrato pelo próprio documento. A pode
começar a máquina de estados usando fake de CoreKeeperServer imediatamente.
Depois de sua API estar estável, A substitui a factory fake pelo adapter e integra
os sinais da UI. Juntos revisam quit/update/restore/manual save. Nenhuma branch
ganha permissão para sobrescrever a outra ou a master. A publicação destas
branches não dispara automaticamente a CI atual (push só em master; PR não faz
parte desta sessão).

## Prompt curto de retomada

> Trabalhe no fork https://github.com/VinicKMx/worldsync, branch
> feature/core-keeper-server, como desenvolvedor B. Leia README, DESIGN e
> docs/development/CORE_KEEPER_HANDOFF.md, SHARED_CONTRACT.md, AUDIT.md e
> SERVER_REFERENCES.md antes de editar. Implemente incrementalmente o adaptador
> local Core Keeper Dedicated Server (prepare/start/health/stop), começando por
> subprocessos e dados sintéticos com mocks. A cuida de coordenação, leases,
> fencing, R2 e orquestração; não implemente esses componentes. Preserve perfis e
> sync atuais, use stdlib/psutil/PySide6 existentes e integre UI minimamente.
> Inspecione Git e preserve alterações. Não use saves reais, não inicie jogo ou
> servidor sem autorização específica de teste real, não faça deploy, merge na
> master, PR ou force push. Execute a suite offscreen e documente gates ainda
> pendentes de build/encerramento em hardware.
