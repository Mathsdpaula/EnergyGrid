// Bibliotecas
#include <Arduino.h>
#include <Wire.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>    
#include <DHT.h>  // Sensor DHT11

// Definições de Wi-Fi
#define SSID "Nome da rede"
#define SSID_PASSWORD "senha da rede"

// Bot Telegram
const char* botToken = "7958964880:AAHbYzyYyUcr4ViyY967w7bmoh3OLVbFlqg";

// Cliente e bot
WiFiClientSecure client;
UniversalTelegramBot bot(botToken, client);
String chatId = ""; // Preenchido automaticamente

// Lista de usuários autorizados
const int NUM_USERS = 2; // Quantos IDs autorizados
String allowedChatIds[NUM_USERS] = {
  "6962292470", // substitua pelos seus chat_ids reais
  "1928001830"
};

// Tempo entre verificações do bot
unsigned long lastTimeBotRan;
const unsigned long BOT_MTBS = 1000; // 1 segundo

// Pinos de saída
#define LuzFrontal 21
#define LuzFundo 19
#define LuzPrincipal 18


// Sensor DHT11
#define DHTPIN 13
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// Verifica se chat_id está autorizado
bool isAuthorized(String chat_id) {
  for (int i = 0; i < NUM_USERS; i++) {
    if (chat_id == allowedChatIds[i]) {
      return true;
    }
  }
  return false;
}

// Função de tratamento de mensagens do Telegram
void handleNewMessages(int numNewMessages) {
  for (int i = 0; i < numNewMessages; i++) {
    String chat_id = bot.messages[i].chat_id;
    String text = bot.messages[i].text;
    String from_name = bot.messages[i].from_name;

    Serial.println("Mensagem de: " + from_name + ": " + text);
    Serial.println("Chat ID: " + chat_id);

    if (!isAuthorized(chat_id)) {
      bot.sendMessage(chat_id, "⛔ Acesso negado. Você não está autorizado.", "");
      continue;
    }

    // Comandos de controle
    if (text == "/LuzFrontal_on") {
      digitalWrite(LuzFrontal, HIGH);
      bot.sendMessage(chat_id, "🌀 LuzFrontal LIGADO", "");
    } else if (text == "/LuzFrontal_off") {
      digitalWrite(LuzFrontal, LOW);
      bot.sendMessage(chat_id, "🛑 LuzFrontal DESLIGADO", "");

    } else if (text == "/LuzFundo_on") {
      digitalWrite(LuzFundo, HIGH);
      bot.sendMessage(chat_id, "🌀 LuzFundo LIGADO", "");
    } else if (text == "/LuzFundo_off") {
      digitalWrite(LuzFundo, LOW);
      bot.sendMessage(chat_id, "🛑 LuzFundo DESLIGADO", "");

    } else if (text == "/LuzPrincipal_on") {
      digitalWrite(LuzPrincipal, HIGH);
      bot.sendMessage(chat_id, "🌀 LuzPrincipal LIGADO", "");
    } else if (text == "/LuzPrincipal_off") {
      digitalWrite(LuzPrincipal, LOW);
      bot.sendMessage(chat_id, "🛑 LuzPrincipal DESLIGADO", "");

    } else if (text == "/status") {
      float temperatura = dht.readTemperature();
      float umidade = dht.readHumidity();

      if (isnan(temperatura) || isnan(umidade)) {
        bot.sendMessage(chat_id, "❌ Falha ao ler o sensor DHT11", "");
      } else {
        String mensagem = "🌡 Temperatura: " + String(temperatura, 1) + " °C\n";
        mensagem += "💧 Umidade: " + String(umidade, 1) + " %";
        bot.sendMessage(chat_id, mensagem, "");
      }

    } else {
      String welcome = "Olá, " + from_name + "!\n\n";
      welcome += "🔧 Comandos disponíveis:\n";
      welcome += "/LuzFrontal_on /LuzFrontal_off\n";
      welcome += "/LuzFundo_on /LuzFundo_off\n";
      welcome += "/LuzPrincipal_on /LuzPrincipal_off\n";

      welcome += "/status (ver temperatura e umidade)\n";
      bot.sendMessage(chat_id, welcome, "");
    }
  }
}

void setup() {
  Serial.begin(115200);

  // Inicializa Wi-Fi
  WiFi.begin(SSID, SSID_PASSWORD);
  Serial.print("Conectando-se ao Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConectado ao Wi-Fi!");

  // Inicializa conexão segura
  client.setInsecure();

  // Mensagem inicial (para debug, pode ser removido se chatId estiver vazio)
  bot.sendMessage(chatId, "🤖 ESP32 conectado ao Telegram!", "");

  // Define pinos como saída
  pinMode(LuzFrontal, OUTPUT);
  pinMode(LuzFundo, OUTPUT);
  pinMode(LuzPrincipal, OUTPUT);

  // Inicializa sensor DHT11
  dht.begin();
}

void loop() {
  // Verifica novas mensagens
  if (millis() - lastTimeBotRan > BOT_MTBS) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    while (numNewMessages) {
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }
    lastTimeBotRan = millis();
  }

  // Leitura no monitor serial
  float umidade = dht.readHumidity();
  float temperatura = dht.readTemperature();

  if (!isnan(umidade) && !isnan(temperatura)) {
    Serial.print("Umidade: ");
    Serial.print(umidade);
    Serial.print(" %\t");
    Serial.print("Temperatura: ");
    Serial.print(temperatura);
    Serial.println(" °C");
  } else {
    Serial.println("Falha ao ler os dados do sensor DHT!");
  }
}
