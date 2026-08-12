# Celestian Browser

Navegador para desktop construído com **Electron** (Chromium + Node.js), com bloqueador de anúncios nativo, atualização automática e build multiplataforma automatizado via GitHub Actions.

![Tela inicial do Celestian Browser](https://hebbkx1anhila5yf.public.blob.vercel-storage.com/image-h7rBFNgo7dAN7Mt7fqO9nud9eMqfOf.png)

---

## Visão geral da arquitetura

O aplicativo roda em dois "mundos" que se comunicam de forma segura:

- **Processo principal (`src/main.js`)** — o cérebro. Roda em Node.js e controla janelas, abas, sessões, cabeçalhos de rede e o bloqueador de anúncios.
- **Processo de interface (renderer)** — a barra do navegador, as abas e os pop-ups que o usuário vê.
- **Ponte segura (`src/preload.js`)** — o único canal entre os dois mundos, via **IPC** (mensagens), com `contextIsolation` ativado. A página nunca acessa o Node diretamente.

Cada aba não é uma janela separada: é uma **WebContentsView** anexada à janela principal, todas compartilhando a mesma sessão (cookies e logins válidos entre abas).

---

## Bloqueio de anúncios

Utiliza o motor **`@cliqz/adblocker-electron`** (a mesma engine usada pelo Brave/Ghostery), em **duas camadas**:

![Escudo Celestian com contador de anúncios e rastreadores bloqueados](https://hebbkx1anhila5yf.public.blob.vercel-storage.com/image-IbrDThaaP4v0eyIBwENghziiyX3Pot.png)

### 1. Bloqueio de rede

Intercepta cada requisição e cancela as que correspondem às listas de anúncios e rastreadores (EasyList + EasyPrivacy, via `fromPrebuiltAdsAndTracking`). O anúncio nem chega a ser baixado, resultando em economia de dados e maior velocidade.

### 2. Filtragem cosmética

Remove os anúncios que fazem parte da própria página (banners, overlays no player de vídeo, blocos "Patrocinado"), que o bloqueio de rede sozinho não consegue remover:

```js
blocker.config.loadCosmeticFilters = true
blocker.config.enableMutationObserver = true // captura anúncios injetados posteriormente
```

### Cache em disco

Na primeira execução, as listas são baixadas e salvas em `adblock-engine.bin`, na pasta do usuário. Nas execuções seguintes, o motor carrega o arquivo local, proporcionando uma abertura mais rápida e sem depender da rede.

### Otimização de desempenho

Uma página pode gerar centenas de bloqueios por segundo. Atualizar a interface a cada bloqueio prejudicaria o desempenho do navegador. Por isso, o contador é agrupado (*debounce*) e a interface é atualizada, no máximo, a cada 800 ms:

```js
function scheduleBlockedFlush() {
  if (blockedFlushTimer) return
  blockedFlushTimer = setTimeout(() => {
    blockedFlushTimer = null
    store.set("blockedCount", blockedCount)
    sendState()
  }, 800)
}
```

O usuário pode ativar ou desativar o bloqueio pelo ícone de escudo, e o estado é persistido (`enableBlockingInSession` / `disableBlockingInSession`).

---

## Atualização automática

Utiliza **`electron-updater`**, com o feed servido pelos **GitHub Releases** (repositório `inside-exe/celestian-browser-releases`). Quem já instalou o navegador **não precisa baixá-lo novamente**.

1. **Verificação** — ocorre na inicialização do aplicativo e, posteriormente, **a cada 3 horas** (funciona apenas no aplicativo empacotado, não durante `npm start`):

   ```js
   const check = () => autoUpdater.checkForUpdates().catch(() => {})
   check()
   setInterval(check, 3 * 60 * 60 * 1000)
   ```

2. **Download automático** em segundo plano assim que uma nova versão é encontrada (`autoDownload = true`), sem interromper o usuário.

3. **Notificação não intrusiva** — em vez de diálogos nativos, o aplicativo envia estados via IPC (`checking` → `available` → `downloading` → `ready`), exibidos em um painel interno com versão atual, novidades e barra de progresso.

4. **Instalação** — ao clicar em **"Reiniciar agora"** ou automaticamente ao fechar o aplicativo (`autoInstallOnAppQuit = true`):

   ```js
   function quitAndInstall() {
     autoUpdater.quitAndInstall()
   }
   ```

As notas de versão (`releaseNotes`) são normalizadas para texto simples (sem HTML) antes de serem exibidas no painel.

---

## Build e publicação (CI/CD)

Automação via **GitHub Actions** (`.github/workflows/build-browser.yml`). Não é necessário instalar nada localmente. Para lançar uma nova versão:

1. Atualize o campo `version` em `electron-app/package.json` (ex.: `1.2.5` → `1.2.6`).
2. Crie e envie a tag:

   ```bash
   git tag v1.2.6 && git push origin v1.2.6
   ```

Isso dispara um build em **três sistemas operacionais simultaneamente** (*matrix strategy*):

| Sistema | Instalador gerado |
| --------------- | ---------------------- |
| windows-latest | `.exe` (Windows) |
| macos-14 | `.dmg` (macOS) |
| ubuntu-latest | `.AppImage` / `.deb` |

Cada máquina instala as dependências, compila e publica automaticamente (`--publish always`) os instaladores **e** os metadados de atualização (`latest.yml`, `latest-mac.yml` e `latest-linux.yml`) no repositório de releases.

O `electron-updater` de todas as instalações consulta esse mesmo feed. Ou seja: **basta publicar uma nova tag para que todos os usuários recebam a atualização nas próximas 3 horas**.

---

## Compatibilidade e segurança

- **Sites reais:** ajusta cabeçalhos (User-Agent e Client Hints) para se apresentar como Chrome e trata corretamente pop-ups de login/SSO (`window.open`), preservando a ligação `window.opener`, essencial para logins corporativos e fluxos OAuth.
- **AJAX:** preserva o cabeçalho `X-Requested-With: XMLHttpRequest`, permitindo que bibliotecas como jQuery e DataTables recebam JSON corretamente.
- **Segurança:** `contextIsolation` ativado, `nodeIntegration` desativado nas páginas e comunicação exclusivamente via IPC.

---

## Recursos

- Abas com sessão compartilhada
- Bloqueador de anúncios com contador
- Controle de volume por página (Web Audio)
- Zoom com **Ctrl + roda do mouse**
- Busca por voz
- Atalhos:
  - **F5** ou **Ctrl + R** — recarregar página
  - **F12** ou **Ctrl + Shift + I** — abrir DevTools
- Painel de atualização integrado

---

## Desenvolvimento local

```bash
cd electron-app
npm install
npm start          # Executa em modo de desenvolvimento (auto-update desativado)
```

Gerar o instalador manualmente (compila para o sistema operacional em uso):

```bash
npm run dist:win    # Windows (.exe)
npm run dist:mac    # macOS (.dmg)
npm run dist:linux  # Linux (.AppImage/.deb)
```

O instalador será gerado na pasta `dist/`.

> Sem um certificado de assinatura de código, o Windows SmartScreen poderá exibir um aviso na primeira instalação ("Editor desconhecido"). O aplicativo continuará funcionando normalmente, e as atualizações posteriores via `electron-updater` serão aplicadas sem novos avisos. Esse comportamento é comum em aplicativos Electron não assinados.

---

## Stack

- **Electron** — Runtime (Chromium + Node.js)
- **@cliqz/adblocker-electron** — Motor de bloqueio de anúncios
- **electron-updater** — Atualização automática
- **electron-builder** — Empacotamento multiplataforma
- **GitHub Actions** — CI/CD

---

## Desenvolvedor

Projeto desenvolvido por **Void**.
