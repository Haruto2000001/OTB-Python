## 仕様調査 2026-01-14

Read_sessantaquattroplus.py の デフォルトそのままで動作している。

### 機械設定パラメータの意味と値

- FSAMP: サンプリング周波数の選択 (0-3)
  - MODE=0 のときの周波数テーブル: 0=500, 1=1000, 2=**2000**, 3=4000 (Hz)
  - 設定値: 2
- NCH: チャンネル数の選択 (0-3)
  - MODE=0 のときのチャンネル数: 0=16, 1=24, 2=40, 3=**72**
  - 設定値: 3
- MODE: 動作モード (0-3)
  - 設定値: 0 (標準モード)
- HRES: 高分解能モード
  - 設定値: 0 (無効)
- HPF: ハイパスフィルタ
  - 設定値: 0 (無効)
- EXTEN: 外部トリガ
  - 設定値: 0 (無効)
- TRIG: トリガモード
  - 設定値: 0 (無効)
- REC: 録音
  - 設定値: 0 (無効)
- GO: 取得開始
  - 設定値: 1 (有効)

上記の設定から、device.frequency は **2000** Hz、device.nchannels は 72 になる。

## 信号の取得から描画までのフロー

機械からの信号は `DataReceiverThread.run()` で受信され、トラックごとのバッファへ分配され、`QTimer` で定期描画される。
処理の流れは以下。

1. 受信: ソケットからサンプルブロックを読み込み、int16 で展開して (ch, samples) に整形

```python
data = self.client_socket.recv(
    self.device.nchannels * 2 * (self.device.frequency // 16)
)
unpacked_data = struct.unpack(f">{len(data) // 2}h", data)
reshaped_data = np.array(unpacked_data).reshape((-1, self.device.nchannels)).T
```

- `self.client_socket` は `SessantaquattroPlus.start_server()` 内で `self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)`として定義されており、中身は`socket.socket`である。(すでに `device.start_server()`で TCP 接続接続が開始されている) [【Python】プロセス間通信の基本(ソケット)](https://zenn.dev/shimiyu/articles/336a06c3a65f0a)

- `self.device.nchannels * 2 * (self.device.frequency // 16)` は、「一度に送られてくるパケットには、チャンネル数 ×2 バイト(int16)×1/16 秒分のサンプル数」という意味である.

  - `nchannels`: チャンネル数
  - `2`: 1 サンプルあたり 2 バイト (`struct.unpack(..., "h")`の h は int16)
  - `frequency // 16`: 1/16 秒分のサンプル数（1 回で受け取るブロック長）

- `reshape((-1, self.device.nchannels)).T` は、フラットな int16 配列を「(サンプル数, チャンネル数)」に整形してから転置することで、「(チャンネル数, サンプル数)」にする。
  例: `nchannels=72` で `frequency//16=125` のとき、受信した int16 の総数は `72 * 125 = 9000`。
  `np.array(unpacked_data)` の shape は `(9000,)`、`reshape((-1, 72))` で `(125, 72)`、`.T` で `(72, 125)` になる。

まとめると、一度の受信で 72 チャンネルそれぞれに 125 サンプルの int16 が入る。これは 1/16 秒分のデータである。

2. 分配: トラックごとのチャンネル範囲に切り出して `Track.feed()` へ投入

```python
channel_index = 0
for track in self.tracks:
    track.feed(
        reshaped_data[channel_index : channel_index + track.num_channels, :]
    )
    channel_index += track.num_channels
```

- `self.tracks`には`Soundtrack.init_tracks()`で追加された 6 種類のトラックが含まれている。

`track_info` に含まれるトラック名と意味（72ch 構成時）は以下。

- `HDsEMG 64 channels`: 主信号の HDsEMG（0-63ch）
- `AUX 1`: 補助入力 1（64ch）
- `AUX 2`: 補助入力 2（65ch）
- `Quaternions`: 姿勢クォータニオン（4ch, 66-69ch）
- `Buffer`: バッファ/状態系の単一チャンネル（70ch）
- `Ramp`: ランプ/状態系の単一チャンネル（71ch）

それぞれの track は`self.buffer`というインスタンス変数を持っており、
`self.buffer = np.zeros((num_channels, int(plot_time * frequency)))`で初期化されている。
つまりこのバッファに、グラフに表示する時間分の信号値が保存されている。(バッファがないと無限長になってしまう)
`plot_time`は`QtWidgets.QComboBox()`で`["100ms", "250ms", "500ms", "1s", "5s", "10s"]`から選択することができる。

3. バッファリング: `Track.feed()` がリングバッファに書き込み

reshaped_data を packet としてバッファに書き込む(feed())

```python
if self.buffer_index + packet_size > self.buffer.shape[1]:
    end_space = self.buffer.shape[1] - self.buffer_index
    if end_space > 0:
        self.buffer[:, self.buffer_index :] = packet[:, :end_space]
    self.buffer[:, : packet_size - end_space] = packet[:, end_space:]
    self.buffer_index = packet_size - end_space
else:
    self.buffer[:, self.buffer_index : self.buffer_index + packet_size] = packet
    self.buffer_index = (self.buffer_index + packet_size) % self.buffer.shape[1]
```

バッファの長さは表示時間分なので、最大まで行ったら idx=0 に戻らなくてはならない。
どの位置から更新するかを、`self.buffer_index`で持っている。
今回のパケットがバッファに収まりきらない場合は、飛び出た部分だけ idx=0 の方に詰め込む。

4. 描画: `QTimer` が `update_plot()` を呼び、各トラックの `Track.draw()` がバッファを描画

```python
def update_plot(self):
    if not self.is_paused:
        for track in self.tracks:
            track.draw()

def draw(self):
    for index, curve in enumerate(self.curves):
        curve.setData(
            self.time_array,
            self.buffer[index, :] * self.conv_fact + (self.offset * index),
        )
```

補足: `conv_fact` はスケーリング係数、`offset` はチャンネルごとの縦方向オフセット。

## 元の説明

OT Bioelettronica Python Communication Scripts
This repository contains Python scripts developed for communicating with OT Bioelettronica devices. Two versions are provided, based on different graphical libraries: PyQt and Matplotlib.

Folder Structure
PyQt/
This folder contains communication scripts using the PyQt framework.
It provides a more optimized and responsive interface compared to the Matplotlib version, especially for real-time plotting and interaction. This version is recommended for practical use.

Matplotlib/
This folder contains an alternative implementation using Matplotlib.
Although the plotting performance is less efficient, this version is kept for completeness and educational purposes.

It includes the following subfolders:

device_communication/
Scripts for establishing and managing communication with OT Bioelettronica devices.

example/
Example scripts demonstrating how to send commands, receive data, and manage basic I/O operations.

Notes
The PyQt version is generally more suitable for real-time data acquisition.

The Matplotlib version may be useful for testing or environments where PyQt is not available.
