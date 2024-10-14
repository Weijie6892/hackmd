# 利用Azure運行機器學習

## §建立虛擬機器並連線

### 利用cmd-ssh連線虛擬機器
1.  將[下載]中的私密金鑰(通常是`.pem`檔)放進 `C:\Users\user\.ssh`
2.  cmd輸入 `cd C:\Users\user\.ssh`
3.  cmd輸入 `ssh -i 私密金鑰名稱 使用者名稱@主機位址`
    　範例： `ssh -i 0909pym.pem azureuser@127.0.0.1`

### VS code連接虛擬機器
1. 下載**Remote-SSH**套件
    ![image](https://hackmd.io/_uploads/H1YFptc1kl.png)

2. 安裝完成後在VS code左方工作列會多出一個項目可點擊，點擊即可進入
![image](https://hackmd.io/_uploads/H1Ge0Kq11x.png)


4. 按下**齒輪**圖案打開`config`文件並設定以下參數：
    - `host`專案名稱；
    - `hostname`主機名稱；
    - `user`管理者帳號；
    - `IdentityFile`私密金鑰位址
    　  範例：`IdentityFile "C:\Users\user\.ssh\0521test_key.pem"`

>[!tip]Tips：
>`IdentityFile`檔名若有空白則加雙引號

![image](https://hackmd.io/_uploads/SJgklTCaA.png)


### VS code編輯虛擬機器終端

1. 按下`ctrl + K` 接續 `ctrl + O`開啟資料夾（或直接從歡迎頁面選擇）
2. 使用`pwd` 檢查當前目錄



#### 相關指令

1. 創建目錄
    
    ```bash!
     mkdir 工作目錄名稱
    ```
2. 刪除目錄
    ```bash
    rm -rf 要刪除的目錄名稱
    ```

3.  按下`Ctrl + Shift + P`：
    - 輸入 `Python: Select Interpreter`：選取編譯器
    - 輸入 `Create Environment`：建立虛擬環境
    
4. 切換目錄
    ```bash!
    cd 要切換的目錄名稱
    cd ..    # 回到上一層目錄
    ```
5. 查看當層所有檔案
    ```bash!
    ls -l
    ls -al    # 查看包括隱藏檔案的所有檔案
    ```
6. 移動檔案
    ```bash!
    mv 檔案名稱 位置
    ```


<br/>

<div style="page-break-after:always"></div>

## §儲存體掛載
掛載（mounting）是指由作業系統使一個儲存裝置（諸如硬碟、CD-ROM或共享資源）上的電腦檔案和目錄可供使用者通過電腦的檔案系統訪問的一個過程。

在Azure中，掛載可以將儲存體連接到虛擬機、容器或其他運算資源，使它們可以像本地磁碟一樣存取資料。透過掛載，我們可以==從終端機直接讀取、寫入儲存體中的資料==，無需額外的資料傳輸過程。
### 前置作業
- 虛擬機器設定完畢並成功使用ssh連線(可以參考"[利用cmd-ssh連線虛擬機器](#利用cmd-ssh連線虛擬機器)"及"[利用vscode連接虛擬機器](#利用vscode連接虛擬機器)"
- 於Azure開起 ==[儲存體帳戶]== 並建立 ==[容器]==
>[!tip]Tips：
>建議直接用跟虛擬機器**一樣**的資源群組

### 操作步驟
#### step1 檢查版本
```bash!
cat /etc/*-release
```

#### step2 設定微軟封裝儲存庫
```bash!
sudo wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb

sudo dpkg -i packages-microsoft-prod.deb

sudo apt-get update

sudo apt-get install libfuse3-dev fuse3
```


#### step3 安裝blobfuse2
```bash
sudo apt-get install blobfuse2
```
#### step4 存放設定參數&參數修改
- 創建一個設定檔，副檔名為`.yaml`
    範例：`nano xxx.yaml`
  
>[!Tip]Tips：
>沒有nano的解決辦法：在終端機輸入以下指令安裝nano
>```bash!
> sudo apt updates
> sudo apt install nano
>```

```yaml=
# Refer ./setup/baseConfig.yaml for full set of config parameters

logging:
  type: syslog
  level: log_debug

components:
  - libfuse
  - file_cache
  - attr_cache
  - azstorage

libfuse:
  attribute-expiration-sec: 120
  entry-expiration-sec: 120
  negative-entry-expiration-sec: 240

file_cache:
  path: /<PATH>/<TO>/<CACHE_DIR>
  timeout-sec: 120
  max-size-mb: 4096

attr_cache:
  timeout-sec: 7200

azstorage:
  type: block
  account-name: <ACCOUNT_NAME>
  account-key: <ACCOUNT_KEY>
  endpoint: https://<ACCOUNT_NAME>.blob.core.windows.net
  mode: key
  container: <CONTAINER_NAME>
```

- `file_cache` 的快取路徑設定：`/mnt/ramdisk/blobfuse2tmp`
- `account-name`：儲存體名稱（具體名稱會依情況而定）
![image](https://hackmd.io/_uploads/BJ3oliApC.png)
- `account-key`：儲存體的存取金鑰（如圖所示）
- `container`：儲存體容器名稱


#### step5 使用RAM磁碟
```bash!
sudo mkdir /mnt/ramdisk
sudo mount -t tmpfs -o size=16g tmpfs /mnt/ramdisk
sudo mkdir /mnt/ramdisk/blobfuse2tmp
sudo chown 使用者名稱 /mnt/ramdisk/blobfuse2tmp
```

#### step6 使用SSD
```bash!
sudo mkdir /mnt/resource/blobfuse2tmp -p
sudo chown 使用者名稱 /mnt/resource/blobfuse2tmp
```

#### step7 建立空目錄以裝載BLOB容器
```bash!
mkdir ~/mycontainer
```


#### step8 將設定檔中指定的容器裝載到位置 `~/mycontainer`
```bash!
blobfuse2 mount ~/mycontainer --config-file=./自己的設定檔名稱.yaml
```
#### step9 檢查掛載是否成功
輸入以下指令確認掛載是否成功，若顯示**綠色反底藍字**代表掛載成功
```bash!
ls -l 
```
![image](https://hackmd.io/_uploads/BJ8hgccykg.png)


>[!Tip]Tips：
>#### 情境一：掛載失敗
>解決方法：`cd`至`~/mycontainer`將資料全部刪除重新掛載
>```bash!
>cd mycontainer
>rm -rf
>```
>
>
>解決方法2：創建新的目錄直接掛到新的目錄，不一定要掛在`mycontainer`
>```bash!
>mkdir NewContent
>```
><br>
>
>#### 情境二：掛載時誤用sudo 導致掛載失效
>解決方法：重新使用沒有 `sudo` 的指令重新掛載
>

<BR/>

### 參考文獻
**微軟儲存體掛載教學**：[Azure Blob儲存體掛載教學](https://learn.microsoft.com/zh-tw/azure/storage/blobs/blobfuse2-how-to-deploy?tabs=Ubuntu)

**設定檔(config公版)**：[檔案快取範本設定](https://github.com/Azure/azure-storage-fuse/blob/main/sampleFileCacheConfig.yaml)

<div style="page-break-after:always"></div>

## §在Ubuntu安裝anconda
為了應對在實務開發上的多種需求，我們需要多個不同的python版本作為解譯器來執行python程式，因此需要使用Anaconda來建置虛擬環境，透過虛擬環境執行程式，如此一來就可以將各種不同的套件安裝在所需的虛擬環境中。

### 前置作業
- 虛擬機器設定完畢並成功使用ssh連線(可以參考"[利用cmd-ssh連線虛擬機器](#利用cmd-ssh連線虛擬機器)"及"[利用vscode連接虛擬機器](#利用vscode連接虛擬機器)"
### 操作步驟
在終端機輸入以下指令
1. 安裝 `curl`

    ```bash
    sudo apt install curl
    ```

    ![image](https://hackmd.io/_uploads/rk1zkQ_0A.png)

2. 下載 Anaconda 安裝批次檔

    你可以使用 `curl` 或 `wget` 來下載最新版本的批次檔。

    - 方式 1：`curl` 下載

      ```bash
      curl -O https://repo.anaconda.com/archive/Anaconda3-2021.05-Linux-x86_64.sh
      ```

    - 方式 2：`wget` 下載（範例）

      ```bash
      sudo wget https://repo.anaconda.com/archive/Anaconda3-2024.06-1-Linux-x86_64.sh
      ```

    >[!Note] 補充：
    >可以到 [conda Repo](https://repo.anaconda.com/archive/) 查找最新的批次檔。

    ![image](https://hackmd.io/_uploads/r1771X_CA.png)

3. 使用 `sha256sum` 檢查檔案一致性

    執行下列指令，並將產出的雜湊值與官方檔案進行比對：

    ```bash
    sha256sum Anaconda3-2021.05-Linux-x86_64.sh
    ```

    輸出結果：

    ```bash
    2751ab3d678ff0277ae80f9e8a74f218cfc70fe9a9cdc7bb1c137d7e47e33d53
    ```

    ![image](https://hackmd.io/_uploads/SkfE1XdCR.png)

4. 使用 `bash` 執行安裝批次檔

    ```bash
    bash Anaconda3-2021.05-Linux-x86_64.sh
    ```
    `do you accept the license terms? [yes|no]`：選`yes`後按`enter`
    ![image](https://hackmd.io/_uploads/S104Jm_AR.png)
    >[!Tip]Tips：
    >按**Q**可以快速讀完[--MORE]
    
    > [!Tip] Tips：
    >**下載失敗解決方法：**
    >
    > 如果下載失敗或沒有初始化(未出現`base`)的解決辦法：
    > 
    > **方法一：** 刪掉整個anaconda根目錄重新下載
    > ```bash
    > rm -rf ~/anaconda3
    > ```
    >
    > **方法2：**：手動初始化
    > <font style="color:red">**source目錄應依據當下anaconda所在的相對位置而改變**</font>
    > ```bash
    > source anaconda3/bin/activate
    > conda init
    > ```
    > ![image](https://hackmd.io/_uploads/H10B2_Hk1e.png)

    >[!Tip]如果是在Windows環境中要初始化的話直接打 `conda init` 就好了
    >![image](https://hackmd.io/_uploads/rJgC6Q_yJl.png)
    >成功後一樣要`exit`並重新進入 就可以用ㄌ
    >![image](https://hackmd.io/_uploads/Skok0Q_11e.png)



    
    
    
5. 重新連線至 Anaconda

    重新啟動終端並連線到伺服器：

    ```bash
    exit
    ssh -i 私密金鑰名稱 使用者名稱@主機位址
    ```

6. 相關指令

    - 更新 conda：

      ```bash
      conda update conda
      conda update anaconda
      ```

### 建立虛擬環境
1. 建立新環境
 ```bash
 conda create -n 環境名稱 python=版本
 ```
 範例：`conda create -n py38 python=3.8`
2. 相關指令
- `conda activate 環境名稱`：進入虛擬環境
- `conda deactivate`：離開虛擬環境
- `conda env list`：列出虛擬環境清單
- `conda env remove -n 環境名稱`：刪除虛擬環境
- `conda info`：查看conda資訊
- `python -m pip install 套件名稱`：下載python套件
    - 範例套件
        - openai
        - tensorflow
- `which python`：看自己在哪個環境下的py

    >[!Warning]注意：
    >沒安裝`python`的話會沒有輸出
    


### 回家作業
建立以下`python`版本的虛擬環境並下載對應的套件
- `py3.7`：下載openai、pandas
- `py3.8`：下載requests、numpy
- `py3.10`：下載tensorflow、keras

### 參考文獻
- **在Linux環境中下載Anaconda**：[Installing Anaconda on Linux](https://docs.anaconda.com/anaconda/install/linux/)
、[Ubuntu 20.04安裝Anaconda詳細教學](https://medium.com/@scofield44165/ubuntu-20-04%E4%B8%AD%E5%AE%89%E8%A3%9D-%E8%A7%A3%E5%AE%89%E8%A3%9Danaconda%E5%8F%8A%E8%99%9B%E6%93%AC%E7%92%B0%E5%A2%83%E8%A9%B3%E7%B4%B0%E9%81%8E%E7%A8%8B-install-uninstall-environment-setup-for-anaconda-in-ubuntu-cd64e68335c5)、[如何在 Ubuntu 上安裝 Anaconda](https://www.taki.com.tw/blog/how-to-install-anaconda-on-ubuntu/)

- **Anaconda批次檔迭代倉庫**：[Anaconda Repo](https://repo.anaconda.com/archive/)

- **卷積神經網路機器學習範例**：[【keras】機器學習範例](https://keras.io/examples/vision/mnist_convnet/)
- **機器學習程式碼參考**：[【keras】機器學習程式碼參考](https://github.com/keras-team/keras-io/blob/master/examples/vision/mnist_convnet.py)
- CUDA installing on Linux：[如何在Linux環境中下載CUDA](https://hackmd.io/@joshhu/SyZwyhODB)

<br/>

<div style="page-break-after:always"></div>


## §建立帶有GPU的虛擬機器
在深度學習的環境中，GPU可以大幅加速模型訓練的速度，本章節中會以Data Science虛擬機器(Ubuntu20.04)作為範例建置一個帶有Tesla T4 GPU的機器，並簡單執行一個Mnist數字辨識的機器學習模型
### 前置作業
1. 搜尋`data science`虛擬機器並建立
![image](https://hackmd.io/_uploads/H1VGXq9y1x.png)
2. [VM大小]搜尋T4，選擇4vcpu的虛擬機器
![image](https://hackmd.io/_uploads/HyAxVc9yyg.png)
![image](https://hackmd.io/_uploads/S1wz4cq1kg.png)


3. 安裝anaconda(此類型機器會預先安裝好)
 >[!Tip]Tips：
 > 此類型的虛擬機器已經包含Anaconda，但在第一次啟動時conda可能還未初始化，這時只需依行輸入以下指令即可解決
 > ```bash
 > exit
 > conda init
 > conda activate 
 > exit
 > 重新連線
 > ```
### 操作步驟
1. `nvidia-smi` 查看機器資訊(確定是否有啟用GPU否則讀取速度會很慢)
    - `driver version :535.183.01`：nvidia驅動版本
    - `Tesla T4`：顯卡類別
    - `temp`：溫度
    - `16384Mib`：GPU專屬記憶體
2. `watch -n 1 nvidia-smi`：==實時監控==GPU使用情況
3. `conda env list`：查看所有環境清單  

    ```
    base                     /anaconda
    azureml_py310_sdkv2      /anaconda/envs/azureml_py310_sdkv2
    azureml_py38             /anaconda/envs/azureml_py38
    azureml_py38_PT_and_TF     /anaconda/envs/azureml_py38_PT_and_TF
    py38_default          *  /anaconda/envs/py38_default
    ```
4. `conda activate azureml_py38_PT_and_TF`：進入有**TensorFlow**的虛擬環境(他預設安裝好ㄉ💗)
5. 建立一個`test.py`測試並查看執行結果
    這一步驟是為了驗證程式執行的環境是否是對的
    >[!Important]錯誤範例：
    >```!
    >(base) azureuser@leafrain:/home$  /usr/bin/env /anaconda/envs/py38_default/bin/python /home/azureuser/.vscode-server/extensions/ms-python.debugpy-2024.10.0-linux-x64/bundled/libs/debugpy/adapter/../../debugpy/launcher 51731 -- /home/azureuser/test.py 
    >is leafrain a pussy?
    >```
    >
    >解決辦法：
    >- 用`ctrl + shfit + P`並輸入`Python: Select Interpreter`選有**TensorFlow**所在的Python解譯器版本
    
    


    >[!Warning]**注意：**
    >即使環境選擇正確也可能因為解譯器選擇錯誤而導致程式無法執行(找不到`Tensorflow`和`Keras`)
6. 利用[Keras Mnist簡單範例](https://github.com/keras-team/keras-io/blob/master/examples/vision/mnist_convnet.py)嘗試讓GPU跑起來



<br/>

### keras mnist 範例程式碼解析
```py=
"""
Title: Simple MNIST convnet
Author: [fchollet](https://twitter.com/fchollet)
Date created: 2015/06/19
Last modified: 2020/04/21
Description: A simple convnet that achieves ~99% test accuracy on MNIST.
Accelerator: GPU
"""

"""
## Setup
"""

import numpy as np
import keras
from keras import layers

"""
## Prepare the data
"""

# Model / data parameters
num_classes = 10
input_shape = (28, 28, 1)

# Load the data and split it between train and test sets
(x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()

# Scale images to the [0, 1] range
x_train = x_train.astype("float32") / 255
x_test = x_test.astype("float32") / 255
# Make sure images have shape (28, 28, 1)
x_train = np.expand_dims(x_train, -1)
x_test = np.expand_dims(x_test, -1)
print("x_train shape:", x_train.shape)
print(x_train.shape[0], "train samples")
print(x_test.shape[0], "test samples")


# convert class vectors to binary class matrices
y_train = keras.utils.to_categorical(y_train, num_classes)
y_test = keras.utils.to_categorical(y_test, num_classes)

"""
## Build the model
"""

model = keras.Sequential(
    [
        keras.Input(shape=input_shape),
        layers.Conv2D(32, kernel_size=(3, 3), activation="relu"),
        layers.MaxPooling2D(pool_size=(2, 2)),
        layers.Conv2D(64, kernel_size=(3, 3), activation="relu"),
        layers.MaxPooling2D(pool_size=(2, 2)),
        layers.Flatten(),
        layers.Dropout(0.5),
        layers.Dense(num_classes, activation="softmax"),
    ]
)

model.summary()

"""
## Train the model
"""

batch_size = 128
epochs = 15

model.compile(loss="categorical_crossentropy", optimizer="adam", metrics=["accuracy"])

model.fit(x_train, y_train, batch_size=batch_size, epochs=epochs, validation_split=0.1)

"""
## Evaluate the trained model
"""

score = model.evaluate(x_test, y_test, verbose=0)
print("Test loss:", score[0])
print("Test accuracy:", score[1])
```


**行 2**: `Title: Simple MNIST convnet`
- 這是一個簡單的卷積神經網路（ConvNet）範例，用於 `MNIST` 資料集。

**行 23**: `num_classes = 10`
- 設定模型的分類類別數，因為 `MNIST` 資料集包含 0 到 9 這 10 個數字，因此設定為 10。

**行 24**: `input_shape = (28, 28, 1)`
- 定義輸入資料的形狀，`MNIST` 的圖像是 28x28 的灰階圖像，因此設定為 28x28 像素，1 表示單一通道（灰階）。

**行 27**: `keras.datasets.mnist.load_data()`
- 從 `Keras` 內建資料集中載入 `MNIST` 資料，這段程式碼將資料分為訓練集 `(x_train, y_train)` 和測試集 `(x_test, y_test)`。

**行 30**: `x_train = x_train.astype("float32") / 255`
- 將像素值標準化，將其範圍從 [0, 255] 轉換為 [0, 1]，這有助於提高模型的準確度及穩定性。

**行 34**: `np.expand_dims(x_train, -1)`
- 將圖像資料從 28x28 轉換為 28x28x1 的形狀，這是 Keras 對於卷積神經網路輸入的標準格式，1 代表單通道（灰階圖像）。

**行 41**: `keras.utils.to_categorical(y_train, num_classes)`
- 將標籤轉換為 one-hot 編碼格式，這是分類問題中常用的標籤格式。例如數字 5 將會轉換成 `[0, 0, 0, 0, 0, 1, 0, 0, 0, 0]`。

**行 48**: `keras.Sequential([...])`
- 使用 `Sequential` 模型，這是一個簡單的線性堆疊模型，適合用於逐層構建的神經網路。模型包括多層卷積層、最大池化層、平坦化層、dropout 層以及全連接層。

**行 51**: `layers.Conv2D(32, kernel_size=(3, 3), activation="relu")`
- 增加卷積層，使用 32 個 3x3 的卷積核，`relu` 是常用的激活函數，用於引入非線性。

**行 52**: `layers.MaxPooling2D(pool_size=(2, 2))`
- 使用最大池化層來減少空間維度（圖像大小），`2x2` 的池化視窗會將特徵圖縮小一半大小。

**行 56**: `layers.Dropout(0.5)`
- 增加 dropout 層，==防止過擬合==。`0.5` 表示隨機捨棄 50% 的神經元，==有助於模型正則化==。

**行 70**: `model.compile(loss="categorical_crossentropy", optimizer="adam", metrics=["accuracy"])`
- 編譯模型，使用 `categorical_crossentropy` 作為損失函數，`adam` 作為優化器，並設定評估指標為 `accuracy`（準確率）。

**行 72**: `model.fit(x_train, y_train, batch_size=batch_size, epochs=epochs, validation_split=0.1)`
- 開始訓練模型，批次大小為 128，訓練 15 個迭代週期（epochs），並將 10% 的資料作為驗證集進行評估。

**行 78**: `score = model.evaluate(x_test, y_test, verbose=0)`
- 使用測試資料集進行模型評估，計算損失及準確率。

**行 79**: `print("Test loss:", score[0])`
- 列印測試損失，顯示模型在測試資料集上的損失值。

**行 80**: `print("Test accuracy:", score[1])`
- 列印測試準確率，顯示模型在測試資料集上的準確率。

<br/>

<div style="page-break-after:always"></div>

## §建立並執行一個文生圖網站
具備本篇上述之基礎後，本章節會示範如何架設一個由文字生成圖片的網站
### 前置作業
- 建立一個[data science]虛擬機器並成功用SSH連線
### 操作步驟
1. 將此網址的github倉庫複製至當前使用的環境(虛擬機或本地端)
    ```bash
    git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git
    ```
    >[!Tip]Tips：
    >這裡的連結會根據每一次要執行的模型的不同而改變，建議在每一次執行新模型前都先去`Github`找找看有沒有對應的Web UI可以用
2. 進入對應目錄
    ```bash
    cd stable-diffusion-webui/models/Stable-diffusion/
    ```
    ![image](https://hackmd.io/_uploads/HkXT8N5yJx.png)
    
    `Put Stable Diffusion checkpoints here.txt`：權重(`.ckpt`)檔案擺這裡的意思
    
    >[!Tip]Tips：
    >將目錄切換至這裡是為了等下下載權重檔案的時候可以直接下載在這裡，若不小心跳走了在進行++步驟4++要記得切回來

<br>

3. 註冊hugging face帳號並申請==權杖==

    >[!Warning]注意：
    >帳號要去郵件確認才算開通
    
    >[!Warning]注意：
    >要複製`your-huggingface-token`，這個token不見就沒了


    <br>
    
4. 將權重下載至對應目錄

    將++步驟3++申請到的權杖替換進`<your-huggingface-token>`並在終端機執行
    ```bash!
    curl -H "Authorization: Bearer <your-huggingface-token>" https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.ckpt --location --output v1-5-pruned-emaonly.ckpt
    ```
    - `-H`：用於設置`HTTP`標頭，包含授權資訊
    - `https:...ckpt`：要下載的東西的網址
    - `--location`：自動重新導向，確保下載無誤
    - `--output v...emaonly.ckpt`：指定下載後的檔案名稱
    > 因為這邊我已經有權杖了，所以直接複製下面以下指令
    > ```bash!
    > curl -H "Authorization: Bearer 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" https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.ckpt --location --output v1-5-pruned-emaonly.ckpt
    >```  
    
    >[!Note]補充：
    >1. 檔名輸出`--output`可以直接縮寫成`-o`，語法如下
    >2. 重新導向`--location`可以縮寫為`-L`，語法如下
    >3. 將前兩者整合後，語法如下
    >```bash!
    >curl -H "Authorization: Bearer 權杖名稱" -L -o 下載後的檔名 下載連結
    >```
    >```bash!
    >curl -H "Authorization: Bearer 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" -L -o flux1-dev-bnb-nf4-v2.safetensors https://huggingface.co/lllyasviel/flux1-dev-bnb-nf4/resolve/main/flux1-dev-bnb-nf4-v2.safetensors
    >```



5. 創建一個新的虛擬環境並進入
    ```bash!
    conda create -n py311 python=3.11
    conda activate py311
    ```
    >[!Caution]警告：
    >若使用python3.11以下的版本可能會出錯或產生相容性問題
6. 回到儲存庫的根目錄

    `stable-diffusion-webui`那一層，每一次的目錄不一樣，具體名稱要看++步驟1++下載下來的東西是甚麼而定

    ```bash!
    cd ../..
    ```
    
    >[!Warning] 注意：
    >這裡預設還在 **步驟2** 的目錄中
    
7. 依序下載相依套件
    ```bash!
    pip install -r requirements_versions.txt 
    ```

8. 從conda下載Web UI缺少的相依套件
    ```conda!
    conda install pytorch=1.13 torchvision=0.14 torchaudio=0.13 pytorch-cuda=11.7 -c pytorch -c nvidia -y
    ```
    `conda`的意思代表將此套件下載為==全域套件==，所有環境都可以調用

9. 檢查`PyTorch`是否成功調用GPU

    在瀏覽器搜尋關鍵字下[pytorch gpu detection](https://stackoverflow.com/questions/48152674/how-do-i-check-if-pytorch-is-using-the-gpu)並測試
    - `python`：執行python環境(依行執行)
    - `exit()`：離開python執行環境
    
    ```python=
    >>> import torch

    >>> torch.cuda.is_available()
    True

    >>> torch.cuda.device_count()
    1

    >>> torch.cuda.current_device()
    0

    >>> torch.cuda.device(0)
    <torch.cuda.device at 0x7efce0b03be0>

    >>> torch.cuda.get_device_name(0)
    'Tesla T4'
    ```
    ![image](https://hackmd.io/_uploads/B1SOuw5Jyx.png)


    

10. 開啟git環境限制，允許 Git 信任所有目錄作為安全的工作目錄
    ```git
    git config --global --add safe.directory '*'
    ```
    - `'*'`：作為萬用字元，代表Git信任所有目錄，無論目錄屬於哪個使用者
    >[!Caution]警告：
    >這一步**不能跳過**，因為Git引入「安全目錄」的概念，主要是為了在某些情況下避免非預期的權限升級或其他安全風險；當使用者在一個不屬於當前用戶的目錄使用`git`時，`git`會警告該目錄未被視為「安全」
11. 啟動 Web UI
    
    此步驟會從遠端請求所有第三方套件，所以需要一些時間

    ```bash!
    accelerate launch --mixed_precision=fp16 --num_cpu_threads_per_process=6 launch.py --share --gradio-auth test:test
    ```
    - `--mixed_precision=fp16`：因為目前我們的顯示卡是Tesla T4，所以值應為`fp16`。
    - `test:test`：指的是待會創建起來的網站的登入帳號密碼
<br>    

12. 前往連結網址
![image](https://hackmd.io/_uploads/HkwOn4qyyg.png)

    將輸出網址內容複製到瀏覽器並開啟
    ```bash!
    Running on public URL: https://540e320077f78fb43f.gradio.live
    ```


<br/>

### 實例－建立一個flux1模型
==這個是期中考==，題目需要下載一個最新的文生圖模型`flux1`，並利用Web UI讓他成功運行
#### 前置作業
- 建立有[data science]的Ubuntu20.04機器並利用SSH連線完成
#### 操作步驟
1. 至瀏覽器搜尋[flux 1 webui](https://github.com/lllyasviel/stable-diffusion-webui-forge)並將其複製至虛擬機中
    ```bash!
    git clone https://github.com/lllyasviel/stable-diffusion-webui-forge
    ```

2. 進入對應目錄
    ```bash!
    cd stable-diffusion-webui-forge/models/Stable-diffusion/
    ```
    
3. (跳過hugging face註冊...)

4. 將權重下載至對應目錄

    ```bash!
    curl -H "Authorization: Bearer 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" -L -o flux1-dev-bnb-nf4-v2.safetensors https://huggingface.co/lllyasviel/flux1-dev-bnb-nf4/resolve/main/flux1-dev-bnb-nf4-v2.safetensors
    ```

5. 創建虛擬環境並進入
    ```conda!
    conda create -n py311 python=3.11
    conda activate py311
    ```

6. 回到儲存庫根目錄
    到`stable-diffusion-webui-forge`那層
    ```bash!
    cd ../..
    ```

7. 依序下載相依套件
    ```bash!
    pip install -r requirements_versions.txt 
    ```
    
8. 從conda下載Web UI缺少的相依套件
    ```conda!
    conda install pytorch=1.13 torchvision=0.14 torchaudio=0.13 pytorch-cuda=11.7 -c pytorch -c nvidia -y
    ```
    
9. 檢查`PyTorch`是否能夠成功調用GPU
    - `python` 進入執行環境
    - `exit()` 離開執行環境
    ```py=
    >>> import torch

    >>> torch.cuda.is_available()
    True

    >>> torch.cuda.device_count()
    1

    >>> torch.cuda.current_device()
    0

    >>> torch.cuda.device(0)
    <torch.cuda.device at 0x7efce0b03be0>

    >>> torch.cuda.get_device_name(0)
    'Tesla T4'
    ```

10. 開啟git環境限制，允許 `Git` 信任所有目錄作為安全的工作目錄
    ```command!
    git config --global --add safe.directory '*'
    ```
    
    
11. 啟動Web UI並連線
    ```bash!
    accelerate launch --mixed_precision=fp16 --num_cpu_threads_per_process=6 launch.py --share --gradio-auth test:test
    ```
    ```bash!
    Running on public URL: https://540e320077f78fb43f.gradio.live
    ```
<BR>

### 參考文獻
WebUI建立教學：[AUTOMATIC1111’s Stable Diffusion Web UI Setup](https://vladiliescu.net/stable-diffusion-web-ui-on-azure-ml/)

AUTOMATIC1111的github倉庫：[stable-diffusion-webui Public](https://github.com/AUTOMATIC1111/stable-diffusion-webui)

huggingface官方網站：[huggingface.com](https://huggingface.co/settings/tokens)

pytorch調用GPU測試：[How do I check if PyTorch is using the GPU?](https://stackoverflow.com/questions/48152674/how-do-i-check-if-pytorch-is-using-the-gpu)

AI推理訓練：[AI 推理與訓練：什麼是 AI 推理？](https://www.cloudflare.com/zh-tw/learning/ai/inference-vs-training/)

## 

Web UI Forge GitHub檔案：[stable-diffusion-webui-forge](https://github.com/lllyasviel/stable-diffusion-webui-forge)

hugging face權重檔案：[Upload flux1-dev-bnb-nf4-v2.safetensors](https://huggingface.co/lllyasviel/flux1-dev-bnb-nf4/blob/main/flux1-dev-bnb-nf4-v2.safetensors)

Flux1-dev Ubuntu安裝教學：[From Text to Image: Install FLUX.1-dev via WebUI Forge on Ubuntu](https://zahiralam.com/blog/from-text-to-image-install-flux-1-dev-via-webui-forge-on-ubuntu/)

Flux1 介紹：[FLUX AI Image Generation: Free AI Art with FLUX.1](https://flux1.org/)







<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>
