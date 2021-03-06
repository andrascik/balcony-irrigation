# Balcony plant irrigation system

Balcony automated watering system powered by arduino, esp8266 and node-red via MQTT.

![Valve with servo](irrigation-valve-arduino.jpg)

## Scheduler

The aim is to check soil moisture and weather forecast API on regular basis (4 or 6 hours variant) and find the best algorithm for watering.


### 4 hrs variant

- 00:00
- 04:00
- 08:00
- 12:00
- 16:00
- 20:00	

### 6 hrs variant

- 00:00
- 06:00
- 12:00
- 18:00

API call every 6 hours checks the amount of RainProbability higher than 40%

## Accuweather API


API key: <GET_ONE_FROM_ACCUWEATHER>

http://dataservice.accuweather.com/forecasts/v1/hourly/12hour/300250
http://dataservice.accuweather.com/currentconditions/v1/300250/historical

http://dataservice.accuweather.com/forecasts/v1/hourly/12hour/300250?apikey={YOUR_ACCUWEATHER_API_KEY}&language=en-gb&details=true&metric=true


## Hardware

- Arduino Uno
- ESP8266 esp01s WIFI
- Temperature probe ds18b20
- Ultrasonic sensor hcsr04

## Wiring scheme

![Schematics](balcony-watering_bb.png)
![Board](board.jpg)
![LDR detail](ldr_detail.jpg)

## Libraries for arduino

### Dallas Temperature probe

https://github.com/milesburton/Arduino-Temperature-Control-Library

### OneWire
https://github.com/PaulStoffregen/OneWire


## Arduino Uno code

```c++
#include <Servo.h>

// Libraries for dallas temperature prob
#include <OneWire.h>
#include <DallasTemperature.h>
 
// Data wire is plugged into port 5 on the Arduino
#define ONE_WIRE_BUS 2
 
// Hook up HC-SR04 with Trig to Arduino Pin 10, Echo to Arduino pin 13
#define trigPin 12
#define echoPin 13

int servoPin = 10;
char recstr[5] = ""; //Initialized variable to store recieved data
char oValve[5] = "valv0";
char cValve[5] = "valv1";
int tankHeightCm = 45;
float Celsius=0;

float duration, distance;

OneWire oneWire(ONE_WIRE_BUS);

DallasTemperature sensors(&oneWire);

Servo servo;

void closeValve()
{
  servo.write(10);
}

void openValve()
{
  servo.write(80);
}

void setup() {
  Serial.begin(115200);
  Serial.flush();
  servo.attach(servoPin);
  sensors.begin();
  
  pinMode(ONE_WIRE_BUS, INPUT_PULLUP);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
}

void loop() {
  Serial.readBytes(recstr, 5); //Read the serial data and store in var

  if (strncmp(recstr, cValve, 5) == 0) {
    closeValve();
  }
  if (strncmp(recstr, oValve, 5) == 0) {
    openValve();
  }
  
  Serial.println(recstr);

  // Write a pulse to the HC-SR04 Trigger Pin
  
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
 
  duration = pulseIn(echoPin, HIGH);
  
  // Determine distance from duration
  // Use 343 metres per second as speed of sound
  
  distance = (duration / 2) * 0.0343;
  if (distance >= 400 || distance <= 2) {
     Serial.println("Out of range");
  }
  else {
    float perc = 0.00;
    perc = distance - tankHeightCm;
    perc = abs(perc)*2.5;
    Serial.print("W:");
    Serial.println(String(perc));
    delay(500);
  }

  int lightIntensity=analogRead(A0);
  Serial.print("L:");
  Serial.println(lightIntensity);

  sensors.requestTemperatures(); 
  delay(1000);
  Celsius=sensors.getTempCByIndex(0);
  Serial.print("C:");
  Serial.print(Celsius);
}
```

## ESP8266 code

```c++
#include <ESP8266WiFi.h>
#include <PubSubClient.h>

const char* ssid = "ssid";
const char* password =  "password";
//const char* mqttServer = "ip";
const char* mqttServer = "192.168.1.250";
//const int mqttPort = 5100;
const int mqttPort = 1883;
const char* mqttUser = "";
const char* mqttPassword = "";

WiFiClient espClient;
PubSubClient client(espClient);

long lastMsg = 0;
char msg[50];
int value = 0;
char mystr[] = "Hello"; //String data
char recstr[5]; //Initialized variable to store recieved data

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

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
  for (int i = 0; i < length; i++) {
    //Serial.print((char)payload[i]);
  }

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is acive low on the ESP-01)
    char mystr[] = "valv1"; //String data

    Serial.write(mystr,5); //Write the serial data
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  // Turn the LED off by making the voltage HIGH
    char mystr[] = "valv0"; //String data
    Serial.write(mystr,5); //Write the serial data
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "balkon-esp-1";
    // Attempt to connect
    if (client.connect(clientId.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("balkon/valve", "valve turned: ");
      // ... and resubscribe
      client.subscribe("balkon/valve");
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
  Serial.begin(115200);
  
  setup_wifi();
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    ++value;
  }
}

```

## Photogallery

![Plants](plants-with-watering.jpg)
Some plants

![Water level sensor](ultrasonic-sensor-hcsr04.jpg)
Water level sensor made from ultrasonic hc-sr04

![Water tank](water-tank.jpg)
25 litres plastic water tank

