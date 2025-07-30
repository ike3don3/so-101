AIロボットアーム so-101: 初期セットアップにおける通信障害の記録

このリポジトリは、ロボットアームプロジェクト「so-101」で使用するESP32とFEETECH STS3215サーボ間の基本的な通信を確立する過程で直面した、困難なデバッグ作業の全記録です。ブレッドボード上での高速な半二重UART通信のトラブルシューティングに関するケーススタディとして、この記録を公開します。

1. プロジェクトの目的

本プロジェクトの最終目標は、ロボットアーム「so-101」とROS対応コンピュータ「RDK X3」を統合し、AIによる画像認識を活用した知能型ブロック積み上げロボットを製作することでした。この目的を達成するためには、まずサーボのID設定など、基本的な制御を確立する必要がありました。

2. 直面した問題：深刻な通信障害

プロジェクトの最初のステップである、各サーボへのユニークなIDの設定が、根深い通信障害により失敗しました。様々なハードウェア構成とソフトウェアを試しましたが、ESP32がサーボからの応答パケットを安定して受信できず、「サーボが見つかりません」というエラーが続きました。

3. トラブルシューティングの道のり

根本原因の特定に至るまでの道のりは長く、以下の要因を一つずつ潰していく必要がありました。
- ファームウェア書き込み失敗: USBドライバの再インストールで解決。
- コンパイルエラーの多発: 安定版のArduino IDE (1.8.13)への変更で解決。
- 不適切な部品選定: 当初使用したレベルシフタ(PCA9306, TXS0108E)が、今回の高速半二重通信の用途に不適合であった。
- 物理層の問題: ブレッドボード環境が1Mbpsの高速通信において信号品質を劣化。

4. 根本原因と最終的な解決策

根本原因は単一の問題ではなく、以下の複合的な要因によるものだと結論付けられました。

1.  ロジックレベルの不整合: ESP32(3.3V)とサーボバス(5V)の信号電圧が異なっていた。
2.  信号品質の劣化: ブレッドボードの寄生容量とインダクタンスが、1Mbpsの高速信号の波形を破壊していた。
3.  部品の不適合: 自動方向検出型のレベルシフタは、プッシュプル方式の高速半二重UARTでは信号の衝突を引き起こした。

この問題を解決するための、最終的に成功した構成は以下の通りです。

- ハードウェア: 方向制御ピンを持つ専用のトランシーバICのTC74AC125Pを使用し、半二重通信を物理的に、そして能動的に制御する回路を構築。
- ソフトウェア: トランシーバの方向制御ピンをESP32のGPIOで手動操作し、通信プロトコルのタイミングに合わせて送受信モードを切り替え。


5. システム構成

- MCU: ESP32-DevKitC-32E
- サーボ: FEETECH STS3215
- トランシーバIC: TC74AC125P
- 電源 (サーボ用): 7.4V (LM2596で降圧)
- 電源 (ロジック用): 5V (独立ACアダプタ)
- 開発環境: Arduino IDE 1.8.13

6. 学んだこと

この一連のトラブルシューティングは、デジタル通信における物理層の重要性を学ぶ、非常に貴重な経験となりました。高速な信号は物理法則を無視できず、プロトタイピングの段階であっても、使用するプラットフォーム（ブレッドボードか、ユニバーサル基板か）や部品の選定が、プロジェクトの成否を大きく左右します。データシートを深く読み込み、理論に基づいた堅牢なハードウェアを構築することの重要性を痛感しました。



#so-101
SP32とFEETECH STS3215サーボの通信確立に向けたデバッグの記録

#include <HardwareSerial.h>
#include <SMS_STS.h>

// --- 設定項目 ---
HardwareSerial& ServoSerial = Serial2; // サーボ通信にSerial2を使用
SMS_STS SERVO;

const int RX_PIN = 16;  // ESP32のRXピン (サーボからの受信)
const int TX_PIN = 17;  // ESP32のTXピン (サーボへの送信)
const int DIR_PIN = 4; // 方向制御に使用するGPIOピン (TC74AC125Pの/OEピンへ)
const long SERVO_BAUDRATE = 1000000; // サーボの通信速度 (1Mbps)

// --- 生のデータパケットを方向制御しながら送信する補助関数 ---
void sendPacket(byte id, byte instruction, byte param1 = 0, byte param2 = 0) {
  byte length = 2; // 命令(1バイト) + チェックサム(1バイト)
  // パラメータの数に応じてパケット長を計算
  if (instruction == 0xFE || param1 != 0) length++;
  if (param2 != 0) length++;
  
  byte packet[length + 4];
  packet[0] = 0xFF; // ヘッダ1
  packet[1] = 0xFF; // ヘッда2
  packet[2] = id;
  packet[3] = length;
  packet[4] = instruction;
  
  int paramIndex = 5;
  if (instruction == 0xFE || param1 != 0) packet[paramIndex++] = param1;
  if (param2 != 0) packet[paramIndex++] = param2;

  // チェックサムの計算
  byte checksum = 0;
  for (int i = 2; i < (length + 3); i++) {
    checksum += packet[i];
  }
  packet[length + 3] = ~checksum; // ビット反転

  // ---送受信の切り替え ---
  digitalWrite(DIR_PIN, LOW);       // 送信モードに設定 (バッファを有効化)
  delayMicroseconds(50);            // ドライバが安定するのを待つ
  ServoSerial.write(packet, length + 4);
  ServoSerial.flush();              // 送信が完了するのを待つ
  delayMicroseconds(50);            // 最後のビットが送信されるのを待ってから方向を切り替える
  digitalWrite(DIR_PIN, HIGH);      // 受信モードに戻す (バッファを無効化)
}

// --- メインプログラム ---
void setup() {
  Serial.begin(115200);
  while (!Serial); // シリアルモニタが開くまで待つ

  ServoSerial.begin(SERVO_BAUDRATE, SERIAL_8N1, RX_PIN, TX_PIN);
  SERVO.pSerial = &ServoSerial;

  // 方向制御ピンを初期化
  pinMode(DIR_PIN, OUTPUT);
  digitalWrite(DIR_PIN, HIGH); // デフォルトは受信モード
  
  Serial.println("\n\n===== サーボID設定ツール (TC74AC125P 物理回路版) =====");
  Serial.println("サーボを1つだけ接続し、電源を入れてください。");
  Serial.println("------------------------------------------------------------------");
}

void loop() {
  Serial.println("\nサーボをスキャンしています (ID: 0-253)...");
  int current_id = -1;

  for (int i = 0; i < 254; i++) {
    while(ServoSerial.available()) ServoSerial.read(); // 受信バッファをクリア

    sendPacket(i, 0x01); // Ping命令(0x01)を送信
    delay(5); // サーボが応答するのを少し待つ
    
    if (ServoSerial.available() >= 6) {
      byte response[6];
      ServoSerial.readBytes(response, 6);
      // 正しい応答ヘッダとIDが返ってきたか確認
      if (response[0] == 0xFF && response[1] == 0xFF && response[2] == i) {
        current_id = i;
        Serial.print(">>> 発見！ 現在のサーボID: ");
        Serial.println(current_id);
        break;
      }
    }
  }

  if (current_id == -1) {
    Serial.println("\nエラー: スキャン範囲内にサーボが見つかりません。");
    Serial.println("配線、電源、サーボの状態を確認してください。");
    while(1); // エラー時は停止
  }

  Serial.print("\n新しいIDを入力してください (0-253): ");
  while (Serial.available() == 0) {
    delay(100);
  }
  int new_id = Serial.parseInt();
  Serial.println(new_id);
  
  // シリアルバッファに残った改行文字などをクリア
  while(Serial.available()) Serial.read();

  if (new_id < 0 || new_id > 253) {
      Serial.println("エラー: 無効なIDです。0から253の範囲で指定してください。");
      while(1);
  }

  Serial.println("\nIDの変更処理を開始します...");
  
  // --- FEETECHサーボ公式のID書き込み手順 ---
  Serial.println("1. EEPROMのロックを解除します...");
  sendPacket(current_id, 0xFE, 0); // unLockEprom(0)
  delay(10);

  Serial.println("2. 新しいIDを書き込みます...");
  sendPacket(current_id, 0x03, 0x05, (byte)new_id); // IDレジスタ(アドレス5)に新しいIDを書き込む
  delay(10);
  
  Serial.println("3. 新しいIDでEEPROMをロックします...");
  sendPacket((byte)new_id, 0xFE, 1); // LockEprom(1)
  delay(10);
  
  Serial.println("\nID変更コマンドのシーケンスを送信しました。結果を確認します...");
  
  // --- 最終確認 ---
  while(ServoSerial.available()) ServoSerial.read(); // バッファをクリア
  sendPacket((byte)new_id, 0x01); // 新しいIDにPingを送信
  delay(5);
  
  if (ServoSerial.available() >= 6) {
    byte final_response[6];
    ServoSerial.readBytes(final_response, 6);
    if (final_response[2] == new_id) {
       Serial.println("\n★★★★★ 成功！ 新しいIDでサーボからの応答を確認しました！ ★★★★★");
    }
  } else {
     Serial.println("\n警告: 検証に失敗しました。新しいIDでサーボが応答しませんでした。");
  }

  Serial.println("\n処理は完了しました。別のサーボを設定するには、ESP32のENボタンを押してリセットしてください。");
  while(1); // リセットされるまで停止
}

