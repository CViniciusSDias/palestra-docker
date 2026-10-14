---
marp: true
theme: default
paginate: true
style: |
  section {
    background-color: #ffffff;
    color: #111111;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }
  h1, h2, h3 {
    color: #0a3d91;
  }
  strong {
    color: #b30000;
  }
  code {
    background-color: #f0f0f0;
    color: #b30000;
    padding: 2px 6px;
    border-radius: 4px;
  }
  pre {
    background-color: #f5f5f5 !important;
    border-radius: 8px;
    border: 1px solid #888888;
  }
  pre code {
    color: #111111;
    background-color: #f5f5f5 !important;
  }
  pre code .hljs-string { color: #007a3d !important; }
  pre code .hljs-keyword { color: #7a1fa2 !important; }
  pre code .hljs-comment { color: #5a5a5a !important; font-style: italic; }
  pre code .hljs-variable { color: #0a3d91 !important; }
  pre code .hljs-number { color: #b35900 !important; }
  pre code .hljs-title { color: #7a1fa2 !important; }
  pre code .hljs-built_in { color: #7a1fa2 !important; }
  pre code .hljs-attr { color: #0a3d91 !important; }
  a {
    color: #0a3d91;
    text-decoration: underline;
  }
  table {
    color: #111111;
    border-collapse: collapse;
    width: auto;
  }
  table th {
    background-color: #0a3d91 !important;
    color: #ffffff !important;
    border: 1px solid #111111 !important;
    padding: 8px 12px;
  }
  table td {
    background-color: #ffffff !important;
    color: #111111 !important;
    border: 1px solid #111111 !important;
    padding: 8px 12px;
  }
  table tr:nth-child(even) td {
    background-color: #ececec !important;
  }
  table td strong {
    color: #b30000 !important;
  }
  table td code {
    color: #b30000 !important;
    background-color: #f0f0f0 !important;
  }
  section.lead h1 {
    font-size: 2.5em;
    color: #0a3d91;
  }
  section.lead h2 {
    color: #333333;
  }
  blockquote {
    border-left: 4px solid #0a3d91;
    background-color: #f0f0f0;
    color: #111111;
    padding: 10px 20px;
    border-radius: 0 8px 8px 0;
  }
  img[alt~="center"] {
    display: block;
    margin: 0 auto;
  }
  section::after {
    color: #555555;
  }
---

<!-- _class: lead -->
<!-- _paginate: false -->

# Imagens Docker para Produção

## Boas práticas, performance e segurança em aplicações web e PHP

**Vinicius Dias**

<!--
Boa tarde/noite a todos! Meu nome é Vinicius Dias. Quase todo desenvolvedor PHP hoje usa Docker no dia a dia. Mas existe um abismo enorme entre "subir um container local com bind mount para desenvolver" e "gerar uma imagem imutável que vai rodar sob alta carga, autoscaling e auditoria de segurança em produção". Hoje vamos dissecar exatamente como construir imagens PHP seguras, enxutas e de alta performance.
-->

---

# Quem sou eu

- **PHP Certified Engineer** pela Zend
- Pós-graduado em **Arquitetura de Software**
- Pós-graduando em **Computação Forense**
- **Tech Lead** na SOCi (EUA)
- YouTube: **Dias de Dev** (@DiasDeDev)
- Autor: *Introdução a Redes para Desenvolvedores Web*

<!--
Rapidamente sobre mim para quem não me conhece: sou certificado em PHP pela Zend, pós-graduado em Arquitetura de Software e pós-graduando em Computação Forense. Atuo como Tech Lead na SOCi, empresa americana de marketing multi-local. No YouTube tenho o canal Dias de Dev, onde publico conteúdo técnico sobre PHP, arquitetura, redes e DevOps. E recentemente publiquei o livro de redes para devs web.
-->

---

<!-- _class: lead -->

# O Cenário: Dev vs Produção

<!--
Vamos começar alinhando por que o ambiente de desenvolvimento e o de produção exigem abordagens completamente diferentes na hora de construir imagens.
-->

---

# Dev vs Produção: Objetivos Opostos

- **Ambiente de Desenvolvimento:**
  - Foco em **conveniência** e velocidade de feedback
  - Montagem de volumes em tempo real (`bind mounts`)
  - Ferramentas ativas: Xdebug, Node, Git, Composer, bash
  - Execução como `root` para contornar permissões de host

- **Ambiente de Produção:**
  - Foco em **imutabilidade**, **segurança** e **previsibilidade**
  - Inicialização rápida para autoscaling (K8s, ECS)
  - Menor superfície de ataque possível: **zero ferramentas de build**

<!--
Em desenvolvimento, a gente tolera lentidão e excesso de ferramentas para ter comodidade: queremos alterar um arquivo no VS Code e ver na hora, queremos Xdebug ativo, Composer rodando no container. Mas em produção a regra do jogo inverte: o container precisa ser um artefato imutável. Não se entra no container para alterar código nem rodar composer. Cada ferramenta extra esquecida na imagem final é uma vulnerabilidade em potencial.
-->

---

# Anatomia de uma Imagem "Ingênua"

```dockerfile
# ❌ Erros clássicos de quem traz dev para produção
FROM php:latest

RUN apt-get update && apt-get install -y git zip npm gcc make

COPY . /var/www/html

RUN curl -sS https://getcomposer.org/installer | php -- \
    --install-dir=/usr/local/bin --filename=composer && \
    composer install

CMD ["php-fpm"]
```

<!--
Quem aqui já viu um Dockerfile como esse em algum projeto? Ele funciona localmente, passa nos testes, mas para produção é uma bomba-relógio. Temos imagem base flutuante, ferramentas de compilação C e Node misturadas com PHP-FPM, cache de camadas destruído e container rodando como root.
-->

---

# Os Riscos dessa Imagem em Produção

- **Tag `:latest`**: deploys não são reprodutíveis (quebra a qualquer momento)
- **Compiladores e Git no runtime**: centenas de CVEs desnecessárias
- **`COPY . .` antes do Composer**: invalida cache a cada commit
- **Pacotes de desenvolvimento**: PHPUnit e Faker instalados em produção
- **Execução como `root`**: porta aberta para ataques de *container breakout*

<!--
Essa imagem funciona? Sim, ela sobe e responde requisições perfeitamente. O problema não é se ela funciona, mas o custo operacional e os riscos que ela carrega: se a equipe upstream do PHP lançar uma nova versão hoje, seu build de amanhã pode quebrar sem ninguém ter mexido no código. Se um invasor conseguir um RCE na sua aplicação, ele tem à disposição um compilador gcc, git e privilégio de root. Além disso, cada commit que você faz força o download de todo o vendor novamente no CI/CD.
-->

---

# .dockerignore: O Primeiro Filtro

Antes de executar a primeira linha do build, o Docker envia o **Build Context**

- O cliente compacta o diretório e envia para o daemon do Docker
- **Sem filtro**, enviamos gigabytes de dados desnecessários ou sensíveis
- **O perigo real:**
  - `.git/` → histórico com centenas de MBs (aumenta o tempo de upload do context)
  - `.env` → **vazamento crítico** de senhas e chaves da sua máquina
  - `vendor/` e `node_modules/` locais → conflitos de arquitetura de SO

<!--
Muita gente esquece do .dockerignore. Quando você roda docker build, a primeira coisa que o cliente faz é compactar o diretório atual e mandar para o daemon. Se você não ignorar a pasta .git, você envia centenas de megabytes de histórico. Pior ainda: se você esquecer o .env local com credenciais da sua máquina, ele vai parar dentro da imagem que vai para o registro público ou privado da empresa.
-->

---

# Exemplo Prático: .dockerignore

```dockerfile
.git/
.env*
!.env.example

vendor/
node_modules/
.phpunit.cache/
.idea/
.vscode/

tests/
*.md
docker-compose*.yml
```

> **Resultado:** o envio do contexto cai de **~450 MB** para **~2 MB**

<!--
Esse é um .dockerignore clássico para aplicações PHP. Notem o detalhe: ignoramos .env* mas abrimos exceção com ponto de exclamação para o .env.example, que serve como documentação de variáveis. Ignoramos vendor e node_modules locais para garantir que o container baixe dependências compiladas estritamente para o Linux do container.
-->

---

<!-- _class: lead -->

# Imagem Base e Cache de Camadas

<!--
Agora que filtramos o contexto, vamos falar sobre a fundação da imagem: a escolha do sistema operacional base e como organizar as instruções para tirar o máximo proveito do cache do Docker.
-->

---

# Escolha da Imagem Base

<div style="display: flex; justify-content: center;">

| Critério | Debian Slim (`bookworm`) | Alpine Linux (`alpine`) |
|----------|--------------------------|-------------------------|
| **Biblioteca C** | `glibc` (padrão) | `musl libc` (minimalista) |
| **Compatibilidade** | Total (Oracle, SQL Server, APMs) | Risco de bugs em extensões C |
| **Tamanho Base** | ~80 MB | ~10 MB |
| **DNS em K8s** | Estável e previsível | Problemas históricos de concorrência |

</div>

- **Debian Slim**: a escolha corporativa mais estável para PHP
- **Alpine**: excelente para Go/Rust, mas arriscado para ecossistemas C complexos

<!--
A discussão entre Alpine e Debian Slim no mundo PHP é clássica. Alpine é excelente para binários estáticos em Go, mas com PHP temos dezenas de extensões em C. O Alpine usa musl libc em vez de glibc. Já vi casos reais de bugs estranhos de alocação de memória e timeouts de DNS em Kubernetes com Alpine. No mundo corporativo PHP, a recomendação mais segura e testada em batalha é usar Debian Bookworm Slim.
-->

---

# Imagem Base: Fixando Versões

```dockerfile
# ❌ Imprevisível: o build de amanhã pode instalar nova versão ou novo SO
FROM php:latest
FROM php:8.3-fpm

# ✅ Recomendado: versão do PHP e versão do Debian explícitas
FROM php:8.3.18-fpm-bookworm

# 🔒 Máxima segurança corporativa: travado por SHA256 digest
FROM php:8.3.18-fpm-bookworm@sha256:7a4b1c2d3e4f5a6b7c8d9e0f1a2b3c4d...
```

- Evita atualizações automáticas que quebram retrocompatibilidade
- Garante que o ambiente de teste de hoje seja **exatamente** o de produção amanhã

<!--
Por que php:8.3-fpm não é suficiente? Porque amanhã a equipe do PHP lança a 8.3.19 ou a equipe do Debian atualiza pacotes base da imagem, e seu build de produção ganha mudanças que você nunca testou. Fixando a versão completa como php:8.3.18-fpm-bookworm, você tem total garantia de reprodutibilidade.
-->

---

# Cache de Camadas: A Ordem Importa

- O Docker avalia o cache **de cima para baixo**
- Qualquer alteração invalida a camada atual e **todas as seguintes**
- **Regra:** o que muda com menos frequência fica no topo

```
┌─────────────────────────────────────────┐
│ 1. Imagem Base + Extensões PHP          │  ← Muda raramente
├─────────────────────────────────────────┤
│ 2. composer.json + composer.lock        │  ← Muda esporadicamente
├─────────────────────────────────────────┤
│ 3. Instalação do Vendor                 │  ← Aproveita cache!
├─────────────────────────────────────────┤
│ 4. Código da Aplicação                  │  ← Muda a cada commit
└─────────────────────────────────────────┘
```

<!--
O Docker funciona empilhando camadas. Cada comando COPY ou RUN gera uma camada baseada no hash do comando ou no checksum dos arquivos copiados. Se você copia o código do projeto antes de rodar o composer install, toda vez que alguém altera um controller ou um CSS, o Docker invalida a camada do composer e baixa tudo de novo da internet no CI.
-->

---

# Otimizando o Cache de Dependências

```dockerfile
WORKDIR /var/www/html

# 1. Copia apenas os manifestos de dependência
COPY composer.json composer.lock ./

# 2. Instala dependências (em cache até o composer.lock mudar)
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

# 3. Copia o restante do código da aplicação
COPY . .

# 4. Gera o autoloader final com as classes da aplicação
RUN composer dump-autoload --optimize --no-dev --classmap-authoritative
```

<!--
Aqui está o padrão correto: primeiro copiamos apenas o composer.json e o composer.lock. Instalamos as dependências com --no-autoloader, porque os arquivos da aplicação ainda não existem nesse ponto. O Docker vai manter essa camada em cache por semanas, até que alguém adicione ou atualize um pacote. Depois copiamos o código e rodamos o dump-autoload, que leva meros milissegundos.
-->

---

# Extensões PHP sem Resíduos

- `docker-php-ext-install` manual exige instalar pacotes `-dev`, compilar e limpar
- Se feito em comandos `RUN` separados, o lixo permanece nas camadas intermediárias
- Solução padrão: **`mlocati/docker-php-extension-installer`**

```dockerfile
# Copia o binário oficial do instalador
COPY --from=mlocati/php-extension-installer \
     /usr/bin/install-php-extensions /usr/local/bin/

# Instala, compila e limpa dependências de build na mesma camada
RUN install-php-extensions pdo_mysql redis opcache intl zip bcmath
```

<!--
Compilar extensões PHP no Docker costumava ser um inferno de 20 linhas de apt-get install, flags de compilação e apt-get purge. O projeto do Michele Locati resolveu isso para a comunidade. Esse binário detecta automaticamente quais pacotes do sistema operacional são necessários, compila a extensão, ativa no php.ini e limpa todos os fontes e compiladores temporários na mesma camada.
-->

---

<!-- _class: lead -->

# Multi-Stage Builds e Otimização

<!--
Agora chegamos ao divisor de águas da construção de imagens modernas: o Multi-Stage Build, que nos permite ter tudo o que precisamos para compilar sem levar nada desnecessário para o runtime.
-->

---

# Multi-Stage: O Divisor de Águas

- **O container final de execução NÃO precisa de:**
  - `Node.js`, `NPM`, `Vite`, `Tailwind`
  - `Composer`, `Git`, chaves SSH privadas
  - Pacotes de teste (`phpunit`, `faker`, `mockery`)

- **Princípio da Separação de Responsabilidades:**
  - Estágios de **Construção**: contêm compiladores, ferramentas e dependências de build
  - Estágio de **Execução**: contém apenas o runtime limpo e os artefatos finais

<!--
Parem para pensar: por que raios o container que roda o PHP-FPM em produção precisa ter o Node.js e o npm instalados? Por que precisa do binário do Composer? A resposta é: não precisa! Com Multi-Stage builds, nós criamos containers efêmeros especializados apenas para a etapa de compilação, e copiamos somente os arquivos finais prontos para o runtime definitivo.
-->

---

# O Mecanismo do Multi-Stage

```
┌─────────────────────────────────┐
│ Stage 1: AS builder (Composer)  │ ──┐
│ Instala pacotes, gera vendor    │   │
└─────────────────────────────────┘   │  COPY --from=builder /app/vendor
                                      ├─────────────────────────────────▶ [ Imagem Final ]
┌─────────────────────────────────┐   │  COPY --from=assets /public/build   (Apenas Runtime)
│ Stage 2: AS assets (Node/Vite)  │ ──┘
│ Compila JS/CSS minificado       │
└─────────────────────────────────┘
```

- Cada estágio possui seu próprio ambiente isolado
- Artefatos são transferidos via `COPY --from=<stage>`
- Estágios de build são **descartados** ao final do processo

<!--
Vejam como a arquitetura do Multi-Stage funciona: dividimos o processo em estágios nomeados. Cada estágio roda em uma imagem especializada (Node, Composer, etc.). O estágio final de produção seleciona apenas os artefatos compilados que realmente importam usando o comando COPY --from. Todas as ferramentas de compilação intermediárias são completamente descartadas.
-->

---

# Multi-Stage na Prática: A Sintaxe

```dockerfile
# 1. Estágio descartável de compilação
FROM composer:2.7 AS builder
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist

# 2. Estágio final de produção
FROM php:8.3-fpm-bookworm AS production
WORKDIR /var/www/html

# Resgata apenas o vendor gerado no builder
COPY --from=builder /app/vendor/ ./vendor/
COPY . .
```

<!--
Aqui está a sintaxe direta no Dockerfile: declaramos o primeiro FROM com "AS builder". Ele faz o trabalho pesado de resolução e download do vendor. Em seguida, iniciamos o segundo FROM a partir do php:8.3-fpm-bookworm limpo. Usamos COPY --from=builder para copiar apenas a pasta vendor. O binário do Composer nunca entra no container de produção.
-->

---

# Vendor Shrinking: Poda Cirúrgica

Mesmo com `--no-dev`, o Composer traz centenas de arquivos mortos:

- Suítes de testes unitários das próprias bibliotecas (`/tests`, `/Test`)
- Documentação, markdowns e licenças (`.md`, `.txt`, `/docs`)
- Configurações de CI de terceiros (`.github`, `.circleci`, `.travis`)

> **Impacto real**: economiza **15 a 30 MB** e elimina milhares de arquivos/inodes da imagem final

<!--
Aqui está uma dica que grandes empresas usam e que poucos conhecem no dia a dia: mesmo quando rodamos composer install com a flag --no-dev, muitas bibliotecas de terceiros trazem no seu próprio repositório pastas de testes, documentações e exemplos. Nada disso é executado em produção. Fazendo uma poda cirúrgica dessas pastas no estágio do builder, reduzimos dezenas de megabytes e aliviamos o sistema de arquivos.
-->

---

# Multi-Stage na Prática: Assets & Vendor

```dockerfile
# ── Stage 1: Assets de Frontend (Vite) ─────────────────────
FROM node:20-alpine AS assets
WORKDIR /app
COPY package*.json ./ && RUN npm ci --silent
COPY resources/ vite.config.js ./ && RUN npm run build

# ── Stage 2: Dependências PHP (Composer + Vendor Shrinking) 
FROM composer:2.7 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --prefer-dist --optimize-autoloader --no-scripts && \
    rm -rf vendor/*/*/{tests,Tests,docs,doc,examples,samples,.git,.github} && \
    find vendor/ -type f \( -name "*.md" -o -name "phpunit.xml*" \) -delete
```

<!--
Vejam como fica limpo na prática: no Stage 1, usamos uma imagem oficial do Node para compilar os assets com Vite. No Stage 2, usamos a imagem oficial do Composer para resolver as dependências de produção. Logo após a instalação, rodamos o comando de limpeza que apaga pastas de teste e documentação das bibliotecas. Esse estágio é descartado ao final do build.
-->

---

# Multi-Stage na Prática: Runtime Limpo

```dockerfile
# ── Stage 3: Runtime de Produção ───────────────────────────
FROM php:8.3-fpm-bookworm AS production
WORKDIR /var/www/html

# Extensões compiladas
COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/
RUN install-php-extensions pdo_mysql redis opcache

# Copia código e artefatos prontos dos estágios anteriores
COPY --from=vendor /app/vendor/ ./vendor/
COPY --from=assets /app/public/build/ ./public/build/
COPY . .
```

<!--
E no Stage 3, a mágica acontece: partimos de um Debian Slim limpo, instalamos apenas as extensões necessárias para rodar o PHP, e usamos COPY --from para puxar o vendor já resolvido e podado, além dos assets compilados pelo Node. O container final não faz a menor ideia de quem é o Node ou o Composer; ele só tem o código pronto para rodar.
-->

---

<!-- _class: lead -->

# Segurança em Produção

<!--
Agora vamos falar de segurança prática em containers: permissões de arquivos, evitar a armadilha do chown e como lidar com segredos durante o build.
-->

---

# A Armadilha do RUN chown

Por que `RUN chown -R` duplica o tamanho da sua imagem?

- O Docker usa sistemas de arquivos em camadas (**OverlayFS / UnionFS**)
- Camadas já criadas são **estritamente somente-leitura**
- O Docker utiliza o mecanismo de **Copy-on-Write (CoW)**:
  - Para alterar permissões ou dono (`UID/GID`), o arquivo inteiro precisa ser **duplicado** na camada superior

<!--
Essa é uma das maiores armadilhas de engenharia com Docker. O dev copia os arquivos da aplicação e depois vê que o www-data não consegue ler ou escrever, então roda um RUN chown -R na pasta inteira. O Docker não consegue mudar metadados de uma camada já gravada no passado. Para mudar a permissão de um arquivo de 10MB, ele precisa copiar esses 10MB inteiros para a nova camada. Se sua aplicação tem 120MB, você acabou de criar uma camada extra de 120MB só para mudar uma flag de permissão.
-->

---

# O Custo do Copy-on-Write

O que acontece no `docker history` da imagem:

<div style="display: flex; justify-content: center;">

| Camada | Instrução no Dockerfile | Tamanho Adicionado |
|---|---|---|
| **Camada 3** | `COPY . /var/www/html` | **+120 MB** |
| **Camada 4** | `RUN chown -R www-data:www-data /var/www/html` | **+120 MB (duplicado!)** |

</div>

> **Resultado:** uma aplicação de 120MB gera uma imagem de **240MB**!

- O tempo de `push` e `pull` no cluster dobra desnecessariamente
- A camada antiga continua existindo por baixo, consumindo banda e disco

<!--
Vejam o impacto no docker history: a Camada 3 adicionou 120MB com o código da aplicação. A Camada 4 com o RUN chown adicionou MAIS 120MB! Ou seja, você tem 240MB de dados para transferir pela rede no deploy. A camada antiga não desaparece; ela continua existindo no histórico da imagem, consumindo banda de rede e espaço de disco nos nós do cluster Kubernetes.
-->

---

# A Solução: COPY --chown

```dockerfile
# ❌ INCORRETO: duplica o peso de todos os arquivos no UnionFS
COPY . /var/www/html
RUN chown -R www-data:www-data /var/www/html

# ✅ CORRETO: aplica ownership durante a cópia (CUSTO ZERO de camada)
COPY --chown=root:root . .
COPY --from=vendor --chown=root:root /app/vendor/ ./vendor/
COPY --from=assets --chown=root:root /app/public/build/ ./public/build/

# Apenas diretórios específicos de escrita pertencem a www-data
RUN chown -R www-data:www-data storage bootstrap/cache && \
    chmod -R 775 storage bootstrap/cache
```

<!--
Vejam a diferença: usamos a flag nativa --chown diretamente no comando COPY. O arquivo já é gravado na camada pertencendo ao usuário correto. Não existe camada subsequente duplicando nada. E reparem no detalhe de segurança: a maior parte do código pertence a root:root e fica como somente-leitura. Apenas as pastas que realmente precisam de escrita em tempo de execução, como storage e bootstrap/cache, recebem chown para www-data.
-->

---

# Princípio do Menor Privilégio

- Container rodando como `root` amplia impacto de vulnerabilidades (**RCE**)
- O processo PHP-FPM deve rodar com usuário sem privilégios (`USER www-data`)

```dockerfile
# Garante que o PHP-FPM execute sem privilégios de root
USER www-data

EXPOSE 9000
CMD ["php-fpm", "-F"]
```

- **Se houver brecha no código (RCE):**
  - O invasor **não** pode sobrescrever arquivos PHP (`root:root 755`)
  - O invasor **não** pode instalar ferramentas (`apt-get` bloqueado)
  - O invasor **não** tem privilégios de root no host

<!--
Executar como non-root é a defesa em profundidade mais elementar e ignorada. Se um invasor encontrar uma falha de injeção de código ou upload arbitrário, ele cai dentro do container como www-data. Como o código-fonte pertence a root, ele não consegue injetar um backdoor no index.php nem sobrescrever os controllers. E como não é root, não consegue instalar utilitários nem tentar fugir para o host.
-->

---

# BuildKit: Por que e Como Habilitar

O BuildKit é o motor de build de próxima geração do Docker

- Habilitado por padrão no Docker v23+ ou via flag: `DOCKER_BUILDKIT=1`
- Sintaxe moderna declarada na **primeira linha do Dockerfile**:

```dockerfile
# syntax=docker/dockerfile:1.7
```

- **Recursos indispensáveis para Produção:**
  - Montagem de segredos em memória temporária (`--mount=type=secret`)
  - Cache persistente de dependências entre builds (`--mount=type=cache`)
  - Paralelização automática de estágios independentes

<!--
O BuildKit mudou completamente a forma como construímos imagens. Ele analisa o grafo de dependências do Dockerfile e executa os estágios em paralelo. Se o estágio de Node e o estágio do Composer não dependem um do outro, o BuildKit roda os dois ao mesmo tempo! Além disso, ele introduziu montagens transitórias, que resolvem de uma vez por todas o problema de passar segredos para o build.
-->

---

# BuildKit: Gestão Segura de Segredos

- **O Anti-pattern perigoso:**
  ```dockerfile
  ARG GITHUB_TOKEN
  RUN composer config github-oauth.github.com $GITHUB_TOKEN
  # ❌ Fica visível para sempre em: docker history imagem:tag
  ```

- **A Solução com Secret Mount (BuildKit):**
  ```dockerfile
  # syntax=docker/dockerfile:1.7
  FROM composer:2.7 AS vendor
  WORKDIR /app
  COPY composer.json composer.lock ./

  # Monta o segredo em tmpfs apenas durante a execução do comando
  RUN --mount=type=secret,id=composer_auth,target=/root/.composer/auth.json \
      composer install --no-dev --prefer-dist
  ```

<!--
Quem nunca viu alguém passar um token de repositório privado via ARG? O problema é que qualquer pessoa que der um docker history na imagem final consegue ler o valor do ARG em texto puro. Com o secret mount do BuildKit, o arquivo auth.json é montado como um sistema de arquivos em memória temporária exclusivamente durante a execução daquele RUN. Ele nunca se torna uma camada da imagem e não deixa nenhum rastro.
-->

---

# Executando o Build com Segredos

No pipeline de CI/CD (GitHub Actions / GitLab CI):

```bash
# Opção 1: Passando arquivo local de segredo
$ docker build --secret id=composer_auth,src=auth.json -t app:prod .

# Opção 2: Passando diretamente de uma variável de ambiente do CI
$ docker build --secret id=composer_auth,env=COMPOSER_AUTH_JSON -t app:prod .
```

- O segredo **nunca** é gravado em disco na imagem gerada
- Nenhuma chave ou token vaza para o registro de containers (Docker Hub, ECR)

<!--
E como isso roda no CI/CD? Você não precisa criar arquivos temporários arriscados no disco do runner. Você pode usar a flag --secret com a opção env, apontando diretamente para o secret do GitHub Actions ou variável protegida do GitLab. O BuildKit entrega o segredo para o container de build e descarta logo em seguida.
-->

---

# BuildKit: Cache Mounts para o CI/CD

- Em pipelines de CI, cada build geralmente inicia sem o cache do Composer
- O `cache mount` persiste a pasta de downloads do Composer no runner:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM composer:2.7 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./

# Mantém o cache de pacotes no disco do host/runner entre builds
RUN --mount=type=cache,target=/tmp/composer-cache \
    composer install --no-dev --prefer-dist
```

> **Ganho:** o download de dependências repetidas cai de **~3 minutos** para **~8 segundos**

<!--
O cache mount é o segredo para ter builds ultra-rápidos no CI. Toda vez que o Composer baixa um pacote, ele guarda o zip no cache local. Com --mount=type=cache, essa pasta de cache é mapeada para o host de build sem entrar na imagem final. Se você mudar uma dependência no composer.lock, o Composer só baixa aquele pacote novo; todo o resto já está em cache local no runner.
-->

---

<!-- _class: lead -->

# Calibrando o Runtime PHP

<!--
Com a imagem construída de forma limpa e segura, agora precisamos calibrar o próprio interpretador PHP para extrair a máxima performance dentro de um container imutável.
-->

---

# OPcache para Containers Imutáveis

Em servidores tradicionais, o OPcache valida se o arquivo mudou no disco (`stat()` syscall)
Em um container de produção, **o código nunca muda em runtime**!

```ini
; /usr/local/etc/php/conf.d/opcache.ini
opcache.enable=1
opcache.enable_cli=0
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.save_comments=1

; Preload (Laravel 10+ e Symfony 6+)
opcache.preload=/var/www/html/preload.php
opcache.preload_user=www-data
```

<!--
Aqui está o ganho de performance mais fácil de conseguir em PHP: o opcache.validate_timestamps=0. Em servidores antigos, a cada requisição o PHP checava a data de modificação dos arquivos para ver se alguém tinha alterado código via FTP. Num container Docker de produção, o código é imutável! Se você desliga essa checagem, você elimina milhares de chamadas de sistema stat() por segundo. Em conjunto com o Preload do Laravel ou Symfony, o ganho de throughput é brutal.
-->

---

# Logs Estruturados (12-Factor App)

- Containers **não devem** salvar logs em arquivos locais de disco
- Redirecionar tudo para `stdout` e `stderr`

```ini
; /usr/local/etc/php-fpm.d/docker.conf
[global]
error_log = /proc/self/fd/2

[www]
access.log = /proc/self/fd/1
clear_env = no
catch_workers_output = yes
decorate_workers_output = no
```

- `catch_workers_output = yes` → captura logs emitidos por workers
- `decorate_workers_output = no` → não polui logs JSON

<!--
Segundo o 12-Factor App, logs em container devem ser tratados como streams de eventos. Se o PHP gravar logs em /var/log/laravel.log dentro do container, você tem dois problemas: o disco do container vai encher e os logs morrem quando o container for recriado. Redirecionando para stdout e stderr, orquestradores como Kubernetes ou agentes como Datadog e FluentBit coletam e indexam tudo automaticamente.
-->

---

# Healthcheck & Graceful Shutdown

- Orquestrador precisa saber se o PHP-FPM está pronto para receber tráfego
- PHP-FPM usa **`SIGQUIT`** para término gracioso (o padrão do Docker é `SIGTERM`)

```dockerfile
# Utilitário leve de ping FastCGI
COPY --from=renatomefi/php-fpm-healthcheck \
     /usr/local/bin/php-fpm-healthcheck /usr/local/bin/

HEALTHCHECK --interval=15s --timeout=3s --retries=3 \
    CMD php-fpm-healthcheck || exit 1

# Aguarda requisições ativas terminarem antes de parar
STOPSIGNAL SIGQUIT
```

<!--
Um detalhe operacional crítico: o comando docker stop envia por padrão o sinal SIGTERM. Mas o PHP-FPM interpreta SIGTERM como "encerre imediatamente", derrubando requisições de clientes no meio. O FPM precisa receber SIGQUIT para fazer shutdown gracioso, ou seja, terminar de processar as requisições em andamento antes de descer. Definir STOPSIGNAL SIGQUIT no Dockerfile evita erros 502 no Nginx durante deploys.
-->

---

# Arquitetura em Produção: FPM + Nginx

- **Princípio da Responsabilidade Única:**
  - **Nginx:** SSL, compressão, arquivos estáticos, HTTP/2
  - **PHP-FPM:** Exclusivamente processamento de código PHP

```
[ Cliente ] ──▶ [ Ingress / Gateway ]
                      │
                      ▼
            [ Nginx Container ]  ──(FastCGI :9000)──▶  [ PHP-FPM Container ]
             (Estáticos & Assets)                       (Código PHP + OPcache)
```

- **Alternativa moderna:** *FrankenPHP* (Caddy + PHP integrado em um único binário)

<!--
Em produção, a arquitetura clássica e mais recomendada é separar Nginx e PHP-FPM em containers distintos, comunicando via FastCGI. O Nginx atende arquivos estáticos sem incomodar o PHP e gerencia conexões HTTP. Vale mencionar também o FrankenPHP, uma tecnologia recente muito interessante escrita em Go que embarca o PHP diretamente no servidor Caddy em um único binário com suporte nativo a worker mode.
-->

---

<!-- _class: lead -->

# Resultados e Conclusão

<!--
Vamos ver os números reais consolidados e o checklist final para aplicar na volta ao trabalho.
-->

---

# O Impacto Real das Otimizações

<div style="display: flex; justify-content: center;">

| Métrica | Imagem "Ingênua" | Imagem Otimizada |
|---------|------------------|------------------|
| **Tamanho final** | ~1.45 GB | **~95 MB** *(com Vendor Shrink)* |
| **Tempo de CI (cache)** | 4 a 6 min | **~20 segundos** |
| **Vulnerabilidades (CVEs)** | 380+ detectadas | **0 a 2 baixas** |
| **Privilégios** | `root` (risco alto) | **`www-data` (mínimo)** |
| **I/O de Disco** | Alto (valida timestamps) | **Zero (`validate=0`)** |

</div>

> Deploys mais rápidos, menor custo de transferência e muito mais segurança

<!--
Olhem a diferença gritante nos números. Saímos de uma imagem de quase um gigabyte e meio para menos de 100 megabytes. O tempo de build no CI caiu de minutos para segundos, porque o cache das dependências só quebra quando o lock muda. E o scan de vulnerabilidades com Trivy deixa de apontar centenas de CVEs do Node e compiladores para praticamente zero.
-->

---

# Checklist de Bolso para o Deploy

- [ ] **.dockerignore**: sem `.git`, `.env` ou `vendor/` local no build context
- [ ] **Imagem base**: versão específica e imutável (ex: `8.3-fpm-bookworm`)
- [ ] **Camadas**: manifestos do Composer antes do código-fonte
- [ ] **Multi-stage**: Node e Composer descartados; poda de testes do `vendor`
- [ ] **Segurança**: cópia com `COPY --chown` e execução com `USER www-data`
- [ ] **BuildKit**: segredos via `--mount=type=secret` e cache mount
- [ ] **Runtime**: `opcache.validate_timestamps=0` e logs em `stdout`/`stderr`
- [ ] **Operação**: `HEALTHCHECK` configurado e `STOPSIGNAL SIGQUIT`

<!--
Esse é o checklist para vocês salvarem e conferirem antes de qualquer pull request que altere Dockerfiles de produção no time de vocês. Seguindo esses pontos, seus containers estarão prontos para qualquer cluster de alta escala.
-->

---

# Referências & Links Úteis

- 📺 **Canal Dias de Dev** no YouTube: [youtube.com/@DiasDeDev](https://youtube.com/@DiasDeDev)
- ✍️ **Blog**: [dias.dev](https://dias.dev)
- 📦 **Docker PHP Extension Installer**: [github.com/mlocati/docker-php-extension-installer](https://github.com/mlocati/docker-php-extension-installer)
- 🔒 **Snyk: Best practices to containerize web apps**: [snyk.io/blog](https://snyk.io)
- 📘 **Livro**: [*Introdução a Redes para Desenvolvedores Web*](https://leanpub.com/redes-dev-web/)

<!--
Aqui deixo os links e referências para quem quiser se aprofundar. No meu canal no YouTube tem vários vídeos sobre infraestrutura, redes e PHP em produção, além do instalador de extensões do Michele Locati e o artigo da Snyk sobre segurança em containers.
-->

---

<!-- _class: lead -->

# Perguntas?

### Obrigado pela atenção!

**Vinicius Dias**
YouTube: **@DiasDeDev** · Blog: **dias.dev**

<!--
Chegamos ao fim da apresentação! Muito obrigado a todos pela presença. Agora vou abrir para perguntas e dúvidas que vocês tiverem sobre OPcache, permissões, FrankenPHP ou qualquer outro ponto que vimos hoje.
-->
