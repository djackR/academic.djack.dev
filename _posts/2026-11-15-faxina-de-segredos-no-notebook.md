---
title: 'Faxina de Segredos no Notebook: SOPS com TPM, Cofre e Agentes de IA'
date: 2026-11-15
permalink: /posts/2026/11/faxina-segredos-notebook/
lang: pt
uid: faxina-segredos-notebook
tags:
  - seguranca
  - sops
  - tpm
  - linux
  - secrets
  - ia
  - devops
excerpt: 'Você acha que sabe onde estão as credenciais do seu notebook, até olhar para onde as ferramentas copiam tudo em silêncio: undo do editor, histórico de agentes de IA, logs de CLIs de nuvem e rascunhos de debug. Um guia prático para auditar a máquina, criar um cofre de segurança e usar o SOPS com chave presa ao chip TPM e PIN.'
---

# Faxina de Segredos no Notebook: SOPS com TPM, Cofre e Agentes de IA

Sábado à tarde. Li o artigo do Akita sobre [segurança de verdade em apps Linux](https://akitaonrails.com/2026/11/06/adicionando-seguranca-de-verdade-nos-seus-apps-gui-linux-vibecoded/) e pensei: "isso eu já resolvi". Tinha escrito até [um post sobre SOPS com KMS](/posts/2026/05/gerenciando-secrets-sops/). O disco é criptografado, a home tem permissão fechada, o `.env` principal já estava com SOPS.

Resolvi revisar meu ambiente pessoal olhando não para onde eu guardo senhas, mas para onde as ferramentas deixam cópias sem avisar. A surpresa é que quase nenhum vazamento local nasce onde você procuraria primeiro.

---

## O Problema

O ponto do artigo não é "e se roubarem meu notebook". Para isso existe disco criptografado.

A pergunta é: **e se um programa rodando com o meu usuário for malicioso?** Um pacote npm comprometido, uma extensão do editor, um servidor MCP que baixa `@latest` a cada sessão, um script que alguém mandou no Slack. Esse programa enxerga tudo que eu enxergo.

Contra isso, `chmod 600` não adianta. Permissão de arquivo protege de **outro usuário**, e o malware é você. O que protege de verdade:

1. segredo **criptografado** em repouso, com uma chave que o programa não consegue usar sozinho;
2. segredo que **não fica no disco** (SSO, credential helper);
3. credencial com o **menor privilégio** por padrão;
4. chave que exige **um humano** para abrir (PIN, frase-secreta).

## TL;DR

```bash
# 1. Guarde tudo num arquivo criptografado (fora da máquina) ANTES de apagar
tar -C ~ -cpf - <arquivos com credencial> | gpg --symmetric --cipher-algo AES256 -o /media/$USER/usb/bkp-$(date +%F).gpg

# 2. Chave do sops dentro do chip TPM, com PIN (sops >= 3.10)
age-plugin-tpm --generate --pin -o ~/.config/sops/age/keys.txt

# 3. Todo .env com segredo vira NOME.sops.env
sops encrypt --in-place .env.sops.env

# 4. Use sem descriptografar para o disco
sops exec-env .env.sops.env './deploy.sh'

# 5. Ensine os agentes de IA a regra (CLAUDE.md / GEMINI.md + skill + hook)
```

---

## Índice

- [Onde os segredos costumam se esconder](#onde-os-segredos-costumam-se-esconder)
- [Antes de tudo: investigue com um prompt](#antes-de-tudo-investigue-com-um-prompt)
- [Passo 1: cofre antes de apagar](#passo-1-cofre-antes-de-apagar)
- [Passo 2: limpe os lugares não óbvios](#passo-2-limpe-os-lugares-não-óbvios)
- [Passo 3: SOPS sem KMS, com a chave no TPM](#passo-3-sops-sem-kms-com-a-chave-no-tpm)
- [Passo 4: atalhos para o dia a dia](#passo-4-atalhos-para-o-dia-a-dia)
- [Passo 5: endureça o shell e a máquina](#passo-5-endureça-o-shell-e-a-máquina)
- [Passo 6: ensine os agentes de IA](#passo-6-ensine-os-agentes-de-ia)
- [Passo 7: troque o que esteve exposto](#passo-7-troque-o-que-esteve-exposto)
- [Um passo além: segredos em orquestradores de containers](#um-passo-além-segredos-em-orquestradores-de-containers)
- [Checklist de uma tarde](#checklist-de-uma-tarde)
- [Conclusão](#conclusão)

---

## Onde os segredos costumam se esconder

Quando alguém procura credenciais soltas numa máquina de desenvolvimento, normalmente olha só para arquivos `.env`. Na prática, os esconderijos clássicos num Linux de dev são estes:

| Onde | Como o segredo vai parar lá |
|---|---|
| `.env` de projetos antigos e testes locais | senhas de banco local e chaves de API criadas num fim de semana e esquecidas |
| **undo e swap do editor** (`~/.local/state/nvim/undo/`) | a versão **em claro** de um `.env` que depois foi criptografado com sops: o editor guarda o histórico da edição anterior |
| histórico do shell (`~/.bash_history`, `~/.zsh_history`) | `curl` com `Authorization: Bearer` ou token passado por argumento |
| **configurações exportadas para debug** | JSONs ou YAMLs salvos "só para inspecionar" um serviço e que trazem as variáveis de ambiente junto |
| **históricos de agentes de IA** | tudo que o agente lê numa sessão fica salvo em `.jsonl` ou SQLite: se ele abriu um `.env`, o conteúdo está lá |
| logs locais de CLIs de nuvem | ferramentas como o `gcloud` gravam milhares de logs detalhados por padrão; num comando de autenticação ou debug, tokens e credenciais podem parar ali |
| `~/Downloads` | `client_secret_*.json` de OAuth, CSVs de chave de acesso ou `.pem` baixados uma vez e esquecidos |
| chaves SSH antigas | chaves geradas sem passphrase ao longo do tempo |

E três configurações muito comuns que ampliam o estrago caso algo rode com o seu usuário:

- o perfil padrão da CLI de nuvem ter permissão de **administrador**: qualquer comando de qualquer script, inclusive de agente, herda esse poder;
- o `PATH` começar com `./bin`: dentro de um repositório clonado, um `bin/git` malicioso rodaria no lugar do `/usr/bin/git`;
- o **firewall estar desligado parecendo ligado**: no Ubuntu, `systemctl is-active ufw` responde `active` mesmo com o firewall desabilitado; quem mostra o estado real é `sudo ufw status`.

Três lições antes do passo a passo:

- **Criptografar o arquivo não basta se o editor guarda o rascunho.** É a armadilha mais irônica.
- **Agente de IA é uma copiadora de segredos.** Tudo que ele lê vai para o histórico dele, em texto puro.
- **Exportar configuração "só para analisar" é como esses arquivos nascem.** Ninguém decide guardar senha; ela vem junto.

---

## Antes de tudo: investigue com um prompt

Não comece apagando. Comece descobrindo o que existe. Um agente de IA faz essa varredura bem, desde que você o proíba de fazer o que mais dá errado nela: **exibir o valor do segredo**. O agente guarda tudo que lê no histórico dele, então uma auditoria mal escrita cria o mesmo problema que está procurando.

Este é o prompt que eu uso. Cole no seu agente (Claude Code, Antigravity ou outro), ajuste o que estiver entre colchetes e peça que ele rode em modo somente leitura:

```text
Faça uma auditoria de segredos em texto puro nesta máquina Linux. SOMENTE LEITURA.

Contexto: o risco que me preocupa é código malicioso rodando com o meu usuário
(pacote npm, extensão, servidor MCP, script), não roubo de disco. Use como
referência o artigo [LINK DO ARTIGO OU DA POLÍTICA DO SEU TIME].

REGRAS (valem para tudo que você fizer):
1. Não altere, mova nem apague nada. Só leia e reporte. A ÚNICA escrita
   permitida é o arquivo do relatório (veja RELATÓRIO, no fim).
2. NUNCA exiba o valor de um segredo, em resposta, comando ou arquivo.
   Reporte caminho, nome da variável e, no máximo, o formato do valor
   (tamanho e classes de caractere, como "aAA9...").
3. Use comandos que não imprimem conteúdo (rg -l, grep -c). Se precisar
   mostrar uma linha, mascare o valor.
4. Confirme pelo FORMATO do valor, não só pelo nome da variável: prefixos
   conhecidos (AKIA, AIza, ghp_, github_pat_, sk-, GOCSPX-, BEGIN PRIVATE KEY),
   alta entropia, URL com senha embutida. Nome sozinho gera falso positivo.
5. Não descriptografe arquivos criptografados (sops, gpg) nem abra cofres.
6. Antes de sugerir apagar qualquer arquivo, diga se ele é ESTADO ÚNICO
   (terraform.tfstate sem backend remoto, banco local, chave em uso).
7. Antes de qualquer ação além de gravar o relatório, peça minha confirmação.

O QUE INVESTIGAR:
A. Arquivos: .env e variantes, chaves privadas (.pem, id_*), JSON de credencial,
   CSV de chave de acesso, client_secret_*.json, terraform.tfstate. Permissões.
B. Rastros de ferramentas: histórico do shell, undo/swap/backup do editor,
   históricos de agentes de IA, logs de CLIs de nuvem, caches de sessão,
   anexos em Downloads.
C. Arquivos exportados "para analisar" (JSON de infraestrutura, dumps) que
   contenham variáveis de ambiente.
D. A máquina: perfil padrão da CLI de nuvem (é administrador?), PATH com
   entradas relativas, firewall (confira com "ufw status", não só com
   systemctl), portas escutando em todas as interfaces, chaves SSH sem
   passphrase, grupo docker, atualizações pendentes, MCPs com @latest.
E. O que já está bom, para eu manter.

RELATÓRIO:
Ao terminar, GRAVE o relatório num arquivo Markdown. Esta é a única escrita
permitida:
  caminho: ~/relatorio-varredura-segredos-AAAA-MM-DD.md, com permissão 600
  (se o arquivo já existir, não sobrescreva; use outro nome)
O relatório NÃO pode conter nenhum valor de segredo (regra 2) e deve ter
estas seções:
1. Resumo em até 5 linhas.
2. Tabela de achados: severidade | caminho | tipo | confirmado pelo valor? |
   estado único? | ação sugerida.
3. Falsos positivos descartados e por quê.
4. Plano em fases (guardar no cofre -> apagar -> criptografar -> rotacionar), com
   caixas [ ] para eu marcar o que foi feito, separando o que depende de mim.
5. O que você NÃO conseguiu verificar.
6. O que já está bom.
No terminal, mostre só o resumo e o caminho do arquivo.
```

Cada regra existe por um erro real:

- **Regras 2 e 3:** um agente que "só quer mostrar o contexto" imprime o conteúdo do arquivo e o segredo vai parar no histórico da sessão. É o tropeço mais comum numa auditoria com IA.
- **Regra 4:** procurar só pelo nome da variável marca como suspeita a própria conversa em que você *falou* sobre a senha. Uma busca ingênua aponta dezenas de arquivos que nem contêm segredo.
- **Regra 6:** um `terraform.tfstate` parece "arquivo com segredo", mas apagá-lo faz o Terraform esquecer a infraestrutura inteira.
- **Regra 7:** quem apaga em lote sem perguntar acaba apagando o que não devia.

O relatório é um **mapa de onde estão os seus segredos**: trate como sensível. Deixe em `600`, fora de repositório e de pastas sincronizadas, e apague quando o plano estiver concluído. Se for rodar uma segunda varredura depois, peça ao agente que leia o relatório anterior antes de começar, para ele não reportar de novo o que já foi tratado.

Depois da auditoria, apague a sessão do agente se ela tiver registrado algum valor, e só então siga para os passos abaixo.

---

## Passo 1: cofre antes de apagar

A ordem importa: **primeiro guardar, depois apagar, por último trocar.**

Junte todos os arquivos com credencial num único arquivo criptografado com frase-secreta. O texto puro não toca o disco: vai do `tar` direto para o `gpg` por pipe.

```bash
tar -C ~ -cpf - .ssh/minha-chave.pem projetos/app/.env ... \
  | gpg --symmetric --cipher-algo AES256 \
        --s2k-mode 3 --s2k-digest-algo SHA512 --s2k-count 65011712 \
        --no-symkey-cache -o /media/$USER/usb/bkp-$(date +%F).gpg

gpg -d /media/$USER/usb/bkp-AAAA-MM-DD.gpg | tar -tf -     # confira que abre ANTES de apagar
```

Use uma frase longa (palavras aleatórias) e guarde fora do computador. Ela não tem recuperação. Mantenha cópia em outro aparelho, com nome neutro que não entregue o que é.

A vantagem que eu não esperava: com o cofre guardado, apagar deixa de dar medo. Precisa de uma chave SSH antiga uma vez por mês? Ela sai do arquivo criptografado direto para o agente SSH, sem virar texto puro no disco, e expira sozinha:

```bash
gpg -d /media/$USER/usb/bkp-AAAA-MM-DD.gpg | tar -xOf - .ssh/minha-chave.pem | ssh-add -t 900 -   # 15 minutos
```

---

## Passo 2: limpe os lugares não óbvios

**Undo, swap e backup do editor.** Apague os que têm nome de `.env` e configure o editor para não gravar rastro desses arquivos. No Neovim:

```lua
-- ~/.config/nvim/lua/config/options.lua
vim.api.nvim_create_autocmd({ "BufNewFile", "BufReadPre" }, {
  pattern = { "*.env", ".env*", "*.pem", "*.key", "*/.ssh/*", "/tmp/*", "/dev/shm/*", "/run/user/*" },
  callback = function()
    vim.opt_local.undofile = false
    vim.opt_local.swapfile = false
    vim.opt.backup = false
    vim.opt.writebackup = false
    vim.opt.shada = "" -- global: desliga o shada da sessão ao abrir um segredo
  end,
})
```

**Histórico do shell.** Procure `Bearer`, `TOKEN=`, `PASSWORD=` e apague as linhas. Para o futuro:

```bash
export HISTIGNORE='*TOKEN=*:*PASSWORD=*:*SECRET=*:*_KEY=*:*Bearer *:export AWS_*'
```

**Históricos de agentes de IA** (`~/.claude/projects/`, `~/.gemini/.../conversations/` e afins). Procure pelo **formato do valor** (prefixo conhecido como `AKIA`, `AIza`, `ghp_`, alta entropia, URL com senha), não pelo nome da variável. Senão você apaga a conversa em que só *falou* sobre a senha.

**Logs do gcloud:**

```bash
find ~/.config/gcloud/logs -mindepth 1 -delete
gcloud config set core/disable_file_logging true
```

**Arquivos de configuração exportados:** em vez de apagar, redija. Troque o valor das variáveis sensíveis por `<REDACTED>` e mantenha o resto da análise.

**`~/Downloads`:** `client_secret_*.json`, `*accessKeys*.csv`, `*.pem`.

> Antes de apagar, confira se o arquivo não é **estado único**. Um `terraform.tfstate` sem backend remoto parece "arquivo com segredo", mas apagá-lo faz o Terraform esquecer a infraestrutura inteira.

---

## Passo 3: SOPS sem KMS, com a chave no TPM

No [post anterior](/posts/2026/05/gerenciando-secrets-sops/) eu usei SOPS com KMS da AWS. Para repositórios do time, continua sendo a escolha certa. Para os segredos que são só meus, ele tem dois problemas: depende de login na nuvem, e um processo malicioso com a minha sessão SSO ativa também consegue descriptografar.

A alternativa é o **TPM**, o chip de segurança da placa-mãe que quase todo notebook moderno tem. A chave do sops é criada **dentro do chip**, nunca sai de lá e só funciona com um PIN. O processo malicioso pode até ler o arquivo da chave, mas sem o chip e sem o PIN ele não serve para nada.

### Ferramentas

Binários dos releases oficiais, conferindo o checksum:

- `sops` **3.10 ou mais novo** (a versão que trouxe suporte a plugins do age);
- `age`, `age-keygen` e **`age-plugin-tag`** (vem no pacote do age; sem ele o sops não reconhece a chave do TPM);
- [`age-plugin-tpm`](https://github.com/Foxboron/age-plugin-tpm).

### Acesso ao chip

```bash
sudo usermod -aG tss $USER     # e saia/entre na sessão
```

> **Pegadinha:** até você relogar, o sops mostra um erro enganoso, `tpm plugin: failed to read line: EOF`. Não é o PIN; é o grupo. No meio tempo, rode com `sg tss -c '...'`.

### A chave do dia a dia, no TPM

```bash
mkdir -p ~/.config/sops/age
age-plugin-tpm --generate --pin -o ~/.config/sops/age/keys.txt
```

O PIN pode ser curto: o chip trava depois de alguns erros seguidos, então não dá para adivinhar por força bruta.

### A chave de recuperação (obrigatória)

Se a placa-mãe morrer, a chave do TPM morre junto. Gere uma segunda chave comum, em memória, guarde a parte privada criptografada fora do notebook e use as duas em todo arquivo:

```bash
D=$(mktemp -d /dev/shm/rk.XXXXXX)
age-keygen -o $D/recuperacao.txt
age-keygen -y $D/recuperacao.txt          # anote o destinatário (age1...)
# criptografe $D/recuperacao.txt com gpg + frase-secreta, leve para fora, e então:
rm -rf $D
```

Escreva também um roteiro de recuperação: "o notebook morreu", "esqueci o PIN", "troquei a placa". Sem senha nenhuma dentro. Daqui a seis meses você não vai lembrar dos comandos.

### As regras

Em `~/.sops.yaml`:

```yaml
creation_rules:
  - age: "age1tag1...CHAVE_DO_TPM...,age1...CHAVE_DE_RECUPERACAO..."
```

Repositórios do time com `.sops.yaml` próprio (com KMS compartilhado) continuam como estão: o sops usa o `.sops.yaml` mais próximo do arquivo.

### Converter um `.env`

```bash
install -m 600 .env .env.sops.env && sops encrypt --in-place .env.sops.env
grep -vE '^sops_|^#|^$' .env.sops.env | grep -vc '=ENC\['   # tem que dar 0
rm .env
```

Repare no nome novo. O antigo deixa de existir **de propósito**: um comando velho que procura `vars.env` falha alto, em vez de mandar valores criptografados para um serviço.

Migrar um arquivo que estava no KMS é um comando, e o conteúdo não muda:

```bash
sops updatekeys -y arquivo.env
```

---

## Passo 4: atalhos para o dia a dia

Ninguém usa segurança que dá trabalho. Estes ficam no `~/.bash_aliases`:

```bash
# roda um comando com as variáveis do arquivo criptografado (pede o PIN)
comenv() { local f="$1"; shift
  if [[ $# -eq 1 ]]; then sops exec-env "$f" "$1"
  else sops exec-env "$f" "$(printf '%q ' "$@")"; fi; }

# para script que recebe o CAMINHO do arquivo: {} vira um temporário em memória
comarq() { local f="$1"; shift; TMPDIR=/dev/shm sops exec-file --no-fifo "$f" "$*"; }

# chave SSH criptografada com sops, carregada no agente por N minutos
chave() { sops decrypt --input-type json --output-type binary ~/.ssh/"$1".sops \
          | ssh-add -t "$(( ${2:-15} * 60 ))" -; }
```

No uso:

```bash
comenv .env.sops.env ./deploy.sh
comarq vars.sops.env './atualiza-config.sh -f {} meu-servico'
chave servidor && ssh servidor
```

Para criptografar uma chave SSH: `sops encrypt --input-type binary --output-type json --output ~/.ssh/servidor.sops servidor.pem`.

Um detalhe sobre o `comenv`: o `sops exec-env` evita gravar texto puro no disco, mas as variáveis ainda existem em `/proc/<pid>/environ` **enquanto aquele comando estiver rodando**. Por isso, use-o para envolver comandos curtos (`./deploy.sh`), nunca para abrir um `bash` interativo pelo resto do dia.

Trocar o PIN depois: o plugin não troca o PIN de uma chave existente. O caminho é gerar uma chave nova no TPM e rodar `sops updatekeys` em cada arquivo. Mantenha uma lista dos arquivos criptografados para isso.

---

## Passo 5: endureça o shell e a máquina

- **`umask 077`** no `~/.bashrc`: arquivo novo nasce `600`, não `664`.
- **Tire `./bin` do PATH.** Se o seu framework de dotfiles põe lá, sobrescreva depois de carregá-lo. Em projetos que guardam scripts em `bin/`, chame-os com o caminho (`bin/script`).
- **Aliases perigosos com trava:** o alias que abre shell num ambiente crítico e o do ambiente de teste não podem diferir por poucas letras (como `hml` e `prd`). Transforme em função que pede para digitar `prod`.
- **Firewall de verdade:**

  ```bash
  sudo ufw default deny incoming && sudo ufw default allow outgoing && sudo ufw enable
  sudo ufw status verbose        # este é o que diz a verdade
  ```

  Porta publicada pelo Docker passa por fora do ufw: use `-p 127.0.0.1:8080:80`.
- **Atualizações** (`sudo apt upgrade`) e **reinício**, para a `libc` nova valer nos processos.
- **Perfil de nuvem:** crie um perfil só de leitura e use o de administrador de propósito.

> Antes de mexer num processo desconhecido escutando numa porta, descubra de quem é (`sudo ss -tlnp`). Muitas vezes a porta "estranha" é só um serviço local legítimo, mas só dá para ter certeza conferindo.

---

## Passo 6: ensine os agentes de IA

Esta é a parte que eu mais recomendo, porque é onde segredos costumam vazar sem ninguém perceber. São três camadas, e cada uma cobre o furo da outra.

### Uma regra global

Lida em toda sessão. No Claude Code é `~/.claude/CLAUDE.md`; no Antigravity (`agy`), `~/.gemini/GEMINI.md`:

```markdown
## Segredos
- Segredo nunca fica em texto puro no disco. `.env` com segredo é sempre `NOME.sops.env`.
  Antes de criar, editar ou usar um `.env`, siga a skill `env-sops`.
- Nunca exiba o valor de um segredo, nem o coloque na linha de comando.
- Descriptografar pede o PIN, que só o dono digita. Se precisar do valor, peça para ele rodar o comando.
- Não salve em arquivo a saída inteira de comandos que trazem segredos
  (configuração exportada de serviços, `terraform show`, `kubectl get secret`).

## Nuvem
- Use o perfil só de leitura por padrão. Para alterar algo, peça confirmação antes.
```

### Uma skill com o passo a passo

Criar sem o texto puro tocar o disco, converter, usar. Fica em `~/.agents/skills/env-sops/SKILL.md` e é ligada por symlink em `~/.claude/skills/` e em `~/.gemini/config/skills/`, então uma edição vale para os dois agentes.

O ponto mais importante dela: **o agente consegue criptografar, mas não descriptografar.** Criptografar usa só a chave pública; descriptografar pede o PIN. Isso coloca um humano no meio de todo acesso a segredo, que é exatamente o que queremos.

### Um hook que pergunta antes de gravar `.env` em claro

No Claude Code, em `~/.claude/settings.json`:

```json
"hooks": {
  "PreToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{ "type": "command", "command": "~/.claude/hooks/env-em-claro.sh", "timeout": 10 }]
  }]
}
```

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/env-em-claro.sh: pergunta antes de gravar .env em texto puro
f="$(jq -r '.tool_input.file_path // empty')"; b="${f##*/}"
case "$b" in .env|.env.*|*.env) ;; *) exit 0 ;; esac
case "$b" in *.sops.*|*.example|*.sample|*.template|*exemplo*) exit 0 ;; esac
case "$f" in /dev/shm/*) exit 0 ;; esac
jq -n --arg f "$f" '{hookSpecificOutput:{hookEventName:"PreToolUse",permissionDecision:"ask",
  permissionDecisionReason:("Arquivo .env em texto puro: "+$f+". Use NOME.sops.env.")}}'
```

Ele pergunta em vez de bloquear, porque existem `.env` legítimos (valores padrão de desenvolvimento, iguais ao `.example` versionado).

### O perfil de nuvem só de leitura, imposto

No terminal, o perfil `default` já deve ser só de leitura (para nenhum `npm install` ou script local rodar como administrador por acidente). Mesmo assim, trave explicitamente no `settings.json` do agente:

```json
"env": { "AWS_PROFILE": "seu-perfil-readonly" }
```

O agente nasce só de leitura; o perfil de administrador só entra em cena quando você o chama de propósito (`--profile admin`).

**Limites honestos:** o hook só vê as ferramentas de escrita de arquivo (um `echo > .env` pelo shell passa); a regra global é instrução, não trava; e cada agente tem seu próprio formato de hook.

---

## Passo 7: troque o que esteve exposto

Criptografar e apagar protege daqui para frente. **Não desfaz** o tempo em que a senha esteve em claro, em undo de editor, em histórico de agente ou em SSD (com ZFS ou Btrfs, apagar não garante que o bloco sumiu). A única coisa que resolve é rotacionar.

Priorize pelo custo: chave de API de terceiros e token pessoal se trocam em minutos; senha de banco precisa de janela de manutenção e de combinar com quem depende dela.

---

## Um passo além: segredos em orquestradores de containers

Na nuvem vale o mesmo princípio do notebook. Um hábito comum na indústria é colocar credenciais direto nas variáveis de ambiente da definição do serviço (a *task definition* do ECS, o `env:` de um Deployment no Kubernetes). O valor escrito ali fica visível para quem tem permissão de leitura, permanece em revisões antigas e vem junto sempre que alguém exporta o manifesto para debugar.

A alternativa é guardar só a **referência**. No ECS, por exemplo, o bloco `secrets` aponta para o Secrets Manager ou para o Parameter Store:

```json
"secrets": [
  { "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:REGIAO:CONTA:secret:ambiente/servico:DB_PASSWORD::" }
]
```

A aplicação não muda; continua lendo a mesma variável. No Kubernetes, o equivalente é `secretKeyRef` junto de um gerenciador de segredos externo. Vale conferir como o seu time guarda isso, e lembrar que, ao exportar uma definição de serviço para analisar, o valor das variáveis vai junto.

---

## Checklist de uma tarde

- [ ] Auditoria feita por um agente em modo somente leitura (prompt do início)
- [ ] Cofre com tudo que tem credencial, conferido, com cópia fora do notebook
- [ ] Undo/swap do editor limpos e configurados para não guardar `.env`
- [ ] Histórico do shell limpo e `HISTIGNORE` configurado
- [ ] Históricos de agentes de IA e logs do gcloud varridos
- [ ] Arquivos de configuração exportados redigidos
- [ ] `~/Downloads` sem `client_secret`, `.pem` ou CSV de chave
- [ ] sops 3.10+, age, plugins, grupo `tss`, chave no TPM com PIN
- [ ] Chave de recuperação fora do notebook, testada, e roteiro de recuperação escrito
- [ ] Todo `.env` com segredo virou `*.sops.env`; os originais apagados
- [ ] Chaves SSH com passphrase ou criptografadas com sops
- [ ] `umask 077`, `./bin` fora do PATH, trava nos aliases perigosos
- [ ] Firewall ligado (conferido com `sudo ufw status`)
- [ ] Atualizações aplicadas e máquina reiniciada
- [ ] Regra global, skill e hook para os agentes de IA
- [ ] Perfil de nuvem só de leitura como padrão dos agentes
- [ ] Credenciais que já ficaram em texto puro listadas para rotação

---

## Conclusão

Segurança de notebook não é um interruptor; é parar de deixar rastro. Os segredos raramente vazam onde você os guarda; vazam onde as ferramentas os copiam: o editor, o shell, o agente, o comando de exportação.

A correção também é ferramenta:

1. **Cofre:** guarde antes de apagar, e apagar deixa de dar medo.
2. **TPM + PIN:** a chave do dia a dia mora num chip que não deixa ela sair.
3. **Recuperação:** uma frase longa, guardada fora, para o dia em que o chip morrer.
4. **Agentes que sabem a regra:** antes de começar, não depois do vazamento.

Se for fazer, comece pelo cofre. Ele é o que deixa todo o resto sem medo.

---

## Referências

- [Adicionando segurança de verdade nos seus apps GUI Linux vibecoded - Akita On Rails](https://akitaonrails.com/2026/11/06/adicionando-seguranca-de-verdade-nos-seus-apps-gui-linux-vibecoded/)
- [Gerenciando Secrets com SOPS: KMS, GCP e GPG](/posts/2026/05/gerenciando-secrets-sops/)
- [SOPS - GitHub](https://github.com/getsops/sops)
- [age encryption](https://github.com/FiloSottile/age)
- [age-plugin-tpm](https://github.com/Foxboron/age-plugin-tpm)
- [Amazon ECS: passando dados sensíveis para containers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data.html)
