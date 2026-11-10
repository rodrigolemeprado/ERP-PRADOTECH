# Histórico do projeto (até a criação do projeto ERP - RPRADOTECH)

Resumo do trabalho feito antes deste projeto existir, para quem chega agora
(pessoa ou Claude). Datas em 2026.

## 1. Inventário e backup do servidor original (29/08)
- Servidor original: Debian 8 (Jessie), Apache 2.4.10 + mod_php 5.6.40,
  MySQL 5.5.62, domínio erp.gruporedes.global.
- `scripts/01-inventario.sh`: levantamento só de leitura (pacotes, vhosts,
  php -i, my.cnf, crons, serviços).
- `scripts/02-backup-completo.sh`: backup do docroot (`www_files_*.tar.gz`,
  ~9,1 GB) e de todos os bancos (`mysql_all_databases_*.sql.gz`, ~63 MB),
  com SHA256SUMS. Cópia em `C:\rprado\erp-gr\20260829_131106\`.

## 2. Dockerização com paridade (projeto erp-gr)
- Dois serviços sobre `debian/eol:jessie`:
  - **app**: Apache 2.4 + PHP 5.6.40 (mod_php), com os vhosts reais
    (`erp.raicom.com.br.conf` em HTTP, `default-ssl.conf` em HTTPS). O
    entrypoint liga o HTTPS só se o certificado estiver em `data/cert/`.
    `php-overrides.ini` com os valores reais medidos em produção.
  - **db**: MySQL 5.5.62 instalado dos repositórios jessie + jessie-security
    (archive.debian.org), com o `my.cnf` literal de produção. O entrypoint
    restaura o dump de `data/mysql-init/` na primeira subida (volume vazio) e
    normaliza o host/senha de `root` e `erp` depois do restore.
- O código fica montado por volume (`data/www`), não dentro da imagem.
- `config.php` do legado foi ajustado para ler `DB_HOST` do ambiente (cópia
  anterior em `config.php.bak-antes-docker`).
- Detalhes, armadilhas e passo a passo: `infra-docker/README.md` e
  `RUNBOOK.md`.

## 3. Subida no ambiente do cliente
- Servidor `docker-services`, pasta `/opt/Docker/ERP-GR`, funcionando.
- Usuário `noc` adicionado ao grupo `docker` (é a conta de operação e já tinha
  sudo; precisa de novo login para o grupo valer).
- Para o contratado (`dev-erp`) foi usado Docker Rootless, restrito a uma
  pasta, porque ele não pode ter acesso equivalente a root.

## 4. Auditoria do código para PHP 8 (31/08)
- Resultado completo em `AUDITORIA_PHP8.md`. Pontos principais:
  - 5.012 .php no docroot: 2.696 próprios e 2.316 de bibliotecas de terceiros.
  - ext/mysql em 181 arquivos (1.360 chamadas), com conexão implícita/global.
  - 158 arquivos com split/each/ereg/eregi/create_function.
  - 3 ferramentas de administração de banco expostas no docroot (incluindo
    phpMyAdmin 3.3.10 em `/mysql/`): risco de segurança no legado em produção.
  - Duplicatas/aplicações escondidas: `Temp/`, `v1/`, `painel_porto/`.
  - Dados em latin1.
- Decisões tomadas na época: auditoria primeiro, Percona Server 8.4 LTS,
  tirar phpMyAdmin e libs de terceiros do docroot, usar o dump completo nos
  testes.

## 5. Mudança de rumo: ERP novo (10/11)
- A ideia deixou de ser migrar o código antigo para PHP 8. Agora é construir
  um **ERP novo** com todas as funcionalidades do antigo, aparência
  totalmente nova e mais opções de customização.
- Princípios: funcionalidade total, elegância, totalmente responsivo,
  minimalista.
- Requisito explícito: enviar as notas fiscais como o antigo fazia, com todas
  as opções de envio, mas no padrão mais novo da prefeitura e do estado.

## 6. Levantamento fiscal do legado (10/11)
- **NFS-e São Paulo**: o legado gera lote de RPS em TXT (layout 1) para upload
  manual no portal e importa o retorno (`legado-fiscal/ExportarNFSE.php`,
  `FuncoesNFSE.php`, `ImportarNFSE.php`). O último arquivo em `loteNFSE/` é de
  ago/2023.
  - Padrão atual: desde 01/01/2026 São Paulo tem o layout 2 (ISS + IBS/CBS)
    só por Web Service ou emissão online; o TXT só aceita o layout 1.
- **SCM**: o legado emite NF modelo 21 com os arquivos do Convênio ICMS 115/03
  (`legado-fiscal/ControlSCM.php`, `PDF_SCM.php`, `Talonario.php`).
  - Padrão atual: NFCom modelo 62 (Ajuste SINIEF 07/2022), obrigatória para
    provedores desde 01/11/2025 (prorrogação por regime especial encerrada em
    01/08/2026). **Confirmar com o cliente como ele emite hoje.**
- SVA e Nota de Débito: natureza fiscal a confirmar com o contador.
- Fontes consultadas: Prefeitura de SP (orientações NFS-e 2026), TecnoSpeed
  (novo layout SP), portal NFCom/SVRS (FAQ), SCM Engenharia (NFCom para ISPs).

## 7. Estrutura do banco (10/11)
- Extraída do dump, sem dados: `schema_erp_raicom.sql` (217 tabelas) e
  `schema_novabase.sql` (133 tabelas). Os dois bancos estão em latin1.
- O nome `novabase` não aparece em nenhum arquivo PHP próprio: uso a
  confirmar.
- Senhas de usuário no legado: `md5('R@' + senha + ';')`.

## 8. Próximos passos sugeridos
1. Responder as pendências do Contexto (stack, emissão fiscal atual, pastas
   e banco de uso duvidoso, MVP).
2. Começar o catálogo de funcionalidades, área por área do menu, com o fiscal
   primeiro.
3. Definir stack, design system e modelo de dados.
