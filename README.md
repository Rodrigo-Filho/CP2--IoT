# Projeto IoT com ESP32, DHT22, LCD e Node-RED

Este projeto utiliza um **ESP32** conectado a um sensor de umidade e temperatura **DHT22**, além de um display **LCD 16x2** para exibir os valores. Os dados são enviados para o broker MQTT **test.mosquitto.org** e processados por uma aplicação local **Node-RED**. A aplicação Node-RED concatena os dados de temperatura e umidade e os reenviam para o ESP32 que exibe os dados.

## Visão Geral

### Objetivo
O objetivo deste projeto é demonstrar a comunicação entre um ESP32 que coleta dados de um sensor de umidade e temperatura **DHT22** e uma aplicação **Node-RED** que processa e reenvia esses dados. O **Node-RED** é responsável por receber os dados de umidade e temperatura via MQTT, processá-los e enviá-los de volta para um ESP32 que pode realizar ações com base nos dados recebidos.

## Arquitetura

![Arquitetura feita no Lucidchart](img/Blank%20diagram.jpeg)


1. **ESP32 (DHT22 e LCD)**: Coleta dados de temperatura e umidade e envia via MQTT para o broker **test.mosquitto.org**.
2. **Node-RED**: Conecta-se ao broker MQTT, recebe os dados, concatena as informações e reenviá-las.
3. **Broker MQTT**: O broker **test.mosquitto.org** é utilizado para intermediar a comunicação entre o ESP32 e o Node-RED.

## Configuração do Node-RED

1. **Adicione dois nós MQTT In**:
   - **Tópico**: `esp32/temperatura` e `esp32/umidade`
   - **Broker**: `test.mosquitto.org`

2. **Adicione um nó Function**:
   - **Função**: Concatenar os dados de temperatura e umidade, por exemplo:
     ```js
     var temperatura = msg.payload;
     var umidade = msg.payload; // Supondo que os dados venham como um array ou objeto
     return { payload: "Temp: " + temperatura + "C, Umid: " + umidade + "%" };
     ```

3. **Adicione um nó MQTT Out**:
   - **Tópico**: `esp32/recebe_dados`
   - **Broker**: `test.mosquitto.org`

## Código do ESP32 (DHT22, LCD, e MQTT)

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include "DHTesp.h"
#include <LiquidCrystal_I2C.h>

#define I2C_ADDR    0x27
#define LCD_COLUMNS 16
#define LCD_LINES   2
#define DHT_PIN 15

// Atualize essas variáveis com as configurações corretas da sua rede WiFi e broker MQTT
const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "test.mosquitto.org";

DHTesp dhtSensor;
LiquidCrystal_I2C lcd(I2C_ADDR, LCD_COLUMNS, LCD_LINES);

WiFiClient Wokwi_Client;
PubSubClient client(Wokwi_Client);

#define MSG_BUFFER_SIZE 50  // Definindo o tamanho do buffer para as mensagens
char msg[MSG_BUFFER_SIZE];  // Buffer para armazenar mensagens

// Função para conectar ao WiFi
void setup_wifi() {
  delay(10);
  Serial.println();
  Serial.print("Conectando a ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);  // Corrigido WiFi com maiúsculas
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi conectado");
  Serial.println("Endereço IP: ");
  Serial.println(WiFi.localIP());
}

// Função para reconectar ao broker MQTT
void reconnect() {
  while (!client.connected()) {
    Serial.print("Tentando conexão MQTT...");

    if (client.connect("Wokwi_Client")) {
      Serial.println("conectado");
    } else {
      Serial.print("falhou, rc=");
      Serial.print(client.state());
      Serial.println(" tentando novamente em 5 segundos");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  
  // Inicializando WiFi, MQTT e LCD
  setup_wifi();
  client.setServer(mqtt_server, 1883);

  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  lcd.init();
  lcd.backlight();
}

void loop() {
  reconnect();  // Garante a reconexão ao MQTT

  // Leitura de temperatura e umidade
  TempAndHumidity data = dhtSensor.getTempAndHumidity();
  Serial.println(String(data.temperature, 1));
  Serial.println(String(data.humidity, 1));
  Serial.println("---");
  
  // Atualizando o display LCD
  lcd.setCursor(0, 0);
  lcd.print("  Temp: " + String(data.temperature, 1) + "\xDF" + "C  ");
  lcd.setCursor(0, 1);
  lcd.print(" Humidity: " + String(data.humidity, 1) + "% ");

  // Publicando dados no broker MQTT
  snprintf(msg, MSG_BUFFER_SIZE, "%.2f", data.temperature);
  client.publish("esp32/temperatura", msg);
  snprintf(msg, MSG_BUFFER_SIZE, "%.2f", data.humidity);
  client.publish("esp32/umidade", msg);

  delay(2000);  // Delay para próxima leitura e publicação
}
````

## Como Testar
1. Conecte o ESP32 à rede Wokwi-GUEST.
2. Suba o código no ESP32 e certifique-se de que os dados de temperatura e umidade estão sendo enviados para o broker MQTT.
3. Verifique o Node-RED para garantir que ele está recebendo e reenviando os dados para o tópico correto.
4. Se houver uma segunda ESP32 (ou outro dispositivo), ela pode assinar o tópico esp32/recebe_dados e agir com base nas informações recebidas.
