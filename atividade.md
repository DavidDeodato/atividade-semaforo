# Projeto de Semáforo Inteligente para Smart Cities

## Introdução
Nesta atividade ponderada, nos foi atribuído o desafio de desenvolver um sistema de semáforo inteligente, que simula a visão de uma Smart City onde semáforos não apenas controlam o tráfego, mas também se comunicam entre si para ajustar o fluxo de veículos de forma autônoma e eficiente. O sistema utiliza um sensor LDR (Light Dependent Resistor) para detectar a presença de veículos com base em variações de luminosidade e adotar um comportamento adequado para diferentes condições, incluindo um "modo noturno". Este documento descreve o processo de montagem, programação e integração do semáforo com a plataforma Ubidots, oferecendo uma visão prática do conceito de cidades inteligentes.

## Estrutura do Projeto
- **Parte 1: Montagem Física e Programação do LDR**  
  Nesta etapa, foram montados dois semáforos conectados ao sensor LDR, que detecta a passagem de veículos simulados através da variação de luz. O sistema foi configurado para mudar automaticamente para o modo noturno quando a luminosidade cai abaixo de um certo limite, ajustando os LEDs de acordo com as condições.
  
- **Parte 2: Configuração da Interface Online**  
  A interface foi configurada para possibilitar o ajuste remoto do semáforo e visualização das leituras do sensor LDR. Utilizamos o Ubidots para centralizar e monitorar os dados em tempo real, o que ajuda na gestão inteligente do fluxo de tráfego.

- **Extra: Implementação com ESP32 e Ubidots**  
  Cada semáforo foi conectado a um ESP32, permitindo a comunicação com o dashboard do Ubidots. Isso facilita o gerenciamento e controle remoto dos semáforos, embora a interface final ainda necessite de melhorias para torná-la mais responsiva.

---

## Documentação do Protótipo

### Montagem e Programação
<div align="center">
<sub>Figura 1 - atividade </sub><br>
<img src="assets/atividade_semafaro.jpeg" width="80%" ><br>
<sup>Fonte: Material produzido pelos autores (2024)</sup>

[Assista ao vídeo demonstrativo](https://youtube.com/shorts/dx9qKVgZ_Qc?feature=share)




### Explicação Geral
Utilizamos um sensor LDR para detectar a intensidade de luz, permitindo que o sistema mude para um modo noturno quando a luminosidade está abaixo do limite pré-definido. Para os LEDs do semáforo, configuramos um ciclo de alternância entre vermelho, amarelo e verde, com intervalos específicos para simular o controle de tráfego. A integração com o Ubidots permite monitorar remotamente o modo do semáforo e os valores capturados pelo sensor, além de ativar o modo noturno manualmente, se necessário.

### Código-Fonte e Explicação

#### Código
```cpp
#include <WiFi.h>
#include <UbidotsEsp32Mqtt.h>

#define LDR_PIN 34          // Pino do LDR no ESP32
#define RED_PIN 27          // LED vermelho
#define YELLOW_PIN 32       // LED amarelo
#define GREEN_PIN 33        // LED verde
#define THRESHOLD 500       // Limite de luz para o modo noturno
#define TOKEN "BBUS-veCoBVrAsiykWhV0az7GqtT7AnbTQx"  // Token do Ubidots
#define DEVICE_LABEL "esp32_t12_g01"    // Identificador do dispositivo no Ubidots
#define VARIABLE_LDR "luminosidade"  // Variável de luminosidade no Ubidots
#define VARIABLE_MODE "modo_noturno" // Variável para ativar o modo noturno

// Configuração do WiFi
const char* WIFI_SSID = "POCOX3ProDavid";
const char* WIFI_PASS = "david12345";

Ubidots ubidots(TOKEN);

bool modoNoturno = false;    // Controle do modo noturno baseado no Ubidots
unsigned long previousMillis = 0;
unsigned long interval = 5000; // Tempo inicial para o LED vermelho
int lightState = 0;            // Estado inicial do semáforo

// Função de callback do Ubidots
void callback(char *topic, byte *payload, unsigned int length) {
  String message;
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.print("Mensagem recebida no Ubidots: ");
  Serial.println(message);

  // Verifica se o tópico é o modo_noturno e atualiza o valor de modoNoturno
  if (String(topic).endsWith(VARIABLE_MODE)) {
    modoNoturno = (message == "1");
    Serial.print("Modo noturno atualizado para: ");
    Serial.println(modoNoturno);
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(LDR_PIN, INPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(YELLOW_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);

  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    Serial.println("Conectando ao WiFi...");
    delay(1000);
  }
  Serial.println("Conectado ao WiFi!");

  ubidots.setDebug(true);
  ubidots.setCallback(callback);
  ubidots.setup();

  // Publica um valor inicial para garantir que as variáveis existam no Ubidots
  ubidots.add(VARIABLE_LDR, 0);
  ubidots.add(VARIABLE_MODE, 0);
  ubidots.publish(DEVICE_LABEL);

  // Inscreve-se para receber o último valor do modo noturno
  ubidots.subscribeLastValue(DEVICE_LABEL, VARIABLE_MODE);
  Serial.println("Conexão com o Ubidots configurada e inscrito na variável modo_noturno.");
}

void loop() {
  if (!ubidots.connected()) {
    ubidots.reconnect();
    ubidots.subscribeLastValue(DEVICE_LABEL, VARIABLE_MODE);
  }

  ubidots.loop();

  int ldrValue = analogRead(LDR_PIN);

  // Se o modo noturno estiver ativado via switch no Ubidots ou pelo sensor LDR
  if (modoNoturno || ldrValue < THRESHOLD) {
    Serial.println("Modo Noturno Ativado");
    digitalWrite(RED_PIN, LOW);
    digitalWrite(YELLOW_PIN, HIGH);
    digitalWrite(GREEN_PIN, LOW);
  } else {
    // Controle dos LEDs em modo normal (semáforo)
    unsigned long currentMillis = millis();
    if (currentMillis - previousMillis >= interval) {
      previousMillis = currentMillis;

      switch (lightState) {
        case 0: // Vermelho
          digitalWrite(RED_PIN, HIGH);
          digitalWrite(YELLOW_PIN, LOW);
          digitalWrite(GREEN_PIN, LOW);
          interval = 5000;  // Tempo de espera no vermelho
          lightState = 1;
          break;
        case 1: // Verde
          digitalWrite(RED_PIN, LOW);
          digitalWrite(YELLOW_PIN, LOW);
          digitalWrite(GREEN_PIN, HIGH);
          interval = 5000;  // Tempo de espera no verde
          lightState = 2;
          break;
        case 2: // Amarelo
          digitalWrite(RED_PIN, LOW);
          digitalWrite(YELLOW_PIN, HIGH);
          digitalWrite(GREEN_PIN, LOW);
          interval = 2000;  // Tempo de espera no amarelo
          lightState = 0;
          break;
      }
    }
  }

  Serial.print("LDR Value: ");
  Serial.println(ldrValue);
  Serial.print("Modo Noturno: ");
  Serial.println(modoNoturno);

  ubidots.add(VARIABLE_LDR, ldrValue);
  ubidots.add(VARIABLE_MODE, modoNoturno);
  ubidots.publish(DEVICE_LABEL);

  delay(500); // Reduz o delay para testar a atualização
}
