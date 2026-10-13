# Fontes externas e decisões sobre o Dedicated Server

Consulta em 2026-10-13. Distinguir documentação verificada, proposta de projeto
e validação pendente. Nenhum pacote servidor foi instalado/executado nesta sessão.

## Core Keeper: evidência primária

A [FAQ publicada pela equipe do jogo no Steam](https://steamcommunity.com/app/1621690/discussions/0/758430465206037848/)
confirma instalação via Library > Tools > Core Keeper Dedicated Server, geração
de Game ID na janela e entrada dos jogadores pelo menu Join Game usando esse ID.
Para encerrar, instrui pressionar `q` na janela de comando e alerta que fechar
a janela pode deixar o servidor executando em background.

Diretórios padrão de mundos indicados na mesma FAQ:

| Plataforma | Caminho |
| --- | --- |
| Windows | `%USERPROFILE%\AppData\LocalLow\Pugstorm\Core Keeper\DedicatedServer\worlds` |
| Linux | `~/.config/unity3d/Pugstorm/Core Keeper/DedicatedServer/worlds` |

Esses são caminhos documentados, **não destinos de testes nem diretórios a serem
adotados automaticamente pelo WorldSync**. Nosso adaptador deve exigir raiz
privada explícita. O ID de sessão não deve ser confundido com slot, app ID Steam
ou ID canônico do WorldSync. A FAQ não especifica protocolo de automação de `q`,
API de readiness, confirmação de flush nem como encerrar o exe diretamente.

A [documentação Steamworks de distribuição de servidores](https://partner.steamgames.com/doc/sdk/uploading/distributing_gs)
é a referência de publicação/obtenção de aplicativos servidor. A página
[SteamCMD da Valve](https://developer.valvesoftware.com/wiki/SteamCMD) retornou
403 no leitor web desta sessão; não foi tratada como conteúdo validado.

## Gate de evidência da implementação B

Antes de habilitar execução real, obter o pacote oficial via Steam/SteamCMD em
diretório de instalação isolado e ler integralmente README.txt, ARGUMENTS.txt
(se existir), Launch.bat/Launch.ps1 ou scripts equivalentes da build escolhida.
Registrar app ID, build ID, plataforma, data e SHA-256 desses arquivos. Não
versionar binários do jogo. A instalação local inspecionada tinha o cliente,
não o Dedicated Server; não havia documentação do pacote servidor a validar.

Os itens abaixo são **candidatos de implementação a confirmar no pacote**, não
garantias de uma build testada nesta auditoria:

| Item | Hipótese a verificar e critério para liberar |
| --- | --- |
| Distribuição | App servidor candidato `1963720`, distinto do cliente `1621690`; confirmar appmanifest e saída oficial SteamCMD. |
| Comando de instalação | `steamcmd.exe +force_install_dir <instalacao-absoluta> +login anonymous +app_update 1963720 validate +quit`; validar suporte a login anônimo, erros de licença, exit code e arquivos esperados. Não incluir credenciais pessoais em argv. |
| Diretório/slot | Confirmar `-datapath <raiz-privada>` e `-world <slot>`; garantir que argumentos têm precedência e que dados não escapam para o default. |
| Inicialização | Executável candidato `CoreKeeperServer.exe`; confirmar argumentos do launcher. Não adicionar `-nographics`, flags de geração ou instalação do app runtime 1007 por suposição. |
| Descoberta | Confirmar se a build escreve GameID.txt, GameInfo.txt ou outra saída; parser limitado à build/formatos testados. Nunca aceitar arquivo residual como readiness. |
| Encerramento | A FAQ documenta `q` para o launcher. Inspecionar como ele implementa isso; não presumir que `Popen.stdin.write('q')` funciona para o exe. Testar parada graciosa, filhos e flush. |
| Conteúdo canônico | Inventariar dados de mundo e mapas de servidor, incluindo necessidade de `servermaps`, worldinfos/worldgenparams e backups do jogo. Separar configuração de host, Game ID, logs, mods e saves de personagens. |

O formato de Game ID é opaco no contrato compartilhado. Não fixar uma regex
retirada de um guia antigo. Limites de tamanho/encoding do transporte são de A;
validação de formato/readiness por build é de B. No primeiro incremento suportar
Windows + Steam e ingresso manual por ID; Linux, crossplay, IP direto, RCON e
ingresso automático ficam fora até evidência específica. Esses limites reduzem
o escopo, não afirmam ausência de capacidades no jogo.

## Cloudflare: adequação técnica

As fontes oficiais consultadas sustentam a escolha, com estas condições:

- [Storage dos Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/access-durable-objects-storage/): estado persistente por objeto, armazenamento transacional; usar um objeto por mundo, com backend SQLite para namespace novo.
- [API SQLite](https://developers.cloudflare.com/durable-objects/api/sqlite-storage-api/): a mudança de owner/lease/fence/head/recibo precisa ocorrer em uma única transação local ao objeto.
- [Regras dos Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/): I/O externo permite interleaving. Após upload/validação em R2, reler e validar lease/estado/base/fence dentro da transação final.
- [Consistência R2](https://developers.cloudflare.com/r2/reference/consistency/): objetos são fortemente consistentes, mas escritas concorrentes na mesma chave não são nosso protocolo de commit. Não usar URL cacheada como fonte do head.
- [API R2](https://developers.cloudflare.com/r2/api/workers/workers-api-reference/): backend pode armazenar/ler objetos e aplicar precondições. Chaves exclusivas e imutabilidade devem ser impostas pelo serviço; não confiar em metadados de hash enviados pelo cliente nem confundir ETag com SHA-256.

**Decisão de projeto:** bytes são gravados antes, em objeto imutável; somente
depois o DO troca atomicamente o ponteiro canônico. Não há transação única entre
DO e R2. Uma falha pode deixar objeto órfão, mas nunca deve deixar head apontando
para upload incompleto. Versionamento é do WorldSync por chaves/manifestos,
sem pressupor versioning S3 nativo ou usar R2 como mutex.

Não foi provisionado Cloudflare, definido plano comercial, criado token/bucket,
nem adicionado SDK, Worker, wrangler ou configuração de deploy. Limites de
upload/CPU, autenticação e compatibilidade do runtime escolhido continuam gates
do plano de A antes de qualquer rollout.
