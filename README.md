
# ASCII 3D Cube

このプログラムは C++ を用いて ASCII アートで 3D の立方体を描画し、回転させるデモンストレーションです。  
ターミナル上で立方体がゆっくり回転していく様子を楽しむことができます。

!(CubeRotation.gif)

## デモの概要

- ASCII 文字を使って画面上に 3D 立方体を描画  
- A, B, C という 3 つの回転角度を用いて、X, Y, Z 軸方向に回転を加えています  
- 各面に異なる文字 (`@`, `$`, `~`, `#`, `;`, `+`) を割り当てることで、立方体であることがわかりやすくなっています  
- Z バッファを用い、奥にある面が手前の面に上書きされないように描画を管理しています

## 仕組み

1. **座標変換 (calculateX, calculateY, calculateZ)**
   - 3D 空間上の `(x, y, z)` 座標を回転行列で変換する処理です。
   - それぞれ \( A, B, C \) (X, Y, Z 軸回転角) をもとに回転後の位置を計算し、視点方向に投影します。

2. **投影 (ooz = 1 / z)**
   - 距離に応じた奥行きを考慮するために、`1 / z` を用いる簡易的な透視投影を行っています。
   - これにより、奥にある点が手前よりも小さく描画されるようになります。

3. **Z バッファ (zBuffer)**
   - 画面上の各ピクセルに対して、いま描画しようとしている点の“奥行き”を比較し、  
     より手前にある点 (`ooz` 値が大きい点) を優先して描画しています。
   - これにより、立体感を保ったまま正しい重なり順で表示されます。

4. **アニメーション**
   - メインループ内で A, B, C の値を少しずつ変化させ (例: `A += 0.05; B += 0.05; C += 0.01;` )、  
     ループを回し続けることで連続的に回転しているように見えます。

## コードについて

```cpp
// 回転角度 (グローバル変数)
float A, B, C;

// 画面サイズ、立方体の幅等の設定
int width = 160;
int height = 44;
float cubeWidth = 20;

// Z バッファと画面バッファ
float zBuffer[160 * 44];
char buffer[160 * 44];
int backgroundASCIICode = ' ';  // バックグラウンドの文字 (空白)

// カメラからの距離や投影用の係数
int distanceFromCam = 100;
float K1 = 40;
```

- **`A, B, C`** は X, Y, Z 軸回転の角度です。  
- **`zBuffer` と `buffer`** は、描画時の Z バッファと ASCII バッファです。  
  インデックスで対応する画面座標 \((x + y * width)\) に保管します。  
- **`distanceFromCam`** はカメラ (視点) と投影面の距離。  
- **`K1`** は透視投影のスケーリング係数です。

```cpp
float calculateX(int i, int j, int k) {
    // X 座標変換
    return j * sin(A) * sin(B) * cos(C)
         - k * cos(A) * sin(B) * cos(C)
         + j * cos(A) * sin(C)
         + k * sin(A) * sin(C)
         + i * cos(B) * cos(C);
}

float calculateY(int i, int j, int k) {
    // Y 座標変換
    return j * cos(A) * cos(C)
         + k * sin(A) * cos(C)
         - j * sin(A) * sin(B) * sin(C)
         + k * cos(A) * sin(B) * sin(C)
         - i * cos(B) * sin(C);
}

float calculateZ(int i, int j, int k) {
    // Z 座標変換
    return k * cos(A) * cos(B)
         - j * sin(A) * cos(B)
         + i * sin(B);
}
```

- **`i, j, k`** がそれぞれ物体座標系における X, Y, Z を表し、  
  これらを **A, B, C** の回転分だけずらして新しい X, Y, Z を求めています。

```cpp
void calculateForSurface(float cubeX, float cubeY, float cubeZ, int ch) {
    x = calculateX(cubeX, cubeY, cubeZ);
    y = calculateY(cubeX, cubeY, cubeZ);
    z = calculateZ(cubeX, cubeY, cubeZ) + distanceFromCam;

    ooz = 1 / z;
    xp = (int)(width / 2 - 2 * cubeWidth + K1 * ooz * x * 2);
    yp = (int)(height / 2 + K1 * ooz * y);

    idx = xp + yp * width;
    if(idx >= 0 && idx < width * height) {
        if(ooz > zBuffer[idx]) {
            zBuffer[idx] = ooz;
            buffer[idx] = ch;
        }
    }
}
```

- 回転後の `(x, y, z)` を基に投影座標 `(xp, yp)` を計算し、  
  該当インデックス `idx` に描画する文字 `ch` を設定します。  
- **`if (ooz > zBuffer[idx])`** によって、より手前にある画素のみ描画するようにしています。

```cpp
while (1) {
    // バッファの初期化
    memset(buffer, backgroundASCIICode, width * height);
    memset(zBuffer, 0, width * height * sizeof(float));

    // 立方体各面の点を描画
    for(float cubeX = -cubeWidth; cubeX < cubeWidth; cubeX += incrementSpeed){
        for(float cubeY = -cubeWidth; cubeY < cubeWidth; cubeY += incrementSpeed){
            calculateForSurface(cubeX,  cubeY, -cubeWidth, '@');
            calculateForSurface(cubeWidth,  cubeY,  cubeX, '$');
            calculateForSurface(-cubeWidth, cubeY, -cubeX, '~');
            calculateForSurface(-cubeX,     cubeY,  cubeWidth, '#');
            calculateForSurface(cubeX,  -cubeWidth, -cubeY, ';');
            calculateForSurface(cubeX,   cubeWidth,  cubeY, '+');
        }
    }

    // 描画 (バッファの出力)
    printf("[H");  // カーソルを先頭へ移動
    for(int k = 0; k < width * height; ++k){
        putchar(k % width ? buffer[k] : 10);
    }

    // 回転角度を更新
    A += 0.05;
    B += 0.05;
    C += 0.01;

    // 適度な休止 (回転速度調整)
    usleep(8000 * 2);
}
```

- **`memset`** でバッファを初期化し、立方体各面のピクセルを更新しては  
  画面に出力するループを無限に繰り返します。  
- ループ内で少しずつ **`A, B, C`** を変化させることで、回転アニメーションが実現されています。

---
