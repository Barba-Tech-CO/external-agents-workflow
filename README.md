# external-agents-workflow

Skill do Claude Code que descreve um workflow para delegar tarefas de código a CLIs de agentes externos, supervisionados por um Haiku "tech lead": triagem por complexidade, cadeia de ping, comandos por CLI, regras de parada, fallback para Claude e registro de avaliações.

## Instalar
Copie a pasta para `~/.claude/skills/` (ou `~/.agents/skills/` e aponte um symlink de `~/.claude/skills/`):

    cp -r external-agents-workflow ~/.claude/skills/

## Pré-requisitos
Só o Claude Code é obrigatório. Os CLIs abaixo são opcionais; instale os que tiver conta. Sem nenhum, o workflow cai no fallback (agentes Claude leves).
- GitHub Copilot CLI (`gh copilot`)
- Command Code (`cmd`)
- Codex (`codex`)
- Antigravity (`agy`)
- opencode (`opencode`)
- Hermes (`hermes`)

Ids de modelo e flags mudam: descubra com `cmd --list-models`, `gh copilot -- --help`, `agy models`, `opencode models` e `--help` de cada CLI.

Aviso: ids de modelo citados na skill são só exemplos ilustrativos e perecíveis, não recomendação nem ranking.
