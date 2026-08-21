# Arquitetura — MDFWizard (CutWizard)

> Versão: MVP acadêmico (PAC 7/8 — 2026)
> Status: Escopo travado para início de implementação

## 1. Visão Geral

O CutWizard é um Web App **open-source**, **self-hosted**, para geração paramétrica de listas de corte de marcenaria. O sistema resolve o problema do cálculo manual de descontos estruturais (espessura de MDF, fitas de borda, folgas de gaveta) enfrentado por marceneiros autônomos, transferindo essa carga cognitiva para o backend.

O MVP adota as seguintes restrições de escopo, definidas para viabilizar a entrega dentro do cronograma acadêmico:

- **Plataforma-alvo:** Desktop Web apenas (sem suporte responsivo/mobile no MVP).
- **Padrão de montagem:** único e fixo (ex.: laterais sempre cobrindo a base). Customização do usuário limitada a espessura de MDF (ex.: 15mm/18mm) e espessura da fita de borda.
- **Módulos base:** conjunto fechado de 3 módulos pré-definidos. Sem criação dinâmica de módulos pelo usuário nesta versão.
- **Hospedagem:** self-hosted via Docker, executado localmente na máquina do marceneiro (on-premises). Zero custo de infraestrutura em nuvem.
- **Multiusuário/autenticação:** sistema single-tenant local; autenticação ausente ou mínima, já que a aplicação roda isolada na máquina do próprio usuário.

Essas decisões priorizam a corretude e testabilidade do motor de cálculo — o núcleo de valor do projeto — em detrimento de funcionalidades de plataforma (multiusuário, mobile, customização de módulos) que podem ser endereçadas em trabalhos futuros.

## 2. Componentes

### 2.1 Frontend (React)
Responsável exclusivamente pela camada de apresentação e entrada de dados:
- Seleção do módulo base (dentre os 3 disponíveis).
- Formulário de medidas externas finais e parâmetros de customização (espessura de MDF, espessura de fita de borda).
- Exibição do resultado e disparo do download do artefato (PDF/CSV) gerado pelo backend.
- **Não contém lógica de cálculo estrutural nem de formatação do documento final** — atua como cliente puro da API.

### 2.2 Backend (Node.js)
Concentra toda a "carga cognitiva" do sistema, dividido em responsabilidades internas claras:
- **Camada de domínio/cálculo:** implementa as regras de dedução estrutural (recuos, folgas, espessuras) como código estruturado, isolado de controllers e rotas, e testável via TDD.
- **Camada de geração de documentos:** produz PDF e CSV a partir do resultado do cálculo, garantindo que o que é calculado e o que é exportado nunca divirjam.
- **Camada de API:** expõe endpoints RESTful consumidos pelo frontend.
- **Camada de persistência:** grava configurações de usuário e histórico de listas geradas no PostgreSQL.

Manter as regras matemáticas como código (e não como dados em banco) foi uma decisão deliberada: como o padrão de montagem é único e fixo no MVP, não há ganho de flexibilidade que justifique a complexidade extra de um motor de regras orientado a dados — o código estruturado é mais simples de testar e versionar via Git/CI.

### 2.3 Banco de Dados (PostgreSQL)
Armazena apenas o que precisa de persistência entre sessões:
- Configurações do usuário (espessura de MDF/fita padrão, preferências).
- Histórico de listas de corte geradas.

Os 3 módulos base do MVP **não** são modelados como dados dinâmicos no banco — são parte da lógica fixa do backend, coerente com a decisão de escopo fechado.

### 2.4 Docker
Papel duplo no projeto:
- **Ambiente de desenvolvimento/CI:** garante paridade entre ambientes durante o desenvolvimento (PAC 8).
- **Estratégia de distribuição do produto final:** o Docker Compose empacota Frontend + Backend + PostgreSQL para que o marceneiro suba o sistema localmente com um único comando, sem depender de nuvem ou de conhecimento técnico de infraestrutura.

## 3. Fluxo de Comunicação

```
[Usuário] 
   │
   │ 1. Seleciona módulo base + informa medidas/parâmetros
   ▼
[Frontend - React]
   │
   │ 2. Requisição HTTP (REST) com payload de entrada
   ▼
[Backend - Node.js: Camada de API]
   │
   │ 3. Delega para a Camada de Domínio/Cálculo
   ▼
[Camada de Domínio/Cálculo]
   │
   │ 4. Aplica regras de dedução (MDF, fita de borda, folgas)
   │ 5. Retorna estrutura de peças calculadas
   ▼
[Camada de Geração de Documentos]
   │
   │ 6. Formata PDF e/ou CSV a partir do resultado calculado
   ▼
[Camada de Persistência] ──► [PostgreSQL]
   │ 7. Grava histórico da lista gerada
   ▼
[Backend - Camada de API]
   │
   │ 8. Retorna documento gerado (ou link/stream de download)
   ▼
[Frontend - React]
   │
   │ 9. Disponibiliza o download ao usuário
   ▼
[Usuário]
```

Ponto-chave do fluxo: o cálculo (passo 4-5) e a geração do documento (passo 6) ocorrem **inteiramente no backend**, na mesma transação lógica — o frontend nunca recalcula nem reformata nada, eliminando o risco de divergência entre o que é mostrado na tela e o que é impresso na lista de corte.

## 4. Decisões Tomadas

| # | Decisão | Justificativa |
|---|---|---|
| 1 | MVP restrito a Desktop Web | Responsividade mobile não é crítica para viabilidade do prazo acadêmico |
| 2 | Único padrão de montagem fixo | Reduz drasticamente a complexidade do motor de cálculo no MVP; customização por usuário fica restrita a espessuras |
| 3 | 3 módulos base fixos, sem criação dinâmica | Mantém o motor de regras como código testável, sem a complexidade de um sistema de regras configurável |
| 4 | Hospedagem self-hosted via Docker (on-premises) | Elimina custo de infraestrutura em nuvem e reforça a proposta de acessibilidade financeira do projeto |
| 5 | Geração de PDF/CSV 100% no backend | Garante consistência entre cálculo e artefato final; evita duplicação de lógica de formatação no cliente |
| 6 | Regras matemáticas implementadas em código no backend (não em banco) | Coerente com padrão fixo de montagem; prioriza testabilidade (TDD, cobertura >80%) sobre flexibilidade não requerida no MVP |
| 7 | Sistema single-tenant, autenticação mínima/ausente | Aplicação roda localmente na máquina do próprio usuário; multiusuário e autenticação robusta ficam como trabalho futuro |

## 5. Fora de Escopo (Trabalhos Futuros)

- Suporte a múltiplos padrões de montagem.
- Criação dinâmica de módulos pelo usuário.
- Interface responsiva/mobile (PWA).
- Hospedagem em nuvem multiusuário.
- Integração com softwares de otimização de plano de corte (nesting).

