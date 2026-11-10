# Catálogo de funcionalidades fiscais do ERP legado (módulo fiscal)

Fonte: 15 arquivos PHP de `legado-fiscal/` (App_pes/), `schema_erp_raicom.sql` (217 tabelas) e `CONTEXTO_PROJETO.md`.
Análise somente leitura; nenhum PHP foi executado.

## Convenções deste documento

- **[obs]** = observado diretamente no código ou no schema (com arquivo e linha aproximada).
- **[inf]** = inferido por dedução; precisa de confirmação.
- **[nf]** = depende de arquivo que **não foi fornecido** (não é possível descrever o comportamento).
- Números de linha são aproximados e referem-se ao arquivo convertido para UTF-8.
- Os arquivos fornecidos estão em UTF-8 (alguns ASCII), embora as páginas declarem `charset=iso-8859-1` e o banco seja latin1. O legado mistura `utf8_decode()` em alguns pontos e não em outros (risco de caracteres corrompidos na migração).

### Arquivos citados no código mas NÃO fornecidos [nf]

`ControlNotaFiscalServico.php` (ações `exportar` e `importar` da NFS-e), `NotaFiscalServico.php` (tela que monta `$_SESSION['where_nfse']`), `CnsSCM.php`, `CnsSVA.php`, `CnsNotaDebito.php` (listagens/consultas), `formSVA.php`, `formNotaDebito.php`, `LogEnvio.php` (`keysLogEnvio`, `insertLogEnvio`), `funcoes.php` (`executar`, `convertDateToBr/Bd`, `toDataSQL`, `mascara`, `mesExtenso`, `redireciona`, `a`, `charComplete`, `verificaSituacao`, `decryptIt`), `conexao.php`, `config.php`, `toolbarFinanc.php`, a biblioteca FPDF, e a pasta `loteNFSE/`.
Consequência: **a geração do arquivo de lote (que junta cabeçalho + detalhes + rodapé), a gravação do arquivo em disco, o parser do retorno da NFS-e e a origem dos registros de `nota_fiscal_servico` não puderam ser lidos.** Só as funções auxiliares (`FuncoesNFSE.php`) estão disponíveis.

---

## 1. Visão geral do fluxo fiscal do legado

### 1.1 Matéria-prima comum: parcelas do Contas a Receber

SCM, SVA e Nota de Débito nascem da mesma origem: **parcelas do contas a receber** (`payment_detalhe`, ligada a `payment` e `conta_receber`), do cliente (`payment_fornecedor`). O tipo de documento é decidido pelo **item do plano de contas** do lançamento (`payment.nr_item`), com códigos fixos no código [obs]:

| Documento | Itens de plano de contas (`nr_item`) | Marca de "já faturado" em `payment_detalhe` | Contador em `info_fiscal_empresa` |
|---|---|---|---|
| SCM (Mod. 21) | 323 e 175 (175 = 20% do valor) | `gerado_scm`='S', `nr_scm` | `nr_ultima_scm` |
| SVA (fatura) | 324, 175 (80% do valor) e 467 | `gerado_svascm`='S', `nr_svascm` | `nr_ultima_sva` |
| Nota de Débito | 174, 467 e 430 | `gerado_svascm`='S', `nr_svascm` (mesma coluna do SVA) | `nr_ultima_nd` |
| NFS-e (RPS) | não vem de `payment_detalhe` nos arquivos fornecidos [nf] | `nota_fiscal_servico.emitiu_nf` | `nr_ultima_rps`, `nr_ultima_nfse` (uso não visto) |

Os significados dos itens 323/324/175/467/174/430 **não constam do schema** (sem dados); ver seção 5.

### 1.2 SCM, SVA e Nota de Débito (fluxo comum, 3 etapas) [obs]

1. **Consulta de clientes/parcelas a faturar** (botão "Consulta Clientes" nas telas `SCM.php`, `SVA.php`, `NotaDebito.php`): filtros por contrato, cliente, período de emissão, período de vencimento (e, no SCM/SVA, situação do pagamento "Em Aberto"/"Pago" e unidade de negócio). A tela faz POST via AJAX (`action=gerar`) para o `Control*.php`, que devolve uma tabela HTML com checkbox por parcela.
2. **Seleção e geração** ("Gerar SCM/SVA/Nota Débito"): o usuário marca as parcelas (individual ou "marcar todos"), informa a data de emissão (padrão = hoje) e envia (`action=atualiza`). Em **uma transação** (`BEGIN`/`COMMIT`/`ROLLBACK`), para cada linha selecionada: calcula o próximo número, grava o registro (`scm`/`sva`/`nota_debito`), gera um PDF em disco (`PDF_Scm/`, `PDF_Sva/`, `PDF_Nota_Debito/`), insere linha em `log_envio` (controle do e-mail de cobrança), marca a parcela como faturada e avança o contador em `info_fiscal_empresa`.
3. **Consulta/edição posterior**: "Consulta SCM/SVA/Nota de Débito" grava um `WHERE` em sessão (`where_scm`, `where_sva`, `asEDs`) e redireciona para `Cns*.php` [nf]. A edição individual permite alterar vencimento, valor e descrição; cancelar e reativar. A impressão do SCM é feita por `ImpressaoSCM.php`.

Não há, nesses três fluxos, **nenhuma transmissão a SEFAZ ou prefeitura** nos arquivos fornecidos. O "documento fiscal" é o PDF gerado localmente (e, no SCM, um código de autenticação MD5 próprio e funções para arquivos do Convênio ICMS 115/03 que não estão ligadas a nenhuma tela fornecida).

### 1.3 NFS-e São Paulo (RPS em lote TXT, layout 1) [obs + nf]

```
nota_fiscal_servico (RPS cadastrados em tela não fornecida; situacao 'A')
   -> ExportarNFSE.php: lista RPS (WHERE vindo da sessão + situacao='A'), usuário marca e clica "Gerar Lote"
   -> POST para ControlNotaFiscalServico.php (action=exportar)  [nf]
        usa FuncoesNFSE: headerRpsLote() + detalheRpsLote() por RPS + footerRpsLote()
        detalheRpsLote() marca nota_fiscal_servico.emitiu_nf='S'
   -> arquivo TXT em loteNFSE/ -> upload MANUAL no portal da Nota Fiscal Paulistana
   -> ImportarNFSE.php: usuário envia o arquivo de retorno -> ControlNotaFiscalServico.php (action=importar) [nf]
```

O contexto do projeto informa que, há 2 a 3 anos, a emissão é feita manualmente fora do ERP e que os últimos arquivos em `loteNFSE/` são de ago/2023 [CONTEXTO_PROJETO.md].

---

## 2. Catálogo de funcionalidades

Legenda do tipo: **NFS-e** = NFS-e São Paulo; **Mod.21** = nota fiscal de serviço de comunicação (SCM); **SVA** = fatura de serviço de valor adicionado; **ND** = nota de débito; **Transversal**.

### Bloco A. NFS-e São Paulo

#### FIS-001 Listagem e seleção de RPS para exportação
- **Tipo:** NFS-e
- **Origem:** `ExportarNFSE.php` (linhas 19-25 consulta; 103-133 grade; 155-222 JS). Chama `consultaNFSE()` de `FuncoesNFSE.php`. A tela anterior `NotaFiscalServico.php` (que define `$_SESSION['where_nfse']`) não foi fornecida [nf].
- **O que o usuário faz:** chega da tela de consulta de NFS-e com um filtro já aplicado (guardado em sessão), vê a grade com checkbox por RPS e "marcar todos"; o botão "Gerar Lote" fica desabilitado até haver pelo menos um item marcado; "Voltar" retorna a `NotaFiscalServico.php`. Colunas: RPS, Cliente, Valor (R$), ISS, PCC, Emissão, Vencimento.
- **Regra de negócio [obs]:**
  - `SELECT * FROM nota_fiscal_servico` + `$_SESSION['where_nfse']`; se o filtro não for vazio, acrescenta ` AND situacao = 'A'`. **Se o filtro for vazio, o `WHERE` nem é criado** e a consulta lista todos os registros (inclusive não-ativos), pois a cláusula de situação só é anexada quando já existe filtro.
  - PCC exibido = `round(valor_cofins + valor_pis + valor_csll, 2)`.
  - Nome do cliente buscado por RPS (`payment_fornecedor.Empresa`, N+1 consultas).
  - O campo oculto `empresa` do formulário usa `$nf['nr_empresa']` do **último** RPS iterado (se a lista misturar empresas, o cabeçalho do lote usará a empresa do último).
  - O botão "Voltar" usa `$_SERVER['where_nfse']` (provável erro; deveria ser `$_SESSION`), então não repassa o filtro [inf].
- **Tabelas/colunas:** lê `nota_fiscal_servico` (`id, nr_rps, nr_pessoa, valor_total, valor_iss, valor_pis, valor_cofins, valor_csll, dt_emissao, dt_vencimento, situacao, nr_empresa`) e `payment_fornecedor.Empresa` (`nr_pessoa` = `nr_payment_fornecedor`). `verificaSituacao()` não fornecida.
- **Equivalente no ERP novo:** fila/lista de "documentos a emitir" por empresa com filtros e seleção em lote, mas sem arquivo TXT: o lote seleciona os RPS e dispara o envio ao Web Service (síncrono ou assíncrono), com status por documento (pendente, enviado, autorizado, rejeitado, cancelado). Filtro de situação sempre aplicado no servidor.

#### FIS-002 Geração do lote de RPS em TXT (layout 1)
- **Tipo:** NFS-e
- **Origem:** `FuncoesNFSE.php`: `detalheRpsLote()` (l. 14-91), `headerRpsLote()` (l. 93-106), `validaHeader()`, `footerRpsLote()` (l. 119-126), `validaFooter()`. Acionadas por `ControlNotaFiscalServico.php`, `action=exportar`, `type=notafiscalservico`, `chckRps[]`, `empresa` (arquivo não fornecido [nf]). Layout completo na seção 3.1.
- **O que o usuário faz:** seleciona RPS e clica "Gerar Lote"; o modal de sucesso tem botões "Baixar o arquivo!" e "Sair" (na tela, a ação de download está vazia).
- **Regra de negócio [obs]:**
  - Registro tipo `1` (cabeçalho), tipo `6` (um por RPS), tipo `9` (rodapé).
  - Cabeçalho = 28 caracteres validados; se diferente, grava o texto `header invalido`. Rodapé = 38 caracteres; senão `footer invalido`. Ou seja, a falha vai **para dentro do arquivo**, sem exceção.
  - Cabeçalho usa `empresa.nr_inscriMun` (8 posições, zeros à esquerda) e `date("Ymd")` duas vezes (início e fim do período = hoje). Os campos de primeiro/último RPS estão comentados.
  - **Data de emissão do RPS no arquivo = data de hoje** (`date("Ymd")`), e não `nota_fiscal_servico.dt_emissao`.
  - Série do RPS fixa `00000`; tipo `RPS`.
  - Valores monetários sem separador (centavos), zeros à esquerda (15 posições). Depende de o driver devolver `decimal` com 2 casas ("1234.56" vira `123456`) [inf].
  - Código de serviço = `empresa.cod_servico` (um único por empresa, 5 posições), **não** `nota_fiscal_servico.cod_servico`.
  - **Alíquota: o código lê `$rps[0]['ali_iss']`, mas a coluna do schema é `aliq_iss`** (também `info_fiscal_empresa.aliq_iss`). Com o nome errado, o campo resulta em `0000` em todos os RPS [inf; confirmar no arquivo real em `loteNFSE/`].
  - ISS retido: `iss_retido = 'N'` => `2`; qualquer outro valor => `1`.
  - Tomador: `TipoPessoa = 'F'` => indicador `1` (CPF, 11 dígitos); qualquer outro valor => `2` (CNPJ, 14 dígitos). **O CPF ocupa 11 posições, não 14, o que desloca todo o restante da linha em 3 colunas.**
  - IE do tomador: `ISENTO` (após `trim`/`upper`) vira `0`; senão remove pontuação e completa com zeros (12). IM do tomador sempre `00000000`.
  - Razão social cortada em 75; e-mail sem quebras de linha e cortado em 75 (o campo de e-mail do cliente pode ter vários endereços; vai o texto cortado, incluindo `;`).
  - Endereço: tipo de logradouro = 3 primeiras letras (`r.`, `r,`, `r:`, `r ` => `RUA`); depois `substr(trim(Endereco), 2 ou 3, 49)` e `clear()` (remove `/ - . : espaço , < > ;`). **A remoção de espaços une as palavras e o início do corte (posição 2 para RUA) deixa uma letra a mais do tipo** (ex.: "Rua das Flores, 123" => `ADASFLORES123`) [inf pela leitura do código]. Número e complemento não têm campo próprio; seguem embutidos no texto do logradouro.
  - `formtText` e `formtInt` usam `str_pad`, que **não trunca**: bairro (30), cidade (50), UF (2) e razão do logradouro não são cortados explicitamente, então valores longos deslocam as colunas seguintes [obs].
  - Tributos: envia PIS, COFINS e CSLL (`valor_*`); INSS e IR vão zerados mesmo existindo `valor_inss`, `valor_ir` no schema [obs].
  - Discriminação (final da linha, tamanho variável) = `desc_servico` sem quebras + ` VENCIMENTO: dd/mm/aaaa` (`dt_vencimento`), sem acentos (`tirarAcentos`).
  - Cada linha de detalhe é **uma consulta + um UPDATE**: `UPDATE nota_fiscal_servico SET emitiu_nf='S' WHERE id=...`. Se o UPDATE falha, o "detalhe" passa a ser a mensagem `Não foi possivel atualizar a RPS do id: ...` (texto de erro entra no retorno no lugar da linha).
  - `extraiLogradouro()` existe mas não é usada nos arquivos fornecidos.
- **Tabelas/colunas:** lê `nota_fiscal_servico` (`nr_rps, valor_servico, valor_deducao, iss_retido, valor_pis, valor_cofins, valor_csll, desc_servico, dt_vencimento, nr_pessoa, nr_empresa, id`), `empresa` (`cod_mun_ibge, cod_servico, nr_inscriMun`), `payment_fornecedor` (`EMail, TipoPessoa, Empresa, INSCRG, CGCCPF, Endereco, Bairro, Cidade, Estado, CEP`). Grava `nota_fiscal_servico.emitiu_nf`.
- **Equivalente no ERP novo:** substituído por montagem do XML de RPS/NFS-e **layout 2** (ISS + grupo IBSCBS) assinado com certificado A1 e envio ao Web Service (`nfews.prefeitura.sp.gov.br`, manual v3.4 ou mais novo, a revalidar). Código de serviço, alíquota, retenções e indicadores IBS/CBS (CST/cClassTrib) vêm de parametrização com vigência por empresa/serviço, nunca de colunas soltas. Endereço estruturado (logradouro, número, complemento, bairro, código IBGE do município) em vez de texto cortado. Data de emissão = a do documento. Falhas de validação geram erro visível e log, nunca texto dentro do payload.

#### FIS-003 Tributação T/F pelo município (IBGE 3550308) e ISS retido
- **Tipo:** NFS-e
- **Origem:** `FuncoesNFSE.php`, `detalheRpsLote()` l. 15-17 e l. 45.
- **O que o usuário faz:** nada na tela; regra aplicada na geração do lote.
- **Regra de negócio [obs]:** `$situacao_rps = empresa.cod_mun_ibge == '3550308' ? 'T' : 'F'` (T = tributado no município de São Paulo; F = fora [inf pelo CONTEXTO_PROJETO.md]). **A variável é calculada, mas o arquivo recebe a letra `T` fixa** (`$detalhe .= "T";` na l. 45). Ou seja, a regra T/F **não tem efeito** no layout; todo RPS sai como `T`. A base do cálculo é o município da **empresa prestadora**, não o do tomador.
- **Tabelas/colunas:** lê `empresa.cod_mun_ibge`.
- **Equivalente no ERP novo:** o local de incidência/tributação passa a ser decisão fiscal explícita por serviço (prestador x tomador x município da prestação), parametrizável e validada com o contador; o XML layout 2 traz os campos próprios. Não replicar o `T` fixo.

#### FIS-004 Marcação de RPS emitida
- **Tipo:** NFS-e
- **Origem:** `FuncoesNFSE.php` l. 82 (`UPDATE ... SET emitiu_nf = 'S'`).
- **Regra [obs]:** o RPS é marcado como emitido **no momento em que a linha é gerada**, antes de qualquer confirmação da prefeitura. Não há reversão se o usuário não enviar o arquivo; também não há bloqueio visível para reexportar (a filtragem por `emitiu_nf` não aparece em `ExportarNFSE.php`).
- **Tabelas:** grava `nota_fiscal_servico.emitiu_nf`.
- **Equivalente no ERP novo:** estados do documento controlados pelo retorno do Web Service (rascunho, na fila, enviado, autorizado, rejeitado, cancelado), reemissão controlada por regra, com histórico e log de cada transmissão.

#### FIS-005 Download do arquivo de lote
- **Tipo:** NFS-e
- **Origem:** `FuncoesNFSE.php`, `download($local, $arquivoNome)` (l. 174-182); botão "Baixar o arquivo!" em `ExportarNFSE.php` (l. ~224-246, função `success`).
- **Regra [obs]:** envia `Content-Disposition: attachment` com `text/plain`; `filesize($local.$arquivoNome)` e `readfile($caminho."/loteNFSE/".$arquivoNome)` usam caminhos diferentes e `$caminho` não existe dentro da função (inconsistência) [inf: a função provavelmente não funciona isoladamente]. A ação do botão no JS está vazia.
- **Tabelas:** nenhuma.
- **Equivalente no ERP novo:** download do XML/PDF autorizado a partir do repositório de documentos; sem arquivo TXT.

#### FIS-006 Importação do retorno da NFS-e
- **Tipo:** NFS-e
- **Origem:** `ImportarNFSE.php` (formulário). Processamento em `ControlNotaFiscalServico.php`, `action=importar`, `type=notafiscalservico`, arquivo `importa`, `where` (hidden) [nf].
- **O que o usuário faz:** escolhe o arquivo de retorno do portal (label mostra o nome), clica "Importar"; "Voltar" retorna a `NotaFiscalServico.php?where=100`.
- **Regra [obs]:** só o formulário (upload `multipart`, campo obrigatório). **O parser não foi fornecido.** Pelas colunas `nota_fiscal_servico.nr_nfse` e `nr_verificacao_nfse` e pelos campos `ctrt.nr_nfse`/`nfse_emitida`, o retorno provavelmente grava número da NFS-e e código de verificação por RPS [inf].
- **Tabelas:** `nota_fiscal_servico` (`nr_nfse, nr_verificacao_nfse`) [inf].
- **Equivalente no ERP novo:** eliminado como passo manual; o retorno vem na resposta/consulta do Web Service (número, código de verificação, XML da NFS-e), gravado automaticamente. Manter, se necessário, importação de XMLs antigos para histórico.

### Bloco B. SCM, nota de comunicação modelo 21

#### FIS-010 Consulta de parcelas a faturar (SCM)
- **Tipo:** Mod.21
- **Origem:** `SCM.php` (tela de filtros) e `ControlSCM.php`, `action=gerar` (l. 14-216), com `naoUnifica()`/`unifica()` (l. 766-816).
- **O que o usuário faz:** informa Contrato/SCM (número), Cliente (texto, maiúsculas), Emissão (de/até), Vencimento (de/até), Situação do Pagamento (Em Aberto/Pago), Unidade de Negócio (lista de `empresa`, padrão = empresa da sessão) e clica "Consulta Clientes". Vê, por cliente, a grade: Contrato, Parcela, SCM ("Ainda nao possui"), Cliente, Valor, Data emissão, Vencimento, e o campo "Ultima SCM emitida" (`nr_ultima_scm`) e a "Data de Emissão" (padrão hoje).
- **Regra de negócio [obs]:**
  - Parcelas elegíveis: `PD.gerado_scm = 'N'` e `P.nr_item IN (323, 175)`; "aberto" => `PD.nr_situacao_pagamento <> 2`, "pago" => `= 2`. A consulta de clientes é `SELECT DISTINCT ... payment_fornecedor.unif_notas` ordenada por vencimento e nome.
  - Para cada cliente, nova consulta das parcelas restringe por `empresa_conta.nr_empresa = empresa da sessão`. **Nessa segunda consulta o código remove `AND PD.nr_situacao_pagamento <> 2` do filtro** (comentário "CORREÇÃO BUG"), então, no modo "Em Aberto", as parcelas de um cliente que tenha ao menos uma parcela em aberto são listadas **mesmo se já pagas**.
  - Item 175 aparece com **valor x 0,2** (20%); o restante é tratado no SVA (x 0,8).
  - Número da parcela: extraído do final da descrição (remove `. - _ ,` e espaço; últimos 7 caracteres; divide por `/`; pega 2 últimos dígitos antes de `/`, sem `[`); convenção `[n/m]` no fim da descrição [inf].
  - Se `unif_notas = 'S'` agrupa parcelas do mesmo vencimento (ver FIS-012).
  - Validação JS ineficaz: compara objetos `Date` com `''` e interpreta datas `dd/mm/aaaa` como `mm/dd/aaaa`; na prática só o servidor processa [inf].
  - O valor do contador mostrado (`nr_ultima_scm`) vem de `empresa.nr_fiscal -> info_fiscal_empresa.nr_info_fiscal` da empresa da **sessão**, mas a gravação (FIS-011) usa outra consulta (`info_fiscal_empresa WHERE nr_empresa = POST empresa ORDER BY nr_info_fiscal DESC LIMIT 1`).
- **Tabelas/colunas:** lê `payment_detalhe` (`nr_payment_detalhe, nr_payment_parcela, gerado_scm, nr_scm, dt_emissao, dt_vencto, descricao, nr_valor, nr_situacao_pagamento, liberado, nr_empresa_conta`), `payment` (`nr_item, nr_ctrt, nr_payment_fornecedor`), `conta_receber`, `empresa_conta` (`nr_empresa`), `plano_conta_itens`/`plano_conta_principal` (`nr_principal, frmtd`), `payment_fornecedor` (`Empresa, Empresa_complemento, CEP, CGCCPF, Endereco, Cidade, Estado, unif_notas`), `empresa` (`nr_fiscal`, `nm_razaosocial`), `info_fiscal_empresa.nr_ultima_scm`.
- **Equivalente no ERP novo:** fila de "faturamento a emitir" por empresa, com origem no contas a receber/contrato (recorrência), filtros equivalentes e seleção individual ou em lote; o tipo de documento (NFCom, NFS-e, outro) é definido por **regra parametrizável** (produto/serviço), não por ID fixo de plano de contas.

#### FIS-011 Geração/emissão de SCM
- **Tipo:** Mod.21
- **Origem:** `ControlSCM.php`, `action=atualiza` (l. 223-431); `PDF_SCM.php` (`pdfSCM`); `LogEnvio.php` [nf].
- **O que o usuário faz:** marca parcelas (individual, vários, "marcar todos"), ajusta a data de emissão e clica "Gerar SCM". Sucesso: redireciona para `SCM.php?G=1` ("SCM geradas com sucesso!"). Erro: **desfaz tudo e não redireciona nem mostra mensagem** (o redirect de erro está comentado, l. 429).
- **Regra de negócio [obs]:**
  - Cada valor de checkbox é um id de `payment_detalhe` ou uma lista separada por `;` (parcelas unificadas); vira `IN (...)`.
  - **Numeração:** `nr_scm = nr_ultima_scm (campo oculto, lido ao listar) + 1` por documento gerado, na ordem do loop; ao final de cada documento grava `info_fiscal_empresa.nr_ultima_scm = nr_scm`. O número **não é relido do banco dentro da transação** (risco de duplicidade com dois operadores ou página antiga) e não há índice único em `scm.nr_scm` [obs no schema: só PK em `id`].
  - Valor do documento = soma dos valores das parcelas; item 175 x 0,2 por parcela ("GAMBIARRA necessária pois recebimentos lançados com esse Plano de Contas é apenas em parte SCM").
  - Empresa emissora = `payment.nr_empresa`; se 0, usa a empresa da tela (POST).
  - CFOP = `payment_fornecedor.cfop` do cliente no momento da emissão (gravado em `scm.cfop`).
  - Fixos: `modelo='21'`, `serie='U'`, `situacao='N'`; `func_emissao = $_SESSION['login']['nr_func']`.
  - `dt_emissao` = data digitada (`toDataSQL`), `dt_vencimento` = vencimento da primeira parcela do grupo; `nr_ctrt` da primeira parcela.
  - Descrição: ver FIS-013; código de autenticação: ver FIS-014.
  - Após o INSERT: `pdfSCM(LAST_INSERT_ID())` gera PDF em `../PDF_Scm/{id}_SCM.pdf` e `insertLogEnvio()` grava `log_envio` com `tipo='scm'`, `id_tipo`, `vencimento`, `path`, `nr_func_criacao`, `dt_criacao`.
  - Marca `payment_detalhe.nr_scm = N` e `gerado_scm = 'S'` e atualiza o contador.
  - Se qualquer INSERT/log falha, `nErro` acumula: o `UPDATE` de marcação não roda para esse e para os próximos grupos (porque `nErro` não é zerado) e ao final `ROLLBACK`. Os PDFs já criados em disco **não** são removidos.
  - `action=faturar-scm` existe só como `echo "entrou if"` (stub).
- **Tabelas/colunas:** lê `payment_detalhe`, `payment`, `payment_fornecedor` (`cfop, Empresa, CGCCPF`), `empresa` (`nr_cnpj`), `plano_conta_itens` (`ds_item`), `info_fiscal_empresa`. Grava `scm` (`nr_scm, cod_autenticacao, nr_empresa, nr_payment_fornecedor, nr_payment_detalhe, dt_emissao, dt_vencimento, nr_valor_total, descricao, nr_ctrt, cfop, modelo, serie, situacao, func_emissao`), `log_envio`, `payment_detalhe` (`nr_scm, gerado_scm`), `info_fiscal_empresa.nr_ultima_scm`. `scm.bc_icms` e `scm.nr_icms` existem mas **não são preenchidos** (ficam 0.00).
- **Equivalente no ERP novo:** emissão de **NFCom modelo 62** (Ajuste SINIEF 07/2022): XML assinado, autorização em tempo real pela SEFAZ, chave de acesso, protocolo, DANFE-COM, eventos (cancelamento etc.) e contingência. Numeração por série com reserva atômica no banco (sem campo oculto). Registro de cada transmissão (requisição, resposta, tentativas) e reprocessamento via fila. A conta de 20% do item 175 deve virar regra explícita de rateio por item de faturamento (a validar com o contador). Mensagens de erro sempre visíveis ao usuário.

#### FIS-012 Unificação de parcelas no mesmo documento
- **Tipo:** Mod.21 / SVA / ND
- **Origem:** `unifica()`/`naoUnifica()` em `ControlSCM.php` (l. 766-816), `ControlSVA.php` (l. 592-643), `ControlNotaDebito.php` (l. 493-542).
- **Regra [obs]:** flag por cliente em `payment_fornecedor`: `unif_notas='S'` (SCM e SVA) ou `unif_svascm='S'` (ND). Quando 'S', parcelas do mesmo cliente com a **mesma data de vencimento** (comparação de texto `dd/mm/aaaa`, sobre lista já ordenada por vencimento) viram uma única linha; os ids ficam em `nr_pd` separados por `;` e os valores são somados. Na grade, `unifica()` não leva o campo `contrato` (coluna "Contrato" fica vazia) e aplica o fator do item 175 pelo `nr_item` da primeira parcela do grupo, enquanto a geração aplica por parcela (grade e documento podem divergir quando o grupo mistura itens) [inf].
- **Tabelas:** `payment_fornecedor.unif_notas`, `unif_svascm`; `payment_detalhe`.
- **Equivalente no ERP novo:** opção por cliente/contrato "agrupar cobranças por vencimento" com itens de documento (um item por serviço), em vez de concatenar descrições.

#### FIS-013 Composição da descrição do SCM e textos fixos por cliente
- **Tipo:** Mod.21
- **Origem:** `ControlSCM.php`, `preparaDescricao()` (l. 829-875) e bloco de exceções por cliente (l. 315-335).
- **Regra [obs]:**
  - A descrição da parcela (campo livre do financeiro) é interpretada por convenção: corta em `[` (indicador de parcela), isola trecho ` PER...` (período), separa por `*`, remove do texto o nome do item de plano de contas (`plano_conta_itens.ds_item`) a partir da segunda parcela, e acrescenta " - R$ valor" quando há mais de um `*`. Uso de `strpos` sem comparar com `false` (posição 0 conta como "não achou"). Há um ramo morto (`if(0 != 0)`) que buscaria `ctrt.nm_ctrt`.
  - **Descrições totalmente substituídas, escritas no código, para os clientes `payment_fornecedor` 1581, 2126, 1862, 2551, 2378, 2548, 2351, 2673, 2311/2757 e 2758**, com período calculado a partir do mês do vencimento (ex.: "PERIODO 21/mm-1/aaaa A 20/mm/aaaa") e **valores em reais escritos no texto** (ex.: "R$ 311,06"). Os textos são fixos; se o preço mudar, o código precisa ser editado.
  - Mês anterior: janeiro => mês 12 do ano anterior.
  - Após montar: `addslashes()` e INSERT concatenado.
- **Tabelas:** `plano_conta_itens.ds_item`, `payment_detalhe.descricao`, `scm.descricao`.
- **Equivalente no ERP novo:** descrição estruturada a partir do contrato/itens do serviço (circuito, endereço, velocidade, valor, período de referência), com modelo de texto por cliente editável em cadastro (não no código).

#### FIS-014 Código de autenticação do SCM
- **Tipo:** Mod.21
- **Origem:** `ControlSCM.php`, `calculaCodAutenticacao()` (l. 898-904); exibição `mascara($cod, 'autenticacao-scm')` (`funcoes.php` [nf]).
- **Regra [obs]:** `strtoupper(md5( CGCCPF_do_cliente . nr_scm . valor . '000000' . emissao(Y-m-d) . nr_cnpj_empresa ))`. O comentário lista a ordem a) CNPJ/CPF do tomador; b) nº do documento; c) valor total; d) BC ICMS; e) ICMS; f) data de emissão; g) CNPJ do emitente. Na implementação, BC e ICMS viram o texto fixo `000000`, o valor entra "cru" (sem 2 casas fixas), a data entra com hífens, e CNPJ/CPF entram como estão no cadastro (com ou sem máscara) [inf: não segue a formatação de largura fixa usada em `documentoMestreFiscal`]. O código é gravado em `scm.cod_autenticacao` (varchar 35) e **não é recalculado** quando o valor é editado depois (FIS-016).
- **Equivalente no ERP novo:** substituído pela chave de acesso e protocolo de autorização da NFCom; não migrar o MD5 (apenas preservar o histórico).

#### FIS-015 Consulta de SCM emitidas
- **Tipo:** Mod.21
- **Origem:** `SCM.php` (botão "Consulta SCM", `id=consulta`) e `ControlSCM.php`, `action=consulta` (l. 436-495). Listagem em `CnsSCM.php` [nf].
- **O que o usuário faz:** usa os mesmos filtros; o formulário é submetido (POST) e o usuário vai para a lista de SCM.
- **Regra [obs]:**
  - Monta `$_SESSION['where_scm']` (fragmento SQL em sessão) com: `nr_scm = <campo Contrato/SCM>` (**aqui o campo filtra o número da SCM, não o contrato**, diferente da consulta de parcelas), vencimento entre datas, emissão entre datas, cliente (`Empresa LIKE` ou `RazaoSocial LIKE`, vira `nr_payment_fornecedor IN (...)`; se nada casar, gera `IN ()` e erro de SQL).
  - Filtro de empresa: se a empresa da sessão é 1 ou 4 => `nr_empresa IN (1, 4, 0)`; caso contrário `nr_empresa = <sessão>`. O seletor "Unidade de Negócio" da tela (`opt`) é lido mas **não é usado** no SCM.
- **Tabelas:** `scm`, `payment_fornecedor`.
- **Equivalente no ERP novo:** listagem de documentos fiscais com filtros por número, cliente, período, empresa, status (autorizada, cancelada, rejeitada, contingência), exportação e ações em lote; sem fragmento SQL em sessão.

#### FIS-016 Edição de SCM emitida
- **Tipo:** Mod.21
- **Origem:** `formSCM.php`; `ControlSCM.php`, `action=gravar` (l. 500-531).
- **O que o usuário faz:** abre a SCM por `?id=`, vê status (Ativo/Cancelado), número, código de autenticação, emissão, cliente; pode alterar **vencimento, valor e descrição (textarea)** e "Gravar"; botões "Imprimir SCM", "Cancelar SCM"/"Ativar SCM" (conforme status) e "Voltar". Mensagens de sucesso/erro via `?res=`.
- **Regra [obs]:** `UPDATE scm SET dt_vencimento, nr_valor_total, descricao, dt_atualizacao, func_atualizacao`; atualiza `log_envio.vencimento`; refaz o PDF (`pdfSCM`). **Não há restrição para editar SCM cancelada nem recálculo de `cod_autenticacao`**; permite alterar o valor de um documento fiscal já emitido. Na exibição, `||` vira `<br/>`; na gravação a descrição volta com `<br/>` e `||` não são restaurados [inf]. `dt_atualizacao` é `date` no schema, mas o código grava data+hora (a hora é perdida) [obs].
- **Tabelas:** `scm`, `log_envio`, `payment_fornecedor.Empresa`.
- **Equivalente no ERP novo:** documento autorizado é **imutável**; correção só por evento (carta de correção quando existir, cancelamento e substituição) conforme o modelo 62. Antes da transmissão, edição livre em rascunho.

#### FIS-017 Cancelamento e reativação de SCM
- **Tipo:** Mod.21
- **Origem:** `ControlSCM.php`, `action=cancela-scm` (l. 537-552) e `action=ativa-scm` (l. 558-574); botão em `formSCM.php` com confirmação JS.
- **Regra [obs]:** cancelar => `situacao='S'`, `dt_cancelamento`, `func_cancelamento` (sem motivo). Reativar => `situacao='N'`, **`dt_emissao = hoje`**, `dt_ativacao`, `func_ativacao`. É só uma marca local (sem evento fiscal). **Cancelar não libera a parcela** (`payment_detalhe.gerado_scm` continua 'S'), então a parcela não volta à lista de faturamento. `ativa-scm` ainda faz `echo "Entrou no if"` antes do redirect. `dt_cancelamento`/`dt_ativacao` são `date` no schema e o código grava data+hora.
- **Tabelas:** `scm` (`situacao, dt_cancelamento, func_cancelamento, dt_emissao, dt_ativacao, func_ativacao`).
- **Equivalente no ERP novo:** **evento de cancelamento** transmitido à SEFAZ (NFCom) com justificativa obrigatória, protocolo e prazo legal; sem reativação (refaturar = nova emissão). Regra explícita sobre liberar ou não a parcela para novo faturamento.

#### FIS-018 Impressão da SCM (visualização de nota)
- **Tipo:** Mod.21
- **Origem:** `ImpressaoSCM.php` (`?id=` para uma nota; sem `id` imprime todas as de `$_SESSION['where_scm']`). Botão "Imprimir SCM" em `formSCM.php` (abre em nova janela).
- **O que o usuário faz:** abre o PDF no navegador (uma página por SCM; em lote, um PDF com todas as notas da consulta).
- **Regra [obs]:**
  - Layout FPDF: cabeçalho da emissora (logo `logo_cew_mini.jpg`, razão social `UPPER(nm_razaosocial)`, CNPJ, IE, endereço e telefone **escritos no código**: "Rua CORREIA SALGADO, 147 A, IPIRANGA, São Paulo - SP, CEP 04267-010", e-mail financeiro@gruporedes.global; para `nr_empresa = 8`, "Conj. 01" e telefone (11) 3195-6395; demais (11) 5061-9288).
  - Título "Nota Fiscal de Serviço de Comunicação Via Unica"; bloco "Serviço de Comunicação" com dados do tomador (nome até 70 caracteres + "...", CNPJ/CPF mascarado, IE/RG, endereço, bairro, CEP, município, UF).
  - Bloco da nota: "Nota: nº - Serie", Modelo, CFOP (da SCM; se vazio, do cliente), Natureza "Prestação de serviço de comunicação a estabelecimento comercial", emissão, "Período Prestação: mês por extenso/ano do **vencimento**", vencimento, valor total.
  - Grade: SEQ 1, "SERVIÇO DE COMUNICAÇÃO", CFOP, BC ICMS "0,00", Aliq ICMS "0,00", valor total (sempre um único item, a soma).
  - Rodapé de impostos: "Base de cálculo do ICMS" = valor total (atualizado em 19/04 segundo comentário), "Valor do ICMS" = **`round(valor total x 0,25, 2)`**, "Valor Aproximado dos Tributos" 0,00. Contradiz a linha da grade (BC e alíquota 0,00) e o texto "optante pelo Simples Nacional" (montado em `$observaCAo`, mas **não impresso**).
  - "Reservado ao Fisco": código de autenticação mascarado + situação "ACEITA" (`situacao='N'`) ou "CANCELADA".
  - Observação = `descricao` + "* VENCIMENTO dd/mm/aaaa".
  - Em impressão individual **sobrescreve** `../PDF_Scm/{id}_SCM.pdf` (o arquivo guardado passa a ter a versão de `ImpressaoSCM`, que difere da de `PDF_SCM`: ver FIS-019).
  - `strpos(';', $scm['nr_payment_detalhe'])` tem os argumentos invertidos e o resultado (`$itens`) não é usado.
- **Tabelas:** `scm`, `empresa` + `ender` (LEFT JOIN, mas endereço do banco não é usado), `payment_fornecedor`.
- **Equivalente no ERP novo:** **DANFE-COM** gerado a partir do XML autorizado (com QR Code/chave de acesso), reimpressão a qualquer momento; dados do emitente vindos do cadastro da empresa; sem ICMS calculado por fator fixo.

#### FIS-019 PDF armazenado do SCM e registro para e-mail de cobrança
- **Tipo:** Mod.21 / SVA / ND
- **Origem:** `PDF_SCM.php` `pdfSCM($id)`; `PDF_SVA.php` `pdfSVA($id)`; `PDF_Nota_Debito.php` `pdfNotaDebito($id)`; `LogEnvio.php` [nf] (`keysLogEnvio`, `insertLogEnvio`).
- **Regra [obs]:** os PDFs são criados na emissão e recriados na edição, em `PDF_Scm/`, `PDF_Sva/`, `PDF_Nota_Debito/` (`{id}_SCM.pdf`, `{id}_SVA.pdf`, `{id}_NotaDebito.pdf`). `log_envio` guarda `tipo` ('scm', 'sva', 'nota_debito'), `id_tipo`, `vencimento`, `path`, `enviado` ('N' padrão), `nr_func_criacao`, `dt_criacao`, `nr_func_envio`, `dt_envio` e datas de lembrete/inadimplente — usado por rotinas de e-mail de cobrança fora do escopo destes arquivos.
  - `PDF_SCM.php` replica `ImpressaoSCM.php` com diferenças: usa `$scm['nr_valor_total']` onde `$scm` é lista (deveria ser `$scm[0]`), então o "Valor do ICMS" do PDF guardado sai **0,00** [inf]; `str_replace($cliente[0]['Telefone'])` com um só argumento; `$i` indefinido. O PDF guardado e o PDF impresso são, portanto, **dois documentos diferentes para o mesmo SCM**.
  - O corte de razão social do tomador é 80 (PDF_SCM) x 70 (ImpressaoSCM).
- **Tabelas:** `log_envio` (todas as colunas citadas), `scm`/`sva`/`nota_debito`.
- **Equivalente no ERP novo:** repositório de documentos (XML autorizado + PDF/DANFE-COM) com guarda pelo prazo legal; envio por e-mail como serviço separado ligado ao evento "documento autorizado", com histórico de envios.

#### FIS-020 Arquivos do Convênio ICMS 115/03 (mestre e item)
- **Tipo:** Mod.21
- **Origem:** `ControlSCM.php`: `documentoMestreFiscal()` (l. 647-694), `documentoItemFiscal()` (l. 593-639), `gravaArquivo()` (l. 725-740), `autDigital()` (l. 749-757), `compactFile()` (l. 702-715). **Nenhuma tela ou `action` fornecida chama essas funções** [obs]; o chamador (provavelmente `CnsSCM.php`) não foi fornecido [nf].
- **Regra [obs]:** monta registro de largura fixa com os campos da seção 3.2, nome do arquivo `UF + CNPJ emitente + modelo + série + AAMM da emissão + situação + "M.001"`. Problemas visíveis: (1) `documentoItemFiscal` é cópia quase idêntica de `documentoMestreFiscal` (mesmos campos, nome de arquivo termina em `M.001` também, sem campos próprios de item); (2) `autDigital()` itera `$registros` (inexistente; o parâmetro é `$registro`) e devolve sempre o MD5 de string vazia (`d41d8cd98f00b204e9800998ecf8427e`) [inf]; (3) `gravaArquivo()` usa `$path_scm` indefinido, ignora a string `$info` montada e grava o **array** `$data` com `file_put_contents` e faz `echo` do resultado; (4) `compactFile()` usa `$path_compact` indefinido; (5) **não há função para o arquivo de cadastro de destinatários**; (6) a ordem final dos campos depende da ordem das chaves do array montado pelo chamador, que não foi fornecido. Conclusão: **o código dos arquivos do Convênio 115 está incompleto/não funcional nos fontes fornecidos**.
- **Tabelas:** nenhuma lida diretamente (recebe array).
- **Equivalente no ERP novo:** não se aplica ao modelo 62 (NFCom é XML via SEFAZ). Confirmar com o contador se há obrigação residual para documentos modelo 21 do passado (ver seção 5).

#### FIS-021 Talonário / relatório de notas (SCM, SVA e ND)
- **Tipo:** Mod.21 / SVA / ND
- **Origem:** `Talonario.php` (recebe POST `tabela` e `campos`; chamado por tela não fornecida).
- **O que o usuário faz:** gera um relatório HTML com Num. NF, Valor Total, Data Emissão, Data Prestação, Cliente, CNPJ/CPF, CFOP, Status e linha de Total; título conforme a tabela: "Relatório de Notas Fiscais SVA", "...Modelo 21/22" ou "...Notas de Débito".
- **Regra [obs]:** executa `SELECT $campos FROM $tabela $where ORDER BY $campo` onde `$tabela` e `$campos` vêm **direto do POST**; o `$where` vem de `$_SESSION['where_sva' | 'where_scm' | 'where_nota_debito']`. Espera aliases `nf, valor, emissao, vencimento, cliente, cpf_cnpj, cfop, status`. "Data Prestação" exibe o alias `vencimento`. CPF/CNPJ mascarado conforme tamanho (>11 => CNPJ). Soma o total antes de formatar. A sessão de ND usada na consulta (`asEDs`) é **diferente** de `where_nota_debito` (o relatório de ND provavelmente ignora o filtro) [inf]. O teste `strpos(UPPER(where),'WHERE') ? ...` falha se "WHERE" estiver na posição 0.
- **Tabelas:** `scm`, `sva` ou `nota_debito` (via POST).
- **Equivalente no ERP novo:** relatório/livro de saídas de documentos fiscais por período, empresa e tipo, com exportação (CSV/PDF/XML em lote para o contador). Sem SQL vindo do navegador.

### Bloco C. SVA, fatura de serviço de valor adicionado

#### FIS-030 Consulta de parcelas a faturar (SVA)
- **Tipo:** SVA
- **Origem:** `SVA.php`; `ControlSVA.php`, `action=gerar` (l. 18-236).
- **O que o usuário faz:** idêntico a FIS-010 (cópia da tela): Contrato/SVA, Cliente, Emissão, Vencimento, Situação do Pagamento, Unidade de Negócio; botões "Consulta Clientes" e "Consulta SVA".
- **Regra [obs]:** `PD.gerado_svascm = 'N'` e `P.nr_item IN (324, 175, 467)`; "aberto" `<> 2` / "pago" `= 2`; mesma remoção do filtro de situação na segunda consulta; item 175 x **0,8**. Parcela extraída da descrição só se contiver `]`. Coluna "Contrato" exibe `payment.nr_ctrt` (0 se vazio). Usa `unif_notas`. Liga `display_errors` e `E_ALL` no topo do `ControlSVA.php` (erros expostos ao usuário).
- **Tabelas:** como FIS-010, com `payment_detalhe.gerado_svascm`, `nr_svascm`, `info_fiscal_empresa.nr_ultima_sva`.
- **Equivalente no ERP novo:** depende da decisão de natureza fiscal (seção 5). Se for serviço tributável, entra na fila de NFS-e SP layout 2; se for documento gerencial, entra como "fatura" sem efeito fiscal.

#### FIS-031 Geração de SVA
- **Tipo:** SVA
- **Origem:** `ControlSVA.php`, `action=atualiza` (l. 243-436); `PDF_SVA.php`.
- **Regra [obs]:** igual ao SCM em estrutura (transação, número = `nr_ultima_sva` + 1, soma de valores, item 175 x 0,8, PDF, `log_envio` `tipo='sva'`, marcação em `payment_detalhe` com `nr_svascm`/`gerado_svascm='S'`), com diferenças: **sem CFOP, série, modelo nem código de autenticação**; `cond_pgto` fixo 1; descrições de várias parcelas unidas com `||`; ao final do documento `echo` da descrição (saída antes do redirect); em caso de erro redireciona para `SVA.php?err=<cliente>` (mensagem "Erro ao gerar scm do cliente"). INSERT em `sva` (`nr_sva, nr_empresa, nr_cliente_payment, nr_payment_detalhe, cond_pgto, dt_emissao, dt_vencimento, nr_valor, observacao, nr_ctrt`); `situacao` assume o padrão 'A'.
- **Tabelas:** `sva`, `payment_detalhe`, `payment`, `log_envio`, `info_fiscal_empresa`, `plano_conta_itens`.
- **Equivalente no ERP novo:** ver FIS-030; numeração por série, atômica.

#### FIS-032 Composição da descrição do SVA e textos fixos por cliente
- **Tipo:** SVA
- **Origem:** `ControlSVA.php`, `preparaDescricao()` (l. 656-693) e exceções por cliente (l. 328-348).
- **Regra [obs]:** variante de FIS-013 com separador `||`. Textos fixos por cliente: 2126, 1581, 1862, 2551, 2378, 2548, 2351, 2673 (valores em R$ no texto). **Os blocos de 2311/2757 e 2758 testam `$arr_scm` (variável do SCM) e nunca executam no SVA** [obs: copy-paste]. Os textos usam "SERVIÇO DE VALOR ADICIONADO REFERENTE ..." ou "LOCAÇÃO DE INFRAE GERENCIAMENTO" etc.
- **Equivalente no ERP novo:** igual a FIS-013.

#### FIS-033 Consulta de SVA emitidas
- **Tipo:** SVA
- **Origem:** `ControlSVA.php`, `action=consulta` (l. 441-499); `CnsSVA.php` [nf].
- **Regra [obs]:** monta `$_SESSION['where_sva']`: `nr_sva = <campo>` (filtra número do SVA), vencimento, emissão, cliente (`nr_cliente_payment IN (...)`), e **empresa a partir do seletor `opt`** (1 ou 4 => `IN (1,4,0)`), diferente do SCM.
- **Tabelas:** `sva`, `payment_fornecedor`.
- **Equivalente no ERP novo:** igual a FIS-015.

#### FIS-034 Edição de SVA emitida
- **Tipo:** SVA
- **Origem:** `ControlSVA.php`, `action=gravar` (l. 504-533); `formSVA.php` [nf].
- **Regra [obs]:** `UPDATE sva SET dt_vencimento, nr_valor, observacao, dt_alteracao, func_alteracao`; atualiza `log_envio.vencimento` (`tipo='sva'`); refaz o PDF. **Descrição e valor entram no SQL sem escape** (quebra com aspas e permite injeção).
- **Equivalente no ERP novo:** edição só em rascunho; fatura emitida só por cancelamento/substituição.

#### FIS-035 Cancelamento e reativação de SVA
- **Tipo:** SVA
- **Origem:** `ControlSVA.php`, `cancela-sva` (l. 539-554) e `ativa-sva` (l. 560-575).
- **Regra [obs]:** cancelar => `situacao='C'`, `dt_cancelamento`, `func_cancelamento`. Reativar => `situacao='A'`, `dt_emissao = hoje`, `dt_ativacao`, `func_ativacao`. Não libera a parcela (`gerado_svascm` permanece 'S'). O PDF mostra " - CANCELADA" ao lado do número quando `situacao='C'`.
- **Equivalente no ERP novo:** igual a FIS-017, conforme o documento adotado.

#### FIS-036 PDF/impressão da fatura SVA
- **Tipo:** SVA
- **Origem:** `PDF_SVA.php`, `pdfSVA($id)`.
- **Regra [obs]:**
  - Documento "FATURA — 1ª VIA CLIENTE", Nº com 8 dígitos (zeros à esquerda), data de emissão e vencimento; "Cond. Pagamento: À prazo"; "Natureza da Operação: Serviço de Valor Adicionado"; descrição "- Serviço de Valor Adicionado"; texto fixo **"Atividade sem incidência de ISSQN Lei Complementar Federal 116/2.003"**; valor e total; bloco "Observação" com as linhas separadas por `||` (em maiúsculas, com paginação para segunda página).
  - Dados da emissora escritos no código, por `nr_empresa`: 10 (GR CIBERSEGURANÇA GERENCIMENTO E SERVIÇOS LTDA, CNPJ 48.503.670/0001-57), 8 (GR SERVIÇOS E TELECOMUNICAÇÕES LTDA, CNPJ 23.923.034/0001-53), demais (CEW - SERVIÇOS E TELECOMUNICAÇÕES EIRELI - EPP, CNPJ 09.530.264/0001-21). **O `switch` testa `$sva['nr_empresa']` onde `$sva` é lista**, então sempre cai no `default` (CEW) [inf].
  - Rótulos invertidos: o código trata o bloco do cliente como "Prestador" (comentários) e o da emissora como "Tomador"; na impressão o cliente aparece com "Razão Social, Endereço, Município, Inscr. CNPJ(MF), Cond. Pagamento, CEP, Estado, Insc. Est." (CNPJ sempre com máscara de CNPJ, mesmo para CPF).
- **Tabelas:** `sva`, `payment_fornecedor`.
- **Equivalente no ERP novo:** se mantido como fatura gerencial, modelo de PDF parametrizável por empresa (dados do cadastro, não do código); se NFS-e, DANFSe/PDF oficial.

### Bloco D. Nota de Débito

#### FIS-040 Consulta de parcelas a faturar (ND)
- **Tipo:** ND
- **Origem:** `NotaDebito.php`; `ControlNotaDebito.php`, `action=gerar` (l. 11-198).
- **O que o usuário faz:** filtros Contrato/Nota de Débito, Cliente, Emissão, Vencimento, Unidade de Negócio (**sem** filtro de situação do pagamento); botões "Consulta Clientes" e "Consulta Nota de Débito".
- **Regra [obs]:** `PD.gerado_svascm = 'N'` e `P.nr_item IN (174, 467, 430)`; o filtro de situação de pagamento está comentado (`/*PD.nr_situacao_pagamento <> 2 AND*/`), então traz abertas e pagas. Usa `unif_svascm` (e não `unif_notas`). **Reaproveita as colunas `gerado_svascm`/`nr_svascm` do SVA** e o item 467 aparece nas duas listas (SVA e ND): quem for faturado primeiro some da outra lista.
- **Tabelas:** `payment_detalhe`, `payment`, `payment_fornecedor.unif_svascm`, `info_fiscal_empresa.nr_ultima_nd`.
- **Equivalente no ERP novo:** ver seção 5 (natureza); no mínimo, flags de faturamento separados por tipo de documento.

#### FIS-041 Geração de Nota de Débito
- **Tipo:** ND
- **Origem:** `ControlNotaDebito.php`, `action=atualiza` (l. 199-378); `PDF_Nota_Debito.php`.
- **Regra [obs]:**
  - Número: `nr_ultima_nd` (campo oculto) + 1 **por grupo de vencimento** dentro da seleção; grava contador.
  - Descrição: usa o texto da parcela em maiúsculas, dividido por `*` e `-` por posição (espera `...*TEXTO*ITEM - DETALHE`): `dd[1]||dd2[0]||dd2[1] (VALOR) <valor>`; para mais parcelas, anexa pedaços equivalentes. **`$nd['valor']` (sem índice) é indefinido**, então o valor da primeira linha sai vazio após "(VALOR)" [inf]. Se `nr_ctrt <> 0`, busca `ctrt.nm_ctrt` com `WHERE nr1_ctrt = ...`, **coluna que não existe em `ctrt`** (a chave é `nr_ctrt`) [obs no schema].
  - INSERT em `nota_debito` com `nr_payment_detalhe = '$payments'`: **`$payments` nunca é definida**, logo a coluna vai vazia e se perde o vínculo documento-parcela [inf]. `nr_ctrt` gravado vem de `$nd[$yz]['nr_ctrt']` (índice de grupo aplicado ao array de parcelas) [inf: pode errar o contrato].
  - Soma de valores e agrupamento por igualdade de vencimento dentro da seleção, independente do flag de unificação.
  - Após o INSERT: PDF, `log_envio` `tipo='nota_debito'`; depois `UPDATE payment_detalhe SET nr_svascm = <último nr_nd>, gerado_svascm='S'` e contador.
  - Erro: `ROLLBACK` e redireciona para `NotaDebito.php?err=<cliente>`; sucesso `?G=1`.
  - Data de emissão digitada (`toDataSQL`).
- **Tabelas:** `nota_debito` (`nr_nd, nr_empresa, nr_cliente_payment, nr_payment_detalhe, cond_pgto, dt_emissao, dt_vencimento, nr_valor, observacao, nr_ctrt`), `payment_detalhe`, `ctrt` (leitura com coluna inexistente), `log_envio`, `info_fiscal_empresa`.
- **Equivalente no ERP novo:** ver seção 5.

#### FIS-042 Consulta de Nota de Débito emitidas
- **Tipo:** ND
- **Origem:** `ControlNotaDebito.php`, `action=consulta` (l. 379-420); `CnsNotaDebito.php` [nf].
- **Regra [obs]:** monta `$_SESSION['asEDs']` (fragmento ` WHERE ...`): `nr_nd = <campo>` (número da ND), emissão, vencimento, cliente. **Cliente: `implode(", ", $clie)` sobre linhas (arrays) gera "Array, Array"**, quebrando o filtro [inf]. A empresa (`opt`) é lida e não usada.
- **Tabelas:** `nota_debito`, `payment_fornecedor`.
- **Equivalente no ERP novo:** igual a FIS-015, para o documento adotado.

#### FIS-043 Edição de Nota de Débito
- **Tipo:** ND
- **Origem:** `ControlNotaDebito.php`, `action=gravar` (l. 422-452); `formNotaDebito.php` [nf].
- **Regra [obs]:** altera `dt_vencimento`, `nr_valor`, `observacao`, `obs`; atualiza `log_envio.vencimento`; refaz PDF. Sem escape (injeção/aspas). Não existe `dt_alteracao`/`func_alteracao` em `nota_debito` (sem auditoria).
- **Tabelas:** `nota_debito`, `log_envio`.
- **Equivalente no ERP novo:** igual a FIS-034, com trilha de auditoria.

#### FIS-044 Cancelamento e reativação de Nota de Débito
- **Tipo:** ND
- **Origem:** `ControlNotaDebito.php`, `cancela-nd` (l. 453-468) e `ativa-nd` (l. 469-484).
- **Regra [obs]:** cancelar => `situacao='C'`, `dt_cancelamento`, `func_cancelamento`; reativar => `situacao='A'`, `dt_ativacao`, `func_ativacao` (**não** muda a data de emissão, diferente de SCM/SVA). Não libera a parcela.
- **Tabelas:** `nota_debito`.
- **Equivalente no ERP novo:** igual a FIS-017/FIS-035, conforme o documento adotado.

#### FIS-045 PDF da Nota de Débito
- **Tipo:** ND
- **Origem:** `PDF_Nota_Debito.php`, `pdfNotaDebito($id)`.
- **Regra [obs]:** título "NOTA DE DÉBITO", "Nº NOTA" com 5 dígitos, emissão e vencimento, emissora (`empresa.nm_razaosocial`, CNPJ, endereço de `ender` montado com `tp_logradouro, nm_logradouro, nr_logradouro, nm_bairro, nm_cidade, nm_estado, nr_cep`), cliente (razão social, endereço, CEP, CNPJ sempre com máscara de CNPJ, IE), tabela DESCRIÇÃO/VALOR (itens separados por `||`; "(VALOR)" separa descrição e valor), TOTAL, e "Periodo:" (texto após "PERÍODO" na observação) + campo `obs`. A consulta de contratos (`ctrt` + `ctrt_valor`) é executada e **não é usada**. O valor total é impresso **sem formatação** ("R$ 1234.56"). O laço de itens percorre `$cnd - 1` elementos (ignora o último trecho) [obs]. Logo `logo_cew_120.jpg`.
- **Tabelas:** `nota_debito`, `empresa`, `ender`, `payment_fornecedor`, `ctrt`, `ctrt_valor`.
- **Equivalente no ERP novo:** igual a FIS-036 (modelo de PDF parametrizável) até a decisão de natureza fiscal.

### Bloco E. Transversal

#### FIS-050 Numeração fiscal por empresa
- **Tipo:** Transversal
- **Origem:** `ControlSCM.php`, `ControlSVA.php`, `ControlNotaDebito.php` (leituras e gravações de `info_fiscal_empresa`).
- **Regra [obs]:** um registro por empresa (`info_fiscal_empresa`, `nr_empresa`; ligação `empresa.nr_fiscal`) guarda `nr_ultima_rps`, `nr_ultima_nfse`, `nr_ultima_sva`, `nr_ultima_scm`, `nr_ultima_nd`, e parâmetros `reter_iss`, `aliq_iss/pis/cofins/inss/ir/csll`, `url_nfse`, `path_nfse`. Não há tabela de séries; SCM usa série fixa 'U'. Dois métodos de localizar a linha: por `empresa.nr_fiscal` (tela) e por `nr_empresa` + `ORDER BY nr_info_fiscal DESC LIMIT 1` (gravação). `AUTO_INCREMENT=5` indica poucas linhas (4 criadas) [obs]. Os parâmetros `aliq_*`, `reter_iss`, `url_nfse`, `path_nfse` e `nr_ultima_rps/nfse` **não são usados** nos arquivos fornecidos.
- **Equivalente no ERP novo:** tabela de séries por empresa e tipo de documento (NFS-e, NFCom), com reserva atômica do número, índice único (empresa, série, número) e parâmetros fiscais com vigência.

#### FIS-051 Seleção da empresa emissora
- **Tipo:** Transversal
- **Origem:** início de todos os arquivos (`$_SESSION['login']['nr_empresa']`), listas de "Unidade de Negócio" em `SCM.php`, `SVA.php`, `NotaDebito.php`.
- **Regra [obs]:** sem empresa na sessão, redireciona para `ConfiguracaoFinanc.php`. As empresas 1 e 4 são tratadas como um grupo nas consultas de SCM e SVA (`IN (1,4,0)`); `nr_empresa = 0` em `payment` é tratado como "empresa da tela". Há dados fixos por empresa nos PDFs (8, 10 e padrão) [obs].
- **Equivalente no ERP novo:** multiempresa com CNPJ, inscrições, certificado digital A1, séries e parâmetros por empresa; grupos de empresas configuráveis.

---

## 3. Layouts de arquivo gerados

### 3.1 RPS em TXT para a Nota Fiscal Paulistana (layout 1)

Arquivo texto, uma linha por registro, terminada por `PHP_EOL`, em `loteNFSE/` [nf para o nome e a montagem final]. Origem: `FuncoesNFSE.php`. Posições calculadas somando os tamanhos montados no código (caso tomador CNPJ). Nomes de campo marcados * são inferidos pela ordem, não pelo código.

**Registro 1, cabeçalho (28 posições, validado)**

| Pos. | Tam. | Conteúdo no código |
|---|---|---|
| 1 | 1 | `1` |
| 2-4 | 3 | `$info['versao']` (3 posições, deduzido do total 28) [inf] |
| 5-12 | 8 | `empresa.nr_inscriMun` (só dígitos, zeros à esquerda) |
| 13-20 | 8 | data atual `AAAAMMDD` (início) |
| 21-28 | 8 | data atual `AAAAMMDD` (fim) |

**Registro 6, detalhe (um por RPS)**

| Pos. | Tam. | Campo | Conteúdo no código |
|---|---|---|---|
| 1 | 1 | tipo de registro | `6` |
| 2-6 | 5 | tipo RPS | `RPS` + espaços |
| 7-11 | 5 | série* | `00000` |
| 12-23 | 12 | número RPS | `nr_rps`, zeros à esquerda |
| 24-31 | 8 | data emissão | **data de hoje** `AAAAMMDD` |
| 32 | 1 | tributação* | **`T` fixo** |
| 33-47 | 15 | valor dos serviços | `valor_servico` sem separador (centavos) |
| 48-62 | 15 | valor das deduções | `valor_deducao` |
| 63-67 | 5 | código do serviço | `empresa.cod_servico` sem pontuação |
| 68-71 | 4 | alíquota | lê `ali_iss` (inexistente; ver FIS-002) |
| 72 | 1 | ISS retido | 2 = não retido (`N`); 1 = retido |
| 73 | 1 | indicador tomador | 1 = CPF (`TipoPessoa='F'`); 2 = CNPJ |
| 74-87 | 14 | CPF/CNPJ do tomador | **11 posições (74-84) se CPF**; zeros à esquerda |
| 88-95 | 8 | IM do tomador | `00000000` |
| 96-107 | 12 | IE do tomador | `INSCRG` sem pontuação; `ISENTO` => 0 |
| 108-182 | 75 | razão social | `Empresa` cortado em 75, espaços à direita |
| 183-185 | 3 | tipo de logradouro | `RUA` ou 3 primeiras letras |
| 186-275 | 90 | logradouro | texto sem espaços/pontuação, maiúsculas (ver FIS-002) |
| 276-305 | 30 | bairro | maiúsculas, sem truncar |
| 306-355 | 50 | cidade | maiúsculas, sem truncar |
| 356-357 | 2 | UF | `Estado` |
| 358-365 | 8 | CEP | só dígitos |
| 366-440 | 75 | e-mail | sem quebras, cortado em 75 |
| 441-455 | 15 | valor PIS* | `valor_pis` |
| 456-470 | 15 | valor COFINS* | `valor_cofins` |
| 471-485 | 15 | (INSS*) | zeros |
| 486-500 | 15 | (IR*) | zeros |
| 501-515 | 15 | valor CSLL* | `valor_csll` |
| 516-530 | 15 | zeros | zeros |
| 531-535 | 5 | zeros | zeros |
| 536-545 | 10 | brancos | espaços |
| 546-569 | 24 | zeros | zeros |
| 570-586 | 17 | zeros | zeros |
| 587-596 | 10 | brancos | espaços |
| 597-611 | 15 | zeros | zeros |
| 612-786 | 175 | brancos | espaços |
| 787-... | variável | discriminação | `desc_servico` + ` VENCIMENTO: dd/mm/aaaa`, sem acentos |

Total fixo antes da discriminação: 786 caracteres (CNPJ) ou 783 (CPF).

**Registro 9, rodapé (38 posições, validado)**

| Pos. | Tam. | Conteúdo no código |
|---|---|---|
| 1 | 1 | `9` |
| 2-8 | 7 | `linhasD` (quantidade de linhas de detalhe), zeros à esquerda |
| 9-23 | 15 | `totalS` (total de serviços) formatado com `number_format(,2)` sem `.` e `,` |
| 24-38 | 15 | `totalD` (total de deduções), `clear()` + zeros à esquerda |

Funções de formatação: `formtInt` (zeros à esquerda, sem truncar), `formtText` (espaços à direita, sem truncar), `clear` (remove `/ - . : espaço , < > ;`), `extraiTipo`, `tirarAcentos` (a/A, e/E, i/I, o/O, u/U, c/C).

### 3.2 Arquivos do Convênio ICMS 115/03 (SCM)

Origem: `ControlSCM.php`. As funções existem, mas nenhuma tela fornecida as chama (FIS-020). Há funções para **mestre** e **item** (praticamente idênticas); **não há** função para o arquivo de **cadastro**.

**Nome do arquivo (mestre e item):** `UF` + `CNPJ do emitente (14)` + `modelo` + `série` + `AAMM da emissão` + `situação` + `M.001` (o comentário descreve: UF + CNPJ + Modelo + Série + Ano + Mês + Status + Tipo + Volume).

**Campos montados (`charComplete(valor, tamanho, 'N'|'X')`; supondo 'N' = numérico com zeros à esquerda e 'X' = alfanumérico com espaços à direita, a função está em `funcoes.php` [nf])**

| Campo | Tam. | Tipo | Valor no código |
|---|---|---|---|
| CPF/CNPJ do destinatário | 14 | N | sem `- _ . , espaço` |
| IE do destinatário | 14 | X | mestre: `ISENTO` se vazia; item: sem esse tratamento |
| Razão social | 35 | X | |
| UF | 2 | X | |
| Classe de consumo | 1 | | `0` (modelo 21) |
| Tipo de utilização | 1 | | `4` |
| Grupo de tensão | 2 | | `00` |
| Código do consumidor | 12 | X | código do consumidor no ERP |
| Data de emissão | 8 | | `AAAAMMDD` |
| Modelo | 2 | N | |
| Série | 3 | X | |
| Número | 9 | N | |
| Código de autenticação | 32 | X | `cod_autenticacao` |
| Valor total | 12 | N | centavos |
| BC ICMS | 12 | N | centavos |
| ICMS | 12 | N | centavos |
| Operações isentas | 12 | N | centavos |
| Outros valores | 12 | N | centavos |
| Situação | 1 | X | |
| Referência de apuração | 4 | X | `AAMM` do **vencimento** |
| Item | 9 | N | |
| Telefone da unidade consumidora | 12 | X | brancos |
| Tipo de CPF/CNPJ | 1 | | 1 = CNPJ (mais de 11 dígitos); 2 = CPF |
| Tipo de cliente | 2 | | `01` |
| Subclasse de consumo | 1 | | `0` |
| Telefone principal | 12 | X | brancos |
| CNPJ do emitente | 14 | N | |
| Fatura comercial | 20 | X | |
| Valor da fatura comercial | 12 | N | centavos |
| Leitura anterior | 8 | N | `0` |
| Leitura atual | 8 | N | `0` |
| Brancos | 93 | X | |
| Autenticação do registro | 32 (MD5) | | `autDigital()` (ver FIS-020; defeito) |

A ordem de gravação é a ordem das chaves do array, que o chamador define [nf]; os campos acima seguem a ordem em que o código os trata, não necessariamente a ordem do arquivo final.

---

## 4. Tabelas fiscais envolvidas

Confirmadas no `schema_erp_raicom.sql`.

| Tabela | Papel | Uso nos arquivos fornecidos |
|---|---|---|
| `nota_fiscal_servico` | RPS/NFS-e: valores, tributos (ISS, PIS, COFINS, INSS, IR, CSLL), `nr_rps`, `nr_nfse`, `nr_verificacao_nfse`, `emitiu_nf`, `situacao`, `nr_pessoa`, `nr_payment_detalhe`, cancelamento | leitura e `emitiu_nf`; demais colunas via telas [nf] |
| `nota_fiscal_servico_excluidas` | histórico de exclusão | não usada nos arquivos |
| `scm` | SCM/Mod. 21 emitidas (nº, autenticação, parcelas, vencimento, CFOP, modelo, série, valor, `bc_icms`, `nr_icms`, situação, auditoria) | leitura/escrita |
| `scm_excluido`, `scm_BKP_2021-10-29` | histórico/backup | não usadas |
| `sva` | faturas SVA (`nr_sva`, `observacao`, valor, vencimento, situação 'A'/'C') | leitura/escrita |
| `sva_excluidos`, `sva_BKP_2021-10-29` | histórico/backup | não usadas |
| `nota_debito` | Notas de Débito (`nr_nd`, `observacao`, `obs`, situação 'A'/'C') | leitura/escrita |
| `nota_debito_excluidos` | histórico | não usada |
| `info_fiscal_empresa` | contadores de numeração e parâmetros fiscais por empresa | leitura/escrita dos contadores |
| `info_fiscal_empresa_excluidos` | histórico | não usada |
| `empresa` | emissora: CNPJ, razão social, IE, IM (`nr_inscriMun`), `cod_mun_ibge`, `cod_servico`, `nr_fiscal`, `nr_ender` | leitura |
| `ender` | endereço da empresa | leitura (ND, SCM) |
| `empresa_conta` | liga a parcela à empresa (`nr_empresa`) | leitura (filtro) |
| `payment_fornecedor` | cliente/tomador: nome, CNPJ/CPF, IE, endereço, e-mail, `TipoPessoa`, `cfop`, `unif_notas`, `unif_svascm` | leitura |
| `payment`, `payment_detalhe`, `conta_receber` | contas a receber: origem das parcelas; flags `gerado_scm`, `nr_scm`, `gerado_svascm`, `nr_svascm` | leitura e marcação |
| `plano_conta_itens`, `plano_conta_principal` | classificação do lançamento (define SCM/SVA/ND); `ds_item` usado nas descrições | leitura |
| `ctrt`, `ctrt_valor` | contrato (`nm_ctrt`, `nr_nfse`, `nr_sva`, `nr_scm`, `nfse_emitida`, `scmsva_emitida`, `notafiscal`); `mensal_valor` | leitura (ND); colunas de flag sem uso visto |
| `log_envio` | controle de PDFs e envio de e-mail de cobrança por documento | gravação |
| `cfop_data` | tabela de CFOP (`cd_cfop`, `ds_cfop`; 6 linhas pelo AUTO_INCREMENT=7) | não referenciada |

Diferenças de tipo relevantes para a migração: `scm.cod_autenticacao varchar(35)`; `scm.situacao char(3)` ('N'/'S'), `sva.situacao char(1)` ('A'/'C'), `nota_debito.situacao char(1)` ('A'/'C'); `scm.dt_cancelamento/dt_atualizacao/dt_ativacao` são `date` (o código grava data e hora); `sva` usa `datetime`; `nota_debito` não tem campos de alteração; `payment_detalhe.nr_svascm` é compartilhado entre SVA e ND, sem indicar o tipo (ambíguo); `nr_payment_detalhe` em `scm`/`sva` é `varchar(300)` com ids separados por `;`; em `nota_debito` é `varchar(50)`; em `nota_fiscal_servico` é `int` (uma parcela). Não há índice único em `nr_scm`, `nr_sva`, `nr_nd` ou `nr_rps`.

---

## 5. Dúvidas e pontos a confirmar com o cliente e o contador

1. **Documento atual do SCM:** hoje sai NFCom (modelo 62) ou ainda modelo 21? Por onde é emitido (portal SEFAZ, outro sistema)? Se o modelo 21 não é mais usado, descartar FIS-018/019/020 do escopo de paridade e manter só leitura do histórico (pendência já registrada no CONTEXTO_PROJETO.md).
2. **Natureza fiscal do SVA:** a fatura traz o texto "Atividade sem incidência de ISSQN, LC 116/2003". Esse tratamento está correto e aceito pelo contador? O SVA deve virar NFS-e (SP layout 2), permanecer documento gerencial ou ser eliminado? Qual o enquadramento atual (item da lista de serviços)?
3. **Natureza fiscal da Nota de Débito:** é cobrança de reembolso/despesa (documento gerencial) ou fato gerador de ISS/ICMS? Quais são os itens 174, 467 e 430?
4. **Significado dos itens de plano de contas** 323 (SCM), 324 (SVA), 175 (rateio 20% SCM / 80% SVA), 467 (SVA e ND ao mesmo tempo), 174, 430. O rateio 20/80 é fixo ou contratual? Qual a base fiscal? O que ocorre quando um item 467 é faturado em SVA e deveria ser ND?
5. **CFOP:** quais CFOPs estão em uso em `payment_fornecedor.cfop` (campo livre) e no conteúdo de `cfop_data`? O CFOP do cliente pode mudar por operação (interestadual etc.)?
6. **ICMS e Simples Nacional:** o PDF do SCM mostra "Valor do ICMS = 25% do total" (ImpressaoSCM) ou 0,00 (PDF guardado) e o código tem o texto "empresa optante pelo Simples Nacional" (não impresso). Qual é o regime de cada empresa (CEW, GR Serviços e Telecomunicações, GR Cibersegurança, empresas 1 e 4)? Há ICMS destacado ou não?
7. **Código de serviço, alíquota e retenções NFS-e:** `empresa.cod_servico` é único por empresa. Há mais de um código por empresa? Qual a alíquota real de ISS (o lote levava `0000` por erro de nome de coluna)? Quando há retenção de PIS/COFINS/CSLL (PCC)? Retenção de INSS/IR (hoje sempre zero no TXT)?
8. **Tributação T/F:** confirmar o critério correto (município do prestador, do tomador ou do local da prestação) para o layout 2; a regra atual era calculada e ignorada.
9. **Origem dos RPS (`nota_fiscal_servico`):** como os registros eram criados (tela `NotaFiscalServico.php`, contrato com `notafiscal='S'`)? Solicitar `NotaFiscalServico.php`, `ControlNotaFiscalServico.php` e uma amostra de arquivo de lote e de retorno de `loteNFSE/`.
10. **Arquivos não fornecidos que precisam ser lidos:** `CnsSCM.php`, `CnsSVA.php`, `CnsNotaDebito.php`, `formSVA.php`, `formNotaDebito.php`, `LogEnvio.php`, `funcoes.php`.
11. **Convênio 115:** os arquivos mestre/item/cadastro eram realmente gerados e entregues ao fisco? Por qual rotina? Há obrigação residual de entregar arquivos dos períodos em que o modelo 21 foi emitido?
12. **Séries e numeração:** a série 'U' (única) é a única em uso? Há necessidade de manter a sequência histórica dos números de SCM/SVA/ND e RPS na migração (o novo ERP precisa continuar a partir do último número emitido)?
13. **Datas:** a data de emissão é digitada e pode ser retroativa; reativar SCM/SVA reinicia a data de emissão para hoje. Qual comportamento deve existir no novo (a NFCom usa a data/hora da autorização)?
14. **Cancelar e liberar parcela:** ao cancelar um SCM/SVA/ND, a parcela deve voltar à fila de faturamento? Hoje não volta.
15. **Descrições fixas por cliente** (ids 1581, 2126, 1862, 2551, 2378, 2548, 2351, 2673, 2311, 2757, 2758): ainda estão ativos? Os valores no texto estão atualizados? Migrar como modelo de descrição por cliente/contrato.
16. **Empresas emissoras:** quais empresas (1, 4, 8, 10 e outras do cadastro; `AUTO_INCREMENT=11` em `empresa`) estarão ativas no novo ERP e com que certificado A1, regime e inscrições? As empresas 1 e 4 formam um grupo (`IN (1,4,0)`): regra de negócio?
17. **Clientes pessoa física e IE:** `INSCRG` trata "ISENTO"; como tratar tomadores PF, estrangeiros e sem CPF/CNPJ no NFS-e e na NFCom?
18. **Dois flags de unificação** (`unif_notas` x `unif_svascm`): são o mesmo conceito? O ND deve usar `unif_notas`?
19. **Faturar parcela "Paga":** por que a tela permite gerar SCM/SVA de parcelas já pagas (opção "Pago" e a remoção do filtro na segunda consulta)? É faturamento posterior ao recebimento?
20. **Prazo de guarda e contingência:** confirmar com o contador prazos de guarda dos XMLs e a política desejada de contingência para a NFCom.
21. **Validação dos padrões:** conferir layout 2, IBS/CBS, manual do Web Service v3.4 (ou mais novo), NFCom/SVRS e regras de transição nos portais oficiais antes de implementar (o CONTEXTO_PROJETO.md já exige essa revalidação).

---

## 6. Riscos e bugs observados que NÃO devem ser replicados

### 6.1 Segurança
- **Injeção de SQL generalizada:** valores de `$_REQUEST`/`$_POST`/`$_GET` concatenados em SQL em todos os `Control*.php`, `formSCM.php` (`?id=`), `ImpressaoSCM.php` (`?id=`), `SCM.php/SVA.php/NotaDebito.php` (`?err=`), `ControlSCM` (`cliente`, `ctrtScm`, `empresa`, `chckSCM[]`, `nr_valor`, `id`), `ControlSVA` (`descricao`, `nr_valor` sem escape), `ControlNotaDebito` (`descricao`, `obs`).
- **`Talonario.php` executa `SELECT $campos FROM $tabela`** com nome de tabela e de colunas vindos do POST: permite ler qualquer tabela do banco (incluindo credenciais) por qualquer usuário autenticado (e sem checagem de permissão vista).
- **`WHERE` guardado em sessão** (`where_nfse`, `where_scm`, `where_sva`, `asEDs`) e reaproveitado em outras páginas; `decryptIt` foi comentado (`ExportarNFSE.php`, l. 18).
- Falta de autorização por perfil nas ações; `display_errors` ligado em `ControlSVA.php`; saída de HTML sem escape (XSS) com nomes de cliente e parâmetros GET.
- Ações destrutivas por GET/POST sem token CSRF.
- Scripts jQuery de `code.jquery.com` via HTTP (sem TLS) em várias telas.

### 6.2 Cálculo e regra fiscal
- RPS: **alíquota lida de coluna inexistente (`ali_iss`)**, **tributação `T` fixa** (a regra T/F é calculada e descartada), **data de emissão = hoje**, cabeçalho com mesmo dia nos dois períodos.
- CPF ocupa 11 posições no registro 6 (deslocamento); campos sem truncamento (bairro, cidade, UF); logradouro deformado (início do corte errado e remoção de espaços); número e complemento sem campo próprio; mensagens de erro gravadas dentro do arquivo (`header invalido`, `footer invalido`, texto de erro no lugar da linha).
- `emitiu_nf='S'` marcado na geração do arquivo, não na confirmação; filtro `situacao='A'` não aplicado quando o `WHERE` da sessão está vazio.
- Fator fixo 0,2/0,8 do item 175 sem fonte legal explícita; aplicado diferente na grade (`unifica`) e na geração.
- Valor de ICMS = 25% do total no PDF impresso, 0,00 no PDF guardado, e `bc_icms`/`nr_icms` nunca gravados: **o documento tem valores fiscais incoerentes entre telas**.
- Código de autenticação MD5 próprio, sem relação com chave de acesso/protocolo; não é recalculado ao editar valor.
- Edição e reativação de documento emitido (muda valor, vencimento e, na reativação, a data de emissão); cancelamento sem motivo, sem evento e sem liberar a parcela.
- Documentos `ND` e `SVA` compartilham `gerado_svascm/nr_svascm`; item 467 nas duas listas (disputa entre documentos).
- ND: `$payments` e `$nd['valor']` indefinidos; `nr1_ctrt` inexistente; `implode` de arrays na consulta; `$nd[$yz]['nr_ctrt']`; PDF com valor sem formatar.
- SVA: blocos de 2311/2757/2758 usam `$arr_scm` (mortos); PDF escolhe a emissora por `$sva['nr_empresa']` inexistente (sempre CEW).
- `Talonario.php`: ND filtra por sessão diferente da gravada; `strpos(...) ?` com posição 0.

### 6.3 Integridade e concorrência
- Numeração via campo oculto da página (sem bloqueio; sem índice único) pode duplicar números. As tabelas fiscais são InnoDB, mas os PDFs gravados em disco não participam do `ROLLBACK`.
- `nErro` acumulado impede a marcação dos demais grupos e o `ROLLBACK` final não informa o usuário (SCM): falha silenciosa.
- `LAST_INSERT_ID()` obtido por `SELECT` separado do INSERT; só é correto se `executar()` usar a mesma conexão [inf; `conexao.php` não fornecido].
- Cliente sem match na consulta gera `IN ()` (erro de SQL) em SCM/SVA.
- Dois caminhos diferentes para obter o mesmo contador (`nr_fiscal` x `nr_empresa` + `DESC LIMIT 1`).

### 6.4 Manutenção
- Dados da emissora, endereços, telefones, e-mail e CNPJs escritos nos PDFs; ids de clientes e valores em reais escritos no código; ids de plano de contas fixos (323/324/175/467/174/430).
- Duplicação de código (SCM/SVA/ND; telas e PDFs quase idênticos), funções com o mesmo nome (`naoUnifica`, `unifica`, `preparaDescricao`) em arquivos diferentes (conflito se incluídos juntos), código morto (`0 != 0`, `extraiLogradouro`, `faturar-scm`, `compactFile`, `$res_contratos`).
- Validação de datas em JS ineficaz e dependente do idioma do navegador.
- Mistura de `utf8_decode` e UTF-8 puro, meta `iso-8859-1` e banco latin1: risco de corrupção de acentos na migração de textos (`descricao`, `observacao`).
- Arquivos do Convênio 115 incompletos (hash de registro sempre igual; gravação de array; caminhos indefinidos).
