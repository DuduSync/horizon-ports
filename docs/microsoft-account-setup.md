# Configurar o login Microsoft no Switch

O launcher do Horizon entra com uma conta Microsoft pelo fluxo de **device code** do OAuth 2.0.
Ele não traz um client ID embutido: você registra o seu próprio aplicativo no Azure e coloca o
**Application (client) ID** no SD card. Sem esse arquivo, a aba **Conta** mostra
"Login Microsoft não configurado." e o jogo continua funcionando offline (single player, LAN e
servidores offline).

O desenho técnico do fluxo está em [microsoft-account.md](microsoft-account.md).

> **Não use o client ID do launcher oficial nem de outro launcher.** Registre o seu e use apenas ele.

## 1. O que você precisa

- Uma conta Microsoft que tenha **Minecraft: Java Edition** (comprado ou via Game Pass). É essa
  conta que será autenticada.
- Acesso ao [portal do Azure](https://portal.azure.com) com essa conta. O registro de aplicativo é
  gratuito; não precisa de assinatura paga.
- Um celular ou PC com navegador, para abrir a página da Microsoft e aprovar o código.
- Acesso ao SD card do Switch: FTP, USB/MTP ou um leitor de cartão.
- Opcional, mas recomendado: um **PIN** para o cofre de tokens.

## 2. Registrar o aplicativo no Azure

1. Abra <https://portal.azure.com> e entre com a conta Microsoft que vai usar no jogo. O
   [Microsoft Entra admin center](https://entra.microsoft.com) também serve; os nomes das telas
   podem variar um pouco entre os dois.
2. Vá em **Microsoft Entra ID** → **App registrations** → **New registration**.
3. Preencha:

   - **Name**: algo como `Horizon Switch`. Esse nome aparece na tela de consentimento da Microsoft.
   - **Supported account types**: **Personal Microsoft accounts only**.
     O launcher usa o endpoint `/consumers`, então contas pessoais são o caso certo. Se essa opção
     não aparecer, escolha **Accounts in any organizational directory and personal Microsoft
     accounts**; contas pessoais continuam funcionando.
   - **Redirect URI**: deixe em branco. O device code não usa redirect URI. Se o formulário exigir
     um, escolha **Mobile and desktop applications** e
     `https://login.microsoftonline.com/common/oauth2/nativeclient` — mas isso não é necessário.

4. Clique em **Register**.
5. Abra **Authentication** → **Advanced settings** (o bloco pode estar em outro lugar na sua
   versão do portal) e marque **Allow public client flows** como **Yes**. Salve.
   - Isso habilita o device code. Não crie client secret: o Horizon é um **cliente público**.
6. Não é preciso adicionar permissões de API no portal. Veja a seção 3.

## 3. Permissões e escopos

No login, o código pede exatamente estes escopos:

```text
XboxLive.signin offline_access
```

- `XboxLive.signin` — delega a autenticação à Xbox Live.
- `offline_access` — faz a Microsoft devolver o **refresh token**, que o cofre guarda para não pedir
  login a cada boot.

Eles são escopos **delegados**, solicitados em tempo de execução no endpoint:

```text
https://login.microsoftonline.com/consumers/oauth2/v2.0/devicecode
```

Na prática, você **não** precisa adicioná-los manualmente em **API permissions**. O consentimento
aparece para o jogador durante o login. Em uma conta pessoal, o próprio jogador aprova. Em contas
corporativas ou escolares, o administrador do tenant pode bloquear o consentimento do usuário.

## 4. Copiar o Application (client) ID e colocar no SD

1. No aplicativo registrado, abra **Overview**.
2. Copie **Application (client) ID**. É um GUID no formato
   `00000000-0000-0000-0000-000000000000`.
   - Não copie o **Directory (tenant) ID** nem o **Object ID**.
3. No SD card, crie o arquivo:

```text
sdmc:/switch/horizon/msa-client-id.txt
```

Conteúdo — uma linha, sem aspas e sem espaços:

```text
00000000-0000-0000-0000-000000000000
```

O leitor do arquivo aceita:

- a primeira linha não vazia que não comece com `#` (linhas com `#` são comentários);
- espaços nas pontas e BOM são removidos; **aspas não são removidas**, então não use aspas;
- uma quebra de linha no fim é aceita.

O arquivo precisa ser texto simples e legível pelo jogo.

Para transferir:

- **FTP**: use um servidor FTP homebrew no Switch (por exemplo, ftpd) e envie para
  `switch/horizon/msa-client-id.txt` na raiz do SD.
- **USB/MTP**: use o modo MTP/USB do seu instalador (por exemplo, DBI) ou coloque o SD no PC.
- **Leitor de cartão**: com o Switch desligado, copie para `switch/horizon/`.

Em builds de desenvolvimento, a variável de ambiente `HORIZON_MSA_CLIENT_ID` tem prioridade sobre o
arquivo. No build do Switch, use o arquivo.

## 5. Aprovação da Mojang

A API do Minecraft recusa aplicativos novos no Azure até a Mojang aprovar o client ID. Enquanto isso,
o login chega até a etapa do Minecraft e falha com:

```text
The Minecraft services refused this launcher: its app is not approved by Mojang yet.
```

Peça a aprovação em <https://aka.ms/mce-reviewappid>, informando o client ID, o nome do app, o link
do projeto e uma descrição curta. A aprovação pode levar dias ou semanas. Isso é separado do
registro no Azure.

## 6. Testar no Switch

1. Abra o Horizon e vá na aba **Conta**.
2. Opcional, mas recomendado: digite um **PIN do cofre (opcional)** antes de entrar.
   - Com PIN, o refresh token fica salvo cifrado no SD.
   - Sem PIN, a sessão vale só até fechar o jogo.
3. Toque em **Entrar com Microsoft**.
4. Aguarde. A tela mostra:
   - um QR code do endereço de verificação (normalmente `https://www.microsoft.com/link`);
   - o código para digitar, normalmente no formato `ABCD-1234`;
   - o status "Aguardando aprovação...".
5. Escaneie o QR com o celular ou abra o endereço no navegador. Digite o código, escolha a conta
   Microsoft e aprove.
6. A tela muda para "Aprovado. Conectando à Xbox e ao Minecraft..." e depois para
   **"Sessão Microsoft ativa"**, com o nome e o UUID do jogador. O próximo mundo ou servidor usa
   essa conta.

Se você definiu PIN, o cofre fica em `sdmc:/switch/horizon/auth/` (`account.bin` e `device.key`).
No próximo boot, use **Carregar conta salva** com o mesmo PIN. Sem PIN, aparece o aviso de que o
login vale só até fechar o jogo.

### Se falhar

A mensagem vermelha na tela é a fonte principal. Alguns textos do código aparecem em inglês mesmo
com o launcher em português. Para diagnosticar:

- `sdmc:/switch/horizon/logs/latest.log` — log do jogo/launcher;
- `sdmc:/switch/horizon/stderr.txt` — mensagens de runtime e panics que o Switch não mostraria;
- `sdmc:/switch/horizon/fatal.txt` — relatório de crash, se houver.

O fluxo de login **não** escreve linhas próprias no `latest.log` e nunca grava tokens. A mensagem
de erro do login aparece na tela; o log e o `stderr.txt` servem para falhas gerais do processo
(panic, falha ao iniciar, problema de rede/TLS que derrube a thread). Antes de publicar um log,
revise: ele pode conter nome de jogador, UUID e caminhos.

## 7. Problemas comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| "Login Microsoft não configurado." | Arquivo ausente, caminho errado, vazio ou só com comentários. | Confira `sdmc:/switch/horizon/msa-client-id.txt`, com uma linha contendo o GUID. Verifique o caminho e se não há aspas. |
| "Microsoft refused this launcher's app registration (client ID)." | Client ID errado, app excluído, tipo de conta incompatível ou public client flows desligado. | Copie de novo o **Application (client) ID**; confirme **Allow public client flows = Yes** e o tipo de conta. |
| O login recusa a conta pessoal | App registrado como **Accounts in this organizational directory only**. | Mude para **Personal Microsoft accounts only** (ou multitenant + personal) em **Authentication**, ou crie outro registro. |
| "The sign-in code expired. Start again." / "O código expirou." | Passaram cerca de 15 minutos. | Toque em **Gerar novo código** ou **Tentar novamente**. |
| "Sign-in was declined in the browser." | A aprovação foi recusada. | Tente de novo e aprove a conta. |
| "Could not reach Microsoft/Xbox Live/the Minecraft services. Check your internet connection." | Sem rede, DNS ou TLS. | Teste a conexão do Switch, desligue VPN/proxy e tente de novo. |
| "The Minecraft services refused this launcher: its app is not approved by Mojang yet." | Falta a aprovação da Mojang. | Veja a seção 5. |
| "This account doesn't own Minecraft: Java Edition." | A conta não tem o jogo nem Game Pass. | Use a conta que comprou o jogo ou que tem Game Pass. |
| "This account has no Minecraft profile yet..." | A conta tem o jogo, mas não criou o perfil. | Crie o perfil em <https://minecraft.net> e entre de novo. |
| "This Microsoft account has no Xbox profile yet..." | Não existe perfil Xbox. | Entre uma vez em <https://xbox.com> para criar o perfil. |
| "This is a child account..." | Conta infantil sem família. | Um adulto precisa adicionar a conta a uma família Microsoft. |
| "Xbox Live is not available in your country or region." | Região sem suporte da Xbox Live. | Não há contorno pelo launcher. |
| "Too many sign-ins in a short time..." | Limite de tentativas (HTTP 429). | Espere um minuto e tente de novo. |
| "Could not unlock the saved sign-in (wrong PIN, or the storage was reset)." | PIN errado, `device.key` removido ou SD trocado. | Digite o PIN certo; se perdeu, apague `sdmc:/switch/horizon/auth/` e faça login de novo. |
| "Nenhuma conta salva com esse PIN." | Não há cofre ou o PIN não abriu. | Use **Entrar com Microsoft**. |
| "Sessão encerrada. Sem PIN, o cofre no SD não foi apagado." | **Sair** foi usado sem PIN. | Para remover o cofre, entre com o PIN e use **Sair**, ou apague a pasta `auth/`. |
| Relógio errado | Data/hora muito fora atrapalha TLS e OAuth. | Acerte o relógio nas configurações do Switch ou use sincronização de rede; reinicie o Horizon. |
| Trocou o client ID e a conta salva não renova | O cofre guarda o client ID usado no login. | Saia com o PIN ou apague `auth/` e entre de novo. |

Depois de corrigir o arquivo ou o registro no Azure, basta tocar em **Tentar novamente** na aba
**Conta**. Não é preciso reinstalar o Horizon.

## 8. Segurança

- O **Application (client) ID** não é segredo: ele aparece no login e não dá acesso à conta. Mesmo
  assim, não comite o arquivo `msa-client-id.txt` no repositório. Ele é configuração da sua
  instalação, e o repositório não deve carregar o ID de terceiros.
- **Tokens são segredos.** O refresh token fica cifrado em `account.bin`; o `device.key` viaja junto.
  Não publique a pasta `auth/` nem uma cópia do SD.
- O PIN protege uma cópia do SD, não um console comprometido nem um keylogger. Use um PIN só seu e
  não o reutilize. O PIN não é armazenado: se esquecer, não há recuperação; apague
  `sdmc:/switch/horizon/auth/` e faça login de novo.
- Este fluxo não usa client secret. Se você criou um em **Certificates & secrets**, apague: ele não é
  necessário e aumenta o risco.
- Nunca use o client ID do launcher oficial ou de outro launcher. Registre o seu e respeite os termos
  da Microsoft e da Mojang.
- Se for pedir ajuda, não cole tokens, o conteúdo de `account.bin` nem o `device.key`. O código do
  device code é temporário, mas evite compartilhá-lo enquanto estiver ativo.
- O arquivo do client ID fica no SD, fora do git. Confira com `git status` que nada disso foi para o
  repositório.

## Checklist rápido

- [ ] App registrado com **Personal Microsoft accounts only**
- [ ] **Allow public client flows** = **Yes**
- [ ] **Application (client) ID** copiado
- [ ] `sdmc:/switch/horizon/msa-client-id.txt` com uma linha, sem aspas
- [ ] Aprovação da Mojang, se necessário
- [ ] PIN definido (opcional)
- [ ] Aba **Conta** mostrando **"Sessão Microsoft ativa"**
