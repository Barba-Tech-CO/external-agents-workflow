# external-agents-workflow

Skill do Claude Code (Markdown puro) que descreve um workflow para delegar tarefas de código a CLIs de agentes externos, supervisionados por um Haiku "tech lead".
Cobre triagem por complexidade, cadeia de ping, comandos por CLI, regras de parada, fallback para Claude e registro de avaliações.

## Instalação (Claude Code)
1. Clonar direto na pasta de skills:

       git clone https://github.com/Barba-Tech-CO/external-agents-workflow ~/.claude/skills/external-agents-workflow

   Alternativa: clonar em `~/.agents/skills/` e criar symlink:

       git clone https://github.com/Barba-Tech-CO/external-agents-workflow ~/.agents/skills/external-agents-workflow
       ln -s ~/.agents/skills/external-agents-workflow ~/.claude/skills/external-agents-workflow

2. Atualizar:

       git -C <destino> pull

## Verificar
1. Rodar (troque `<destino>` pelo caminho usado):

       test -f <destino>/SKILL.md && head -3 <destino>/SKILL.md

   Deve mostrar o frontmatter com `name: external-agents-workflow`.
2. Reiniciar a sessão do Claude Code e conferir que a skill aparece na lista de skills.

## Outros agentes (Codex, Agy, opencode, Hermes, Copilot)
Não há diretórios assumidos aqui. A skill é Markdown puro:
- Se o cliente tem pasta de skills, coloque a pasta lá (consulte a doc do cliente).
- Senão, referencie ou cole o conteúdo de `SKILL.md` no arquivo de instruções do cliente (AGENTS.md, GEMINI.md ou equivalente).

## Pré-requisitos
- Obrigatórios: Claude Code, `git`.
- Opcional: `gh`.
- CLIs executores (todos opcionais; sem nenhum, o workflow cai no fallback com agentes Claude leves):

| Produto | Verificação |
|---|---|
| GitHub Copilot CLI | `gh copilot --help` |
| Command Code | `cmd --version` |
| Codex | `codex --version` |
| Antigravity (Agy) | `agy --version` |
| opencode | `opencode --version` |
| Hermes | `hermes --help` |

Instalação e login de cada CLI seguem a documentação oficial do produto.

## Primeiro uso
Peça ao agente: "use a skill external-agents-workflow para delegar <tarefa>".
A pasta `~/.claude/agent-evals/` é criada na primeira avaliação.

## Aviso
Ids de modelo citados na skill são só exemplos ilustrativos e perecíveis, não recomendação nem ranking. Descubra os seus com `cmd --list-models`, `gh copilot -- --help`, `agy models`, `opencode models`.
