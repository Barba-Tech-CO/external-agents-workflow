---
name: external-agents-workflow
description: Use quando uma tarefa de código/pesquisa/review com escopo fechado puder ser delegada a CLIs de agentes externos (Copilot, Command Code, Codex, Agy, opencode, Hermes) supervisionados por um Haiku tech lead, em vez de gastar modelos Claude maiores.
---
# Workflow de agentes externos

## Papéis
- Orquestrador (sessão principal): triagem, despacho, conferência final (testes, typecheck, diff) e relato. Não edita código em massa.
- Executor externo: CLI de outro provedor que faz a tarefa (lista abaixo).
- Haiku tech lead: Agent `model: "haiku"` que roda o executor, confere o resultado contra o pedido, reenvia correção (máx. 2 rodadas) e devolve veredito. Nunca despachar o executor direto, sempre via tech lead.

## Triagem
1. Dimensionar em 5 níveis (Baixa, Média, Alta, Avançada, Muito Avançada), cada um com sub-nível baixa/média/alta (15 degraus).
2. Quebrar em tarefas independentes; arquivos disjuntos rodam em paralelo (um Agent por ponto).
3. Tarefa que exige MCP ou contexto da sessão (browser, tracker, memória): Agent Claude normal, não executor externo.
4. Modelo Claude por nível (tech lead e fallback): Haiku = Baixa e Média; Sonnet = Alta; Opus = Avançada e Muito Avançada baixa/média. Sub-nível alto que tende ao próximo nível sobe um modelo; fora disso, o menor que dá conta.
5. Distribuir executores de forma igual entre os disponíveis e variar a complexidade; descobrir quem rende melhor medindo com as suas próprias avaliações (evals).

## Cadeia de ping (antes de lançar)
Testar o executor com ping curto e lançar a tarefa no primeiro que responder. Ordem é sua escolha (ex.: Copilot, Command Code, Agy, Codex, opencode, Hermes). Nenhum respondeu: parar, não relançar em loop, aplicar fallback.
- Copilot: `gh copilot -p "reply ok" -s`
- Command Code: `cmd -p "reply ok" -m <model> --yolo --no-session --skip-onboarding`
- Codex: `codex exec --skip-git-repo-check "reply ok"`
- Agy: `agy --model <model> --dangerously-skip-permissions --print "reply ok"`
- opencode: `opencode run --auto -m <model> "reply ok"`
- Hermes: `hermes -z "reply ok" -m <model> --provider <provider> --yolo --in <dir>`
Cota pode ser por modelo: erro de cota num modelo não derruba o CLI inteiro; teste outro modelo do mesmo CLI antes de trocar.

## Comandos (cwd no projeto; stdin fechado; log em arquivo)
Placeholders: `<dir>` projeto, `<log>` arquivo de log, `<model>` id (descobrir, não chutar; ex.: ver exemplos abaixo), `<prompt>` texto curto ou caminho de um spec.md.
- Copilot: `cd <dir> && gh copilot -p "<prompt>" --allow-all-tools --add-dir <dir> -s --model <model> < /dev/null > <log> 2>&1`. Modelos: `gh copilot -- --help`. Cada modelo tem multiplicador de cota; prefira os de menor multiplicador que dão conta. `--reasoning-effort` baixo em Baixa/Média.
- Command Code: `cd <dir> && timeout 900 cmd -p "<prompt>" -m <model> --yolo --no-session --skip-onboarding --add-dir <dir> --max-turns 40 < /dev/null > <log> 2>&1; echo EXIT=$? >> <log>`. Modelos: `cmd --list-models`. `--max-turns` estourado sai com exit 8. Prefira modelos médios a baratos; caro só quando a tarefa exige entendimento alto (dizer por quê no relatório). Modelo desconhecido: começar em tarefa Baixa e subir um degrau por vez só se as notas forem boas.
- Codex: `codex exec -m <model> -c model_reasoning_effort="<low|medium|high>" --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check -C <dir> "<prompt>" < /dev/null > <log> 2>&1`. Esforço conforme dificuldade; descer se a janela de cota esgotar rápido.
- Agy: `cd <dir> && agy --model <model> --dangerously-skip-permissions --add-dir <dir> --print "<prompt>"` (prompt vem DEPOIS de `--print`; o log só sai no fim, acompanhar por `git status`). Modelos: `agy models`. Esforço no sufixo do id (low/medium/high). Cota: `agy -p /usage --output-format json`.
- opencode: `cd <dir> && opencode run --auto -m <model> "<prompt>" < /dev/null > <log> 2>&1`. Modelos: `opencode models`. Sem esforço configurável.
- Hermes: `cd <dir> && timeout 900 hermes -z "<prompt>" -m <model> --provider <provider> --yolo --in <dir> < /dev/null > <log> 2>&1; echo EXIT=$? >> <log>`. Sempre passar `-m`, `--provider` e `--yolo`. Log só aparece no fim e o processo pode ficar pendurado após gravar: vigiar `git status` e esperar o `timeout`. Alguns modelos ignoram `--in`: conferir onde o arquivo caiu. Mandar o prompt proibir regex se o repo não quiser.
Ressalva: flags mudam entre versões; confirmar com `--help` no primeiro uso e corrigir esta lista.

### Exemplos de modelos (ILUSTRATIVOS, não são recomendação nem ranking; ids mudam, descubra os seus)
- Command Code: `deepseek/deepseek-v4.1-flash`, `qwen/qwen3.8-flash`, `z-ai/glm-5.3-flash`, `claude-haiku-5-5`
- Copilot: `gpt-6-luna` (com `--reasoning-effort <nível>`)
- Agy: `gemini-4-argon-low|medium|high`, `claude-sonnet-5-5-medium`
- Codex: `gpt-6-luna`
- opencode: `opencode/deepseek-v4.1-flash-free`
- Hermes: `poolside/laguna-s-2.1:free` com `--provider nous`

## Parada e timeout
- Não julgar progresso antes de ~15 min.
- Travado = 8 min seguidos sem linha de tool no log E sem arquivo alterado no repo (Agy e Hermes não escrevem log até sair: só `git status`).
- NUNCA matar processo que o tech lead não lançou (vários rodam em paralelo). Deixar o `timeout` do próprio comando agir.

## Agent vs Workflow
- Um tech lead, tarefa curta: Agent em background.
- 2+ tech leads, tarefa complexa/demorada: Workflow com um `agent()` por tech lead, em paralelo.
- Dentro de Workflow o prompt do Haiku DEVE mandar rodar o executor em FOREGROUND bloqueante (com `timeout` generoso, sem `&` nem `run_in_background`) e só retornar depois que o processo sair e o resultado for conferido. Executor que exige background (Agy/Hermes): o Haiku espera em loop até o processo sair. Senão o Haiku retorna antes e o processo morre junto.
- Agente em background: esperar o retorno final; não consultar status a cada notificação.

## Template do prompt do tech lead
```
[bloco fixo de estilo do time: relatório curto, YAGNI, menor diff]
TAREFA (literal): <pedido original>
CRITÉRIOS DE PRONTO: <lista verificável>
PATHS: <arquivos que pode tocar> | NÃO TOCAR: <lista>
RESTRIÇÕES DO REPO: <regras do projeto; repassar também ao executor>
EXECUTOR: <comando exato com <dir>/<log>/<model>> (rodar ping antes; ordem de fallback: ...)
PROCESSO: rodar o executor, conferir git diff + typecheck/lint/testes + requisito original;
se faltar, reenviar correção ao executor (máx. 2 rodadas); regra de parada acima.
RETORNO: veredito, nota 0-10 do executor, feedback em 1-2 frases, modelo/effort usado,
rodadas, paths alterados, executores que falharam o ping.
```
Só o que foi pedido: não acrescentar condições extras ao repasse. Instrução longa vai num `spec.md` temporário em `<projeto>/.claude/tmp/<tarefa>/`, apagado após a conferência.

## Fallback
- Sempre tentar os executores externos primeiro, inclusive em correção pós-review pequena.
- Executor que passou do escopo: avaliar cada mudança, manter o que faz sentido, reverter só o excedente.
- Todos os externos fora (cota, ping falhou, travou): NÃO esperar reset; ir direto para Agent Claude do menor modelo que dá conta (Alta = Sonnet, Média/Baixa = Haiku).
- Agent Claude já rodando não é parado para reroteamento.

## Avaliação (evals.jsonl)
Ao receber o retorno, anexar UMA linha JSON por tarefa por executor em `~/.claude/agent-evals/evals.jsonl` (criar a pasta se faltar):
`{"date","project","task","executor","model","effort","complexity","score","feedback","rounds","fallback"}`
- `executor`: copilot|cmd|codex|agy|opencode|hermes; `complexity`: baixa|media|alta|avancada; `effort`: `n/a` quando não existe; `fallback`: true se usou modelo/executor substituto; `score`: nota 0-10 do tech lead (conferir contra o diff).
Depois de ~10 avaliações por executor, resumir nota média, rodadas e taxa de retrabalho por complexidade e propor o roteamento. Não usar ranking de terceiros: o seu próprio histórico manda.

## Verificação final
É do orquestrador: testes, typecheck, lint e diff, nunca só o relato do executor.
