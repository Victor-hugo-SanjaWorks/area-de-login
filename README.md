# Área de Login — TechFala

Novo design da tela de `/login` (glassmorphism, marca TechFala + carrossel do Grupo Sanja Works), pronto pra substituir a `PAGINA_LOGIN` atual em cada instância `ferramenta-<cliente>`.

## Arquivos deste repo

- **`index.html`** — preview standalone (documento completo, com `<head>`). Abra direto no navegador só pra visualizar.
- **`PAGINA_LOGIN.html`** — o mesmo HTML, pronto pra colar dentro do `servidor.ts` de cada instância.
- **`assets/login/`** — todos os arquivos externos que a página referencia (fontes, logos, foto de fundo). Precisam ser copiados pra `app/public/assets/login/` de cada instância antes da página funcionar.

## O que mudou em relação ao login atual

- **Visual novo**: painel de vidro (glassmorphism) sobre a foto da ponte, marca TechFala no canto da tela, marca Sanja Works dentro do cartão, carrossel com as 7 marcas do grupo (Sanja Works, TechFala, abaré, DuRole, exata, sanjafree, Sanja Smart), ícones de WhatsApp/TikTok/Instagram e atalhos pras marcas do ecossistema — todos com link real.
- **Removido um bug de segurança**: o `salvarLoginLocal`/`carregarLoginLocal` que salvava a senha em texto puro no `localStorage` (chave `mp_painel_login`) foi tirado. Ele existia em **todas** as instâncias auditadas e era redundante — "manter conectado" já funciona certo via cookie de sessão (`ttlSessao(lembrar)` no backend).
- **Contrato com o backend não muda**: o formulário continua mandando `POST /login` com `{ email, senha, lembrar }`, e trata a resposta do mesmo jeito (200 → redireciona pra `?proximo=` ou `/phone.html?painel=whatsapp`; não-200 → mostra "E-mail ou senha incorretos."). Nenhuma rota nova é necessária.
- **"Esqueceu senha?"** continua sem funcionalidade real (`href="#"`) — igual à versão anterior, que também não tinha isso implementado. Não é uma regressão, é um gap que já existia.

## Dois padrões de instância — confira qual é o seu antes de aplicar

Auditei 5 instâncias (divane, alumax, esportevalle, patty, revistaurbanova) e existem **dois formatos diferentes** de `servidor.ts`. Abra o arquivo da instância e veja qual bate:

### Padrão A — `PAGINA_LOGIN` (constante fixa)
**Instâncias confirmadas:** divane, patty, revistaurbanova.

Procure por `const PAGINA_LOGIN = \`...\`` no `app/src/servidor.ts`. É uma troca direta de string.

1. Copie a pasta `assets/login/` deste repo pra `app/public/assets/login/` da instância.
2. Abra `app/src/servidor.ts` e localize a constante `PAGINA_LOGIN`.
3. Substitua **todo o conteúdo entre os backticks** pelo conteúdo de `PAGINA_LOGIN.html` deste repo.
4. Não mexa em mais nada nas rotas `GET /login` / `POST /login` — o contrato do form é o mesmo.
5. `npm run build` (ou `npm run dev`) local pra conferir que não quebrou o TypeScript.
6. Teste: login válido, senha errada, "lembrar" ligado/desligado.
7. Commit + push pra `main` (dispara o deploy automático).

**Atenção pra revistaurbanova especificamente:** essa instância tinha `<title>` e favicon customizados ("Revista Urbanova — Acesso"). Se quiser manter isso, ajuste o `<title>` e o `<link rel="icon">` no topo do `PAGINA_LOGIN.html` antes de colar — o resto do visual (inclusive os logos dentro da página) já era genérico "TechFala" antes da troca, então não há branding específico sendo perdido aí.

### Padrão B — `paginaLogin()` (função parametrizada)
**Instâncias confirmadas:** alumax, esportevalle.

Procure por `function paginaLogin(): string {` no `app/src/servidor.ts`. Essa função hoje monta a marca em runtime via `config.appDisplayName` / `config.appLogoUrl` / `config.appHeroUrl`, e pode ter um prefixo de base path pra modo "hub" (`config.publicBasePath`).

Como o novo design é uniforme (mesma marca TechFala em todas as instâncias), a parametrização de marca não é mais necessária — mas **duas coisas do handler não podem ser perdidas**:

- A guarda `if (config.painelAcessoBloqueado) return reply.code(403).send(...)` no início do `POST /login` — é um kill-switch de acesso, não relacionado ao design.
- O prefixo `config.publicBasePath`, **se** essa instância estiver rodando atrás do hub reverso (`ia.sanjaworks.com/{slug}/...`). Confira o valor de `publicBasePath` no `config.ts` da instância antes de decidir se precisa prefixar os caminhos `/login` e `/phone.html` no JavaScript da página nova.

Passos:

1. Copie a pasta `assets/login/` pra `app/public/assets/login/`.
2. Troque o corpo de `paginaLogin()` pra simplesmente `return \`...\`;` com o conteúdo de `PAGINA_LOGIN.html`, removendo as variáveis `marca`/`logo`/`hero` (não são mais usadas).
3. Se `config.publicBasePath` não estiver vazio nessa instância, ajuste o `fetch('/login', ...)` e o redirect no `<script>` pra incluir esse prefixo.
4. **Não** mexa no restante do `POST /login` — mantenha a guarda `painelAcessoBloqueado` intacta.
5. Teste local, depois commit + push.

## Checklist de verificação (todas as instâncias)

- [ ] Login com credencial válida redireciona certo
- [ ] Login com senha errada mostra "E-mail ou senha incorretos."
- [ ] "Lembrar" mantém a sessão mais longa (cookie) — sem mais cache de senha no localStorage
- [ ] Fontes, logos e a foto de fundo carregam (checar a aba Network, sem 404 em `/assets/login/...`)
- [ ] Layout empilha certo em mobile (< 760px)
- [ ] Ícones sociais (WhatsApp/TikTok/Instagram) e as marcas do ecossistema abrem os links certos em aba nova

## Próximas instâncias

Antes de aplicar em uma instância nova, primeiro abra o `servidor.ts` dela e veja se usa o Padrão A ou o Padrão B (veja acima) — daí é só seguir o passo a passo correspondente. Se a instância divergir dos dois padrões conhecidos, vale conferir com calma antes de colar.
