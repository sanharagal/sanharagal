
#include <WiFiClient.h>
#include <WiFiServer.h>
#include <WiFiUdp.h>
#include <Servo.h>
#include<ESP8266WiFi.h>
#include<ThingSpeak.h>

const int trigpin =D1;
const int echopin =D2;
const int buzzer =D5;
const int led =12;
const int servopin =13;

const char*ssid="Redmi9";
const char*password="jesus great";
const char*writeAPIkey="5804GWLMOHFXMD40";
WiFiClient client;
unsigned long channelID=123456;

long duration;
int distance;
int startAngle =-1;
int stopAngle =-1;
bool ObjectDetected = false;

Servo myServo;

unsigned long lastTime =0;
unsigned long delayTime =15000;

void sendDataToThingSpeak();
void setup()
{
  Serial.begin(115200);

  pinMode(trigpin,OUTPUT);
  pinMode(echopin,INPUT);
  pinMode(buzzer,OUTPUT);
  pinMode(led,OUTPUT);

  myServo.attach(servopin);

  myServo.write(90);
  WiFi.begin(ssid,password);
  while(WiFi.status()!=WL_CONNECTED)
  {
    delay(1000);
    Serial.println("connecting to WiFi...");
  }
  Serial.println("connected to WiFi!");
  
  ThingSpeak.begin(client);
}

void loop()
{
  rotateRadarAndDetectObject();
  if(millis()-lastTime>delayTime)
  {
    sendDataToThingSpeak();
    lastTime=millis();
  }
  delay(100);
}
int getDistance()
{
  digitalWrite(trigpin,LOW);
  delayMicroseconds(2);

  digitalWrite(trigpin,HIGH);
  delayMicroseconds(10);
  digitalWrite(trigpin,LOW);

  duration=pulseIn(echopin,HIGH);
  distance=duration*0.034/2;
  return distance;
}
void rotateRadarAndDetectObject()
{
  for(int pos=0;pos<=180;pos+=1)
  {
    myServo.write(pos);
    delay(20);
    distance=getDistance();
    Serial.print("Angle");
    Serial.print(pos);
    Serial.print("|distance");
    Serial.print(distance);
    Serial.print("cm");

    if(distance<10)
    {
      if(!ObjectDetected)
      {
        startAngle=pos;
        ObjectDetected=true;
        Serial.print("Object detected starting at angle:");
        Serial.println(startAngle);
      }
      digitalWrite(buzzer,HIGH);
      digitalWrite(led,HIGH);
    }
    else
    {
      if(ObjectDetected)
      {
        stopAngle=pos;
        ObjectDetected=false;
        Serial.print("object detected stopping at angle:");
        Serial.print(stopAngle);
        sendDataToThingSpeak();
      }
      digitalWrite(buzzer,LOW);
      digitalWrite(led,LOW);
    }
  }
  for(int pos=180;pos>=0;pos-1)
  {
    myServo.write(pos);
    delay(20);

    distance= getDistance(); 
    Serial.print("Angle:");
    Serial.print(pos);
    Serial.print("Distance:");
    Serial.print(distance);
    Serial.println("cm");
  
   if(distance<=30)
   {
    if(!ObjectDetected)
   {
    startAngle=pos;
    ObjectDetected=true;
    Serial.print("Object detected starting at angle");
    Serial.println(startAngle);
   }
   digitalWrite(buzzer,HIGH);
   digitalWrite(led,HIGH);
  }
  else
  {
    if(ObjectDetected)
    {
      stopAngle=pos;
      ObjectDetected=false;
      Serial.print("Object detected stopping at angle");
      Serial.println(stopAngle);
      sendDataToThingSpeak();
    }
    digitalWrite(buzzer,LOW);
    digitalWrite(led,LOW);
  }
}
}

void sendDataToThingSpeak()
{
  if(startAngle!=-1&&stopAngle!=-1)
  {
   ThingSpeak.setField(1,startAngle);
   ThingSpeak.setField(2,stopAngle);
   ThingSpeak.setField(3,distance);

   int responseCode=ThingSpeak.writeFields(channelID,writeAPIkey);
   if(responseCode==200)
   {
    Serial.println("Datasent to ThingSpeak successfully");
   }
   else
   {
    Serial.println("Problem sending data to ThingSpeak:"+String(responseCode));
   }
   startAngle=-1;
   stopAngle=-1;
  }
}
    
    
