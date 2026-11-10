# ERP-PRADOTECH

ERP novo que substitui o ERP legado do Grupo Redes (marca antiga RAICOM), com paridade funcional total, interface elegante, minimalista e totalmente responsiva, mais opções de configuração e customização, e emissão fiscal integrada aos padrões vigentes (NFS-e São Paulo via Web Service e NFCom modelo 62).

## Stack
Laravel (PHP 8.4) + Vue 3 via Inertia + Tailwind CSS, Percona Server 8.4 LTS, deploy em Docker. Motivo em [docs/decisoes/ADR-001-stack.md](docs/decisoes/ADR-001-stack.md).

## Documentação
- [docs/CONTEXTO.md](docs/CONTEXTO.md): meta, princípios, requisitos e fases.
- [docs/HISTORICO.md](docs/HISTORICO.md): o que já foi feito até aqui.
- [docs/catalogo/](docs/catalogo/): catálogo de funcionalidades do legado, área por área (fiscal primeiro).
- [docs/decisoes/](docs/decisoes/): decisões de arquitetura e escopo, com o motivo.
- [docs/legado/](docs/legado/): análise técnica do sistema legado.

## Fora do repositório
O código do legado, os dumps de banco, certificados digitais e qualquer `.env` ficam fora do Git: contêm dados do cliente e credenciais.

## Status
Fase 1: levantamento funcional do legado. Ainda não há código da aplicação.
