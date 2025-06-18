# IR_esp8266
通过mqtt自动配置，在homeassistant上自动添加空调，和温湿度检测 

## 硬件
ESP8266-e12开发板      X1  
DHT11 温湿度传感器     X1  
红外发射二极管         X1  
对应电路的电阻和杜邦线 若干

## 软件
homeassistant环境  
Arduino IDE  
mqtt 服务器  

### 主要功能
1.每分钟读取一次温湿度，自动上传到homeassistant  
2.控制空调（格力，其他品牌要修改）  
3.homeassistant自动发现空调和温湿度传感器  


### 代码

```arduino
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <IRremoteESP8266.h>
#include <IRsend.h>
#include <ir_Gree.h>
#include <DHT.h>
#include <ArduinoJson.h>
#include <EEPROM.h>

// ==== WiFi & MQTT 配置 ====
const char* ssid = "PING";
const char* password = "kk123456";
const char* mqtt_server = "192.168.0.254";
const int mqtt_port = 1883;
const char* mqtt_user = "";
const char* mqtt_pass = "";

WiFiClient espClient;
PubSubClient client(espClient);

// ==== 红外 & DHT 传感器引脚设置 ====
#define IR_LED_PIN 4   // D2 -> GPIO4
#define DHTPIN 2       // D4 -> GPIO2
#define DHTTYPE DHT11

IRGreeAC ac(IR_LED_PIN);
DHT dht(DHTPIN, DHTTYPE);

// ==== 空调状态结构体，用于掉电保存 ====
struct AcState {
  bool isValid;  // 标志：是否已初始化
  bool power;
  uint8_t mode;
  uint8_t fan;
  uint8_t temp;
} acState;

const int EEPROM_ADDR = 0;  // EEPROM 地址

// ==== MQTT Topic 定义 ====
const char* climate_config_topic = "homeassistant/climate/gree_ac/config";
const char* temp_config_topic = "homeassistant/sensor/gree_temp/config";
const char* humi_config_topic = "homeassistant/sensor/gree_humi/config";

const char* ac_state_topic = "home/gree/ac/state";
const char* ac_cmd_topic = "home/gree/ac/set";
const char* temp_state_topic = "home/gree/temp";
const char* humi_state_topic = "home/gree/humi";

unsigned long lastDhtMillis = 0;
const unsigned long DHT_INTERVAL = 60000;

// ==== 状态保存到 EEPROM ====
void saveAcStateToEEPROM() {
  acState.isValid = true;  // 写入前设置为有效
  EEPROM.put(EEPROM_ADDR, acState);
  EEPROM.commit();
  Serial.println("✅ 状态已保存到 EEPROM");
}

// ==== 从 EEPROM 恢复状态 ====
void loadAcStateFromEEPROM() {
  EEPROM.get(EEPROM_ADDR, acState);
  if (acState.isValid &&
      acState.temp >= 16 && acState.temp <= 30 &&
      acState.mode <= kGreeAuto &&
      acState.fan <= kGreeFanAuto) {
    ac.setPower(acState.power);
    ac.setMode(acState.mode);
    ac.setFan(acState.fan);
    ac.setTemp(acState.temp);
    ac.send();
    Serial.println("✅ EEPROM 中读取状态并已发送红外命令");
    publish_ac_state_from_eeprom();
  } else {
    // 设置默认值
    acState.power = false;
    acState.mode = kGreeCool;
    acState.fan = kGreeFanAuto;
    acState.temp = 26;
    Serial.println("⚠️ EEPROM 无效，使用默认空调设置");
    publish_ac_state_from_eeprom();  // 上报默认状态
  }
}

// 断电后重启发送状态给homeassistant
void publish_ac_state_from_eeprom() {
  StaticJsonDocument<128> doc;
  doc["mode"] = acState.mode == kGreeCool   ? "cool" :
                acState.mode == kGreeHeat   ? "heat" :
                acState.mode == kGreeDry    ? "dry" :
                acState.mode == kGreeFan    ? "fan_only" :
                acState.mode == kGreeAuto   ? "auto" : "off";
  doc["fan"] = acState.fan == kGreeFanMin   ? "low" :
               acState.fan == kGreeFanMed   ? "medium" :
               acState.fan == kGreeFanMax   ? "high" :
               acState.fan == kGreeFanAuto  ? "auto" : "auto";
  doc["temp"] = acState.temp;
  doc["power"] = acState.power ? "on" : "off";

  char buf[128];
  serializeJson(doc, buf);
  client.publish(ac_state_topic, buf, true);
}



// ==== MQTT 消息回调函数 ====
void callback(char* topic, byte* payload, unsigned int length) {
  StaticJsonDocument<256> doc;
  String msg;
  for (unsigned int i = 0; i < length; i++) msg += (char)payload[i];
  DeserializationError error = deserializeJson(doc, msg);
  if (error) return;

  const char* mode = doc["mode"] | "";
  const char* fan = doc["fan"] | "";
  const char* power = doc["power"] | "";
  float tempFloat = doc["temp"] | 0.0;
  int temp = (int)tempFloat;

  if (strcmp(power, "off") == 0) acState.power = false;
  else if (strcmp(power, "on") == 0) acState.power = true;

  if (strcmp(mode, "cool") == 0) acState.mode = kGreeCool;
  else if (strcmp(mode, "heat") == 0) acState.mode = kGreeHeat;
  else if (strcmp(mode, "dry") == 0) acState.mode = kGreeDry;
  else if (strcmp(mode, "fan_only") == 0) acState.mode = kGreeFan;
  else if (strcmp(mode, "auto") == 0) acState.mode = kGreeAuto;

  if (strcmp(fan, "low") == 0) acState.fan = kGreeFanMin;
  else if (strcmp(fan, "medium") == 0) acState.fan = kGreeFanMed;
  else if (strcmp(fan, "high") == 0) acState.fan = kGreeFanMax;
  else if (strcmp(fan, "auto") == 0) acState.fan = kGreeFanAuto;

  if (temp >= 16 && temp <= 30) acState.temp = temp;

  ac.setPower(acState.power);
  ac.setMode(acState.mode);
  ac.setFan(acState.fan);
  ac.setTemp(acState.temp);
  ac.send();

  // 保存到ROM
  saveAcStateToEEPROM();
}

void init_wifi() {
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
}

void init_mqtt() {
  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);
}

void init_ac() {
  ac.begin();
}

void init_dht() {
  dht.begin();
}

// ==== Home Assistant 自动发现配置 ====
void publish_discovery() {
  char buf[1024];
StaticJsonDocument<1024> climate;

climate["name"] = "格力空调";
climate["unique_id"] = "gree_ac_8266";

climate["min_temp"] = 16;
climate["max_temp"] = 30;
climate["temp_step"] = 1;

climate["mode_command_topic"] = ac_cmd_topic;
climate["mode_command_template"] = "{ \"mode\": \"{{ value }}\" }";

climate["temperature_command_topic"] = ac_cmd_topic;
climate["temperature_command_template"] = "{ \"temp\": {{ value }} }";

climate["fan_mode_command_topic"] = ac_cmd_topic;
climate["fan_mode_command_template"] = "{ \"fan\": \"{{ value }}\" }";

// 状态读取配置
climate["mode_state_topic"] = "home/gree/ac/status";
climate["mode_state_template"] = "{{ value_json.mode }}";

climate["temperature_state_topic"] = "home/gree/ac/status";
climate["temperature_state_template"] = "{{ value_json.temp }}";

climate["fan_mode_state_topic"] = "home/gree/ac/status";
climate["fan_mode_state_template"] = "{{ value_json.fan }}";

// 当前温度（环境温度）
climate["current_temperature_topic"] = "home/gree/roomtemp";

// 模式列表
JsonArray modes = climate.createNestedArray("modes");
modes.add("off");
modes.add("cool");
modes.add("heat");
modes.add("dry");
modes.add("fan_only");
modes.add("auto");

// 风速列表
JsonArray fan_modes = climate.createNestedArray("fan_modes");
fan_modes.add("auto");
fan_modes.add("low");
fan_modes.add("medium");
fan_modes.add("high");

// 设备信息
JsonObject device = climate.createNestedObject("device");
JsonArray identifiers = device.createNestedArray("identifiers");
identifiers.add("gree_ac_8266");
device["name"] = "格力空调";
device["manufacturer"] = "Gree";
device["model"] = "智能空调";
device["sw_version"] = "1.0";

// 序列化并发布MQTT
serializeJson(climate, buf);
client.publish(climate_config_topic, buf, true); 

  // 温度传感器配置
  StaticJsonDocument<256> temp;
  temp["name"] = "卧室温度";
  temp["unique_id"] = "gree_temp_8266";
  temp["state_topic"] = temp_state_topic;
  temp["unit_of_measurement"] = "°C";
  temp["device_class"] = "temperature";
  serializeJson(temp, buf);
  client.publish(temp_config_topic, buf, true);

  // 湿度传感器配置
  StaticJsonDocument<256> humi;
  humi["name"] = "卧室湿度";
  humi["unique_id"] = "gree_humi_8266";
  humi["state_topic"] = humi_state_topic;
  humi["unit_of_measurement"] = "%";
  humi["device_class"] = "humidity";
  serializeJson(humi, buf);
  client.publish(humi_config_topic, buf, true);
}

// 上报空调状态
void publish_ac_state() {
  StaticJsonDocument<128> doc;
  doc["mode"] = ac.getMode() == kGreeCool   ? "cool" :
                ac.getMode() == kGreeHeat   ? "heat" :
                ac.getMode() == kGreeDry    ? "dry" :
                ac.getMode() == kGreeFan    ? "fan_only" :
                ac.getMode() == kGreeAuto   ? "auto" : "off";
  doc["fan"] = ac.getFan() == kGreeFanMin   ? "low" :
               ac.getFan() == kGreeFanMed   ? "medium" :
               ac.getFan() == kGreeFanMax   ? "high" :
               ac.getFan() == kGreeFanAuto  ? "auto" : "auto";
  doc["temp"] = ac.getTemp();
  doc["power"] = ac.getPower() ? "on" : "off";

  char buf[128];
  serializeJson(doc, buf);
  client.publish(ac_state_topic, buf, true);  // Retain 状态
}


// ==== 发布 DHT 温湿度数据 ====
void publish_dht_state() {
  float h = dht.readHumidity();
  float t = dht.readTemperature();
  if (isnan(h) || isnan(t)) return;
  char buf[16];
  dtostrf(t, 4, 1, buf);
  client.publish(temp_state_topic, buf);
  dtostrf(h, 4, 1, buf);
  client.publish(humi_state_topic, buf);
}

void handle_dht_reading() {
  if (millis() - lastDhtMillis >= DHT_INTERVAL) {
    lastDhtMillis = millis();
    publish_dht_state();
  }
}

void reconnect_mqtt() {
  while (!client.connected()) {
    if (client.connect("ESP8266_AC_DHT")) {
      client.subscribe(ac_cmd_topic);
      publish_discovery();
    } else {
      delay(2000);
    }
  }
}


void setup() {
  Serial.begin(115200);
  EEPROM.begin(sizeof(AcState));
  init_wifi();
  init_mqtt();
  init_ac();
  init_dht();
  loadAcStateFromEEPROM();
}

void loop() {
  if (!client.connected()) reconnect_mqtt();
  client.loop();
  handle_dht_reading();
  yield();
}

```
