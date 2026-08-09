# DeviceLink

App Android (Kotlin + Jetpack Compose) para trocar **mensagens de texto e arquivos** diretamente
entre dois aparelhos, sem servidor central:

- **Mesma rede Wi-Fi:** descoberta automática dos outros aparelhos (NSD/mDNS).
- **Redes diferentes:** adicione o dispositivo manualmente pelo IP dado por uma VPN como
  **Tailscale** ou **WireGuard** — depois que os dois aparelhos entram na mesma VPN, eles enxergam
  um ao outro como se estivessem na mesma rede local, e o mesmo protocolo funciona sem mudanças.

Não existe nenhum servidor/nuvem envolvido: um aparelho conecta diretamente no outro por TCP.

## Como gerar o APK

Este projeto **não vem compilado** — é o código-fonte completo de um projeto Android Studio.
Escolha uma das duas formas abaixo.

### Opção A — Android Studio (no seu computador)

1. Instale o [Android Studio](https://developer.android.com/studio) (versão recente).
2. Abra a pasta `DeviceLink` (File → Open).
3. Deixe o Android Studio sincronizar o Gradle (ele baixa o `gradle-wrapper.jar` automaticamente
   na primeira sincronização, já que este projeto não inclui o binário do wrapper).
4. Build → Build Bundle(s)/APK(s) → Build APK(s). O APK fica em
   `app/build/outputs/apk/debug/app-debug.apk`.
5. Alternativa por linha de comando (se tiver o Gradle instalado): `gradle wrapper && ./gradlew assembleDebug`.

### Opção B — GitHub Actions (sem instalar nada, gera o .apk pronto pra baixar)

O projeto já inclui `.github/workflows/build-apk.yml`, que compila o APK automaticamente na nuvem:

1. Crie um repositório novo (pode ser privado) em github.com e suba esta pasta nele
   (`git init && git add . && git commit -m "DeviceLink" && git push`, ou pelo botão
   "Add file → Upload files" no site do GitHub).
2. Vá na aba **Actions** do repositório — o build começa sozinho (ou clique em "Run workflow").
3. Quando terminar (ícone verde), abra a execução e baixe o artefato **DeviceLink-debug-apk**
   em "Artifacts" — é um .zip contendo o `app-debug.apk`.
4. Transfira o .apk pro celular e instale (é preciso permitir "instalar apps de fontes
   desconhecidas" para o app usado na transferência, ex: Arquivos, navegador, etc).

Instale o APK em pelo menos dois aparelhos para testar a troca de mensagens.

## Como testar

**Mesma Wi-Fi:** abra o app nos dois aparelhos. Cada um se anuncia automaticamente na rede; o
outro aparece na seção "Encontrados na rede local" — toque em "Adicionar" e comece a conversar.

**Via VPN (Tailscale/WireGuard), redes diferentes:**
1. Instale e conecte o Tailscale (ou WireGuard) nos dois aparelhos, na mesma conta/rede.
2. Veja o IP que a VPN deu a cada aparelho (no app do Tailscale, ou em
   `login.tailscale.com/admin/machines`) — algo como `100.101.102.103`.
3. No DeviceLink, toque no "+" → "Adicionar dispositivo" → informe um nome e esse IP → Salvar.
4. Pronto: as mensagens/arquivos passam pelo túnel da VPN, não pela internet aberta.

O app roda um serviço em primeiro plano (notificação fixa) para conseguir **receber** mensagens
mesmo com a tela apagada ou o app em segundo plano — é assim que o outro lado consegue te alcançar
a qualquer momento.

## Arquitetura (resumo)

- **Protocolo:** TCP simples na porta `58217`. Cada mensagem é um cabeçalho JSON com prefixo de
  tamanho; para arquivos, os bytes crus vêm logo depois do cabeçalho.
- **Recebimento:** `ConnectionService` (foreground service) mantém um `ServerSocket` sempre aberto.
- **Envio:** `PeerClient` abre uma conexão TCP nova para o IP:porta salvo e escreve a mensagem.
- **Descoberta local:** `NsdHelper` usa a API `NsdManager` do Android (mDNS) — só funciona dentro
  do mesmo domínio de broadcast (mesma Wi-Fi), por isso não alcança dispositivos atrás de VPN.
- **Persistência:** Room (SQLite) guarda dispositivos salvos e histórico de mensagens.
- **Arquivos recebidos** ficam em `Android/data/com.devicelink.app/files/received/` (área privada
  do app, sem precisar de permissão de armazenamento).

## Limitações conhecidas / próximos passos

- Sem fila de reenvio automática: se o envio falhar, a mensagem fica marcada como "falhou" e pode
  ser reenviada tocando nela; não há retomada automática quando o dispositivo volta a ficar online.
- Sem criptografia própria — dentro da mesma Wi-Fi os dados trafegam em texto simples; usando
  VPN (Tailscale/WireGuard) o túnel já vem criptografado ponto a ponto.
- Sem notificação por mensagem individual (só a notificação fixa do serviço); dá para adicionar
  depois usando `NotificationCompat` a cada mensagem recebida.
- Sem retomada de transferência de arquivos grandes interrompidas no meio.
- A rede precisa permitir a porta 58217 (algumas redes públicas/corporativas bloqueiam conexões
  ponto a ponto entre aparelhos).

## Sobre este projeto

Este código foi gerado em um ambiente sem SDK do Android nem acesso à internet, então **não foi
compilado nem testado em um dispositivo real** — revise antes de confiar nele para uso crítico, e
rode o `assembleDebug` para pegar qualquer erro de compilação que a revisão manual não tenha visto.
