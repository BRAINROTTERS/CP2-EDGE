# CP2-EDGE
Grupo- Guiherme Amaral,Enrico Bagli,Rafael Moraes,Matheus Antunez,Joao Cazzarine.

 Projeto-Vinharia-Agnello

 Descricao do projeto: Nosso projeto consiste em um sistema embarcado que tem como objetivo analisar a temperatura do ambiente,luminosidade e umidade contendo informacoes e com a proposta de mostrar informacoes para o usuario de forma facil e objetiva,
 servindo principalmente para o monitoramento dos vinhos e alertar caso haja alguma deturpacao avisando o usuario sobre o problema vindo da luminosidade ou umidade o temperatura.

 Dependencias (hardware):
 fios-18x
 resistores-3x 1k onw
 arduino uno-1x
 leds-3x
 buzzer-1x
 sensor dht11-1x
 fotoressistor-1x
 rtc-1x
 Depedencias (software):
 LiquidCrystal I2C
RTClib
DHT sensor library
Como reproduzilo:
Estrtura e ligacoes:https://wokwi.com/projects/431427074556541953

Codigo:
// Inclusão das bibliotecas necessárias
#include <Wire.h>              // Comunicação I2C
#include <LiquidCrystal_I2C.h> // Controle do display LCD via I2C
#include <RTClib.h>            // Relógio em tempo real (RTC)
#include <DHT.h>               // Sensor de temperatura e umidade DHT
#include <EEPROM.h>            // Memória EEPROM (não utilizada neste trecho)

#define SERIAL_OPTION 0        // Ativa/desativa o uso da porta serial (0 = desligado, 1 = ligado)
#define UTC_OFFSET -3          // Fuso horário UTC-3

RTC_DS1307 RTC;                // Objeto do relógio em tempo real

// ----- Pinos e Constantes -----
#define LDR_PIN A0             // Sensor de luz (LDR)
#define LED_VERDE 2
#define LED_AMARELO 3
#define LED_VERMELHO 4
#define BUZZER_PIN 8           // Alarme sonoro (buzzer)
#define DHTPIN 7               // Pino do sensor DHT11
#define DHTTYPE DHT11          // Tipo do sensor
#define NUM_AMOSTRAS 10        // Número de leituras para calcular médias
#define POTENCIOMETRO_PIN A1   // Potenciômetro para ajuste
#define BACKLIGHT_PIN 9        // PWM para brilho do LCD

// Tempos de exibição (em milissegundos)
#define TEMPO_TELA 3000
#define TEMPO_UNIDADE 6000
#define TEMPO_IDIOMA 9000

// ----- Caracteres personalizados para o LCD -----

// Ícones de temperatura e umidade (divididos em partes para visualização gráfica)
byte tempchar1[8] = {B00000, B00001, B00010, B00100, B00100, B00100, B00100, B00111};
byte tempchar2[8] = {B00111, B00111, B00111, B01111, B11111, B11111, B01111, B00011};
byte tempchar3[8] = {B00000, B10000, B01011, B00100, B00111, B00100, B00111, B11100};
byte tempchar4[8] = {B11111, B11100, B11100, B11110, B11111, B11111, B11110, B11000};

byte humchar1[8] = {B00000, B00001, B00011, B00011, B00111, B01111, B01111, B11111};
byte humchar2[8] = {B11111, B11111, B11111, B01111, B00011, B00000, B00000, B00000};
byte humchar3[8] = {B00000, B10000, B11000, B11000, B11100, B11110, B11110, B11111};
byte humchar4[8] = {B11111, B11111, B11111, B11110, B11100, B00000, B00000, B00000};

// ----- Objetos e Variáveis Globais -----
LiquidCrystal_I2C lcd(0x27, 16, 2); // LCD 16x2 no endereço I2C 0x27
DHT dht(DHTPIN, DHTTYPE);           // Objeto DHT

// Variáveis para armazenar dados
int luminosidade_mapeada = 0;
float temperatura_media = 0.0;
float umidade_media = 0.0;

// Configurações de exibição
bool idiomaPortugues = true;  // Alternância entre PT/EN
bool usarCelsius = true;      // Alternância entre C/F

// Controle de ciclos e tempo
unsigned long ultimaAtualizacao = 0;
byte cicloAtual = 0;

// ---------- SETUP ----------
void setup() {
  lcd.init();       // Inicializa o LCD
  lcd.backlight();  // Liga luz de fundo

  if (SERIAL_OPTION) Serial.begin(9600); // Inicia comunicação serial se ativada

  RTC.begin();      // Inicia o relógio em tempo real
  RTC.adjust(DateTime(F(__DATE__), F(__TIME__))); // Sincroniza com a hora da compilação

  // Configura os pinos como entrada ou saída
  pinMode(POTENCIOMETRO_PIN, INPUT);
  pinMode(BACKLIGHT_PIN, OUTPUT);
  pinMode(LED_VERDE, OUTPUT);
  pinMode(LED_AMARELO, OUTPUT);
  pinMode(LED_VERMELHO, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Mensagens de boas-vindas
  lcd.setCursor(0, 0);
  lcd.print("Iniciando...");
  delay(3000);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Seja Bem vindo!");
  lcd.setCursor(0, 1);
  lcd.print("BrainRotters");
  delay(3000);
  lcd.clear();

  dht.begin();  // Inicia o sensor DHT
}

// ---------- FUNÇÃO: Ajustar contraste ----------
void ajustarContraste() {
  int valorPot = analogRead(POTENCIOMETRO_PIN);
  int contraste = map(valorPot, 0, 1023, 0, 100);
  lcd.setContrast(contraste); // Requer LCD com suporte a ajuste de contraste
}

// ---------- FUNÇÃO: Ajustar brilho do LCD ----------
void ajustarBrilho() {
  int valorPot = analogRead(POTENCIOMETRO_PIN);
  int brilho = map(valorPot, 0, 1023, 0, 255);
  analogWrite(BACKLIGHT_PIN, brilho);
}

// ---------- LOOP PRINCIPAL ----------
void loop() {
  DateTime now = RTC.now();

  // Ajuste de fuso horário manual
  int offsetSeconds = UTC_OFFSET * 3600;
  now = now.unixtime() + offsetSeconds;
  DateTime adjustedTime = DateTime(now);

  if (SERIAL_OPTION) {
    // Código opcional para comunicação serial
  }

  ajustarContraste(); // Ajusta contraste dinamicamente

  unsigned long agora = millis();
  if (agora - ultimaAtualizacao >= TEMPO_TELA) {
    ultimaAtualizacao = agora;
    cicloAtual++;

    // Alterna entre as telas de exibição (0, 1, 2)
    byte telaAtual = cicloAtual % 3;

    // Alterna o idioma a cada 9 segundos (3 ciclos)
    if (cicloAtual % (TEMPO_IDIOMA / TEMPO_TELA) == 0) {
      idiomaPortugues = !idiomaPortugues;
    }

    lcd.clear();
    atualizarTela(telaAtual);
  }

  atualizarSensores();       // Atualiza os sensores
  controlarLedsEBuzzer();    // Controla os LEDs e o buzzer

  delay(100); // Pequeno atraso para estabilidade
}

// ---------- FUNÇÃO: Atualizar sensores ----------
void atualizarSensores() {
  int valorLDR = analogRead(LDR_PIN);
  luminosidade_mapeada = map(valorLDR, 0, 1023, 0, 100);

  float soma_temp = 0.0;
  float soma_umid = 0.0;
  for (int i = 0; i < NUM_AMOSTRAS; i++) {
    soma_temp += dht.readTemperature();
    soma_umid += dht.readHumidity();
    delay(50);
  }
  temperatura_media = soma_temp / NUM_AMOSTRAS;
  umidade_media = soma_umid / NUM_AMOSTRAS;
}

// ---------- FUNÇÃO: Atualizar tela ----------
void atualizarTela(byte tela) {
  switch (tela) {
    case 0: // Tela de luminosidade
      if (idiomaPortugues) {
        lcd.setCursor(0, 0);
        lcd.print("O - Nivel de Luz:");
        lcd.setCursor(0, 1);
        lcd.print(luminosidade_mapeada);
        lcd.print("% - ");
        lcd.print(luminosidade_mapeada >= 70 ? "Muito alto!" :
                  luminosidade_mapeada >= 30 ? "Alerta" : "Bom");
      } else {
        lcd.setCursor(0, 0);
        lcd.print(" O - Light Level:");
        lcd.setCursor(0, 1);
        lcd.print(luminosidade_mapeada);
        lcd.print("% - ");
        lcd.print(luminosidade_mapeada >= 70 ? "Very High" :
                  luminosidade_mapeada >= 30 ? "Alert" : "Great");
      }
      break;

    case 1: // Tela de temperatura
      {
        bool exibirCelsius = idiomaPortugues;
        float tempExibida = exibirCelsius ? temperatura_media : (temperatura_media * 1.8 + 32);

        // Criação dos ícones
        lcd.createChar(1, tempchar1);
        lcd.createChar(2, tempchar2);
        lcd.createChar(3, tempchar3);
        lcd.createChar(4, tempchar4);

        // Exibição gráfica do ícone
        lcd.setCursor(0, 0


