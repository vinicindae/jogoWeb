# Projeto técnico — Sudoku Pausa

**Versão:** 0.1  
**Status:** proposta para implementação do MVP  
**Direção visual:** Calma  
**Referência de interface:** [mockup atual](./index.html)

## 1. Objetivo

Este documento descreve uma arquitetura técnica para transformar o mockup em um jogo web de Sudoku clássico 9×9, casual e relaxante. O produto é direcionado prioritariamente a desktop, sem impedir o uso responsivo em telas menores.

A proposta preserva a linguagem visual **Calma**: superfície marfim, verde-sálvia, tipografia editorial em títulos, grade central e painéis discretos. A experiência deve enfatizar conforto e clareza, e não velocidade, punição ou competição.

O mockup é uma referência de direção visual e interação. Ele não implementa persistência, geração de partidas, adaptação real de dificuldade, acessibilidade completa, estatísticas reais nem telemetria.

## 2. Escopo do MVP

### Incluído

- Sudoku clássico 9×9, com solução única por puzzle.
- Interface em português do Brasil e inglês.
- Quatro níveis iniciais: fácil, médio, difícil e especialista.
- Seleção explícita do nível pelo jogador.
- Adaptação do nível selecionado para a **próxima** partida; nunca alterar a dificuldade de uma partida em andamento.
- Validação imediata de conflitos e contador de erros, sem limite punitivo.
- Seleção por mouse, navegação por teclado e operação compatível com leitor de tela.
- Desfazer, apagar, anotações e dica opcional.
- Desafio diário, sequência de dias e conquistas.
- Tema claro/escuro, com preferência do sistema como padrão inicial e escolha manual persistida.
- Salvamento local, sem conta e sem sincronização.
- Gratuito e sem publicidade.
- Interface preparada para eventos de produto minimizados e anônimos; envio real depende de consentimento e decisão sobre infraestrutura.

### Fora do escopo inicial

- Variantes de Sudoku, multiplayer, ranking e comparação social.
- Conta de usuário, sincronização entre dispositivos ou recuperação em nuvem.
- Compras, anúncios, notificações push e conteúdo sazonal.
- Telemetria que registre valores digitados, conteúdo dos tabuleiros ou comportamento individual identificável.
- Geração procedural irrestrita de puzzles em produção sem validação de unicidade e dificuldade.

## 3. Decisões e premissas

| Tema | Proposta técnica | Estado |
|---|---|---|
| Plataforma | Aplicação web estática, responsiva e instalável como PWA numa etapa posterior | Proposto |
| UI | React + TypeScript + Vite; componentes acessíveis e CSS com tokens | Proposto |
| Motor de jogo | Módulo TypeScript puro, sem dependência de framework | Proposto |
| Persistência | IndexedDB para estado da partida e progresso; `localStorage` para preferências pequenas | Proposto |
| Dados de puzzle | Pacote curado e versionado no MVP, com verificador de solução | Proposto |
| Desafio diário | Puzzle determinístico por data, sujeito à decisão de fuso horário | A confirmar |
| Métricas | Adaptador desligado até consentimento, política de privacidade e fornecedor serem aprovados | A confirmar |
| Adaptação | Avaliar a partida concluída e atualizar o nível selecionado para a seguinte | Alinhado ao PRD; limites a validar |
| Erros | Mostrar conflitos e contador; não bloquear conclusão nem impor máximo | Alinhado ao PRD |

O workspace contém somente o mockup HTML no momento desta proposta, sem framework ou dependências instaladas. A primeira tarefa de implementação deverá criar o projeto e fixar versões das ferramentas.

## 4. Arquitetura de alto nível

Separar interface, regras do jogo, persistência e serviços para que decisões visuais ou de armazenamento não contaminem a lógica de Sudoku.

```text
src/
  app/
    App.tsx
    routes.ts
    providers/
  features/
    game/
      components/
      hooks/
      gameReducer.ts
      gameTypes.ts
    daily/
      dailyChallenge.ts
      streak.ts
    progress/
      achievements.ts
      adaptiveDifficulty.ts
    settings/
      SettingsPanel.tsx
  engine/
    board.ts
    candidates.ts
    conflicts.ts
    solver.ts
    difficulty.ts
    types.ts
  data/
    puzzles/
    puzzleRepository.ts
  i18n/
    catalog.ts
    formatters.ts
  persistence/
    database.ts
    migrations.ts
    repositories/
  analytics/
    consent.ts
    analyticsClient.ts
  styles/
    tokens.css
    themes.css
    global.css
tests/
```

### Responsabilidades

- **`engine/`**: regras puras do Sudoku, sem DOM, React, armazenamento ou analytics.
- **`features/game/`**: estado interativo, comandos do jogador e componentes.
- **`data/`**: seleção e validação de puzzles disponíveis.
- **`persistence/`**: leitura, gravação, versionamento e migrações do estado local.
- **`features/progress/`**: sequência diária, conquistas e ajuste de nível para partidas futuras.
- **`i18n/`**: mensagens e formatação local; nenhum texto de interface hardcoded em componentes.
- **`analytics/`**: coleta opt-in, minimizada, atrás de uma interface substituível.

## 5. Tecnologias propostas

- **React + TypeScript** para UI e tipos de domínio.
- **Vite** para desenvolvimento local, build e servidor de preview.
- **CSS Modules ou CSS organizado por feature**, usando custom properties para cores, espaçamento e tipografia. Evitar introduzir biblioteca visual pesada para a interface pequena e customizada.
- **Vitest** para testes unitários do motor e reducers.
- **Testing Library** para fluxos de interface e verificações semânticas.
- **Playwright** para smoke tests de navegação, responsividade e persistência.
- **ESLint + TypeScript strict mode** para qualidade estática.

As versões e scripts devem ser fixados em `package.json` e no lockfile ao inicializar o projeto. Esta seleção é recomendação de implementação, não uma limitação do produto.

## 6. Modelo de domínio

Representar o tabuleiro como arrays de tamanho fixo. Não usar strings concatenadas como estado operacional.

```ts
type Digit = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
type CellIndex = number; // inteiro de 0 a 80
type Difficulty = "easy" | "medium" | "hard" | "expert";
type NoteMask = number; // bits 0..8 representam candidatos 1..9

interface Puzzle {
  id: string;
  version: number;
  difficulty: Difficulty;
  givens: Array<Digit | null>;
  solution: Array<Digit>;
  source: "curated" | "generated";
}

interface GameState {
  gameId: string;
  puzzleId: string;
  puzzleVersion: number;
  mode: "free" | "daily";
  startedAt: number;
  elapsedSeconds: number;
  status: "playing" | "completed" | "abandoned";
  values: Array<Digit | null>;
  notes: Array<NoteMask>;
  selectedCell: CellIndex | null;
  mistakes: number;
  hintCount: number;
  undoStack: GameCommand[];
  redoStack: GameCommand[];
}

interface PlayerProgress {
  schemaVersion: number;
  selectedDifficulty: Difficulty;
  adaptationReason?: string;
  recentResults: GameResult[];
  completedDailyIds: string[];
  currentStreak: number;
  bestStreak: number;
  achievements: string[];
  settings: PlayerSettings;
}
```

`solution` é útil para validar entradas localmente, mas pode ser inspecionada no cliente. Como o jogo é offline e não competitivo, isso é aceitável no MVP. Não alegar que dados enviados ao navegador são secretos. Se integridade competitiva vier a ser requisito, validação e classificação precisarão de serviço confiável no servidor.

## 7. Motor de Sudoku

### Invariantes

1. O puzzle tem exatamente 81 células e cada dígito é `1..9` ou `null`.
2. Uma casa inicial (`given`) não pode ser editada nem apagada.
3. Cada solução publicada satisfaz linhas, colunas e regiões 3×3.
4. Cada puzzle publicado tem exatamente uma solução.
5. Anotações não contam como valores e não alteram validade.
6. Um conflito é definido pelas regras do Sudoku, independente de o valor digitado coincidir com a solução pretendida.

### Funções puras recomendadas

```ts
isValidBoard(values): boolean
getConflicts(values): Set<CellIndex>
getCandidates(values, index): Digit[]
countSolutions(givens, limit = 2): number
isComplete(values): boolean
applyCommand(state, command): GameState
```

`getConflicts` compara a casa preenchida com as outras da mesma linha, coluna e região. Para indicar erros imediatos, não comparar apenas com a solução secreta: um valor diferente da solução pode ser válido temporariamente em puzzles com alternativas locais. O puzzle de solução única permite que a conclusão final seja verificada contra a solução.

### Banco de puzzles e geração

Para o MVP, usar puzzles curados, versionados em JSON e testados em CI. Metadados de dificuldade devem ser associados ao puzzle, não inferidos somente pelo número de pistas. Aplicar `countSolutions` no build ou em teste para assegurar uma única solução.

Gerador procedural é uma evolução separada. Se aprovado, deverá:

1. Criar uma grade completa válida com backtracking e escolha aleatória de candidatos.
2. Remover pistas em ordem determinística/aleatória.
3. Executar contagem de soluções após cada remoção, interrompendo ao haver mais de uma.
4. Medir dificuldade por técnicas lógicas ou métricas acordadas, e não apenas pelo número de casas vazias.
5. Ter limite de tempo/iterações e fallback explícito para puzzle pré-gerado.

Não gerar puzzle no render da tela. Selecionar previamente ou executar em worker caso o custo de CPU seja relevante.

## 8. Estado de partida e comandos

Usar reducer ou máquina de estados explícita. A UI emite comandos como `SELECT_CELL`, `ENTER_DIGIT`, `TOGGLE_NOTE`, `ERASE_CELL`, `UNDO`, `REDO`, `USE_HINT`, `PAUSE`, `RESUME` e `COMPLETE`.

Fluxo básico:

```text
ready → playing → completed
                 ↘ abandoned
```

- O nível exibido no seletor é o nível efetivo para a próxima partida e fica persistido.
- Ao terminar uma partida, a adaptação pode atualizar esse nível efetivo, com base no histórico. Informar a alteração de forma transparente e permitir que a pessoa escolha outro nível manualmente.
- A partida em andamento conserva o puzzle e nível originais, mesmo após fechar e reabrir a página.
- Desfazer e refazer operam sobre comandos reversíveis. Limpar histórico ao começar outra partida.
- O contador de erros aumenta quando uma ação cria ou mantém conflito, seguindo regra consistente. Evitar incrementar de novo ao selecionar uma célula com conflito existente.
- A dica do MVP deve explicar uma possibilidade ou estratégia sem inserir valor silenciosamente. Um uso de dica não deve tornar impossível terminar o jogo.
- A conclusão exige grade completa e comparação com a solução. Mostrar celebração discreta e atualizar estatísticas/diário uma única vez, de forma idempotente.

## 9. Dificuldade adaptativa

Manter separados:

- `selectedDifficulty`: nível efetivo escolhido pelo jogador ou atualizado pela adaptação para a próxima partida.
- `adaptationReason`: motivo resumido da última alteração automática, para explicar a mudança na interface.
- `currentPuzzle.difficulty`: nível fixo da partida aberta.

Não alterar dificuldade automaticamente enquanto a pessoa joga. A alteração acontece somente ao concluir uma partida e passa a valer na seguinte; a pessoa pode substituir o nível pelo seletor a qualquer momento. Para estimar desempenho, armazenar localmente resultados recentes por dificuldade: conclusão, abandono, tempo ativo, erros e uso de dicas. Excluir tempo de pausa.

Uma regra inicial simples para experimento pode usar as últimas cinco partidas concluídas no mesmo nível, movendo no máximo um nível por avaliação. Não aumentar dificuldade devido somente a uma partida rápida. Limiares de tempo e peso de erros precisam ser confirmados com produto e testados com jogadores antes de serem fixados.

## 10. Desafio diário, sequência e conquistas

- O desafio diário deve ser estável durante o dia e reutilizável após refresh.
- Chave de conclusão sugerida: `daily:<date>:<puzzleVersion>`.
- Registrar conclusão uma única vez; reabrir o desafio concluído não aumenta sequência ou conquistas novamente.
- O serviço `dailyChallenge` deve receber uma `Clock`/data como dependência para ser testável.
- A sequência deve ser derivada de dias de conclusão consecutivos, com fuso horário explicitamente definido; não calcular usando a hora atual durante render.
- Conquistas são IDs versionados e avaliados por eventos de domínio, por exemplo `daily-completed`, `streak-reached` e `games-completed`.
- Persistir a lista de IDs já conquistados para idempotência.

**Decisão de produto pendente:** definir se o dia do desafio usa UTC ou horário local, o que acontece ao perder um dia, se há congelamento de sequência e se a mesma grade é compartilhada por todas as pessoas. O modelo deve suportar a regra escolhida sem migrar os dados de progresso.

## 11. Persistência local

Usar IndexedDB para dados que podem crescer (estado de jogos, resultados, progresso) e `localStorage` somente para preferências pequenas se a simplicidade for útil. Uma única camada de repositório oculta o mecanismo do restante da aplicação.

Armazenamento inicial:

- `activeGames`: estado de jogo em andamento, indexado por `gameId`.
- `gameResults`: resultado final, nível, duração ativa, erros, dicas e modo.
- `progress`: streak, conquistas, nível selecionado e datas concluídas.
- Preferências pequenas (`locale`, tema e assistência) em `localStorage`, ou em store `settings` do IndexedDB; escolher uma única fonte de verdade na implementação.

Requisitos:

- Persistir ao concluir uma ação de domínio, com debounce apenas em atualizações de alta frequência como cronômetro.
- Executar migrações sequenciais com `schemaVersion`; não descartar dados em erro de migração.
- Apresentar falha de armazenamento ao usuário com possibilidade de continuar a sessão em memória e aviso de que ela não ficará salva.
- Não guardar dados pessoais, email, IP em analytics ou texto digitado.
- Ter ação para apagar progresso e configurações, com confirmação explícita.
- Tratar quota/armazenamento indisponível sem produzir falso estado de “salvo”.

## 12. Interface e identidade visual Calma

### Tokens visuais

Usar custom properties nomeadas por função, evitando espalhar valores hexadecimais:

```css
:root {
  --color-page: #edf0e8;
  --color-surface: #fffefa;
  --color-ink: #25332b;
  --color-muted: #778177;
  --color-accent: #4e7c64;
  --color-soft: #e7efe6;
  --color-selection: #dcebe0;
  --color-error: #fde8e3;
  --color-grid-strong: #8c9d8d;
}
```

Tema escuro deve redefinir tokens semanticamente, não duplicar regras de componente. Validar contraste de texto, foco, grade, erro e seleção nos dois temas. Seleção, conflito e estados nunca devem ser comunicados somente por cor.

### Componentes

- `AppHeader`: marca, seletor de idioma e tema.
- `GameLayout`: áreas de tabuleiro, ações e progresso; reorganização responsiva.
- `SudokuGrid`, `SudokuCell`: grade e célula acessíveis.
- `NumberPad`: entrada por ponteiro/toque e feedback de disponibilidade.
- `GameToolbar`: desfazer/refazer, apagar, modo de nota e dica.
- `DifficultyPicker`: escolha explícita de nível e informação de ajuste para a próxima partida.
- `GameStatus`: cronômetro, erros, progresso e anúncios acessíveis.
- `DailyProgress`: sequência e conquistas.
- `SettingsDialog`: preferências, privacidade e apagamento de dados.
- `CompletionDialog`: resultado e próxima ação.

O tabuleiro precisa caber sem rolagem horizontal em larguras menores. Em desktop, priorizar grade central e painéis laterais; em viewport estreito, empilhar tabuleiro e painéis. Não comprimir casas a ponto de dificultar seleção.

### Temas

Estado de tema: `system | light | dark`. `system` acompanha `prefers-color-scheme`; seleção manual tem precedência e é persistida. Mudanças devem respeitar `prefers-reduced-motion`.

## 13. Acessibilidade

Objetivo de qualidade: WCAG 2.2 AA, com revisão manual e testes com tecnologias assistivas.

- Usar `<main>`, `<header>`, headings e botões HTML nativos sempre que possível.
- Implementar a grade como widget ARIA apenas se a navegação composta tiver comportamento consistente. Nomear grade, linhas e células; anunciar coordenada, valor/pista, candidatos quando útil e conflito.
- Adotar modelo de foco roving: uma única célula no tab order; setas movem uma casa e atualizam foco; `Tab` sai do widget normalmente.
- `1–9` insere número; `Delete`/`Backspace` apaga; atalhos não são interceptados quando foco está em input, seletor ou controle de texto.
- Foco visível, não ocultado por overflow; ordem de foco segue ordem visual.
- Mensagens de validação via região `aria-live="polite"` sem repetir a grade inteira em cada jogada.
- Erros anunciados em texto além do ícone/cor. Células iniciais distinguíveis das editáveis.
- Alvos de controles preferencialmente pelo menos 44×44 CSS px; grade tem exceção de densidade inerente, mas deve oferecer controles numéricos alternativos e foco navegável.
- Suporte a zoom, contraste do sistema, leitores de tela, alto contraste e redução de movimento.
- Idioma da página (`lang`) atualizado em cada mudança PT/EN e mensagens acessíveis traduzidas junto com rótulos.

## 14. Internacionalização

Usar chaves estáveis em catálogos separados, por exemplo `pt-BR.json` e `en.json`. Regras:

- Não montar sentenças concatenando fragmentos.
- Formatar datas, tempos e números com `Intl.DateTimeFormat` e `Intl.NumberFormat`.
- Persistir preferência de idioma; na primeira visita, usar idioma do navegador se suportado, senão `pt-BR`.
- Traduzir labels acessíveis, mensagens de erro, instruções de teclado e confirmações, não somente títulos.
- Ter teste que falhe quando uma chave do idioma padrão estiver ausente em outro catálogo.

## 15. Privacidade e métricas

O objetivo primário informado é aumentar a proporção de partidas concluídas sem abandono. Essa métrica exige definição de início, conclusão e janela de abandono antes da ativação.

Eventos propostos, somente após consentimento:

- `game_started` com modo e dificuldade.
- `game_completed` com duração em faixas, contagem de erros em faixas, número de dicas em faixas e dificuldade.
- `game_abandoned` com duração aproximada, quando a regra de abandono puder ser determinada sem rastreamento invasivo.
- `daily_completed` e `consent_updated`.

Não coletar movimentos célula a célula, valores do tabuleiro, solução, texto livre, identificadores publicitários ou fingerprinting. Evitar IP persistido no fornecedor de analytics; confirmar processamento, retenção e base legal antes da integração. O produto deve funcionar integralmente sem consentimento.

Implementar interface `AnalyticsClient` com implementação `NoopAnalyticsClient` por padrão. Consentimento deve ser granular e revogável; `analyticsClient.track` não envia evento se a preferência estiver ausente ou negada. A política, banner/diálogo e mecanismo de exclusão dos dados analíticos precisam ser decididos antes de ligar um serviço.

## 16. Erros e recuperação

- Falha ao carregar puzzle: mostrar mensagem e oferecer outro puzzle do mesmo nível; registrar erro técnico sem dados da grade.
- Falha ao gravar: preservar jogo em memória, avisar que o salvamento está indisponível e oferecer tentar novamente.
- Puzzle malformado ou solução inválida: rejeitar no build/teste e não iniciar em produção.
- Estado persistido incompatível: migrar; se impossível, preservar cópia recuperável e pedir confirmação antes de reiniciar progresso.
- Erro inesperado de UI: boundary de erro com opção de recarregar, sem apagar automaticamente os dados locais.

## 17. Testes e critérios de aceite

### Testes unitários do motor

- Validação de linha, coluna e região.
- Conflitos múltiplos e limpeza de conflito após edição.
- Imutabilidade de pistas iniciais.
- Contagem de solução zero, uma e múltiplas.
- Tabuleiro completo correto e incorreto.
- Candidatos e máscaras de anotação.
- Reducer, desfazer/refazer e idempotência da conclusão.
- Adaptação não muda puzzle ativo e limita movimento a um nível.
- Sequência em virada de dia, mudança de timezone e conclusão repetida.

### Testes de interface e integração

- Completar uma partida por teclado e por mouse.
- Verificar conflito, contador e anúncio sem depender apenas da cor.
- Alternar nota, dica, desfazer, apagar e tema.
- Alterar PT/EN sem misturar rótulos ou anúncios.
- Fechar/recarregar e recuperar uma partida local.
- Revogar consentimento e verificar que analytics não envia.
- Testar layout em desktop, tablet e viewport estreito.

### Critérios de aceite do MVP

1. Todo puzzle iniciado tem solução única e não permite editar pistas.
2. Mouse e teclado permitem jogar sem bloquear o uso normal de `Tab`.
3. Conflitos são sinalizados visualmente, por texto e pelo contador, sem penalidade que encerre a partida.
4. Uma partida ativa é restaurada após recarregar, ou a pessoa recebe aviso explícito se o armazenamento falhar.
5. Partida concluída atualiza estatísticas e desafios exatamente uma vez.
6. Tema, idioma e progresso sobrevivem a reinício local.
7. Preferência de nível afeta partidas futuras, nunca a partida atual.
8. O jogo permanece funcional sem telemetria e sem rede após os recursos iniciais carregarem.
9. Nenhum fluxo crítico depende exclusivamente de cor, som, movimento ou ponteiro.

## 18. Plano de implementação

### Fase 0 — Fundação

- Criar app Vite/React/TypeScript, scripts, lint e CI.
- Extrair tokens visuais da direção Calma e construir shell responsivo.
- Implementar idioma e tema com preferência do sistema.

### Fase 1 — Motor e partida local

- Criar tipos e motor puro com testes.
- Adicionar lote inicial de puzzles curados validados.
- Implementar grade, entrada, seleção, validação, anotações, apagar e histórico.

### Fase 2 — Persistência e acessibilidade

- Implementar IndexedDB e migrações.
- Restaurar partida; criar apagamento dos dados locais.
- Completar teclado, foco, labels, anúncios e testes com leitor de tela.

### Fase 3 — Progressão e desafio diário

- Implementar seletor de nível, resultados e ajuste automático para partida futura.
- Adicionar identificador diário, sequência e conquistas.
- Resolver política de fuso e dias perdidos antes do lançamento.

### Fase 4 — Privacidade, validação e lançamento

- Validar métricas com pesquisa e definir fornecedor/retenção/consentimento.
- Ativar analytics apenas após decisão e revisão de privacidade.
- Executar testes de navegador, auditoria de acessibilidade, desempenho e revisão de conteúdo PT/EN.
- Publicar aplicação estática via HTTPS e configurar monitoramento sem payload sensível.

## 19. Decisões pendentes antes de congelar a implementação

1. Qual é a regra exata que aumenta ou reduz o nível selecionado automaticamente?
2. O desafio diário segue UTC ou data local? O mesmo puzzle é global?
3. Uma falha de dia zera a sequência, permite recuperação ou usa congelamento?
4. As dicas ensinam técnicas explicando a lógica ou apenas apontam uma jogada?
5. Qual é a regra de conclusão/abandono usada na métrica principal?
6. Qual fornecedor de analytics, retenção de eventos e experiência de consentimento serão aprovados?
7. Qual a política para limpar progresso local sem apagar a preferência de idioma/tema?

Até essas decisões serem tomadas, implementar contratos e módulos com testes, mas manter limiares adaptativos, calendário e envio de analytics configuráveis/desligados.
