# Comandos Úteis - Sessão de Trabalho

## Git - Histórico e Investigação

### Ver último commit de um arquivo
```bash
git log -1 --format="%an (%ae) - %ar%n%h - %s" -- <arquivo>
```

### Ver últimos 5 commits de um arquivo
```bash
git log -5 --format="%h - %an - %ar%n    %s%n" -- <arquivo>
```

### Rastrear histórico de uma linha específica
```bash
git log -L 15,15:<arquivo> --oneline
```

### Ver detalhes de um commit específico
```bash
git show --stat <hash>
```

### Ver autor e data de um commit
```bash
git show --format="%an (%ae)%nData: %ad%nTítulo: %s%n" <hash>
```

### Ver branches que contêm um commit
```bash
git branch -a --contains <hash>
```

### Buscar commits por mensagem
```bash
git log --all --oneline --grep="<palavra-chave>"
```

### Ver commits que modificaram padrão de código
```bash
git log -p --all -S "padrão de código" -- <arquivo>
```

## Git - Branches e Reset

### Resetar branch para origin
```bash
git checkout develop
git reset --hard origin/develop
```

### Ver status resumido
```bash
git status --short
```

## GitHub CLI (gh)

### Ver detalhes de um PR
```bash
gh pr view <número> --json title,author,createdAt,mergedAt,url,body
```

### Ver apenas informações básicas
```bash
gh pr view <número> --json title,author,createdAt,mergedAt,url,number
```

## Commits - Seguindo Padrão do Projeto

### Commit com prefixo fix:
```bash
git add <arquivos>
git commit -m "fix: descrição breve do problema

Descrição mais detalhada se necessário.
Explica o contexto e a solução.

Relacionado: #PR_NUMBER

Co-Authored-By: Claude <noreply@anthropic.com>"
```

### Commit direto (sem prefixo)
```bash
git commit -m "Descrição direta do que foi feito

Detalhes adicionais se necessário.

Co-Authored-By: Claude <noreply@anthropic.com>"
```

## Arquivos Modificados Nesta Sessão

### 1. Dockerfiles (modernização Yarn + PostgreSQL)
```bash
git add resources/docker/rails/Dockerfile.base \
        resources/docker/rails/Dockerfile.devcontainer

git commit -m "fix: substitui repositório Yarn por Corepack nos Dockerfiles

Remove repositório Debian do Yarn (chave GPG expirada) e usa Corepack
para gerenciar Yarn Classic v1. Remove apt-key deprecado do PostgreSQL.

Mudanças:
- Yarn: usa corepack prepare yarn@1 --activate
- PostgreSQL: keyring específico via gpg --dearmor
- Remove warnings de apt-key deprecado

Resolve: EXPKEYSIG 23E7166788B63E1E Yarn Packaging

Co-Authored-By: Claude <noreply@anthropic.com>"
```

### 2. JSONs MacroProcess (correção others_steps)
```bash
git add vendor/vfactoring/db/default_data/macro_processes/*.json \
        vendor/vinvoice/db/default_data/macroprocesses/*.json

git commit -m "fix: atualiza JSONs de MacroProcess para usar others_steps_title

Substitui atributo deprecado 'others_steps' por 'others_steps_title'
em 8 arquivos JSON de default_data, corrigindo NoMethodError no db:seed.

Relacionado: #42210, #42324

Co-Authored-By: Claude <noreply@anthropic.com>"
```

## Grep - Busca em Arquivos

### Buscar padrão em arquivos JSON
```bash
git grep "padrão" -- "*.json"
```

### Buscar com contexto
```bash
git grep -C 3 "padrão" -- <arquivo>
```

## Docker - Verificação de Build

### Build de imagem Docker
```bash
docker build -f resources/docker/rails/Dockerfile.base -t v360-base-test .
```

### Verificar versões instaladas
```bash
docker run --rm <imagem> yarn --version
docker run --rm <imagem> node --version
docker run --rm <imagem> psql --version
```

### Verificar keyrings criados
```bash
docker run --rm <imagem> ls -la /usr/share/keyrings/
```

## Problemas Resolvidos

### 1. Chave GPG expirada do Yarn
**Erro:** `EXPKEYSIG 23E7166788B63E1E Yarn Packaging`

**Solução:** Substituir repositório apt por Corepack
```dockerfile
# Remove linhas do repositório Yarn
# Adiciona:
corepack enable && \
corepack prepare yarn@1 --activate
```

### 2. apt-key deprecado
**Erro:** `Warning: apt-key is deprecated`

**Solução:** Usar gpg --dearmor com keyrings específicos
```dockerfile
# Yarn
curl -sS <url> | gpg --dearmor -o /usr/share/keyrings/yarn-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/yarn-archive-keyring.gpg] ..." | tee /etc/apt/sources.list.d/yarn.list

# PostgreSQL
wget --quiet -O - <url> | gpg --dearmor -o /usr/share/keyrings/postgresql-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/postgresql-archive-keyring.gpg] ..." > /etc/apt/sources.list.d/pgdg.list
```

### 3. NoMethodError: others_steps=
**Erro:** `undefined method 'others_steps=' for Vprocessmanager::MacroProcess`

**Solução:** Renomear atributo nos JSONs
```json
// Antes
"others_steps": "valor"

// Depois
"others_steps_title": "valor"
```

## Referências

### PRs Relacionados
- #42210 - Adiciona configuração de localização para others_steps
- #42324 - Adiciona setter para coluna legado others_steps

### Commits de Referência
- aedbfb6f9e2 - Deprecação de others_steps
- d994a2360aa - Setter retrocompatível
- f42276be0ca - Exemplo de commit com --trace
- 654ab62c617 - Remoção de --trace

## Notas

- Sempre usar Yarn v1 (Classic) com `yarn@1`, não `yarn@stable`
- Corepack já vem incluído no Node.js 16.10+
- Keyrings em `/usr/share/keyrings/` seguem melhores práticas
- Incluir `Co-Authored-By: Claude` em commits assistidos
- Relacionar PRs com `Relacionado: #XXXXX` ou `(#XXXXX)`
