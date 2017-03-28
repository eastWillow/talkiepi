# 將 talkiepi 開機

![assembled1](talkiepi_assembled_1.jpg "Assembled talkiepi 1")

這是一個概述如何安裝talkiepi 到你的RaspberryPi 然後開機。
這個文件假設你已經安裝了raspbian-jessie-lite 在你的SD 卡裡面，並且已經將它更新到最新版。
這個文件也假設你已經設定好了有線網路或是WIFI 的設定。

talkiepi 預設是不使用任何設定運作的，他將會自動產生使用者帳號並且連線到我的mumble server。
你若是想要改變上面的預設，可以透過附加這些變數：`-server YOUR_SERVER_ADDRESS`, `-username YOUR_USERNAME` 到ExecStart `/etc/systemd/system/mumble.service` 這行後面

talkiepi 也接受這些變數： `-password`, `-insecure`, `-certificate` and `-channel` ，這些都定義在`cmd/talkiepi/main.go` ，如果你想要改成自己的mumble 伺服器，這些變數必須由你自己設定。


## 建立使用者

登入root 使用者(`sudo -i`)，建立一個mumble 使用者：
```
adduser --disabled-password --disabled-login --gecos "" mumble
usermod -a -G cdrom,audio,video,plugdev,users,dialout,dip,input,gpio mumble
```

## 安裝

登入root 使用者(`sudo -i`)，安裝go 語言還有其他依賴的安裝包，然後安裝talkeipi：
```
apt-get install golang libopenal-dev libopus-dev git

su mumble

mkdir ~/gocode
mkdir ~/bin

export GOPATH=/home/mumble/gocode
export GOBIN=/home/mumble/bin

cd $GOPATH

go get github.com/layeh/gopus
go get github.com/dchote/talkiepi

cd $GOPATH/src/github.com/dchote/talkiepi

go build -o /home/mumble/bin/talkiepi cmd/talkiepi/main.go 
```


## 開機自動啟動

登入root 使用者(`sudo -i`), 複製mumble.service 去某個地方:
```
cp /home/mumble/gocode/src/github.com/dchote/talkiepi/conf/systemd/mumble.service /etc/systemd/system/mumble.service

systemctl enable mumble.service
```

## 建立憑證(certificate)

這個不一定要做，主要是你想要註冊你的talkiepi 在mumble 伺服器上，並且開啟黑白名單之類的（ACLs）。

```
su mumble
cd ~

openssl genrsa -aes256 -out key.pem
```

輸入一個簡單的密碼，不用擔心，我們很快就會用不到這個密碼...


```
openssl req -new -x509 -key key.pem -out cert.pem -days 1095
```

再次輸入你的簡單密碼，然後選擇一個你喜歡的憑證訊息，這其實不太重要，因為你只是在做一個hacking 的東西

```
openssl rsa -in key.pem -out nopasskey.pem
```

輸入你的簡單密碼最後一次

```
cat nopasskey.pem cert.pem > mumble.pem
```

登入root 使用者(`sudo -i`), 編輯 `/etc/systemd/system/mumble.service` 加上 `-username USERNAME_TO_REGISTER -certificate /home/mumble/mumble.pem` 在最後這行 `ExecStart = /home/mumble/bin/talkiepi`

執行 `systemctl daemon-reload` 然後 `service mumble restart` 然後你就把憑證設定好了


## 使用你的USB 擴音器

If you are using a USB speakerphone such as the US Robotics one that I am using, you will need to change the default system sound device.
As root on your Raspberry Pi (`sudo -i`), find your device by running `aplay -l`, take note of the index of the device (likely 1) and then edit the alsa config (`/usr/share/alsa/alsa.conf`), changing the following:
```
defaults.ctl.card 1
defaults.pcm.card 1
```
_1 being the index of your device_


If your speakerphone is too quiet, you can adjust the volume using amixer as such:
```
amixer -c 1 set Headphone 60%
```
_1 being the index of your device_


I will be adding volume control settings in an upcoming push.
