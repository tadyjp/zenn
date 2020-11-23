---
title: "Nature Remo と M5Stack で自宅の電力と気温のグラフを家族で見る"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["m5stack", "natureremo"]
published: true
---


# Nature Remo と M5Stack で自宅の電力と気温のグラフを家族で見る


## TL;DR

- [Nature Remo mini](https://nature.global/jp/nature-remo-mini)
  - 6,480円 (2020-11-23時点、Amazon.co.jp)
- [Nature Remo E lite](https://nature.global/jp/nature-remo-mini)
  - 14,800円 (2020-11-23時点、Amazon.co.jp)
- [M5Stack Gray](https://www.switch-science.com/catalog/3648/)
  - 4,950円 (2020-11-23時点、Amazon.co.jp)
- 計: 26,230円

を使って、電力使用量と気温を表示するようにした。

![](https://storage.googleapis.com/zenn-user-upload/zbt2jy63v0gccrokva4odhjj2xgr)

## コード

```c
#include "time.h"
#include "float.h"
#include "math.h"
#include <M5Stack.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

const char* ssid     = "<change here>";
const char* password = "<change here>";

const String natureremo_appliances_url = "https://api.nature.global/1/appliances";
const String natureremo_devices_url = "https://api.nature.global/1/devices";
const String natureremo_token = "<change here>";
const String natureremo_mini_id = "<change here>";
const String natureremo_e_lite_id = "<change here>";

struct tm timeinfo;

const unsigned int header_height = 36;
const unsigned int graph_left = 30;
const unsigned int graph_top = header_height + 1;
const unsigned int graph_width = 260;
const unsigned int num_points = graph_width - 2; // excluding the values at both ends
const unsigned int graph_height = 180;
const unsigned int watt_max = 3000;
const unsigned int temp_max = 30;

const unsigned int hour_font_witdh = 22;

float watt_history[num_points] = {};
float temp_history[num_points] = {};

#define FEQ(a, b) (fabsf(a - b) < FLT_EPSILON)
#define INVALID_VALUE -999

void setup(){
  M5.begin();
  M5.Power.begin();

  setup_wifi();
  setup_time();

  M5.Lcd.fillScreen(WHITE);

  srand(10);
}

void setup_wifi() {
  M5.Lcd.printf("Connecting to %s ", ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    M5.Lcd.print(".");
  }
  M5.Lcd.println(" CONNECTED");
}

void setup_time() {
  configTime(9 * 3600, 0, "ntp.nict.jp");
}


void loop() {
  if (WiFi.status() != WL_CONNECTED) { setup_wifi(); }

  update_time();

  if (passed_60_seconds()) {
    Serial.println("passed_60_seconds");
    print_header();
    print_graph();
  }

  delay(1000); // 1 second
}

int last_second = 99;

bool passed_60_seconds() {
  bool updated = false;

  if (timeinfo.tm_sec < last_second) {
    updated = true;
  }

  last_second = timeinfo.tm_sec;

  return updated;
}

void print_str(unsigned int x, unsigned int y, byte color, unsigned int siz, String str) {
  M5.Lcd.setCursor(x, y);
  M5.Lcd.setTextColor(color);
  M5.Lcd.setTextSize(siz);
  M5.Lcd.print(str);
}

void update_time() {
  getLocalTime(&timeinfo);
}

void print_header() {
  String ret = "";

  // time
  char now[5];
  sprintf(now, "%02d:%02d",
    timeinfo.tm_hour,
    timeinfo.tm_min
  );

  ret += now;
  ret += " ";

  // watt
  JsonArray appliances_doc = fetch_natureremo_json(natureremo_appliances_url);
  float watt = get_current_watt(appliances_doc);

  unshift_history(watt_history, watt);

  if (watt != INVALID_VALUE) {
    char watt_str[4];
    sprintf(watt_str, "%4.0f", watt);

    ret += watt_str;
    ret += "W ";
  }

  // temperature
  JsonArray devices_doc = fetch_natureremo_json(natureremo_devices_url);
  float temp = get_current_temperature(devices_doc);

  unshift_history(temp_history, temp);

  if (temp != INVALID_VALUE) {
    char temp_str[4];
    sprintf(temp_str, "%.1f", temp);

    ret += temp_str;
    ret += "C";
  }

  M5.Lcd.fillRect(0, 0, TFT_HEIGHT, header_height, WHITE);
  print_str(5, 5, BLACK, 3, ret);
}

void print_graph() {
  unsigned int hour_width_step = 60;

  // reset
  M5.Lcd.fillRect(0, graph_top, TFT_HEIGHT, TFT_WIDTH, WHITE);

  draw_watt_graph();
  draw_temp_graph();

  // graph line
  M5.Lcd.drawLine(graph_left, graph_top, graph_left, graph_top + graph_height, BLACK); // graph left vertical line
  M5.Lcd.drawLine(graph_left, graph_top + graph_height, graph_left + graph_width, graph_top + graph_height, BLACK); // graph horizontal line
  M5.Lcd.drawLine(graph_left + graph_width, graph_top, graph_left + graph_width, graph_top + graph_height, BLACK); // graph right vertical line
  M5.Lcd.drawLine(graph_left + graph_width, graph_top, graph_left + graph_width, graph_top + graph_height, BLACK); // graph right vertical line
  M5.Lcd.drawLine(graph_left, 40, graph_left + graph_width, 40, LIGHTGREY);// additional line
  M5.Lcd.drawLine(graph_left, 100, graph_left + graph_width, 100, LIGHTGREY);// additional line
  M5.Lcd.drawLine(graph_left, 160, graph_left + graph_width, 160, LIGHTGREY);// additional line

  // graph left vertical label
  print_str(5, 40, BLUE, 2, "3K");
  print_str(5, 100, BLUE, 2, "2K");
  print_str(5, 160, BLUE, 2, "1K");

  // graph right vertical label
  print_str(graph_left + graph_width + 3, 40, RED, 2, "30");
  print_str(graph_left + graph_width + 3, 100, RED, 2, "20");
  print_str(graph_left + graph_width + 3, 160, RED, 2, "10");

  // graph horizontal label
  for (unsigned int i = 0;; i++) {
    int x_hour = graph_left + graph_width - (hour_width_step * i) - timeinfo.tm_min - (hour_font_witdh / 2);
    if (x_hour < 0) { return; }

    int hour = timeinfo.tm_hour - i;
    if (hour < 0) { hour += 24; }

    char hour_str[2];
    sprintf(hour_str, "%02d", hour);
    print_str(x_hour, graph_top + graph_height + 3, RED, 2, hour_str);
  }
}

void unshift_history(float hist[], float val) {
  for (int i = num_points - 1; i > 0 ; i--) {
    hist[i] = hist[i - 1];
  }
  hist[0] = val;

  for (int i = 0; i < num_points; i++) {
    Serial.print(hist[i]);
    Serial.print(",");
  }
  Serial.print("\n");
}

void draw_watt_graph() {
  for (int i = 1; i <= num_points; i++) {
    if (FEQ(watt_history[i], INVALID_VALUE) || watt_history[i] == 0) { continue; }
    M5.Lcd.drawLine(
      graph_left + graph_width - i,
      int(graph_top + graph_height - (watt_history[i] * graph_height / watt_max)),
      graph_left + graph_width - i,
      graph_top + graph_height,
      BLUE
    );
  }
}

void draw_temp_graph() {
  for (int i = 1; i < num_points; i++) {
    Serial.print(i);
    Serial.print(",");
    Serial.print(temp_history[i]);
    Serial.print(",");
    Serial.print(INVALID_VALUE);
    Serial.print(",");
    Serial.print(FEQ(temp_history[i], INVALID_VALUE));
    Serial.print("\n");

    if (FEQ(temp_history[i], INVALID_VALUE) || temp_history[i] == 0) { continue; }
    if (FEQ(temp_history[i - 1], INVALID_VALUE) || temp_history[i - 1] == 0) { continue; }
    M5.Lcd.drawLine(
      graph_left + graph_width - (i - 1),
      int(graph_top + graph_height - (temp_history[i - 1] * graph_height / temp_max)),
      graph_left + graph_width - i,
      int(graph_top + graph_height - (temp_history[i] * graph_height / temp_max)),
      RED
    );
  }
}

JsonArray fetch_natureremo_json(String endpoint) {
    DynamicJsonDocument doc(10000);

    if ((WiFi.status() == WL_CONNECTED)) {
        HTTPClient http;
        http.begin(endpoint);
        http.addHeader("authorization", "Bearer " + natureremo_token);
        int httpCode = http.GET();
        if (httpCode > 0) {
          String json = http.getString();
          DeserializationError err = deserializeJson(doc, json);
          Serial.println(json);
          if (err) {
            Serial.print(F("deserializeJson() failed with code "));
            Serial.println(err.c_str());
          }
        } else {
            Serial.println("Error on HTTP request");
        }
        http.end();
    }
    return doc.as<JsonArray>();
}

float get_current_watt(JsonArray doc) {
  for (JsonObject v : doc) {
    if (v["id"].as<String>() == natureremo_e_lite_id) {
      for (JsonObject p : v["smart_meter"]["echonetlite_properties"].as<JsonArray>()) {
        if (p["name"] == "measured_instantaneous") {
          return atof(p["val"].as<const char*>());
        }
      }
    }
  }
  return INVALID_VALUE;
}

float get_current_temperature(JsonArray doc) {
  for (JsonObject v : doc) {
    if (v["id"].as<String>() == natureremo_mini_id) {
      return v["newest_events"]["te"]["val"].as<float>();
    }
  }
  return INVALID_VALUE;
}

```

## ざっくり説明

- JSON の取り扱いは [ArduinoJson](https://arduinojson.org/) を使った
- [Nature Remo API の Rate limit は 30回/5分](https://developer.nature.global/) なので、10秒おきに更新、とかはできない
- `TFT_WIDTH` と `TFT_HEIGHT` は向きに関係なく、 `TFT_HEIGHT` が長辺を表す
  - see https://github.com/m5stack/M5Stack/blob/db395470881ca8a4e928c995417631c94c645933/src/utility/ILI9341_Defines.h#L3-L4
- 家庭ネットワークの AP が遠いなどして Wifi の電波が弱いと、起動時の Wifi 接続が時間かかるので、携帯のテザリングなど使うとよい


## 効果

趣味でやったことだが、グラフが常に見えるということで自分だけでなく家族で「グラフはねてるから節電しよう！」という気分にさせてくれ、結果として電力消費量も減って来ている。

体感、月間で10%くらいマイナスなので、2年で元が取れるかな(笑)

## 感想

これだけで、プログラム容量の 73% を使ってしまい、デバイス開発における「効率的プログラミング」はなかなか難しい。

```
Sketch uses 967578 bytes (73%) of program storage space. Maximum is 1310720 bytes.
Global variables use 45092 bytes (13%) of dynamic memory, leaving 282588 bytes for local variables. Maximum is 327680 bytes.
```

また、今回は勉強を兼ねて、グラフを自前で実装したが、画面アップデート時の時刻軸の同期や、異常値の取り扱いなど気を使う箇所が多かった。

実際に、温度が異常時になる場合の取り回しはどうするのが効率的なのかわかっていない。
