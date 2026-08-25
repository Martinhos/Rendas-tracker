# Gestor Imobiliário — app Android

Projeto Android completo que embrulha a app HTML numa aplicação nativa.
Funciona **offline**, **sem permissões nenhumas** e os dados ficam no dispositivo.

Basta compilar para obter o APK.

---

## Caminho A — GitHub Actions (sem instalar nada)

1. Cria um repositório novo no GitHub (privado serve).
2. Envia para lá o conteúdo desta pasta, mantendo a estrutura de diretórios.
3. Vai a **Actions** → **Build APK** → **Run workflow** (também corre a cada `push`).
4. Ao fim de ~3 min, descarrega o artefacto **GestorImobiliario-apk**.

Por linha de comandos:

```bash
git init -b main && git add . && git commit -m "Gestor Imobiliario"
git remote add origin https://github.com/<utilizador>/<repo>.git
git push -u origin main
```

O token do GitHub precisa dos âmbitos `repo` **e** `workflow` — sem o segundo,
o push é rejeitado por causa do ficheiro em `.github/workflows/`.

## Caminho B — Android Studio

**File → Open** nesta pasta, deixa sincronizar e **Build → Build APK(s)**.
O APK fica em `app/build/outputs/apk/debug/app-debug.apk`.

## Caminho C — linha de comandos

Precisas do JDK 17 e do Android SDK (`ANDROID_HOME` definido). O Gradle vem pelo wrapper:

```bash
./gradlew assembleDebug
```

---

## Instalar no telemóvel

Copia o `app-debug.apk` para o telefone e abre-o. O Android pede para autorizares
**"Instalar apps desconhecidas"** — é normal fora da Play Store.

---

## O que o wrapper faz pela app

O HTML não foi alterado. Tudo o que segue está no `MainActivity.java`:

| Detalhe | Porquê |
|---|---|
| Página servida em `https://appassets.androidplatform.net/` e não em `file://` | Dá um *secure context* ao WebView. Sem isso o `crypto.randomUUID()` não existe e a app rebentava ao criar imóveis, contratos ou movimentos. |
| `setDomStorageEnabled` e `setDatabaseEnabled` | `localStorage` para os dados e IndexedDB para os anexos dos contratos. |
| `WebChromeClient` definido | Sem ele o WebView ignora os diálogos de JS **e** os `<input type="file">`. |
| `onShowFileChooser` | É o que faz o seletor abrir quando escolhes o CSV do Splitwise ou anexas o PDF de um contrato. |
| Ponte `Android.saveAs()` | Abre o `ACTION_CREATE_DOCUMENT` do Android, onde o **Google Drive** aparece como destino. Não é preciso acesso à internet nem à conta Google: quem envia é o Drive. |
| Ponte `Android.openFile()` | `ACTION_OPEN_DOCUMENT` para repor uma cópia de segurança guardada no Drive. |
| Ponte `Android.shareText()` | O botão Partilhar da Avaliação abre o menu de partilha do sistema. |
| Interceção de downloads `blob:` | O WebView não sabe descarregar blobs. É assim que o CSV, as cópias e os anexos são gravados. |
| Temas `values/` e `values-night/` com `android:isLightTheme` | É o que faz o WebView reportar `prefers-color-scheme` à página, para o tema automático funcionar. |
| `setOnApplyWindowInsetsListener` no WebView | A partir do Android 15 as apps desenham de bordo a bordo e o cabeçalho ficava por baixo da barra de estado. |
| Pontes `linkAuto` / `autoSave` / `autoStatus` / `unlinkAuto` | Cópia automática: escolhes o destino uma vez, guardamos a autorização com `takePersistableUriPermission` e a partir daí a app grava lá sozinha a cada alteração. |

**Nenhuma permissão declarada.** Guardar ficheiros passa pelo seletor do sistema,
que dispensa permissões de armazenamento.

## Especificações

- `applicationId`: `pt.gestorimobiliario.app`
- Mínimo: Android 10 (API 29) · Alvo: Android 15 (API 35)
- AGP 8.7.3 · Gradle 8.9 · JDK 17
- Única dependência: `androidx.webkit`

## Assinatura

O projeto inclui `app/gestor.keystore`, uma chave fixa gerada de propósito e
versionada com o código. Sem ela, cada compilação usaria uma chave de debug
diferente e o Android recusaria instalar a atualização por cima da versão
anterior — obrigando a desinstalar e a **perder todos os dados guardados**.

Não uses esta chave para a Play Store: gera uma tua e substitui o bloco
`signingConfigs` no `app/build.gradle`.

## Atualizar mais tarde

Substitui `app/src/main/assets/index.html` pela versão nova e sobe o `versionCode`
no `app/build.gradle`. Mantém a mesma keystore.

## Cópia automática e o Google Drive

Em **Dados → Cópia automática** escolhes uma vez onde fica o ficheiro. A app guarda a
autorização de escrita (`takePersistableUriPermission`) e, a partir daí, grava lá
sozinha a cada alteração, sem voltar a perguntar nada.

Isto **não é uma integração com a API do Google Drive**. A app não pede acesso à tua
conta Google, não tem permissão de internet e não guarda credenciais: escreve num
ficheiro através do seletor de documentos do Android, e o Drive é um dos destinos
possíveis. Quem sincroniza para a nuvem é a app do Drive.

A consequência prática: se o fornecedor do Google Drive não aceitar reescritas
silenciosas no teu dispositivo, a app avisa logo na ligação e podes escolher outra
pasta (por exemplo uma pasta local que o Drive sincronize).

Uma integração verdadeira com a API do Drive obrigaria a criar um projeto no Google
Cloud, ativar a Drive API e gerar um OAuth client ID ligado ao pacote
`pt.gestorimobiliario.app` e ao SHA-1 da keystore incluída.

## Anexos dos contratos

Ficam em IndexedDB, no dispositivo, e **não entram na cópia de segurança em JSON**
— um PDF de contrato estoiraria a quota do `localStorage` e levaria o resto dos
dados atrás. Guarda os originais no Drive se quiseres uma segunda via.
