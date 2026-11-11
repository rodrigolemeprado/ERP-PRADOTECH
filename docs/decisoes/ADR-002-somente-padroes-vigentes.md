# ADR-002: Emissão somente nos padrões fiscais vigentes

- Data: 10/11/2026
- Status: aceita (decidida por Rodrigo)

## Decisão
O ERP-PRADOTECH é um produto próprio e só emite documentos fiscais no padrão mais novo em vigor. Hoje isso significa:
- NFS-e São Paulo por Web Service no layout mais novo (ISS + IBS/CBS); NFS-e Padrão Nacional onde for aplicável;
- NFCom modelo 62 para serviços de comunicação.

Não existirá nenhum caminho de emissão nos modelos antigos: RPS em TXT para upload manual, importação de retorno em CSV, NF modelo 21, arquivos do Convênio ICMS 115/03 e cancelamento só local.

## Consequências
- O legado é referência de regras de negócio (o que faturar, para quem, com quais valores), não de formato.
- Quando o fisco publicar layout ou versão nova, o ERP adota a nova e desativa a anterior assim que a transição permitir.
- Documentos antigos emitidos pelo legado não são reemitidos; no máximo entram como histórico somente leitura.
- Obrigações de períodos antigos (por exemplo, Convênio 115) ficam fora do ERP novo.
