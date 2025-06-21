# Proyecto-Mezclador-E1 (ESP32-LED-DHT22-HCSR04-NODERED-WSP) 

# Introducción:
Se desea realizar un sistema para relizar un proceso de mezcla de los siguientes elementos: a)[INSERTAR EL PRIMER ELEMENTO] y b)[INSERTAR EL SEGUNDO ELEMENTO], esto sucede en etapas las cuales son:

1. Llenado de tanque (LLENANDO)
2. Calentamiento y mezcla (MEZCLANDO)
3. Vaciado de tanque (VACIANDO)

Adicionalmente se agregaran las siguientes caracteristicas:

- Descripcion de las caracteristicas de la mezcla dentro de una interfaz grafica (DASHBOARD)
  
- Historial de los cambios dentro del tanque dentro de una base de datos (MySQL)

- Notificacion de la fase en la que se encuentra el proceso a traves de un mensajero en linea (TELEGRAM)
   

# Material necesario

- WOKWI (para simular las variables del proceso)
- Una tarjeta ESP32 (para adquisicion de datos)
- Sensor ultrasonico "HC-SR04 Ultrasonic Distance Sensor" 
- Plataforma Node-RED conectada a un broker
- Entorno funcional dentro de Node-RED
- XAMPP Control panel (para Apache y MySQL)

# Desarrollo de la proyecto 

## 1 Entorno simulado 

## 1.1 Codigo para el entorno simulado en WOKWI 

  ```

#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int DHT_PIN = 13;
DHTesp dhtSensor;
// Update these with values suitable for your network.
const int Trigger = 12;   //Pin digital 2 para el Trigger del sensor
const int Echo = 14;
const char* ssid = "Wokwi-GUEST"; //red wifi
const char* password = ""; //contraseña de la red wifi
const char* mqtt_server = "52.28.105.152"; //servidos mqtt adresses
String username_mqtt="DiploIC";
String password_mqtt="12345";

const int Bomba1 = 17;
const int Bomba2 = 16;
const int Calentador = 4;
const int Mezclador = 0;
const int Bomba3 = 15;
String mensaje;

int safetyDistance;

enum Estado { LLENADO, MEZCLA, VACIADO };
Estado estado = LLENADO;

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {

  pinMode(Bomba1, OUTPUT);
  pinMode(Bomba2, OUTPUT);
  pinMode(Calentador, OUTPUT);
  pinMode(Mezclador, OUTPUT);
  pinMode(Bomba3, OUTPUT);

  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0
}

void loop() {
 
 
 
 long t; //timepo que demora en llegar el eco
  long d; //distancia en centimetros

  digitalWrite(Trigger, HIGH);
  delayMicroseconds(10);          //Enviamos un pulso de 10us
  digitalWrite(Trigger, LOW);
  
  t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
  d = t/59;             //escalamos el tiempo a una distancia en cm
  delay(1000);

  TempAndHumidity  data = dhtSensor.getTempAndHumidity();
  safetyDistance = d;
  


  switch (estado) {
    case LLENADO:
      digitalWrite(Bomba1, HIGH);
      digitalWrite(Bomba2, HIGH);
      mensaje = ("LLENANDO");
      if (safetyDistance<= 100) { // Nivel alto alcanzado
        digitalWrite(Bomba1, LOW);
        digitalWrite(Bomba2, LOW);
        estado = MEZCLA;
      }
    break;

    case MEZCLA:
      digitalWrite(Mezclador, HIGH);
      digitalWrite(Calentador, HIGH);
      mensaje = ("MEZCLANDO");
      if (data.temperature > 60) { // Temperatura objetivo
        digitalWrite(Mezclador, LOW);
        digitalWrite(Calentador, LOW);
        estado = VACIADO;
      }
    break;

    case VACIADO:
      digitalWrite(Bomba3, HIGH);
      mensaje = ("VACIANDO");
      if (safetyDistance == 398) { // Tanque vacío
        digitalWrite(Bomba3, LOW);
        estado = LLENADO;
      }
    break;
    
    delay(500);
  }


  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    doc["EQUIPO1"] = "EQUIPO 1";
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["DISTANCIA"] = String(d);
    doc["ESTADO"] = String(mensaje);

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("DiplomadoE1", output.c_str());
  }

}

  ```

## 1.2 Librerias empleadas para el entorno simulado en WOKWI

Instalar las siguientes librerias

- DHT sensor library for ESPx
- ArduinoJson
- WiFi
- PubSubClient

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/LIBRERIA.png?raw=true)

## 1.3 Hacer las conexiones entre los elementos del entorno simulado del siguiente modo como se muestra en la imagen

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/CONEXIONES%20WOKWI.png?raw=true)

## 1.4 Intrucciones de operación

1. Iniciar la simulación.
2. Observar el envio de los datos a traves del monitor serial.
3. Variar los valores de la temperatura y distancia para observar que se actualizan adecuadamente.
4. Comprobar que las fases del proceso operen adecuadamente.
   - Llenando
   - Mezlando
   - Vaciando

# 2 Node-RED

## 2.1 Dentro del Node-RED ya funcionando se agregara la siguiente libreria  (EDITANDO)

- node-red-dashboard

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/ADD%20DASHBOARD.png?raw=true)
  
- node-red-node-mysql

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/ADD%20MYSQL.png?raw=true)

- node-red-contrib-telegrambot

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/ADD%20TELEGRAM.png?raw=true)

## 2.2 Integrar en la interfaz los siguientes elementos

- mqtt in (para hacer de entrada de dats)
- json (para comunicacion entre aplicaciones web y servidores)
- debug 1 y 2 (para observar la informacion que se esta recibiendo)
- function 1 (se empleara para la temperatura)
- function 2 (se empleara para la distancia)
- function 3 (se empleara para enviar informacion a la base de datos)
- function 4 (se empleara para enviar mensaje a telegram)
- mysql (para guardar los eventos y las variables de estado)
- telegram sender (para enviar mensaje al char de Telegram)
- chart 1 (se empleara para poder observar en graficos los cambios de las variables)
- gauge 1 (temperatura)
- gauge 2 (distancia)
- text (para almacenar la variable estado)

## 2.3 Conexiones y codigo interno de las funciones Node-RED

Se interconectaran los elementos tal cual se muestra en la siguiente imagen

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/CONEXIONES%20NODE-RED.png?raw=true)

- mqtt in (para hacer de entrada de dats)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/ADD%20MQTT.png?raw=true)

- json (para comunicacion entre aplicaciones web y servidores)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/JSON.png?raw=true)

- debug 1 y 2 (para observar la informacion que se esta recibiendo)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/DEBUG%201%20Y%202.png?raw=true)

- function 1 (se empleara para la temperatura)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/TEMPERATURA.png?raw=true)

  ```
msg.payload = msg.payload.TEMPERATURA;
msg.topic = "TEMPERATURA";
return msg;
  ```
  
- function 2 (se empleara para la distancia)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/DISTANCIA.png?raw=true)

  ```
msg.payload = msg.payload.DISTANCIA;
msg.topic = "DISTANCIA";
return msg;
  ```
  
- function 3 (se empleara para el estado)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/ESTADO.png?raw=true)

  ```
msg.payload = msg.payload.ESTADO;
msg.topic = "ESTADO";
return msg;
  ```
  
- function 4 (se empleara para enviar informacion a la base de datos)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/BASE%20DE%20DATOS.png?raw=true)

  ```
var query = "INSERT INTO `proyecto1`(`ID`, `FECHA`, `DEVICE`, `TEMPERATURA`, `DISTANCIA`) VALUES (NULL, current_timestamp(), '";
query += msg.payload.DEVICE + "', '";
query += msg.payload.TEMPERATURA + "', '";
query += msg.payload.DISTANCIA + "');";
msg.topic = query;
return msg;

  ```

- function 5 (se empleara para enviar mensaje a telegram)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/EXT%20PARA%20TELEGRAM.png?raw=true)

  ```
msg.payload = {
    chatId: 5276504889,
    type: "message",
    content: msg.payload.ESTADO
};
return msg;
  ```
  
- mysql (para guardar los eventos y las variables de estado)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/MQTT.png?raw=true)

- telegram sender (para enviar mensaje al char de Telegram)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/SENDER%20TELEGRAM.png?raw=true)

- chart 1 (se empleara para poder observar en graficos los cambios de las variables)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/GRAFICOS.png?raw=true)

- gauge 1 (temperatura)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/TEMPERATURA.png?raw=true)

- gauge 2 (distancia)

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/DISTANCIA.png?raw=true)

- text (para almacenar la variable estado)
  
![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/TEXT%20ESTADO.png?raw=true)


# 3 Base de datos

## 3.1 Creacion de la base de datos:

Con la herramienta de XAMPP Control data base ya instalada en el equipo:

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/XAMPP.png?raw=true)

Iniciar los modulos (se muestran en la siguiente imagen):

- Apache
- MySQL

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/START%20APACHE%20Y%20MYSQL.png?raw=true)

Abrir el apartado "Admin" (se muestra en la siguiente imagen):

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/ABRIR%20ADMIN.png?raw=true)

Crear base nueva base de datos (la siguiente imagen es un ejemplo):

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/CREAR%20BASE%20DE%20DATOS%201.png?raw=true)

Crear Tabla para guardar los datos con las siguientes caracteristicas:

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/CREAR%20BASE%20DE%20DATOS%202.png?raw=true)

Configurar del siguiente modo:

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/CREAR%20BASE%20DE%20DATOS%203.png?raw=true)

# Resultados:



## Dashboard:

- Llenando
![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/LLENANDO%201.png?raw=true)

- Mezclando
![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/MEZCLANDO%201.png?raw=true)

- Vaciando
![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/VACIANDO%201.png?raw=true)



## Alertas:

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/MENSAJES%20DE%20TELEGRAM.png?raw=true)

## Registro de base de datos:

![](https://github.com/HV202506/Proyecto-Mezclador-E1-ESP32-LED-DHT22-HCSR04-NODERED-WSP-EDITANDO-/blob/main/RESULTADOS%20BASE%20DE%20DATOS.png?raw=true)


# Créditos

Desarrollado por: 

- Ing. Vega Rodríguez Héctor Angel

- Ing. Salas Chávez Luis Adrián

- Ing. Cardoso Mota Irving Daniel 

