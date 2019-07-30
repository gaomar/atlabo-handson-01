---
id: dist
{{if .Meta.Status}}status: {{.Meta.Status}}{{end}}
{{if .Meta.Summary}}summary: {{.Meta.Summary}}{{end}}
{{if .Meta.Author}}author: {{.Meta.Author}}{{end}}
{{if .Meta.Categories}}categories: {{commaSep .Meta.Categories}}{{end}}
{{if .Meta.Tags}}tags: {{commaSep .Meta.Tags}}{{end}}
{{if .Meta.Feedback}}feedback link: {{.Meta.Feedback}}{{end}}
{{if .Meta.GA}}analytics account: {{.Meta.GA}}{{end}}

---

#LINE Things + LINE Pay連携ハンズオン

## LINE Payサンドボックス環境設定をしよう！

### 1-1. LINE Payのサンドボックス環境を作成する
下記URLからサンドボックス環境を作成します。
[https://pay.line.me/jp/developers/techsupport/sandbox/creation?locale=ja_JP](https://pay.line.me/jp/developers/techsupport/sandbox/creation?locale=ja_JP)

メールアドレスを入力して［Submit］ボタンをクリックします。
![s100](images/s100.png)

### 1-2. LINE Pay Homeにログインする
設定したメールアドレスにログインするためのIDとパスワードが送られてきます。
`LINE Pay Home` 部分をクリックして、ログインします。

[https://pay.line.me/login/](https://pay.line.me/login/)

![s101](images/s101.png)

メールに記載されているIDとパスワードでログインします。

![s102](images/s102.png)

左側メニューの決済連動管理にある、連動キー管理をクリックして、メールに記載されているパスワードを入力してから、［確認］ボタンをクリックします。

![s103](images/s103.png)

### 1-3. IDとSecret Keyをメモする
`Channel ID` と `Channel Secret Key` の値をそれぞれメモしておきます。

![s104](images/s104.png)

## ローカル環境を構築しよう！
### 2-1. GitHubからプロジェクトを取得する
ローカルの環境にプロジェクトを構築して実際の決済処理の動きを確認してみましょう。
適当なフォルダを作成して、GitHubからクローンしてきます。

```shell
$ mkdir my-atlabo-linepay-demo
$ cd my-atlabo-linepay-demo
$ git clone -b step1 https://github.com/gaomar/atlabo-linepay-demo.git
```

### 2-2. Visual Studio Codeに展開する
Visual Studio Codeを開いて、ワークスペースに `atlabo-linepay-demo` のフォルダを指定します。

![s200](images/s200.png)

表示-ターミナルを開いてください

![s201](images/s201.png)

下記コマンドを実行してインストールしてください。

```shell
$ npm install
```

### 2-3. 起動確認
インストールできたら下記コマンドを実行してブラウザで確認してください。

```shell
$ npm run serve
```

無事起動できたらこのURLで確認してください。
[http://localhost:8080/](http://localhost:8080/)

![s202](images/s202.png)

起動確認できたらCTRL + C で終了してください。

### 2-4. モジュールを追加する
LINE Payを実行するために必要なモジュールをインストールします。
下記コマンドを実行してモジュールをインストールしてください。

```shell
$ npm i -s line-pay netlify-lambda uuid
```

### 2-5. 環境変数にLINE Payの設定をする
.envというファイルに環境変数を設定します。
`1-3`で取得した`Channel ID` と `Channel Secret Key` の値を書き込みます

```.env
VUE_APP_LINE_PAY_CHANNEL_ID=XXXXXXXXXX
VUE_APP_LINE_PAY_CHANNEL_SECRET=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

![s203](images/s203.png)

### 2-6. 決済処理を記述する
LINE Payの決済処理を記述します。`src/lambda/pay.js` ファイルを開いて、下記プログラムを記述します。

![s204](images/s204.png)

```javascript
// LINE Pay処理を実装する
"use strict"

import { URLSearchParams } from 'url'
const uuid = require("uuid/v4")

const line_pay = require("line-pay")
const pay = new line_pay({
  channelId: process.env.VUE_APP_LINE_PAY_CHANNEL_ID,
  channelSecret: process.env.VUE_APP_LINE_PAY_CHANNEL_SECRET,
  isSandbox: true  // サンドボックス環境
})

exports.handler = function(event, context, callback) {
  const body = event.body
  let params = new URLSearchParams(body)
  const type = params.get('type')

  const headers = {
    'Access-Control-Allow-Headers': '*',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'POST',
    'Content-Type': 'application/json'
  }

  if (type === 'reserve') {
    // 決済予約
    let options = {
      productName: "ロック解除",
      amount: 1,                 // 金額（この場合は1円）
      currency: "JPY",           // 日本円
      orderId: uuid(),
      confirmUrl: process.env.VUE_APP_LINE_PAY_CONFIRM_URL
    }
    pay.reserve(options).then((response) => {
      let reservation = options
      reservation.transactionId = response.info.transactionId
      reservation.paymentUrl = response.info.paymentUrl.web
  
      callback(null, {
        statusCode: 200,
        body: JSON.stringify(reservation),
        headers: headers
      })  
    })
  } else if (type === 'confirm') {
    // 決済処理
    const transactionId = params.get('transactionId')
    const reservations = JSON.parse(params.get('reservations'))

    let confirmation = {
      transactionId: transactionId,
      amount: reservations.amount,
      currency: reservations.currency
    }

    pay.confirm(confirmation).then((response) => {
      callback(null, {
        statusCode: 200,
        body: '決済完了しました！',
        headers: headers
      })
    })
  } else {
    callback(null, {
      statusCode: 400,
      body: 'APIエラー',
      headers: headers
    })
  }
}
```

### 2-7. 画面遷移を有効にする
`src/router.js` ファイルを編集して、決済画面に遷移できるようにします。
16行目と27行目をコメントアウトします。

![s205](images/s205.png)

### 2-8. ローカル環境で実行する
ターミナルから下記コマンドを実行します。

```shell
$ npm run serve
```
![s206](images/s206.png)

もう一つ新しいターミナルを開いて、下記コマンドを実行します。

```shell
$ npm run lambda

Lambda server is listening on 9000 // ←これが出てくればOK
```

![s207](images/s207.png)

[http://localhost:8080](http://localhost:8080)にアクセスすると決済処理を確認することができます。
携帯からQRコードを読み取っても決済することができます。

※サンドボックス環境なので、実際に決済はされません！

![s208](images/s208.png)

確認できたら、`Ctrl + C` でプログラムを終了しておいてください。

### 2-9. serveoで外部からアクセスしてみる
ターミナルをひらいて、下記コマンドを実行します。
8080ポートを指定します。

```shell
$ ssh -o ServerAliveInterval=120 -R 80:localhost:8080 serveo.net

Forwarding HTTP traffic from https://xxxxxx.serveo.net  // httpsの値をコピー
```

もう一つ新しいターミナルをひらいて下記コマンドを実行します。
9000ポートを指定します。

```shell
$ ssh -o ServerAliveInterval=120 -R 80:localhost:9000 serveo.net

Forwarding HTTP traffic from https://xxxxxx.serveo.net  // httpsの値をコピー
```

発行されたURLを.envファイルに記述します。

|項目|値|
|:--|:--|
|VUE_APP_LINE_PAY_BASE_URL|https://xxxxxx.serveo.net <br/>※9000ポートの値（serveoの値は毎回実行毎に生成されます）|
|VUE_APP_LINE_PAY_CONFIRM_URL|https://xxxxxx.serveo.net/pay/confirm <br/>※8080ポートの値（serveoの値は毎回実行毎に生成されます）|

![s209](images/s209.png)

8080で発行されたserveoのURLにアクセスします。
![s210](images/s210.png)

## LINEチャネルとLIFFを作成しよう！
### 3-1. LINEチャネルを作成する
いよいよLINE Thingsのキモの部分を作成していきます。
LINE Developerにアクセスしてください。

<font color='red'>**※ログインする際はLINEアプリで使っているアカウントと同一のものでログインしてください！**</font>

[https://developers.line.biz/ja/](https://developers.line.biz/ja/)

プロバイダーをまだ持っていない方は新規で作成してください。

新規チャネルを作成します。

![s300](images/s300.png)

MessagingAPIを選択します。

![s301](images/s301.png)

|項目|値|
|:--|:--|
|①アプリ名|M5Pay|
|②説明|M5Pay|
|③大業種|個人|
|③小業種|個人（その他）|
|④メールアドレス|ご自身のメールアドレス|

![s302](images/s302.png)

同意するをクリック
![s303](images/s303.png)

2つのチェックを入れて作成ボタンをクリックします。

![s304](images/s304.png)

作成されたチャネルをクリックします。

![s305](images/s305.png)

### 3-2. アクセストークンを発行する
少し下にスクロールするとメッセージ送信設定という部分があります。
そこの［再発行］ボタンをクリックします。

![s306](images/s306.png)

出てきたポップアップはそのまま［再発行］ボタンをクリックします。

![s307](images/s307.png)

発行されたアクセストークンは後で使うのでメモしておきましょう。

![s308](images/s308.png)

### 3-3. LINE Botと友だちになる
下にスクロールするとQRコードがあるので、LINEアプリからQRを読み取って、
Botと友だちになっておいてください。

![s309-1](images/s309-1.png)

### 3-4. LIFFを設定する
LIFFとは<font color='red'>LI</font>NE <font color='red'>F</font>ront-end <font color='red'>F</font>rameworkの略で、LINE内で動作するウェブアプリのプラットフォームです。

表示できるサイズは3種類あります。用途に合わせて使い分けましょう。

![s309](images/s309.png)

LIFFタブをクリックして、作成ボタンをクリックします。

![s310](images/s310.png)

|項目|値|
|:--|:--|
|①名前|M5Pay|
|②サイズ|Full|
|③エンドポイントURL|https://xxxxxx.serveo.net<br/>※8080ポートの値（serveoの値は毎回実行毎に生成されます）|
|④オプション|必ずONにする|

![s311](images/s311.png)

## サービスUUIDを生成しよう！
### 4-1. サービスUUIDを生成する

[のびすけ](https://twitter.com/n0bisuke)さんが作ったツールをありがたく使います。
下記にアクセスしてください。

[https://n0bisuke.github.io/linethingsgen/](https://n0bisuke.github.io/linethingsgen/)

左側メニューのSettingをクリックして、`3-2`で生成したアクセストークンを貼り付けます。
［保存］ボタンをクリックします。

![s400](images/s400.png)

左側メニューのCreate Productをクリックして、プルダウンメニューから`3-4`で作成したLIFFアプリを指定します。
トライアルプロダクトの名前を入力して［作成］ボタンをクリックします。

![s401](images/s401.png)

生成されたサービスUUIDをメモしておきましょう。

![s402](images/s402.png)

## LINE Thingsに対応しよう！
### 5-1. .envファイルを修正する
`.env`ファイルの`VUE_APP_USER_SERVICE_UUID`に先程生成されたサービスUUIDをコピペします。

### 5-2. プログラムを修正する
Vue.js側のプログラムを修正します。`src/views/Confirm.vue`
下記コードをコピペしてください。

コードはこちらからもコピペできます

[https://raw.githubusercontent.com/gaomar/atlabo-linepay-demo/master/src/views/Confirm.vue](https://raw.githubusercontent.com/gaomar/atlabo-linepay-demo/master/src/views/Confirm.vue)

```javascript
<template>
  <div class="confirm">
    <h1>{{status}}</h1>
    <h2>{{bleStatus}}</h2>
  </div>
</template>

<script>
export default {
  data () {
    return {
      status: '',
      USER_SERVICE_UUID: process.env.VUE_APP_USER_SERVICE_UUID,
      LED_CHARACTERISTIC_UUID: 'E9062E71-9E62-4BC6-B0D3-35CDCD9B027B', /* トライアルは固定値 */
      bleConnect: false,
      bleStatus: 'デバイス未接続',
      characteristic: '',
      code: ''
    }
  },
  methods: {
    confirmAction: function () {
      if (!this.$route.query.transactionId){
        throw new Error("Transaction Id not found.")
      }

      // Retrieve the reservation from database.
      var reservation = JSON.parse(sessionStorage.getItem(this.$route.query.transactionId))
      if (!reservation){
          throw new Error("Reservation not found.")
      }

      var params = new URLSearchParams()
      params.set('type', 'confirm')
      params.set('transactionId', this.$route.query.transactionId)
      params.set('reservations', JSON.stringify(reservation))

      axios.post(process.env.VUE_APP_LINE_PAY_BASE_URL + '/.netlify/functions/pay', params)
        .then(response => {
          sessionStorage.clear()
          this.status = '決済完了しました！'
      
          this.code = '1234'
          const ch_array = this.code.split("");
          for(let i = 0; i < 16; i = i + 1) {
            ch_array[i] = (new TextEncoder('ascii')).encode(ch_array[i]);
          }

          this.characteristic.writeValue(new Uint8Array(ch_array)
          ).catch(error => {
            this.bleStatus = error.message
          })

        })
    },
    // BLEが接続できる状態になるまでリトライ
    liffCheckAvailablityAndDo: async function (callbackIfAvailable) {
      try {
        const isAvailable = await liff.bluetooth.getAvailability();
        if (isAvailable) {
          callbackIfAvailable()
        } else {
          // リトライ
          this.bleStatus = `Bluetoothをオンにしてください。`
          setTimeout(() => this.liffCheckAvailablityAndDo(callbackIfAvailable), 10000)
        }
      } catch (error) {
        this.bleStatus = `Bluetoothをオンにしてください。`
      }
    },
    // サービスとキャラクタリスティックにアクセス
    liffRequestDevice: async function () {
      const device = await liff.bluetooth.requestDevice()
      await device.gatt.connect()
      const service = await device.gatt.getPrimaryService(this.USER_SERVICE_UUID)
      service.getCharacteristic(this.LED_CHARACTERISTIC_UUID).then(characteristic => {
        this.characteristic = characteristic
        this.bleConnect = true
        this.bleStatus = `デバイスに接続しました！`
        this.confirmAction()
      }).catch(error => {
        this.bleConnect = true
        this.bleStatus = `デバイス接続に失敗=${error.message}`
      })
    },
    initializeLiff: async function(){
      await liff.initPlugins(['bluetooth']);
      this.liffCheckAvailablityAndDo(() => this.liffRequestDevice())
    }
  },
  mounted: function () {
    liff.init(
        () => this.initializeLiff()
    )
  }
}
</script>
```

### 5-3. プログラムを実行しておく
修正したら、一度Ctrl + Cでプログラムを終了しておいて、再度実行します。

```shell
$ npm run serve
```

## M5Stackにコードを書き込む
### 6-1. M5Stackにコードを書き込む
Arduino IDEを開いて下記コードを入力します。
11行目のサービスUUIDは4-1で生成した値を入力してください。

コードはこちらからもコピペできます

[https://raw.githubusercontent.com/gaomar/atlabo-linepay-demo/master/src/M5Pay/M5Pay-M5Stack/M5Pay-M5Stack.ino](https://raw.githubusercontent.com/gaomar/atlabo-linepay-demo/master/src/M5Pay/M5Pay-M5Stack/M5Pay-M5Stack.ino)

```c
#include <M5Stack.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

// Device Name: Maximum 30 bytes
#define DEVICE_NAME "M5Pay"

// あなたのサービスUUIDを貼り付けてください
#define USER_SERVICE_UUID "＜あなたのサービスUUID＞"
// Notify UUID: トライアル版は値が固定される
#define NOTIFY_CHARACTERISTIC_UUID "62FBD229-6EDD-4D1A-B554-5C4E1BB29169"
// PSDI Service UUID: トライアル版は値が固定される
#define PSDI_SERVICE_UUID "E625601E-9E55-4597-A598-76018A0D293D"
// LIFFからのデータ UUID: トライアル版は値が固定される
#define WRITE_CHARACTERISTIC_UUID "E9062E71-9E62-4BC6-B0D3-35CDCD9B027B"
// PSDI CHARACTERISTIC UUID: トライアル版は値が固定される
#define PSDI_CHARACTERISTIC_UUID "26E2B12B-85F0-4F3F-9FDD-91D114270E6E"

BLEServer* thingsServer;
BLESecurity* thingsSecurity;
BLEService* userService;
BLEService* psdiService;
BLECharacteristic* psdiCharacteristic;
BLECharacteristic* notifyCharacteristic;
BLECharacteristic* writeCharacteristic;

bool deviceConnected = false;
bool oldDeviceConnected = false;
bool unlockFlg = false;

// 認証コード
int myNumber = 1234;

class serverCallbacks: public BLEServerCallbacks {

  // デバイス接続
  void onConnect(BLEServer* pServer) {
    deviceConnected = true;

    // 一度認証されないとコードは生成しない
    if (unlockFlg) {
      unlockFlg = false;
    }
  };

  // デバイス未接続
  void onDisconnect(BLEServer* pServer) {
    deviceConnected = false;

    if (unlockFlg) {
      unlockFlg = false;
    }
  }
};

// LIFFから送信されるデータ
class writeCallback: public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *bleWriteCharacteristic) {
    // LIFFから来るデータを取得
    std::string value = bleWriteCharacteristic->getValue();
    int myNum = atoi(value.c_str());

    // 認証コードと一致しているか確認
    if (myNumber == myNum) {
      unlockFlg = true;    
    }    
  }
};

void setup() {
  Serial.begin(115200);
  BLEDevice::init("");
  BLEDevice::setEncryptionLevel(ESP_BLE_SEC_ENCRYPT_NO_MITM);

  // Security Settings
  BLESecurity *thingsSecurity = new BLESecurity();
  thingsSecurity->setAuthenticationMode(ESP_LE_AUTH_BOND);
  thingsSecurity->setCapability(ESP_IO_CAP_NONE);
  thingsSecurity->setInitEncryptionKey(ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK);

  setupServices();
  startAdvertising();

  // put your setup code here, to run once:
  // M5Stack LCD Setup
  M5.begin(true, false, false);
  M5.Lcd.clear(BLACK);
  M5.Lcd.setTextColor(YELLOW);
  M5.Lcd.setTextSize(2);
  M5.Lcd.setTextDatum(MC_DATUM);  
  M5.Lcd.drawString("Ready to Connect",160,10);
  Serial.println("Ready to Connect");

}

void loop() {
  if (!deviceConnected && oldDeviceConnected) {
    delay(500); // Wait for BLE Stack to be ready
    thingsServer->startAdvertising(); // Restart advertising
    oldDeviceConnected = deviceConnected;
    Serial.println("Restart!");
    M5.Lcd.clear(BLACK);
    M5.Lcd.setTextColor(YELLOW);
    M5.Lcd.setTextSize(2);
    M5.Lcd.setTextDatum(MC_DATUM);  
    M5.Lcd.drawString("Ready to Connect",160,10);
  }

  // Connection
  if (deviceConnected && !oldDeviceConnected) {
    oldDeviceConnected = deviceConnected;
    M5.Lcd.clear(BLACK);
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setTextSize(2);
    M5.Lcd.setTextDatum(MC_DATUM);  
    M5.Lcd.drawString("Connect",160,10);
    
  }

  if (unlockFlg){
    M5.Lcd.clear(GREEN);
    M5.Lcd.setTextColor(BLACK);
    M5.Lcd.setTextSize(2);
    M5.Lcd.setTextDatum(MC_DATUM);  
    M5.Lcd.drawString("UNLOCK",160,10);

    delay(5000);
    unlockFlg = false;      
  }

  M5.update();

}

// サービス初期化
void setupServices(void) {
  // Create BLE Server
  thingsServer = BLEDevice::createServer();
  thingsServer->setCallbacks(new serverCallbacks());

  // Setup User Service
  userService = thingsServer->createService(USER_SERVICE_UUID);

  // LIFFからのデータ受け取り設定
  writeCharacteristic = userService->createCharacteristic(WRITE_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_WRITE);
  writeCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  writeCharacteristic->setCallbacks(new writeCallback());

  // Notifyセットアップ
  notifyCharacteristic = userService->createCharacteristic(NOTIFY_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_NOTIFY);
  notifyCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  BLE2902* ble9202 = new BLE2902();
  ble9202->setNotifications(true);
  ble9202->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  notifyCharacteristic->addDescriptor(ble9202);

  // Setup PSDI Service
  psdiService = thingsServer->createService(PSDI_SERVICE_UUID);
  psdiCharacteristic = psdiService->createCharacteristic(PSDI_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_READ);
  psdiCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);

  // Set PSDI (Product Specific Device ID) value
  uint64_t macAddress = ESP.getEfuseMac();
  psdiCharacteristic->setValue((uint8_t*) &macAddress, sizeof(macAddress));

  // Start BLE Services
  userService->start();
  psdiService->start();
}

void startAdvertising(void) {
  // Start Advertising
  BLEAdvertisementData scanResponseData = BLEAdvertisementData();
  scanResponseData.setFlags(0x06); // GENERAL_DISC_MODE 0x02 | BR_EDR_NOT_SUPPORTED 0x04
  scanResponseData.setName(DEVICE_NAME);

  thingsServer->getAdvertising()->addServiceUUID(userService->getUUID());
  thingsServer->getAdvertising()->setScanResponseData(scanResponseData);
  thingsServer->getAdvertising()->start();
}
```

`M5Stack-Core-ESP32`にする

![s500](images/s500.png)

コードを書き換えたら書き込みボタンをクリックします

![s501](images/s501.png)

LINEアプリをひらいて、M5Payをクリックして権限を許可してペアリングを完了してください。

![s510](images/s510.png)

M5Payをタップすると、LINE Payボタンが表示されているので、それをクリックして決済を完了させてください。

![s520](images/s520.png)

## M5StickCにコードを書き込む
### 7-1. M5StickCにコードを書き込む
Arduino IDEを開いて下記コードを入力します。
11行目のサービスUUIDは4-1で生成した値を入力してください。

コードはこちらからもコピペできます

[https://raw.githubusercontent.com/gaomar/atlabo-linepay-demo/master/src/M5Pay/M5Pay-M5Stick/M5Pay-M5Stick.ino](https://raw.githubusercontent.com/gaomar/atlabo-linepay-demo/master/src/M5Pay/M5Pay-M5Stick/M5Pay-M5Stick.ino)

```c
#include <M5StickC.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

// Device Name: Maximum 30 bytes
#define DEVICE_NAME "M5Pay"

// あなたのサービスUUIDを貼り付けてください
#define USER_SERVICE_UUID "＜あなたのサービスUUID＞"
// Notify UUID: トライアル版は値が固定される
#define NOTIFY_CHARACTERISTIC_UUID "62FBD229-6EDD-4D1A-B554-5C4E1BB29169"
// PSDI Service UUID: トライアル版は値が固定される
#define PSDI_SERVICE_UUID "E625601E-9E55-4597-A598-76018A0D293D"
// LIFFからのデータ UUID: トライアル版は値が固定される
#define WRITE_CHARACTERISTIC_UUID "E9062E71-9E62-4BC6-B0D3-35CDCD9B027B"
// PSDI CHARACTERISTIC UUID: トライアル版は値が固定される
#define PSDI_CHARACTERISTIC_UUID "26E2B12B-85F0-4F3F-9FDD-91D114270E6E"

BLEServer* thingsServer;
BLESecurity* thingsSecurity;
BLEService* userService;
BLEService* psdiService;
BLECharacteristic* psdiCharacteristic;
BLECharacteristic* notifyCharacteristic;
BLECharacteristic* writeCharacteristic;

bool deviceConnected = false;
bool oldDeviceConnected = false;
bool unlockFlg = false;

// 認証コード
int myNumber = 1234;

class serverCallbacks: public BLEServerCallbacks {

  // デバイス接続
  void onConnect(BLEServer* pServer) {
    deviceConnected = true;

    // 一度認証されないとコードは生成しない
    if (unlockFlg) {
      unlockFlg = false;
    }
  };

  // デバイス未接続
  void onDisconnect(BLEServer* pServer) {
    deviceConnected = false;

    if (unlockFlg) {
      unlockFlg = false;
    }
  }
};

// LIFFから送信されるデータ
class writeCallback: public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *bleWriteCharacteristic) {
    // LIFFから来るデータを取得
    std::string value = bleWriteCharacteristic->getValue();
    int myNum = atoi(value.c_str());

    // 認証コードと一致しているか確認
    if (myNumber == myNum) {
      unlockFlg = true;    
    }    
  }
};

void setup() {
  Serial.begin(115200);
  BLEDevice::init("");
  BLEDevice::setEncryptionLevel(ESP_BLE_SEC_ENCRYPT_NO_MITM);

  // Security Settings
  BLESecurity *thingsSecurity = new BLESecurity();
  thingsSecurity->setAuthenticationMode(ESP_LE_AUTH_BOND);
  thingsSecurity->setCapability(ESP_IO_CAP_NONE);
  thingsSecurity->setInitEncryptionKey(ESP_BLE_ENC_KEY_MASK | ESP_BLE_ID_KEY_MASK);

  setupServices();
  startAdvertising();

  // put your setup code here, to run once:
  // M5Stick LCD Setup
  M5.begin(true, false, false);

  // 横向き
  M5.Lcd.setRotation(3);
  
  M5.Lcd.fillScreen(TFT_BLACK);
  M5.Lcd.setTextColor(YELLOW);
  M5.Lcd.setTextDatum(MC_DATUM);  
  M5.Lcd.drawString("Ready to Connect",80,10);
  Serial.println("Ready to Connect");

}

void loop() {
  if (!deviceConnected && oldDeviceConnected) {
    delay(500); // Wait for BLE Stack to be ready
    thingsServer->startAdvertising(); // Restart advertising
    oldDeviceConnected = deviceConnected;
    Serial.println("Restart!");
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextColor(YELLOW);
    M5.Lcd.setTextDatum(MC_DATUM);  
    M5.Lcd.drawString("Ready to Connect",80,10);
  }

  // Connection
  if (deviceConnected && !oldDeviceConnected) {
    oldDeviceConnected = deviceConnected;
    M5.Lcd.fillScreen(TFT_BLACK);
    M5.Lcd.setTextColor(GREEN);
    M5.Lcd.setTextDatum(MC_DATUM);  
    M5.Lcd.drawString("Connect",80,10);
    
  }

  if (unlockFlg){
    M5.Lcd.fillScreen(TFT_GREEN);
    M5.Lcd.setTextColor(BLACK);
    M5.Lcd.setTextDatum(MC_DATUM);  
    M5.Lcd.drawString("UNLOCK",80,10);

    delay(5000);
    unlockFlg = false;      
  }

}

// サービス初期化
void setupServices(void) {
  // Create BLE Server
  thingsServer = BLEDevice::createServer();
  thingsServer->setCallbacks(new serverCallbacks());

  // Setup User Service
  userService = thingsServer->createService(USER_SERVICE_UUID);

  // LIFFからのデータ受け取り設定
  writeCharacteristic = userService->createCharacteristic(WRITE_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_WRITE);
  writeCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  writeCharacteristic->setCallbacks(new writeCallback());

  // Notifyセットアップ
  notifyCharacteristic = userService->createCharacteristic(NOTIFY_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_NOTIFY);
  notifyCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  BLE2902* ble9202 = new BLE2902();
  ble9202->setNotifications(true);
  ble9202->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);
  notifyCharacteristic->addDescriptor(ble9202);

  // Setup PSDI Service
  psdiService = thingsServer->createService(PSDI_SERVICE_UUID);
  psdiCharacteristic = psdiService->createCharacteristic(PSDI_CHARACTERISTIC_UUID, BLECharacteristic::PROPERTY_READ);
  psdiCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENCRYPTED | ESP_GATT_PERM_WRITE_ENCRYPTED);

  // Set PSDI (Product Specific Device ID) value
  uint64_t macAddress = ESP.getEfuseMac();
  psdiCharacteristic->setValue((uint8_t*) &macAddress, sizeof(macAddress));

  // Start BLE Services
  userService->start();
  psdiService->start();
}

void startAdvertising(void) {
  // Start Advertising
  BLEAdvertisementData scanResponseData = BLEAdvertisementData();
  scanResponseData.setFlags(0x06); // GENERAL_DISC_MODE 0x02 | BR_EDR_NOT_SUPPORTED 0x04
  scanResponseData.setName(DEVICE_NAME);

  thingsServer->getAdvertising()->addServiceUUID(userService->getUUID());
  thingsServer->getAdvertising()->setScanResponseData(scanResponseData);
  thingsServer->getAdvertising()->start();
}
```

![s600](images/s600.png)

`ESP32 Pico Kit`にする

![s601](images/s601.png)

LINEアプリをひらいて、M5Payをクリックして権限を許可してペアリングを完了してください。

![s510](images/s510.png)

M5Payをタップすると、LINE Payボタンが表示されているので、それをクリックして決済を完了させてください。

![s602](images/s602.png)