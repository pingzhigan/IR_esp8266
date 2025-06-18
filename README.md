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

const char* ac_state_topic = "home/gree/ac/status";
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
   // 打印调试信息
  Serial.println("保存空调状态到 EEPROM:");
  Serial.printf("  Power: %s\n", acState.power ? "on" : "off");

  const char* modeStr = acState.mode == kGreeCool   ? "cool" :
                        acState.mode == kGreeHeat   ? "heat" :
                        acState.mode == kGreeDry    ? "dry" :
                        acState.mode == kGreeFan    ? "fan_only" :
                        acState.mode == kGreeAuto   ? "auto" : "unknown";
  Serial.printf("  Mode : %s\n", modeStr);

  const char* fanStr = acState.fan == kGreeFanMin   ? "low" :
                       acState.fan == kGreeFanMed   ? "medium" :
                       acState.fan == kGreeFanMax   ? "high" :
                       acState.fan == kGreeFanAuto  ? "auto" : "unknown";
  Serial.printf("  Fan  : %s\n", fanStr);

  Serial.printf("  Temp : %d\n", acState.temp);
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
    // ac.send();
    publish_ac_state_from_eeprom();
    Serial.println("✅ EEPROM 中读取状态并已发送home命令");
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
  // 检查是否有有效的 acState（你可以结合后续初始化判断更严谨）
  // 若无效数据，则初始化为默认值
  if (acState.temp < 16 || acState.temp > 30) {
    acState.power = false;
    acState.mode = kGreeCool;   // 实际值不会被用到，因为 power = false 会设置 mode = "off"
    acState.fan = kGreeFanAuto;
    acState.temp = 26;
  }

  StaticJsonDocument<128> doc;

  // 如果关机就只上报 mode = "off"
  if (!acState.power) {
    doc["mode"] = "off";
  } else {
    doc["mode"] = acState.mode == kGreeCool   ? "cool" :
                  acState.mode == kGreeHeat   ? "heat" :
                  acState.mode == kGreeDry    ? "dry" :
                  acState.mode == kGreeFan    ? "fan_only" :
                  acState.mode == kGreeAuto   ? "auto" : "off";
  }

  doc["fan"] = acState.fan == kGreeFanMin   ? "low" :
               acState.fan == kGreeFanMed   ? "medium" :
               acState.fan == kGreeFanMax   ? "high" :
               acState.fan == kGreeFanAuto  ? "auto" : "auto";

  doc["temp"] = acState.temp;

  char buf[128];
  serializeJson(doc, buf);
  client.publish(ac_state_topic, buf, true);  // retain 状态
}


// ==== MQTT 消息回调函数 ====
void callback(char* topic, byte* payload, unsigned int length) {
  StaticJsonDocument<256> doc;
  String msg;
  for (unsigned int i = 0; i < length; i++) msg += (char)payload[i];

  DeserializationError error = deserializeJson(doc, msg);
  if (error) {
    Serial.println("JSON 解析失败");
    return;
  }

  // === 提取字段并清理空格 ===
  String modeStr = doc["mode"] | "";
  modeStr.trim();

  String fanStr = doc["fan"] | "";
  fanStr.trim();

  float tempFloat = doc["temp"] | 0.0;
  int temp = (int)tempFloat;

  // === 处理 mode 和 power 状态 ===
  if (modeStr.length() > 0) {
    if (modeStr == "off") {
      acState.power = false;
    } else {
      acState.power = true;

      // 设置工作模式
      if (modeStr == "cool") acState.mode = kGreeCool;
      else if (modeStr == "heat") acState.mode = kGreeHeat;
      else if (modeStr == "dry") acState.mode = kGreeDry;
      else if (modeStr == "fan_only") acState.mode = kGreeFan;
      else if (modeStr == "auto") acState.mode = kGreeAuto;
    }
  }

  // === 处理风速 ===
  if (fanStr.length() > 0) {
    if (fanStr == "low") acState.fan = kGreeFanMin;
    else if (fanStr == "medium") acState.fan = kGreeFanMed;
    else if (fanStr == "high") acState.fan = kGreeFanMax;
    else if (fanStr == "auto") acState.fan = kGreeFanAuto;
  }

  // === 处理温度 ===
  if (temp >= 16 && temp <= 30) {
    acState.temp = temp;
  }

  // === 设置空调状态 ===
  ac.setPower(acState.power);
  ac.setMode(acState.mode);
  ac.setFan(acState.fan);
  ac.setTemp(acState.temp);
  ac.send();

  // === 上报状态 & 保存 ===
  publish_ac_state();
  saveAcStateToEEPROM();

  // === 调试打印 ===
  // Serial.println("保存到 EEPROM 的空调状态：");
  // Serial.printf("  Power: %s\n", acState.power ? "on" : "off");

  // const char* modeDebug = acState.mode == kGreeCool   ? "cool" :
  //                         acState.mode == kGreeHeat   ? "heat" :
  //                         acState.mode == kGreeDry    ? "dry" :
  //                         acState.mode == kGreeFan    ? "fan_only" :
  //                         acState.mode == kGreeAuto   ? "auto" : "unknown";

  // Serial.printf("  Mode : %s\n", modeDebug);

  // const char* fanDebug = acState.fan == kGreeFanMin   ? "low" :
  //                        acState.fan == kGreeFanMed   ? "medium" :
  //                        acState.fan == kGreeFanMax   ? "high" :
  //                        acState.fan == kGreeFanAuto  ? "auto" : "unknown";

  // Serial.printf("  Fan  : %s\n", fanDebug);
  // Serial.printf("  Temp : %d\n", acState.temp);
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

  // 使用保存的状态判断
  if (!acState.power) {
    doc["mode"] = "off";
  } else {
    doc["mode"] = acState.mode == kGreeCool   ? "cool" :
                  acState.mode == kGreeHeat   ? "heat" :
                  acState.mode == kGreeDry    ? "dry" :
                  acState.mode == kGreeFan    ? "fan_only" :
                  acState.mode == kGreeAuto   ? "auto" : "off";
  }

  doc["fan"] = acState.fan == kGreeFanMin   ? "low" :
               acState.fan == kGreeFanMed   ? "medium" :
               acState.fan == kGreeFanMax   ? "high" :
               acState.fan == kGreeFanAuto  ? "auto" : "auto";

  doc["temp"] = acState.temp;

  char buf[128];
  serializeJson(doc, buf);
  client.publish(ac_state_topic, buf, true);  // 保留发布
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
      delay(200);
      loadAcStateFromEEPROM();
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
}

void loop() {
  if (!client.connected()) reconnect_mqtt();
  client.loop();
  handle_dht_reading();
  yield();
}

```
