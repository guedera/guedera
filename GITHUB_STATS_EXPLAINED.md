# Como funciona o GitHub Stats automático

Este documento explica como as imagens de estatísticas do README são geradas e por que essa abordagem foi escolhida.

## O problema

O serviço mais popular para exibir stats no README é o `github-readme-stats.vercel.app`. Ele funciona assim: quando alguém abre seu perfil, o GitHub tenta carregar a imagem fazendo uma requisição para o Vercel, que por sua vez consulta a API do GitHub e gera a imagem na hora.

O problema é que esse serviço é público e usado por milhões de pessoas. O Vercel impõe **rate limiting** — quando o limite de requisições é atingido, o servidor retorna erro. O GitHub recebe esse erro e exibe o ícone de imagem quebrada no README.

## A solução

Em vez de gerar a imagem no momento em que alguém visita o perfil, as imagens são **geradas antecipadamente e salvas como arquivos no próprio repositório**. O README então aponta para esses arquivos estáticos, que o GitHub serve diretamente — sem depender de nenhum serviço externo.

Isso é feito com **GitHub Actions**.

---

## O que é GitHub Actions

GitHub Actions é um sistema de automação nativo do GitHub. Você escreve um arquivo YAML descrevendo tarefas, e o GitHub executa essas tarefas automaticamente em uma máquina virtual temporária — podendo ser disparado por push, pull request, agendamento (cron), ou manualmente.

Os workflows ficam em `.github/workflows/`.

---

## O Personal Access Token (PAT)

A API do GitHub exige autenticação para retornar dados de um usuário. O PAT é uma chave de acesso gerada nas configurações da sua conta:

`github.com/settings/tokens` → **Tokens (classic)** → **Generate new token**

O escopo `repo` foi marcado para que o token tenha permissão de leitura nos repositórios.

### Por que usar Secrets?

O token não pode ficar exposto no código — qualquer pessoa com acesso ao repositório poderia usá-lo. O GitHub oferece **Secrets**: variáveis criptografadas que ficam armazenadas nas configurações do repositório e só ficam acessíveis durante a execução de uma Action. Nem o dono do repositório consegue ver o valor após salvo.

O Secret foi criado em:
`github.com/guedera/guedera/settings/secrets/actions` → **New repository secret** → nome: `ACCESS_TOKEN`

No workflow, ele é referenciado assim:
```yaml
env:
  ACCESS_TOKEN: ${{ secrets.ACCESS_TOKEN }}
```

---

## O arquivo YAML explicado

Arquivo: `.github/workflows/github-stats.yml`

### Gatilhos

```yaml
on:
  schedule:
    - cron: "0 */12 * * *"  # executa a cada 12 horas
  workflow_dispatch:          # permite execução manual pelo site
```

O formato cron segue o padrão Unix: `minuto hora dia mês dia-da-semana`. O valor `0 */12 * * *` significa "no minuto 0, a cada 12 horas".

### Ambiente

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
```

O GitHub sobe uma VM Ubuntu temporária para executar o job. A permissão `contents: write` é necessária para que a Action possa fazer commit de volta no repositório.

### Etapas

**1. Checkout**
```yaml
- uses: actions/checkout@v4
```
Clona o repositório dentro da VM para que os arquivos possam ser modificados e commitados.

**2. Geração dos SVGs**
```yaml
- name: Generate GitHub Stats SVGs
  run: |
    pip install requests
    python generate_stats.py
  env:
    ACCESS_TOKEN: ${{ secrets.ACCESS_TOKEN }}
```
Instala a biblioteca `requests` e executa um script Python que:
- Faz uma requisição à **API GraphQL do GitHub** usando o token
- Busca: total de stars, commits, pull requests, issues e linguagens por repositório
- Calcula a proporção de uso de cada linguagem
- Gera dois arquivos SVG com os dados formatados:
  - `generated/overview.svg` — stats gerais
  - `generated/languages.svg` — linguagens mais usadas

SVG (Scalable Vector Graphics) é um formato de imagem baseado em XML. Foi escolhido porque é texto puro, pode ser gerado sem bibliotecas de imagem, e o GitHub o renderiza nativamente.

**3. Commit e push**
```yaml
- name: Commit and push stats
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add generated/
    git diff --cached --quiet || git commit -m "chore: update github stats"
    git push
```
Configura o git com uma identidade de bot, verifica se houve mudança nos arquivos (para não criar commits vazios) e faz push de volta ao repositório.

---

## O README

As imagens no README apontam para os arquivos gerados:

```markdown
![GitHub Stats](https://raw.githubusercontent.com/guedera/guedera/main/generated/overview.svg)
![Top Languages](https://raw.githubusercontent.com/guedera/guedera/main/generated/languages.svg)
```

`raw.githubusercontent.com` serve arquivos brutos do repositório diretamente — sem processamento, sem rate limit, sempre disponível.

---

## Fluxo completo

```
A cada 12 horas:

GitHub Actions
    └─ sobe VM Ubuntu
        └─ clona o repositório
            └─ script Python consulta API do GitHub (com o PAT)
                └─ gera overview.svg e languages.svg
                    └─ commita e faz push para o repositório

Visitante abre o perfil:

README carrega imagem
    └─ raw.githubusercontent.com serve o SVG do repositório
        └─ imagem aparece sempre, sem depender de serviço externo ✓
```

---

## Para atualizar manualmente

Acesse `github.com/guedera/guedera/actions`, clique em **GitHub Stats** → **Run workflow** → **Run workflow**.
