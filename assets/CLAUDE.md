# Piso de Docker/OCI (schematize-docker) — sempre-on

A imagem é o artefato que vai para produção; o container nasce **sem privilégio**.

1. **Base por digest** (`FROM img:tag@sha256:...`); `latest` e imagem sem tag são vetados. Renovate/Dependabot renova.
2. **Não-root com UID:GID numérico fixo** (`USER 10001:10001`) no estágio final. Sem `USER` o container é root.
3. **Segredo nunca em `ARG`, `ENV` nem camada.** Build: `RUN --mount=type=secret`. Runtime: injete no start (cofre/secret). `COPY .env`/chave vetado.
4. **Multi-stage e imagem mínima** (distroless/scratch/chiseled/slim); toolchain só no estágio de build.
5. **`.dockerignore` obrigatório** (exclui `.env` e `.git` no mínimo). Manifesto/lockfile antes do código; `--mount=type=cache`.
6. **Nada remoto sem verificação:** `ADD <url>` e `curl | sh` vetados; baixe, `sha256sum -c`, execute. `apt`/`apk` limpam na MESMA camada.
7. **SBOM + provenance + scan que trava (HIGH/CRITICAL) + cosign por digest**, e verificação na admissão. Build de release sai do CI.
8. **Tag imutável = sha do git**; deploy referencia o digest; rollback = digest anterior.
9. **Runtime:** `read_only`, `cap_drop: ALL`, `no-new-privileges`, limites. **Nunca** `--privileged`, `/var/run/docker.sock`, `--net=host`.
10. **Banco/cache nunca em `0.0.0.0`**; `.env` fora do git; compose é dev/hml (produção da casa é a `schematize-infra`).
11. <!-- herdado:engineering/orquestracao:curto -->**Orquestrador não desenvolve; subagent barato executa.** O agent principal só planeja, despacha e revisa; ação onerosa vira micro-tasks para o executor mais barato que resolve: `haiku` (mecânico) / `sonnet` (código; falhou → mesmo subagent corrige, até 2 rodadas) → Codex `gpt-5.6-terra` → Cursor → subagent `opus` (motivo) → `park`; **visual vai direto ao Codex**; **imagem nunca pelo Claude: detectar Codex/Gemini e perguntar ao usuário qual usar (Codex é a sugestão), nunca API de imagem paga**; papéis em `agentes.toml`. No overdev, cada item do checklist vai a um subagent e o principal revisa antes do `- [x]`. **Sem frota ociosa:** idle com pendência volta ao trabalho; dependente de outro agent → mata e enfileira com gatilho; terminou → mata (§9.6). Detalhe: `schematize-engineering` → `references/orquestracao.md` §9.<!-- /herdado -->

<!-- herdado:engineering/pisos-22-29:bloco -->**Herda os pisos 22–29 da `schematize-engineering`** (texto completo em `assets/CLAUDE.md` dela): 22 tudo no git — sem remote, push falho ou HEAD≠origin a demanda NÃO se conclui; 23 deploy só do git (VETADO scp/rsync/tar|ssh/docker save|load/git bundle); 24 o ops é o gate da IA — a IA nunca muda a VPS direto nem escreve SQL, só operação nomeada (`ssh vps "ops <op>"`); 25 sem agents inúteis (entregou, morreu); 26 relatório a cada 30 min e handoff a cada 6 h; 27 CI remoto só antes do hml, a pedido ou em tag (`workflow_dispatch` + `hml-*`/`v*`; VETADO `pull_request`, `push` em branch, `workflow_run`, cron sem justificativa); 28 microserviço autônomo (fila durável + 7 testes de resiliência); 29 build/teste na máquina dev/hml quando existe (ponte só no workspace de build, nunca prd) + higiene (`scripts/higiene.sh`).<!-- /herdado -->

Gates: `bash .claude/skills/schematize-docker/scripts/check-dockerfile.sh .` e
`bash .claude/skills/schematize-docker/scripts/check-compose.sh .` — waiver rastreável
`# docker-ok: <motivo>`. Isolamento por serviço, deploy gated e cofre: `schematize-infra`.
