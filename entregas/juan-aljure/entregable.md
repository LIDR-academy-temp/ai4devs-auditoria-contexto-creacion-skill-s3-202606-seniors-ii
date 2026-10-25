# Entregable · Sesión 3 — Copilotos IA

- Juan Pablo Aljure
- 25 oct 2025
- Sand box de S2

---

## 1. Hallazgos de la auditoría (Parte A)

> 3-5 cosas que el agente **no pudo inferir** del código y que tendrías que decirle explícitamente.
> Redáctalas para que otra persona las entienda sin contexto adicional. **Sin código propietario ni secretos.**

1. No pudo inferir los contratos de respuesta de API completos
2. Algunas convenciones tampoco pudo inferirlas, como nombres o prefijos o estructura de los datos
3. Algunas specs como por ejemplo que Solo se actualiza en un login exitoso, y un login fallido no lo toca.

---

## 2. SKILL.md de la skill creada (Parte B)

> Pega aquí el contenido completo de tu `.claude/skills/<nombre-skill>/SKILL.md`
> (o enlaza al archivo en tu repositorio sandbox).

```markdown
--
name: push-code
description: Stage unstaged changes, summarize everything done on the branch, flag risky code for the user to fix, and commit with a summary line plus at most 3 bullets. Use when the user wants to commit or push their work.
---

Stage, review, commit and (optionally) push the current work.

**Input**: optional arguments, e.g. a commit scope or `--push`. Without `--push`, ask before pushing.

## Steps

1. **Inspect the state** (run in parallel)
   - `git status --short`
   - `git branch --show-current` and the base branch (`main` unless the repo says otherwise)
   - `git diff` (unstaged) and `git diff --cached` (staged)
   - `git log --oneline <base>..HEAD` (commits already on the branch)
   - `git diff <base>...HEAD --stat` (what the branch has done so far)

   If there is nothing to commit and nothing ahead of the remote, say so and stop.
   If the branch is the base branch (`main`), ask the user to confirm or create a branch first.

2. **Stage unstaged changes**
   - List the unstaged and untracked files to the user.
   - Stage them by explicit path (`git add <paths>`), never `git add -A` blindly.
   - Never stage: `.env*` (except `.env.example`), `*.sqlite*`, `tmp/`, `node_modules/`, build output, credentials, keys or large binaries. Mention any such file you skipped.
   - Files that are already staged stay staged.

3. **Summarize the whole branch**
   Read the diff of the branch against the base (committed + staged), not only the latest changes. Produce a short summary of what the branch does and why, grouped by theme (feature, fix, tests, docs, tooling).

4. **Risk check — mandatory before committing**
   Scan the staged diff for risky code:
   - Secrets: API keys, tokens, passwords, private keys, `APP_KEY` values, real credentials
   - Committed env/DB files (`.env`, `*.sqlite3`)
   - Debug leftovers: `console.log`, `debugger`, `TODO/FIXME/HACK`, commented-out code
   - Focused or skipped tests: `.only`, `.skip`
   - Security-sensitive changes: auth/middleware bypass, disabled validation, wide-open CORS, SQL built by string concatenation, `eval`, hardcoded URLs or hosts
   - Destructive migrations (`dropColumn`, `dropTable`) without a working `down()`
   - Unrelated changes mixed into the commit
   - Failing checks: run the project's lint/typecheck/tests if cheap (for `backend/`: `npm run lint`, `npm run typecheck`, `npm test`) and report failures

   **If any risk is found: STOP.** Show each finding as `file:line — what — why it is risky`, and ask the user (AskUserQuestion) how to proceed:
   - Fix it (then re-run this check)
   - Unstage / exclude that file
   - Commit anyway (only on an explicit yes)

   Never commit risky code silently. Never fix it without telling the user what changed.

5. **Write the commit message**
   Follow the repo's Conventional Commits style (`feat(scope): ...`, `fix`, `chore`, `docs`, `test`, `refactor`):

   ```
   <type>(<scope>): <summary, imperative, <= 72 chars>

   - <bullet 1>
   - <bullet 2>
   - <bullet 3>
   ```

   Rules:
   - One summary line covering the branch's purpose.
   - **At most 3 bullets**, each one line, describing the most important changes (why over what). Fewer is fine.
   - Code and commit text in English.
   - End with the attribution line required by the session's instructions, if any.

   Show the message to the user and wait for approval or edits before committing.

6. **Commit**
   - `git commit` with the approved message (use a heredoc for multi-line).
   - Never use `--no-verify`. If a pre-commit hook fails (this repo runs lint + typecheck), fix the cause, re-stage and create a **new** commit; do not amend.
   - Show `git log -1 --stat` afterwards.

7. **Push (only with consent)**
   - If `--push` was given or the user says yes: `git push` (use `-u origin <branch>` if there is no upstream).
   - Never force-push. Never push to `main` without an explicit instruction.
   - Report the branch, remote and result.

## Output

End with a short report:
- Files staged / skipped (and why)
- Branch summary (2–4 lines)
- Risks found and how each was resolved
- Commit hash and message
- Push result, or "not pushed"

## Guardrails

- Do not run destructive git commands (`reset --hard`, `clean`, `checkout .`, force push) unless explicitly asked.
- Do not modify code to "fix" risks without the user's go-ahead.
- Stage and commit nothing the user has not seen listed.

```

---

## 3. Diario de decisiones

*Skill creada:* push-code ayuda a añadir los cambios sin stage, resume todo lo hecho en la rama y genera un mensaje de commit con un resumen y un máximo de 3 viñetas. 

*Decisiones de diseño tomadas:*
- Decisión 1: Añadir los cambios de toda la rama puesto que suelo en mis equipos mantener comits atomicos pero en ocaciones mas de un commit. 
- Decisión 2: El skill debe pedir que se rectifique cualquier código riesgoso, si lo hay, antes del push. Para evitar sorpresas con push que potencialmente puedan dañar el código
- Decisión 3: Un maximo de 3 viñetas porque un commit demasiado largo no resulta practico ni entendible.

*Qué me resultó fácil:* 
- La parte geberal del commit mesage y resumen

*Qué me resultó ambiguo o difícil de decidir:*
- Escoger las caracteristicas del código riesgoso y referebciarlo y escoger los comandos no permitidos sin confirmacion previa

*Tiempo real invertido:*
- 2h

*Qué probarías si tuvieras más tiempo:*
- Probaria agregar un estandar especifico ademas una puerta para poder entregar el mensaje y saktarse el resumen. puede que sea útil

*¿Usaste IA para crear la skill?* (qué partes generaste con IA y qué partes decidiste tú)
- Si utulice la ia para crear el skill especifico en la estructura claude pero por medio de un pront que especificaba las decisiones de forma. 

### Resultado de la prueba (Paso 8)

- ¿Se activó cuando lo esperabas? Si
- ¿El resultado fue el que querías? Si, hasta hizo la pausa para validar el mensaje del commit
- Si no, ¿qué crees que falló? (no la "arregles" — documenta el primer intento)
