# Bondlive — Site de Deep Links

Este repositório contém o site estático do subdomínio `app.bondlive.com.br`, que serve dois propósitos:

1. **Landing page** (`/`) — apresenta o Bondlive pra quem abre o link no navegador
2. **Fallback de convite** (`/i/{codigo}`) — redireciona quem clicou num link de convite pra Play Store
3. **App Links** (`/.well-known/assetlinks.json`) — permite que o Android abra links `https://app.bondlive.com.br/i/*` diretamente no app, sem passar pelo navegador

---

## Passo 1 — Gerar a fingerprint SHA-256 do keystore de release

O arquivo `.well-known/assetlinks.json` contém um placeholder (`AA:BB:CC:...`) que você **precisa substituir** pela fingerprint real do seu keystore antes de fazer o deploy.

Execute o comando abaixo no terminal (substitua os valores entre colchetes):

```bash
keytool -list -v -keystore [caminho-completo-do-keystore.jks] -alias [alias] -storepass [senha-da-store]
```

Exemplo:
```bash
keytool -list -v -keystore /Users/vitor/bondlive-release.jks -alias bondlive -storepass minha_senha
```

A saída vai mostrar algo parecido com:
```
Certificate fingerprints:
  SHA1: 12:34:56:...
  SHA-256: AB:CD:EF:01:23:45:67:89:AB:CD:EF:01:23:45:67:89:AB:CD:EF:01:23:45:67:89:AB:CD:EF:01:23:45:67:89
```

Copie o valor do **SHA-256** (a linha mais longa, com 32 pares separados por `:`) e substitua no arquivo `.well-known/assetlinks.json`:

```json
"sha256_cert_fingerprints": [
  "AB:CD:EF:01:23:45:67:89:..."  ← cole aqui a fingerprint real
]
```

> ⚠️ **Importante:** o `assetlinks.json` com a fingerprint errada ou de debug faz os App Links não funcionarem. O Android só valida com a fingerprint do APK assinado com o keystore de **release**.

---

## Passo 2 — Criar o repositório no GitHub

1. Acesse [github.com](https://github.com) e faça login
2. Clique em **New repository** (botão verde no canto superior direito)
3. Preencha:
   - **Repository name:** `bondlive-deeplink`
   - **Visibility:** Public ← obrigatório para GitHub Pages gratuito
   - Deixe as outras opções desmarcadas (sem README, sem .gitignore)
4. Clique em **Create repository**
5. Anote o nome de usuário do GitHub (ex: `vitoremanoel`) — vai ser usado no DNS

---

## Passo 3 — Fazer o upload dos arquivos

No terminal, dentro desta pasta (`site-deeplink/`):

```bash
git init
git add .
git commit -m "Deploy inicial do site Bondlive"
git branch -M main
git remote add origin https://github.com/SEU_USUARIO/bondlive-deeplink.git
git push -u origin main
```

Substitua `SEU_USUARIO` pelo seu usuário do GitHub.

---

## Passo 4 — Ativar o GitHub Pages

1. No repositório, clique em **Settings** (aba superior)
2. No menu lateral, clique em **Pages**
3. Em **Source**, selecione: **Deploy from a branch**
4. Em **Branch**, selecione: `main` / `/ (root)`
5. Clique em **Save**
6. Em **Custom domain**, digite: `app.bondlive.com.br`
7. Clique em **Save**
8. Marque a opção **Enforce HTTPS** (aparece depois de alguns minutos)

O GitHub vai gerar automaticamente um certificado SSL. Aguarde 5-10 minutos.

---

## Passo 5 — Configurar o DNS no Registro.br

1. Acesse [registro.br](https://registro.br) e faça login
2. Clique em **Painel** e depois em **bondlive.com.br**
3. Clique em **DNS** (ou "Editar zona DNS")
4. Adicione um novo registro:
   - **Tipo:** CNAME
   - **Nome:** `app`
   - **Valor:** `SEU_USUARIO.github.io` (substitua pelo seu usuário do GitHub, com `.github.io` no final)
   - **TTL:** 3600 (ou deixe o padrão)
5. Salve as alterações

> O Registro.br costuma propagar em **10 a 30 minutos**. Em alguns casos pode demorar até 1 hora.

---

## Passo 6 — Verificar se está funcionando

Depois da propagação do DNS, teste os três pontos:

### 6.1 — O site está acessível
```bash
curl -I https://app.bondlive.com.br/
```
Deve retornar `HTTP/2 200` com `content-type: text/html`.

### 6.2 — O assetlinks.json está acessível com o Content-Type correto
```bash
curl -I https://app.bondlive.com.br/.well-known/assetlinks.json
```
Deve retornar `HTTP/2 200` com `content-type: application/json`.

> ⚠️ Se o `Content-Type` não for `application/json`, o Android não vai validar os App Links. O GitHub Pages serve `.json` corretamente por padrão, então se você vir outro tipo, verifique se o arquivo está na pasta certa.

### 6.3 — O Android reconhece os App Links
Com o celular conectado via USB e adb habilitado:
```bash
adb shell pm get-app-links com.bondlive.app
```
Procure por `app.bondlive.com.br` na saída. O status deve ser `verified` ou `selected`. Se mostrar `none`, aguarde mais alguns minutos e tente de novo — o Android faz a verificação em background após a instalação do app.

Para forçar a re-verificação:
```bash
adb shell pm verify-app-links --re-verify com.bondlive.app
```

---

## Passo 7 — Testar o Analytics via Firebase DebugView

Para ver os eventos de analytics disparados pelo app em tempo real no Firebase Console:

### Ativar o modo debug no celular
```bash
adb shell setprop debug.firebase.analytics.app com.bondlive.app
```

### Acessar o DebugView
1. Acesse o [Firebase Console](https://console.firebase.google.com)
2. Selecione o projeto `boundlive-26afc`
3. No menu lateral, clique em **Analytics** → **DebugView**
4. Abra o app no celular e navegue pelo onboarding
5. Os eventos aparecem em tempo real na tela, com nome e parâmetros

### Desativar o modo debug quando terminar
```bash
adb shell setprop debug.firebase.analytics.app .none.
```

---

## Estrutura dos arquivos

```
site-deeplink/
├── README.md                    ← este arquivo
├── CNAME                        ← aponta o domínio para o GitHub Pages
├── index.html                   ← landing page principal
├── i/
│   └── index.html               ← fallback de convite (redireciona pra Play Store)
└── .well-known/
    └── assetlinks.json          ← verificação de App Links do Android
```

---

## Checklist de deploy

- [ ] Gerou a fingerprint SHA-256 do keystore de release
- [ ] Substituiu o placeholder no `assetlinks.json`
- [ ] Criou o repositório público `bondlive-deeplink` no GitHub
- [ ] Fez o push de todos os arquivos para `main`
- [ ] Ativou o GitHub Pages com custom domain `app.bondlive.com.br`
- [ ] Adicionou o registro CNAME no Registro.br
- [ ] Aguardou a propagação do DNS (10-30 min)
- [ ] Confirmou `HTTP/2 200` no `assetlinks.json` com `content-type: application/json`
- [ ] Confirmou `verified` no `adb shell pm get-app-links`
- [ ] Testou o deep link abrindo `https://app.bondlive.com.br/i/TESTE` no celular com o app instalado
