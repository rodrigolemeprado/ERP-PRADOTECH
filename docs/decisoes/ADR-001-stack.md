# ADR-001 — Stack do ERP-PRADOTECH

- Data: 10/11/2026
- Status: aceita (escolhida por Rodrigo)

## Decisão
- Back-end: PHP 8.4 + Laravel (versão LTS/estável vigente no início do desenvolvimento).
- Front-end: Vue 3 via Inertia.js, com Tailwind CSS como base do design system.
- Banco: Percona Server 8.4 LTS, utf8mb4.
- Fiscal: bibliotecas NFePHP (sped-common para assinatura/certificado A1, e os pacotes de NFS-e e NFCom), sempre conferidas contra os manuais oficiais vigentes.
- Deploy: containers Docker.

## Motivo
- O módulo fiscal é a parte mais crítica, e o ecossistema aberto mais maduro para documentos fiscais brasileiros (XML assinado, NFCom, NFS-e) está em PHP.
- Laravel cobre autenticação, filas (fila de envio fiscal com reprocessamento), agendamento (crons do legado), permissões e migrations sem montar infraestrutura própria.
- Inertia + Vue dá uma SPA responsiva sem manter uma API separada só para o front; Tailwind facilita um design system único e minimalista.

## Alternativas consideradas
- Node/TypeScript (NestJS + React): uma linguagem só, mas mais código fiscal próprio.
- .NET 8 + React: robusto, mas dependeria de libs fiscais comerciais.
