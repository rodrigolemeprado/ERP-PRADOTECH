# ERP - RPRADOTECH — Meta e Contexto do projeto

Cole a **Meta** na descrição do projeto e o **Contexto** nas instruções do
projeto. Este arquivo também fica no conhecimento do projeto, como referência.

## Meta

Construir o ERP RPRADOTECH: um ERP novo que entregue todas as funcionalidades
do ERP legado do Grupo Redes, com interface elegante, minimalista e totalmente
responsiva, mais opções de configuração e customização, e emissão fiscal
(NFS-e e NFCom) integrada aos padrões vigentes da prefeitura e da SEFAZ.

## Contexto

### Objetivo
Criar um ERP novo do zero com paridade funcional total com o ERP legado do
Grupo Redes (marca antiga RAICOM). NÃO é para portar nem corrigir o código
antigo. O legado é a fonte de verdade sobre funcionalidades, regras de
negócio, relatórios e dados.

### Princípios (valem para toda decisão)
1. Funcionalidade total: todo recurso usado no legado tem equivalente no
   sistema novo, ou uma decisão explícita e registrada de não levar.
2. Elegância: visual limpo e coerente, com um design system único
   (tipografia, espaçamento, cores e componentes padronizados).
3. Totalmente responsivo: toda tela funciona bem do celular ao monitor
   grande, pensada mobile-first. Tabelas e formulários se adaptam (por
   exemplo, viram cards ou etapas) em vez de só encolher.
4. Minimalista: só o essencial à vista, ações secundárias em menus, poucos
   cliques por tarefa, nada de poluição visual.

Customização: perfis e permissões granulares, parâmetros por empresa e por
módulo, campos e relatórios configuráveis, tudo sem mexer no código.

### Arquivos de conhecimento do projeto
- HISTORICO.md — tudo o que já foi feito e decidido até a criação do projeto.
- AUDITORIA_PHP8.md — auditoria técnica do código legado (números, conexões,
  libs, riscos).
- RUNBOOK.md e infra-docker/ — o ambiente Docker do legado (paridade com
  produção) e os scripts de inventário/backup.
- schema_erp_raicom.sql e schema_novabase.sql — estrutura dos bancos do
  legado (sem dados).
- legado_arquivos_php.txt — lista dos 2.696 arquivos PHP próprios do legado.
- legado-fiscal/ — código-fonte do legado para NFS-e, SCM (modelo 21), SVA
  e Nota de Débito.

### O sistema legado (referência)
- PHP 5.6 procedural, ~2.700 arquivos próprios, sem framework.
- Bancos (MySQL 5.5, todos em latin1): erp_raicom (217 tabelas, o principal)
  e novabase (133 tabelas; o nome não aparece no código, uso a confirmar).
- Senhas de usuário no legado: md5('R@' + senha + ';') (ver login.php).
- Roda dockerizado no cliente: servidor docker-services, /opt/Docker/ERP-GR
  (usuário noc). Continua em produção até o cutover.
- Cópia local para análise: C:\rprado\erp-gr\erp-gr\data\www. Dump:
  C:\rprado\erp-gr\20260829_131106\mysql_all_databases_20260829_131106.sql.gz.
- Áreas do menu: Principal, Pré-Vendas, Orçamento, Financeiro, Logística,
  Marketing, Relatórios, RH, Desenvolvimento, CCO.
- Módulos/pastas (conteúdo a mapear): App_cad, App_pes, App_Fin, App_Ger,
  App_Pre, App_Rel, App_List, App_Net, App_Cpd, App_Fer, App_Leads_Propostas,
  Propostas, Revendedor, SCM, SVA, Cco/RelatorioCco, Boletos,
  remessa/remessa_bradesco (CNAB), Nota_Debito, loteNFSE, PDF_*, Marketing,
  comunicado, contratos_canais, painel_porto, v1.
- Integrações e rotinas: RD Station, boletos BB/Bradesco + remessa, e-mails
  de cobrança e lembrete, ordens de serviço/chamados com bloqueio por OS
  atrasada (trava_de_os), monitoramento de hosts, Dashboard, HubPop, crons
  em __scripts/.

### Módulo fiscal (prioridade crítica)
Como o legado faz hoje:
- NFS-e São Paulo (ExportarNFSE.php, FuncoesNFSE.php, ImportarNFSE.php):
  gera lote de RPS em TXT (layout 1, só ISS) para upload manual no portal da
  Nota Fiscal Paulistana e importa o retorno. Situação T/F definida pelo
  código IBGE do município (3550308). Últimos arquivos em loteNFSE/: ago/2023.
- NF de comunicação modelo 21 do SCM (ControlSCM.php, ImpressaoSCM.php,
  PDF_SCM.php, Talonario.php), com arquivos do Convênio ICMS 115/03.
- Faturamento de SVA (ControlSVA/PDF_SVA) e Nota de Débito
  (ControlNotaDebito/PDF_Nota_Debito): natureza fiscal a confirmar com o
  contador.
- Boletos e remessa CNAB (Banco do Brasil e Bradesco).

Situação atual: há 2 a 3 anos o cliente faz a emissão manualmente, fora do
ERP. O novo não precisa conviver com o fluxo antigo: já nasce no padrão
mais novo.

O que o novo precisa fazer (padrões pesquisados em nov/2026, revalidar nos
manuais oficiais antes de implementar):
- NFS-e SP: integração por Web Service (XML assinado com certificado A1
  ICP-Brasil) no layout 2 (ISS + grupo IBSCBS), no endpoint novo
  nfews.prefeitura.sp.gov.br (manual do Web Service v3.4 ou mais novo). Envio
  de lote síncrono e assíncrono, consulta, cancelamento e download de
  XML/PDF. Substitui o upload manual de TXT.
- SCM: NFCom modelo 62 (Ajuste SINIEF 07/2022) no lugar do modelo 21 +
  Convênio 115. Autorização em tempo real pela SEFAZ, DANFE-COM, eventos
  (cancelamento) e contingência.
- Manter todas as opções de envio do legado (por lote e período, seleção
  individual, reemissão, importação de retorno, impressão/PDF), modernizadas
  e automáticas.
- Arquitetura: camada de emissores fiscais plugável (prefeitura de SP hoje,
  NFS-e Padrão Nacional e outros municípios no futuro), fila de envio com
  reprocessamento, log e auditoria de cada transmissão, guarda dos XMLs pelo
  prazo legal, ambiente de homologação antes da produção e certificado
  digital guardado com segurança.
- Regras tributárias (alíquotas, CST/cClassTrib, IBS/CBS, retenções) ficam
  parametrizáveis e são validadas com o contador do cliente.

### Requisitos técnicos gerais
- Migração dos dados do erp_raicom para o modelo novo, com conversão latin1 →
  UTF-8 e conferência de totais, principalmente nos dados financeiros e
  fiscais.
- Senhas: aceitar o hash legado no primeiro login e regravar com hash moderno
  (Argon2id/bcrypt), sem obrigar todo mundo a trocar a senha no cutover.
- Segurança desde o início: autenticação forte, consultas parametrizadas,
  nenhuma credencial no código e nenhuma ferramenta de administração exposta.
- Deploy em containers Docker (banco preliminar: Percona Server 8.4 LTS).

### Fases
1. Levantamento funcional do legado → catálogo de funcionalidades priorizado
   (o fiscal primeiro).
2. Stack, arquitetura, design system e modelagem de dados.
3. Plano de migração de dados (mapeamento tabela a tabela).
4. Desenvolvimento por módulo, validando cada um contra o legado (para o
   fiscal, homologação na prefeitura e na SEFAZ).
5. Operação em paralelo, migração final e cutover.

### Pendências
- Stack (linguagem, framework, front-end).
- Emissão fiscal: há 2 a 3 anos o cliente emite tudo manualmente, fora do
  ERP. O sistema novo já nasce com os padrões mais novos aceitos pela
  prefeitura e pela SEFAZ (NFS-e layout 2 por Web Service e NFCom). Confirmar
  com o cliente qual documento sai hoje para o SCM (NFCom ou ainda modelo 21)
  e por onde ele é emitido.
- Confirmar o uso real de Temp/, v1/, painel_porto/, App_Rel/Velhos e do
  banco novabase.
- Definir o MVP.

### Como trabalhar
- Responder em português, direto ao ponto.
- Analisar o legado só em modo leitura: nunca alterar o código nem o banco de
  produção.
- Toda funcionalidade levantada vai para o catálogo com a origem
  (arquivo/tela) e a regra de negócio observada.
- Decisões de arquitetura e de escopo ficam registradas em docs/, com o
  motivo.
- Padrões fiscais mudam: sempre conferir a versão vigente nos portais
  oficiais (Prefeitura de SP, portal NFCom/SVRS, NFS-e Nacional) antes de
  implementar.
- Em Windows, usar PowerShell. No servidor, Bash (Debian).
