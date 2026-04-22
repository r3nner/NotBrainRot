# Not BrainRot

## 1. Visão do Projeto
### 1.1 Propósito
Desenvolver um aplicativo Android operando em segundo plano via `AccessibilityService` para impor limites rigorosos ao uso do Instagram (`com.instagram.android`). O sistema bloqueia o consumo passivo de vídeos curtos (Reels) e limita a navegação no feed principal (Home) a 5 minutos por janela de 30 minutos. O sistema utiliza uma abordagem de "ejeção" (fechamento forçado via retorno à Home do SO) para qualquer infração, preservando e priorizando exclusivamente o acesso ao Direct (mensagens).

## 2. Arquitetura do Sistema
* **Monitor de Acessibilidade (Listener):** Escuta a árvore de nós (`AccessibilityNodeInfo`) e eventos de mudança de janela.
* **Máquina de Estados (State Manager):** Gerencia os estados `FORA`, `FEED_HOME`, `DIRECT`, `REELS_DM`. A tentativa de entrar no estado `REELS_MAIN` atua como gatilho de ejeção imediata.
* **Motor de Tempo (Chronos):** *Foreground Service* responsável por gerenciar a cota de milissegundos e controlar a janela de 30 minutos de resfriamento.
* **Interceptador de Inicialização (Launch Manager):** Módulo para rotear a abertura do aplicativo para o Direct via Deep Link caso a cota de tempo da Home esteja esgotada.

---

## 3. Requisitos Funcionais (RF)
* **RF01 - Rastreamento de Pacote:** Identificar quando o aplicativo `com.instagram.android` está em primeiro plano.
* **RF02 - Mapeamento do Direct:** Ler a `AccessibilityNodeInfo` para confirmar a entrada segura na área de mensagens buscando assinaturas visuais exclusivas (Nós Âncora).
* **RF03 - Ejeção por Acesso a Reels:** Ao detectar a renderização da aba principal de Reels, ejetar o usuário imediatamente enviando o comando `GLOBAL_ACTION_HOME`.
* **RF04 - Validação de Reel Autorizado:** Ao abrir um vídeo de dentro do Direct, validar continuamente a presença da interface exclusiva de mensagens (ex: barra inferior "Responder").
* **RF05 - Ejeção por Fuga do Direct (Swipe Up):** Se o Nó Âncora desaparecer durante a exibição de um Reel no Direct (fuga para o algoritmo), aplicar o comando `GLOBAL_ACTION_HOME`.
* **RF06 - Contabilização de Tempo na Home:** Incrementar o cronômetro **apenas** quando o estado for verificado como `FEED_HOME`. O tempo pausa ao entrar no Direct ou sair do app.
* **RF07 - Ejeção por Estouro de Tempo:** Ao atingir 5 minutos (300.000 ms) no estado `FEED_HOME`, fechar o Instagram imediatamente. Tentativas de reabrir na Home com o tempo esgotado resultarão em nova ejeção.
* **RF08 - Inicialização Forçada no Direct:** Prover um atalho ou interceptação via `Intent` com Deep Link (ex: `ig://direct_v2`) para forçar o app a abrir nas mensagens quando a Home estiver bloqueada.
* **RF09 - Reset da Janela de Tempo:** A cota de 5 minutos será restaurada exatamente 30 minutos após o registro do início da sessão original.

---

## 4. Requisitos Não Funcionais (RNF)
* **RNF01 - Permissão de Acessibilidade:** Necessária (`android.permission.BIND_ACCESSIBILITY_SERVICE`) para leitura da tela e injeção do comando de ação global.
* **RNF02 - Foreground Service:** Execução contínua com notificação persistente obrigatória para evitar o encerramento do motor de tempo pelo Android.
* **RNF03 - Otimização de Bateria:** O aplicativo deve operar com isenção do modo *Doze* (Bateria Irrestrita) nas configurações do sistema.

---

## 5. Regras de Negócio (RN)
* **RN01 - Isolamento do Direct:** O tempo dentro da área de mensagens ou assistindo a Reels autorizados na DM é isento e não consome a cota de 5 minutos.
* **RN02 - Persistência do Castigo:** Fechar o Instagram manualmente ou limpá-lo da memória não reseta o tempo consumido. A janela de 30 minutos conta em tempo real de forma independente.
* **RN03 - Gatilhos de Fechamento (Kill Triggers):** Ações que resultam em fechamento instantâneo do Instagram (ejeção):
    1. Atingir 5 minutos de navegação na Home.
    2. Tentar visualizar a Home com a cota zerada.
    3. Tentar abrir o feed principal de Reels.
    4. Rolar a tela (swipe) durante um Reel recebido por DM.

---

## 6. Stack Tecnológica
* **Linguagem:** Java.
* **Ambiente de Desenvolvimento (IDE):** Android Studio.
* **Engenharia Reversa Visual:** Layout Inspector e UI Automator Viewer (ferramentas nativas do Android SDK para mapear os Nós Âncora e IDs do Instagram).
* **Armazenamento de Estado:** `SharedPreferences` para persistência local rápida do *timestamp* da janela de 30 minutos e do tempo acumulado.

---

## 7. Estratégia de Implantação e Ciclo de Vida
* **Distribuição (Sideloading):** Devido às políticas estritas do Google Play referentes à API de Acessibilidade, o aplicativo será compilado em um arquivo `.apk` (Release) e instalado diretamente no dispositivo via fontes desconhecidas.
* **Configuração (Setup Inicial):** Após a instalação, o usuário atuará como administrador do sistema para habilitar manualmente o `AccessibilityService` e conceder permissão de uso de bateria irrestrita.
* **Execução Contínua:** Uma vez configurado, o aplicativo roda de forma autônoma (sem interface gráfica principal ativa). O *Foreground Service* mantém a vigilância em segundo plano, acionando a Máquina de Estados unicamente quando o pacote `com.instagram.android` entra em foco.
