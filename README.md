#include <WiFi.h>
#include <PubSubClient.h>
#include "DHT.h"
#include "Arduino_LED_Matrix.h"
#define LED_PIN 13  
#define DHTPIN 2 
#define DHTTYPE DHT11
#define LDR A0
#define LED_PIN1 9
DHT dht(DHTPIN, DHTTYPE);
ArduinoLEDMatrix matrix;

const char* ssid = "🥨";                
const char* password = "55555555";            
const char* mqtt_server = "broker.netpie.io";  
const int mqtt_port = 1883;
const char* mqtt_Client = "ef8832da-c1d0-4680-a92a-0600c8c6199c"; 
const char* mqtt_username = "RCU2Xm3TBfNDt8r6X168THy5PKAqQWfd";   
const char* mqtt_password = "3YfXonpZk73QY3Nwizsb1BAgHNwGSjna";   

WiFiClient espClient;
PubSubClient client(espClient);

long lastMsg = 0;
int value = 0;
char msg[100];
String DataString;

const uint32_t animation[][3] = {
  { 0x70e5f, 0xa402909a, 0x954023fc },
  { 0x70e5f, 0xa4029999, 0x994023fc }
};

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    if (client.connect(mqtt_Client, mqtt_username, mqtt_password)) {  
      Serial.println("connected");
      client.subscribe("@msg/operator");  
      client.subscribe("@msg/toggle");   
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println("try again in 5 seconds");
      delay(1000);
    }
  }
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  String message;
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.println(message);

  if (String(topic) == "@msg/operator") {
    if (message == "ON") {
      digitalWrite(LED_PIN, HIGH);
      Serial.println("LED ON");
    } else if (message == "OFF") {
      digitalWrite(LED_PIN, LOW);
      Serial.println("LED OFF");
    }
  } else if (String(topic) == "@msg/toggle") {
    if (message == "SHOW1") {
      Serial.println("Display Animation 1");
      matrix.loadFrame(animation[0]); 
    } else if (message == "SHOW2") {
      Serial.println("Display Animation 2");
      matrix.loadFrame(animation[1]);  
    }
  }
}
void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  pinMode(LDR, INPUT);
  pinMode(LED_PIN1, OUTPUT);
  dht.begin();
  matrix.begin();
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid, password); 

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());  

  client.setServer(mqtt_server, mqtt_port); 
  client.setCallback(callback);           
  client.subscribe("@msg/operator");
  client.subscribe("@msg/toggle");
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  long now = millis();
  if (now - lastMsg > 5000) {
    lastMsg = now;
    ++value;

    float h = dht.readHumidity();
    float t = dht.readTemperature();
    int l = analogRead(LDR);

    if (l < 10) {
      digitalWrite(LED_PIN1, HIGH); 
    } else {
      digitalWrite(LED_PIN1, LOW);  
    }

    DataString = "{\"data\":{\"temperature\":" + (String)t + ",\"humidity\":" + (String)h + ",\"LDR\": " + (String)l + "}}";
    DataString.toCharArray(msg, 100);
    Serial.println("HELLO NETPIE");
    Serial.print("Publish message : ");
    Serial.println(msg);
    client.publish("@shadow/data/update", msg);
  }
  delay(1);
}
