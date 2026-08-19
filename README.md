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

## 📖 Guia completo: atualizando a área de login de uma instância

Procedimento testado ponta a ponta — da troca do arquivo até o container no ar.

### 1. Identificar o padrão da instância

Abra `app/src/servidor.ts` e veja qual estrutura a rota `/login` usa. Existem **dois padrões** entre as instâncias auditadas (divane, alumax, esportevalle, patty, revistaurbanova):

- **Padrão A — constante fixa.** `const PAGINA_LOGIN = \`...\`` — usado por **divane, patty, revistaurbanova**. Troca direta de string.
- **Padrão B — função parametrizada.** `function paginaLogin(): string { ... }` — usado por **alumax, esportevalle**. Monta a marca em runtime via `config.appDisplayName`/`config.appLogoUrl`/`config.appHeroUrl`, e pode ter prefixo de base path pro modo "hub" (`config.publicBasePath`).

Se a instância for **Padrão B**, guarde estes dois detalhes pro passo 3 — não podem se perder na troca:
- A guarda `if (config.painelAcessoBloqueado) return reply.code(403).send(...)` no início do `POST /login` (kill-switch de acesso, não relacionado ao design).
- O prefixo `config.publicBasePath`, **se** essa instância rodar atrás do hub reverso (`ia.sanjaworks.com/{slug}/...`) — confira o valor no `config.ts` antes de decidir se precisa prefixar `/login` e `/phone.html` no `<script>` da página nova.

### 2. Copiar os assets estáticos

O novo visual usa fontes locais, logotipos das marcas e a foto de fundo, sem depender de CDN externa.

```bash
cp -r /caminho/area-de-login/assets/login/* app/public/assets/login/
```

Arquivos incluídos:
- Plano de fundo: `bg.jpg`
- Marcas fixas: `techfala-mark.png` e `sanja-mark.png`
- Fontes: `fonts/hanken.woff2` e `fonts/plex-mono-500.woff2`
- Carrossel de marcas: `logos/` (`sanjaworks.png`, `techfala.png`, `abare.png`, `durole.png`, `exata.png`, `sanjafree.png`, `sanjasmart.png`)

### 3. Atualizar o código do servidor (`app/src/servidor.ts`)

- **Padrão A:** substitua todo o conteúdo entre os backticks da constante `PAGINA_LOGIN` pelo conteúdo de `PAGINA_LOGIN.html` deste repo.
- **Padrão B:** troque o corpo de `paginaLogin()` pra `return \`...\`;` com o mesmo conteúdo, removendo as variáveis `marca`/`logo`/`hero` (não são mais usadas) — mas mantendo a guarda `painelAcessoBloqueado` e o prefixo de `publicBasePath` do passo 1 intactos no resto do handler.

A rota `GET /login` não muda (nos dois padrões):

```typescript
app.get('/login', async (_req, reply) => {
  reply.header('Cache-Control', 'no-store, must-revalidate');
  return reply.type('text/html').send(PAGINA_LOGIN);
});
```

- **Segurança no formulário:** o salvamento de senha em texto claro no `localStorage` (`mp_painel_login`) foi removido — a sessão é controlada exclusivamente pelo cookie seguro (`ttlSessao`).
- **Contrato de autenticação:** o endpoint `POST /login` permanece idêntico, recebendo `{ email, senha, lembrar }`.

### 4. Sincronizar o arquivo estático (`app/public/login.html`)

Pra garantir compatibilidade caso o arquivo seja acessado diretamente:

```bash
cp /caminho/area-de-login/PAGINA_LOGIN.html app/public/login.html
```

### 5. Compilar o TypeScript

Entre na pasta `app` da instância e rode o build, pra garantir que a tipagem e a sintaxe estão corretas:

```bash
cd app
npm run build
cd ..
```

### 6. Reiniciar apenas o container da instância (sem derrubar as outras)

Como cada instância roda em um container Docker isolado, as alterações nos arquivos só entram em vigor quando a imagem do container é reconstruída. Na raiz da pasta da respectiva instância (onde fica o `docker-compose.yml`):

```bash
docker compose up -d --build
```

**🛡️ Por que este comando é seguro:**
- **Escopo restrito**: o comando lê apenas o `docker-compose.yml` da pasta atual e atua unicamente no container daquela instância.
- **Sem impacto em outras instâncias**: nenhuma outra ferramenta, container de cliente ou serviço compartilhado (Postgres, Redis, Qdrant, N8N, Traefik) é parado ou reiniciado.
- **Zero downtime desnecessário**: o Docker compila a nova imagem em segundo plano e só substitui o container antigo no instante exato em que a nova imagem fica pronta.

### 7. Checklist de validação

Depois da reinicialização do container:

- [ ] `GET /login` retorna status 200 OK com o layout glassmorphism
- [ ] Assets estáticos: `GET /assets/login/bg.jpg`, fontes e logos retornam 200 OK (sem 404)
- [ ] Credenciais inválidas exibem a mensagem de erro
- [ ] Credenciais válidas fazem login e redirecionam pro painel
- [ ] "Lembrar" mantém a sessão mais longa (cookie) — sem mais cache de senha no localStorage
- [ ] Carrossel do Grupo Sanja Works animando, e os botões sociais/marcas do ecossistema abrindo os links certos em aba nova
- [ ] Layout empilha certo em mobile (< 760px)

## Próximas instâncias

Antes de aplicar em uma instância nova, primeiro abra o `servidor.ts` dela e veja se usa o Padrão A ou o Padrão B (passo 1 acima) — daí é só seguir o guia. Se a instância divergir dos dois padrões conhecidos, vale conferir com calma antes de colar.
