# NODE-RED CON DOS SENSORES
## Practica con un sensor de temperatura y humedad como un sensor ultrasonico, en una placa de desarrollo ESP32 y comunicado en Node-RED. 
Este repositorio muestra como podemos programar una **ESP32** con los sensores **DHT22** **HC-SR04** y mostrar los datos optenidos en **Node-RED**.

### Introducción
**Descripción:** 
La **ESP32** la utilizamos en un entorno de adquision de datos, en esta practica ocuparemos un **sensor ultrasonico** para medir la distancia, como tambien un **sensor de temperatura y humedad** los datos optenidos seran visualizados en forma de grafica como indicador en Node-RED.

### Material Necesario
Para realizar esta practica necesitas lo siguiente:
- WOKWI
- Tarjeta ESP32
- Sensor ultrasonico HC-SR04
- Sensor de temperatura y humedad DHT22
- Node-RED

### Requisitos previos:
Para poder usar este repositorio necesitas entrar a la plataforma WOKWI como tener instalado Node-RED.

### Instrucciones de preparación de entorno:
1.-Abrir la terminal de programación en **WOKWI** y colocar la siguente programación:
```
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
const int Trigger = 4;   //Pin digital 2 para el Trigger del sensor
const int Echo = 15;   //Pin digital 3 para el Echo del sensor
#include "DHTesp.h"
const int DHT_PIN = 16;
DHTesp dhtSensor;
// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "35.172.255.228"; //servidor local de la CMD public MQTT
String username_mqtt="DIXZO";
String password_mqtt="238";

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
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);//iniciailzamos la comunicación
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
}

void loop() {


delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
long t; //timepo que demora en llegar el eco
  long d; //distancia en centimetros


  digitalWrite(Trigger, HIGH);
  delayMicroseconds(10);          //Enviamos un pulso de 10us
  digitalWrite(Trigger, LOW);
  
  t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
  d = t/59;             //escalamos el tiempo a una distancia en cm
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
    //doc["Anho"] = 2022;
    //doc["Empresa"] = "Educatronicos";
    doc["DISTANCIA"] = String(d);
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
    doc["NOMBRE"] = "Diego Bahena";

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("HOME", output.c_str()); //nos unimos a un canal
  }
}
```
2.- Instalar las librerias necesarias para el correcto funcionamiento del programa, como se muestra en la siguente imagen.

![image](https://github.com/user-attachments/assets/3d366e6f-a9a6-47e2-83e3-6f51858167a6)

3.- Hacer la conexion de **HC-SR04** con la **ESP32** como tambien, conectar el sensor de temperatura y humedad **DHT22** como se muestra en la siguente imagen.

![image](https://github.com/user-attachments/assets/1e0f3ac5-7820-4b30-921a-cb42708af71c)

4.- abrimos la **CMD** como administrador, para ejecutar el Programa **Node-red** debemos escribir en el **CMD**: ```node-red``` 

5.- Para abrir la aplicación nos vamos algun explorador y colocamos el siguente link: ```localhost:1880```

6.- ejecutamos el **CMD** como administrador escribimos ```nslookup broker.emqx.io``` y copiamos el Address que nos genera.

7.- una vez en **node-red** colocamos un bloque ```mqqtt in``` y configuramos el bloque con el puerto mqtt con el ip ```35.172.255.228``` como se muestra en la imagen.

![image](https://github.com/user-attachments/assets/393d2aff-8efe-4873-8013-7ff0a5d3c131)

8.- Colocar el bloque ```json``` y configurarlo como se muestra en la imagen.

![image](https://github.com/user-attachments/assets/d50013ba-a925-4526-ac1d-7f462584081b)

9.- Colocamos tres bloques ```function``` y los configuramos con los siguentes codigos.

```
msg.payload = msg.payload.TEMPERATURA;
msg.topic = "TEMPERATURA";
return msg;
```

```
msg.payload = msg.payload.HUMEDAD;
msg.topic = "HUMEDAD";
return msg;
```

```
msg.payload = msg.payload.DISTANCIA;
msg.topic = "DISTANCIA";
return msg;
```
10.- Colocamos un bloque de ```chart``` y ```gauge``` para cada funcion para poder visualisar grafica como indicadores.

![image](https://github.com/user-attachments/assets/57a09481-a686-4bd6-9845-9195be95a16b)

### Instrucciónes de operación
- Iniciar simulador WOKWI.
- Seleccionamos DEPLOY en node-red para iniciar la comunicacion. 
- Seleccionamos DASHBOARD en node-red. 
- Abrimos el localhost en el dashboard. 
- Visualizamos las graficas como los parametros.
  
### Resultados
Cuando haya funcionado, verás los valores en le localhost como se muestra en la siguente imagen.

![image](https://github.com/user-attachments/assets/ac10ff61-6ce6-4577-825f-cdf1b0b955db)

### Desarrollado por 
Diego David Bahena Galan

[GitHub](https://github.com/DiegoDBG)
