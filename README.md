# Cadastro de Profissionais de Saúde

Documentação funcional da feature de cadastro de profissionais no sistema de agendamento próprio. Define campos, regras de negócio, planos de trabalho, exceções, regras de conflito e requisitos de auditoria.

---

## Sumário

- [1. Cadastro do Profissional](#1-cadastro-do-profissional)
- [2. Permissões e Acesso do Profissional](#2-permissões-e-acesso-do-profissional)
- [3. Plano de Trabalho (Base)](#3-plano-de-trabalho-base)
- [4. Exceções ao Plano de Trabalho](#4-exceções-ao-plano-de-trabalho)
- [5. Detecção de Conflitos](#5-detecção-de-conflitos)
- [6. Edição e Exclusão de Exceções](#6-edição-e-exclusão-de-exceções)
- [7. Logging e Auditoria](#7-logging-e-auditoria)
- [8. Sugestão de Modelo de Dados](#8-sugestão-de-modelo-de-dados)

---

## 1. Cadastro do Profissional

### 1.1 Campos

| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| Nome | string | **Sim** | Único campo de identificação obrigatório no cadastro inicial. |
| Foto | string (URL) | Não | **Não há upload.** Recebe a URL da foto já hospedada (mesma imagem usada no aplicativo, obtida via admin para garantir match exato com a UX de agendamento). Exibir preview com recorte na interface. |
| E-mail | string | Não | Validação de formato quando preenchido. |
| Telefone | string | Não | Formato BR. |
| Liberado para agendamento | boolean | **Sim** | Controla se o profissional aparece na grade de agendamento. Default sugerido: `false`. |
| Login | string | Não | Necessário apenas se o profissional for acessar o sistema. |
| Senha | string (hash) | Não | Armazenada com hash. Política de senha a definir. |

> **Regra:** no cadastro inicial, apenas **Nome** e **Liberado para agendamento** são obrigatórios. Todos os demais campos — inclusive plano de trabalho — podem ficar em branco e serem preenchidos posteriormente.

### 1.2 Preview da Foto

- Renderizar preview imediatamente após inserção da URL.
- Disponibilizar ferramenta de recorte (crop) para enquadrar a área que aparecerá no app de agendamento.
- Salvar tanto a URL original quanto os parâmetros de recorte (ou a URL recortada, conforme implementação).

---

## 2. Permissões e Acesso do Profissional

Quando o profissional possui Login/Senha, ele acessa o sistema com escopo restrito:

- **Visualizar:** apenas sua própria escala.
- **Criar:** novos agendamentos na sua agenda.
- **Editar:** alterar data/horário de agendamentos da sua agenda.
- **Excluir:** cancelar agendamentos da sua agenda.

> Profissional **não** tem acesso à escala de outros profissionais nem a dados administrativos.

Toda ação realizada pelo profissional (e por qualquer outro usuário) deve ser registrada conforme [Seção 7 - Logging e Auditoria](#7-logging-e-auditoria).

---

## 3. Plano de Trabalho (Base)

O plano de trabalho base define a disponibilidade recorrente e padrão do profissional. É a **fonte de verdade** da agenda — exceções são sobreposições aplicadas a datas específicas, não substituem o plano base globalmente.

### 3.1 Seleção de Dias da Semana

Checkboxes habilitando/desabilitando atendimento em cada dia:

- Domingo
- Segunda
- Terça
- Quarta
- Quinta
- Sexta
- Sábado

### 3.2 Faixas de Horário por Dia

Para cada dia habilitado, o usuário cadastra uma ou mais faixas de horário independentes.

**Requisito crítico:** suportar múltiplas faixas no mesmo dia. Muitos profissionais atendem em janelas distintas (ex.: 08h–12h e 18h–22h).

- Botão na interface: **"Adicionar outra faixa de horário"** (ou equivalente).
- Cada faixa = `hora_inicio` + `hora_fim`.
- Validar que `hora_fim > hora_inicio`.
- Validar que faixas do mesmo dia não se sobreponham.

### 3.3 Aplicação em Massa para Dias Úteis

Atalho para configurar segunda a sexta de uma só vez:

- Usuário define uma ou mais faixas de horário.
- Marca um checkbox **"Aplicar a todos os dias úteis"**.
- O sistema replica essas faixas para Segunda, Terça, Quarta, Quinta e Sexta.
- Também suporta múltiplas faixas independentes (mesma lógica do item 3.2).
- Após aplicar, o usuário ainda pode editar faixas de dias específicos individualmente.

---

## 4. Exceções ao Plano de Trabalho

Exceções permitem ajustes pontuais ou recorrentes-limitados (horas extras, folgas, substituições temporárias) **sem alterar o plano base**.

### 4.1 Tipos de Exceção

O usuário seleciona **um tipo por exceção**:

| Tipo | Comportamento |
|---|---|
| **Adicionar** | Soma faixas de horário ao plano base do dia. Exemplo: terça normal 08h–12h + adicionar 14h–16h → profissional disponível nas duas faixas. |
| **Substituir** | Ignora o plano base nesse dia e usa apenas as faixas da exceção. Exemplo: terça seria 08h–12h, mas hoje será 14h–18h. |
| **Bloquear** | Marca indisponibilidade em horário que normalmente trabalharia (folga, férias, congresso, atestado). |

### 4.2 Modos de Seleção de Datas

O usuário escolhe **um dos três modos**:

#### 4.2.1 Data Única
Seleciona uma data no calendário. Caso simples de ajuste pontual.

#### 4.2.2 Múltiplas Datas Avulsas
Calendário com seleção múltipla de datas não consecutivas (ex.: 03/06, 07/06, 15/06, 22/06).
Resolve padrões irregulares que não se encaixam em data única nem em período.

#### 4.2.3 Período com Filtro de Dias da Semana
Define `data_inicial`, `data_final` e quais dias da semana dentro do período se aplicam.
Exemplo: todas as terças e quintas entre 01/06 e 30/06.
Cobre o caso de recorrência limitada.

### 4.3 Configuração de Horários

Após selecionar as datas, o usuário define as faixas de horário da exceção.

#### Toggle "Usar os mesmos horários para todos os dias"

- **Ligado por padrão.**
- **Quando ligado:** exibe um único bloco de faixas de horário aplicado a todas as datas selecionadas. Suporta múltiplas faixas independentes no mesmo bloco (ex.: 08h–12h e 18h–22h), com botão **"Adicionar nova faixa de horário"**.
- **Quando desligado:** exibe um bloco de faixas **por dia da semana** selecionado. Cada dia tem seu próprio conjunto independente de faixas, também com suporte a múltiplas faixas.

**Exemplo com toggle desligado** (período 01/06 a 30/06, terças/quintas/sábados):

- Terça: 18h–22h
- Quinta: 18h–22h
- Sábado: 08h–12h e 14h–18h

### 4.4 Preview Antes de Salvar

**Crítico** para evitar erro humano nos modos de múltiplas datas e período.

Antes de confirmar, o sistema:

1. Expande a configuração e exibe a lista completa de datas afetadas, com os respectivos horários.
   - Exemplo: *"Será criada exceção em: 03/06 (18h–22h), 05/06 (18h–22h), 07/06 (08h–12h)..."*
2. Permite **desmarcar datas específicas** antes de confirmar (ex.: feriado dentro do range em que o profissional não deseja atender).
3. Só persiste a exceção após confirmação explícita.

---

## 5. Detecção de Conflitos

O sistema deve detectar e exibir avisos nas seguintes situações, **antes** de salvar:

### 5.1 Conflito com o Plano Base
Ao adicionar uma faixa (tipo **Adicionar**) que sobrepõe horário já existente no plano base do dia → exibir aviso. Atendimento sobreposto não deve existir.

### 5.2 Conflito com Outra Exceção
Se já existe exceção cadastrada em qualquer das datas selecionadas:
- Exibir aviso identificando a exceção existente (descrição, tipo, horários).
- Oferecer ao usuário três caminhos: **substituir**, **mesclar** ou **cancelar** a nova.

### 5.3 Conflito com Agendamentos Existentes
Se houver agendamentos confirmados no horário que será removido por uma exceção do tipo **Bloquear** ou **Substituir**:
- Listar todos os agendamentos afetados (data, hora, paciente, especialidade) antes de confirmar.
- Bloquear a confirmação até decisão explícita do usuário.

---

## 6. Edição e Exclusão de Exceções

- Cada exceção é uma **entidade única e editável**, mesmo que afete múltiplas datas.
- Operações suportadas:
  - **Excluir** a exceção inteira.
  - **Desmarcar datas específicas** dentro da exceção (aplicável aos modos "múltiplas datas" e "período com filtro").
  - **Editar** tipo, datas, horários e demais parâmetros.
- Todas as ações (criação, edição, exclusão, desmarcação de datas) devem ser logadas conforme Seção 7.

---

## 7. Logging e Auditoria

> Requisito não-funcional crítico. Já tivemos casos de conflito entre paciente, profissional e suporte em que o log foi a única evidência para identificar o ponto de falha (ex.: paciente cancelou a própria consulta e alegou no suporte que não o fez).

### 7.1 O Que Logar

**Toda ação** de **todo usuário** — não apenas a última. Isso inclui:

- Login e logout.
- Criação, edição e exclusão de cadastros de profissional.
- Alterações no plano de trabalho.
- Criação, edição e exclusão de exceções (incluindo desmarcação de datas individuais).
- Criação, edição, cancelamento e remarcação de agendamentos — independente de quem executa (paciente, profissional, suporte, admin).

### 7.2 Campos Mínimos do Log

- `usuario_id` e papel (paciente, profissional, suporte, admin).
- `timestamp` (data e hora com timezone).
- `acao` (enum: criar, editar, excluir, cancelar, login, etc.).
- `entidade` afetada (profissional, agendamento, exceção, etc.) e seu identificador.
- `valores_antes` e `valores_depois` (snapshot, para auditoria de alterações).
- `origem` (web, app, API, painel interno) — útil para investigação de incidentes.

### 7.3 Padrão Existente
Manter consistência com o formato de log já em uso na plataforma MediQuo. Esta seção descreve apenas o conteúdo mínimo esperado; o schema de persistência segue o padrão da plataforma.

---

## 8. Sugestão de Modelo de Dados

Esboço inicial, sujeito a revisão pelo time de tecnologia.

```
Profissional
├── id
├── nome (obrigatório)
├── foto_url
├── foto_crop_params (json)
├── email
├── telefone
├── liberado_para_agendamento (boolean, obrigatório)
├── login
├── senha_hash
├── criado_em
└── atualizado_em

PlanoTrabalho
├── id
├── profissional_id (FK)
├── dia_semana (0=domingo ... 6=sábado)
├── ativo (boolean)
└── [relação 1:N com FaixaHorario]

FaixaHorario
├── id
├── plano_trabalho_id (FK) — quando pertence ao plano base
├── excecao_id (FK, nullable) — quando pertence a uma exceção
├── dia_semana (nullable, usado em exceções com toggle desligado)
├── hora_inicio
└── hora_fim

Excecao
├── id
├── profissional_id (FK)
├── tipo (enum: adicionar, substituir, bloquear)
├── modo_datas (enum: data_unica, multiplas_datas, periodo_com_filtro)
├── usar_mesmos_horarios (boolean)
├── descricao (opcional, ex.: "férias", "congresso")
├── criado_em
└── atualizado_em

ExcecaoData
├── id
├── excecao_id (FK)
├── data
└── ativa (boolean — permite desmarcar datas sem excluir o registro histórico)

LogAcao
├── id
├── usuario_id
├── papel
├── timestamp
├── acao
├── entidade
├── entidade_id
├── valores_antes (json)
├── valores_depois (json)
└── origem
```

---

## Status

📝 Documentação funcional — pronta para revisão técnica e definição de arquitetura.
