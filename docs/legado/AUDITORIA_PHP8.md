# Auditoria de migração — PHP 5.6 → PHP 8 / MySQL 5.5 → Percona 8.4 LTS

Levantamento estático (grep/inventário de arquivos) do código real do ERP,
feito em cima da cópia extraída em `data/www/` (backup
`www_files_20260829_131106.tar.gz`, ~9,1 GB). **Nenhum código foi
executado nem alterado nesta etapa** — o objetivo aqui é dimensionar o
trabalho antes de tocar em qualquer coisa.

## 1. Tamanho do código

- **5.012 arquivos `.php`** no total dentro do docroot.
- Desses, **2.696 são código próprio do ERP** e **2.316 são bibliotecas de
  terceiros** vendorizadas dentro do próprio docroot (ver seção 6).
- Todos os números das seções seguintes se referem só aos 2.696 arquivos
  próprios, salvo indicação contrária.

## 2. Bloqueador nº1: extensão `mysql_*` (removida no PHP 7)

- **181 arquivos** usam funções da extensão `ext/mysql`, num total de
  **1.360 chamadas**:

  | Função | Ocorrências |
  |---|---|
  | `mysql_query` | 502 |
  | `mysql_fetch_array` | 361 |
  | `mysql_error` | 220 |
  | `mysql_free_result` | 72 |
  | `mysql_num_rows` | 46 |
  | `mysql_real_escape_string` | 32 |
  | `mysql_connect` | 27 |
  | `mysql_select_db` | 25 |
  | `mysql_errno` | 13 |
  | `mysql_fetch_assoc` | 12 |
  | outras (`field_name`, `num_fields`, `insert_id`, `close`, `affected_rows`, `pconnect`, `data_seek`, etc.) | 50 |

- **Boa notícia**: em `conexao.php` (o ponto central de conexão), o
  `mysql_connect()` é armazenado numa variável (`$App_conexao`) que **nunca
  é reaproveitada** nas chamadas seguintes — todo o resto do código chama
  `mysql_query()`, `mysql_select_db()` etc. **sem passar link explícito**,
  contando com o comportamento antigo do PHP de usar "a última conexão
  aberta". Isso é bom para nós: dá pra resolver os 1.360 call-sites com uma
  camada de compatibilidade central (funções `mysql_*` reimplementadas em
  cima de `mysqli`, usando um link global) em vez de reescrever cada uma
  das 181 arquivos na mão.
- **27 pontos distintos de `mysql_connect()`** — ou seja, conexões abertas
  em paralelo/duplicadas, não é só o `conexao.php` principal:
  - `conexao.php` (raiz — principal)
  - `default/common/conexao.php` (uma segunda cópia)
  - `painel_porto/conexao/conexao.php` (uma **quarta** conexão de banco —
    parece um sistema/cliente à parte embutido no mesmo docroot)
  - `App_Fer/atualiza_campanhas.php`, `App_Net/Maquinas.php`,
    `App_Rel/Velhos/{Bilhetes,gera_arquivo_pesquisa,Maquinas}.php`,
    `Boletos/corpo_boleto.php`, `monitoramento_hosts.php`,
    `__scripts/*.php` (scripts de cron) — conexões próprias hardcoded
  - As mesmas 8 chamadas acima **duplicadas de novo** dentro da pasta
    `Temp/` (ver seção 7)
  - `v1/system/database/mysql.php` — driver de banco de uma aplicação
    separada (ver seção 7)
- Já existem **70 arquivos usando `mysqli`** e **1 usando `PDO`** — sinal
  de tentativas anteriores de modernização parcial, provavelmente em
  módulos mais novos.

## 3. Outras funções removidas/depreciadas do PHP (fora do `mysql_*`)

| Função | Arquivos afetados | Status no PHP 8 |
|---|---|---|
| `split()` | 152 | Removida (PHP 7.0) |
| `each()` | 151 | Removida (PHP 8.0) |
| `ereg()` | 22 | Removida (PHP 7.0) |
| `eregi()` | 8 | Removida (PHP 7.0) |
| `create_function()` | 7 | Removida (PHP 8.0) |

Total combinado: **158 arquivos** (pode haver sobreposição, um mesmo
arquivo usando mais de uma função da lista).

Também há **4 arquivos** ainda usando `$HTTP_GET_VARS` / `$HTTP_POST_VARS`
/ `$HTTP_COOKIE_VARS` — resquício da era do `register_globals` (PHP < 4.1),
funcionam como array comum hoje mas indicam código muito antigo que merece
revisão ao lado.

## 4. Risco de charset/collation nos dados

- O `my.cnf` original (copiado literalmente do servidor de produção) **não
  define `character-set-server`/`collation-server`** — isso faz o MySQL
  5.5 cair no padrão da versão: **latin1**.
- `config.php` força
  `Header('Content-Type: text/html; charset=ISO-8859-1')` no app inteiro.
- Essas duas coisas juntas indicam que **os dados reais (nomes de
  clientes, propostas, textos livres) muito provavelmente estão gravados
  em latin1/ISO-8859-1** no banco de produção — o próprio `config.php` já
  mostra um texto corrompido quando lido como UTF-8
  (`"...e Comunicaç..."` com caractere inválido).
- Percona 8.4 usa **utf8mb4 como padrão**. Se o dump for restaurado sem
  cuidado nesse ponto, ou se a conexão `mysqli` nova não fixar
  explicitamente o charset `latin1`, o resultado é **corrupção de
  acentuação em todo o histórico de dados** — não dá pra corrigir isso
  depois de forma trivial.
- **Isso precisa de uma decisão explícita antes de restaurar qualquer
  dump no ambiente novo**: (a) manter `latin1` de propósito no banco novo
  e no charset da conexão (replica o comportamento atual, sem "consertar"
  nada agora), ou (b) fazer uma conversão real latin1→utf8mb4 nos dumps
  antes de restaurar — isso é um projeto à parte, maior que só trocar o
  ambiente Docker, e não está incluído no escopo desta migração a menos
  que você quera incluir.

## 5. Ferramentas de administração de banco expostas no docroot público

Encontrei **três instalações de administração de banco** dentro da mesma
árvore que o Apache serve publicamente:

- `/phpMyAdmin/`
- `/phpMyAdminRaicom/`
- `/mysql/` — na verdade **é o phpMyAdmin 3.3.10** (arquivo
  `RELEASE-DATE-3.3.10` dentro da pasta), lançado em 2010. Essa versão
  específica tem CVEs públicos conhecidos (RCE/LFI). É o achado de maior
  risco de segurança desta auditoria, independente da migração de PHP.

Isso confirma a sua decisão de tirar essas ferramentas do docroot — vale
tratar como prioridade separada da migração de PHP/banco, inclusive antes
dela se possível.

## 6. Bibliotecas de terceiros vendorizadas (fora do código próprio)

Além das 3 cópias de administração de banco acima, o docroot inclui, vendorizadas dentro dele:

- `PHPMailer` — API antiga (`class.phpmailer.php` / `class.smtp.php`,
  pré-Composer/pré-namespace). Versões PHPMailer < 6 tiveram CVEs de RCE
  conhecidos; recomendo trocar pela versão atual via Composer.
- `includes/dompdf/` — inclui uma pasta `.fr-5HYFS7/dompdf-master/`
  (nome típico de conflito/merge de sincronização, ex. Dropbox/rsync) —
  sugere uma extração duplicada ou malfeita que vale limpar.
- `openboleto-master`, `FusionChartsFree` (projeto sem atualização desde
  ~2013), `amcharts4`, `includes/rd-station-api-master`, `FPDF` — cada uma
  precisa de avaliação individual de versão e compatibilidade com PHP 8
  antes de decidir se atualiza, substitui ou remove.

## 7. Duplicações e aplicações escondidas dentro do mesmo docroot

- **`Temp/`** — contém uma cópia de **68 itens de primeiro nível**
  espelhando nomes de pastas/arquivos reais (`App_Fer`, `App_Net`,
  `App_Rel/Velhos`, `Boletos`, `conexao.php`, `monitoramento_hosts.php`
  etc.). Parece um backup ou cópia de trabalho esquecida dentro do docroot
  público — **preciso confirmar se está em uso ou é lixo** antes de
  decidir se migra, arquiva ou descarta.
- **`v1/`** — tem estrutura de framework próprio (`system/`, `admin/`,
  driver de banco próprio em `system/database/mysql.php`) — parece ser uma
  segunda aplicação/API rodando dentro do mesmo domínio, com sua própria
  camada de acesso a dados, independente do `conexao.php` principal.
  Precisa mapeamento à parte antes de migrar.
- **`painel_porto/conexao/conexao.php`** — uma quarta conexão de banco
  separada (já citada na seção 2), possivelmente outro sistema/cliente
  embutido no mesmo docroot.
- **`config.php.bak-antes-docker`** — indica que uma tentativa anterior de
  dockerizar já alterou o `config.php`; vale revisar o diff antes de mexer
  de novo pra não repetir trabalho ou perder um ajuste já feito.

## 8. O que este levantamento NÃO cobre (é estático, não executa o código)

- Comportamento real das ~1.360 chamadas `mysql_*` em runtime (parâmetros
  dinâmicos, uso de `@` pra suprimir erro, etc.).
- Uso de outras extensões PHP possivelmente removidas (ex. `mcrypt`,
  removida no PHP 7.2) — não pesquisado nesta rodada.
- Mudanças de comportamento de tipos do PHP 8 (comparação `==` entre
  string e número, concatenação com `null`, acesso a array em variável
  não-array, etc.) — isso exige um analisador estático de verdade (Rector,
  PHPCompatibility/PHP_CodeSniffer) rodando sobre o código, não só grep.
- O volume de achados específico dentro de `v1/` não foi separado do total
  de 2.696 — ele está incluído, mas por ser aparentemente outra aplicação,
  pode merecer uma auditoria própria.

## 9. Resumo em números

| Item | Valor |
|---|---|
| Arquivos `.php` no docroot | 5.012 |
| Código próprio do ERP | 2.696 arquivos |
| Bibliotecas de terceiros vendorizadas | 2.316 arquivos |
| Arquivos usando `mysql_*` (ext removida) | 181 (1.360 chamadas) |
| Pontos de conexão (`mysql_connect`) distintos | 27 |
| Arquivos já usando `mysqli` | 70 |
| Arquivos já usando `PDO` | 1 |
| Arquivos com `split`/`each`/`ereg`/`eregi`/`create_function` | 158 |
| Arquivos com `$HTTP_*_VARS` legado | 4 |
| Ferramentas de admin de banco expostas no docroot | 3 (phpMyAdmin x2 + phpMyAdmin 3.3.10 disfarçado em `/mysql/`) |
| Cópias/duplicatas suspeitas dentro do docroot | `Temp/` (68 itens), `v1/` (app separada), `painel_porto/` (4ª conexão) |

## 10. Proposta de próximos passos (aguardando seu OK)

Com base nas suas respostas (Percona 8.4 LTS, remover/isolar libs de
terceiros do docroot, usar o dump completo pra testes):

1. **Terceiros e admin tools**: tirar as 3 instalações de admin de banco e
   as libs vendorizadas do docroot; atualizar cada uma pra uma versão
   atual compatível com PHP 8 (via Composer onde der) e recolocá-las fora
   do caminho público.
2. **Charset**: decidir entre manter `latin1` explícito ponta a ponta
   (mais seguro/rápido, sem "arrumar" nada) ou converter os dados para
   utf8mb4 (projeto à parte) — **isso precisa ser decidido antes de
   restaurar o dump no `docker-new`**, porque reverter depois é caro.
3. **Bloqueador `mysql_*`**: construir uma camada de compatibilidade
   central (`mysql_*` reimplementado sobre `mysqli`, com link global) para
   resolver os 181 arquivos sem reescrever call-site por call-site; usar
   uma ferramenta como Rector para tratar `split()`/`each()`/`ereg()`/
   `create_function()` nos 158 arquivos restantes.
4. **Confirmar destino de `Temp/`, `v1/` e `painel_porto/`** antes de
   qualquer um dos três entrar na migração — preciso saber se estão em uso
   real hoje ou são lixo/sistemas paralelos.

Nenhuma dessas ações foi executada ainda. Assim que você confirmar o plano
acima (ou ajustar o que fizer sentido), eu crio o `docker-new/` com
PHP 8 + Percona 8.4 LTS e começo a aplicar as correções.
