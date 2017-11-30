下面开始正式写代码环节。

```c
 #include <ArduinoJson.h>
#include <ESP8266WiFi.h>
#include <ESP8266mDNS.h>
#include <WiFiUdp.h>
#include <ArduinoOTA.h>
#include <ESP8266WiFi.h>
#include <PubSubClient.h>



const char* ssid = "*******";//路由器ssid
const char* password = "*******";//路由器密码
const char* mqtt_server = "********";//服务器的地址 手机热点网关为192.168.43.1


//全局变量区域上界



//全局变量区域下界

WiFiClient espClient;
PubSubClient client(espClient);

int OTA=0;
int OTAS=0;
long lastMsg = 0;//存放时间的变量 
char msg[200];//存放要发的数据
String load;


void setup_wifi() {//自动连WIFI接入网络
  delay(10);
    WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
}



void reconnect() {//等待，直到连接上服务器
  while (!client.connected()) {//如果没有连接上
    int randnum = random(0, 999); 
    if (client.connect("OTADEMO"+randnum)) {//接入时的用户名，尽量取一个很不常用的用户名
      client.subscribe("*******");//接收外来的数据时的intopic
    } else {
      Serial.print("failed, rc=");//连接失败
      Serial.print(client.state());//重新连接
      Serial.println(" try again in 5 seconds");//延时5秒后重新连接
      delay(5000);
    }
  }
}

void callback(char* topic, byte* payload, unsigned int length) {//用于接收服务器接收的数据
  load="";
  for (int i = 0; i < length; i++) {
      load +=(char)payload[i];//串口打印出接收到的数据
  }
   decodeJson();
}

void  decodeJson() {
  DynamicJsonBuffer jsonBuffer;
  JsonObject& root = jsonBuffer.parseObject(load);
   OTA = root["OTA"];
   OTAS =OTA;
   //接收数据json处理区上界

   //添加其他自己的JSON收听处理方式就像这样  int Activity=root["ACT"];

   //接收数据json处理区下界
}

void OTAsetup(){
   if(OTAS){
  ArduinoOTA.onStart([]() {
    Serial.println("Start");
  });
  ArduinoOTA.onEnd([]() {
    Serial.println("\nEnd");
  });
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });
  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) Serial.println("Auth Failed");
    else if (error == OTA_BEGIN_ERROR) Serial.println("Begin Failed");
    else if (error == OTA_CONNECT_ERROR) Serial.println("Connect Failed");
    else if (error == OTA_RECEIVE_ERROR) Serial.println("Receive Failed");
    else if (error == OTA_END_ERROR) Serial.println("End Failed");
  });
  ArduinoOTA.begin();
  Serial.println("Ready");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
  OTAS=0;
   }
}


void setup() {
  //setup代码区域上界



  //填写自己的逻辑代码
  Serial.begin(9600);


  //setup代码区域下界

   setup_wifi();//自动连WIFI接入网络
  client.setServer(mqtt_server, 1883);//1883为端口号
  client.setCallback(callback); //用于接收服务器接收的数据
}

void loop() {
        if(OTA){
          OTAsetup();
        ArduinoOTA.handle();
       }
       else{
        reconnect();//确保连上服务器，否则一直等待。
        client.loop();//MUC接收数据的主循环函数。
        //loop代码上界


        //自己的逻辑代码



        //loop代码下界   
         long now = millis();//记录当前时间
        if (now - lastMsg > 300) {//每隔300毫秒秒发一次数据
           encodeJson();
           client.publish("******",msg);//以OTA为TOPIC对外发送MQTT消息
          lastMsg = now;//刷新上一次发送数据的时间
        }
       }
}

void encodeJson(){
  DynamicJsonBuffer jsonBuffer;
  JsonObject& root1 = jsonBuffer.createObject();
  //发送数据区上界


  //添加其他要发送的JSON包就像这样下面这句代码
  root1["Rs"] =R_speed[1];
  root1["Ls"] =L_speed[1];
  root1["dif"] =L_speed[1]-R_speed[1];


  //发送数据区下界
  root1.printTo(msg);
  }

  //函数区域上界


  //函数区域下界
```

以上代码是一个MQTT以及OTA的框架。你可以先将此代码黏贴到你的ArduinoIDE上，然后我们再慢慢补充完整。先介绍一下这段代码的关键点。这段代码集成了OTA（远程烧写功能）和MQTT协议（用于物联网通信的一种协议）。

```
const char* ssid = "*******";//路由器ssid
const char* password = "*******";//路由器密码
const char* mqtt_server = "********";//服务器的地址
```

将这三段代码的\*号改为你手机热点名称，热点密码。将服务器地址改为“192.168.43.1” 这个服务器地址是Android手机WLAN热点的网关地址。当然如果你有云服务器那你也可以将地址改成你云服务器的地址（前提是你云服务器上装了MQTT broker）。这么一来，小车就有能力连接手机热点，可以通过WLAN网关访问手机内部的MQTT broker了。

接下来，看到reconnect\(\)函数，其中有一行代码是这样的

```c
client.subscribe("*******");//接收外来的数据时的intopic
```

将\*号改为你喜欢的西文字符，这个是小车订阅你手机数据的一种方式，这个Intopic与到时候APP里输入的PubTopic需要保持一致,这样才能确保你手机发出去的消息能被小车接收。然后再loop函数里有这样一个函数

```c
client.publish("*********",msg);//以OTA为TOPIC对外发送MQTT消息
```

同样道理，这些\*号也改为你喜欢的西文字符，但是要注意，不要与intopic相同，否则小车自己发送的数据又被自己收回来了。这个字符串到时候会与APP里的

---

---

保持一致，用于从小车向手机发送消息。

到现在为止，你已经搭好了整个属于你的MQTT以及OTA的框架。

