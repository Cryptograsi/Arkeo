# Arkeo Nod Kurulumu


#### Sistem Gereksinimleri
```
4 CPU
8 GB RAM
250 GB SSD
```


#### Sunucumuzu Güncelleyelim
```
sudo apt-get update && sudo apt-get upgrade -y
```


#### bashrc ve bash_profile Dosyalarını Uygulayalım
```
source $HOME/.bashrc
source $HOME/.bash_profile
```


#### Gerekli Paketleri Yükleyelim
```
sudo apt install curl make clang pkg-config lz4 libssl-dev build-essential git jq ncdu bsdmainutils htop -y
```


#### Go Yükleyelim
```
wget -O go1.22.4.linux-amd64.tar.gz https://golang.org/dl/go1.22.4.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.4.linux-amd64.tar.gz && rm go1.22.4.linux-amd64.tar.gz
echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
echo 'export GO111MODULE=on' >> $HOME/.bash_profile
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile && . $HOME/.bash_profile
```

###### Versiyon Kontrolü

```
go version
```


#### Binary Dosyasını Yükleyelim


NodesGuru'nun binary dosyasını kullandım

```
wget -O arkeod https://snapshots.nodes.guru/arkeo/arkeod
sudo chmod +x $HOME/arkeod
sudo mv $HOME/arkeod /usr/local/bin/arkeod
```


#### Nodumuzu Başlatalım


Dikkat: MonikerName kısmını kendi belirlediğiniz bir moniker adıyla değiştirin!!!

```
arkeod init MonikerName --chain-id arkeo
```


#### Seed Ekleyelim
```
SEEDS=df949a46ae6529ae1e09b034b49716468d5cc7e9@testnet-seeds.stakerhouse.com:11656,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:22856,df0561c0418f7ae31970a2cc5adaf0e81ea5923f@arkeo-testnet-seed.itrocket.net:18656

sed -i.bak -e "s/^seeds =./seeds = "$SEEDS"/" $HOME/.arkeo/config/config.toml
```


#### Gas Ayarlarını Girelim
```
sed -i.bak -e "s/^minimum-gas-prices =./minimum-gas-prices = "0.001uarkeo"/" $HOME/.arkeo/config/app.toml
```


#### Pruning Ekleyelim (opsiyonel)
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"

sed -i -e "s/^pruning =./pruning = "$pruning"/" $HOME/.arkeo/config/app.toml

sed -i -e "s/^pruning-keep-recent =./pruning-keep-recent = "$pruning_keep_recent"/" $HOME/.arkeo/config/app.toml

sed -i -e "s/^pruning-keep-every =./pruning-keep-every = "$pruning_keep_every"/" $HOME/.arkeo/config/app.toml

sed -i -e "s/^pruning-interval =./pruning-interval = "$pruning_interval"/" $HOME/.arkeo/config/app.toml

sed -i -e "s/indexer =./indexer = "null"/g" $HOME/.arkeo/config/config.toml
```


#### Genesis Dosyasını İndirelim
```
wget -O $HOME/.arkeo/config/genesis.json https://snapshots.nodes.guru/arkeo/genesis.json
```


#### Validatör Dosyasını Başlangıç Durumuna Sıfırlayalım
```
arkeod tendermint unsafe-reset-all
```


#### Snapshot İndirelim

Polkachu'nun snapshot'ını kullandım
```
curl https://snapshots.polkachu.com/testnet-snapshots/arkeo/arkeo_3873468.tar.lz4 | lz4 -dc - | tar -xvf - -C $HOME/.arkeo
```


#### Servis Dosyası Oluşturalım
```
echo "[Unit]
Description=Arkeo Node
After=network.target

[Service]
User=root
Type=simple
ExecStart=/usr/local/bin/arkeod start 
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/arkeod.service
```
```
sudo systemctl daemon-reload
sudo systemctl enable arkeod
sudo systemctl start arkeod
```

Logları görmek isterseniz:
```
journalctl -u arkeod -f -o cat
```
Devam etmek için CTRL+C ile logları durduralım


#### Nodumuzu Kontrol Edelim
```
service arkeod status
```
Not: Eğer düzgün çalışıyorsa "active (running)" yazar


#### Cüzdan Oluşturalım

Dikkat: WalletName kısmını kendinize göre düzenleyin. Şifre isteyecek, şifre belirleyin. Size mnemonic kelimeler verecek, onları mutlaka bir yere kaydedin!!!
```
source $HOME/.bash_profile
arkeod keys add WalletName
```

Discorda girip faucetten token isteyin. https://discord.com/invite/BfEHpm6uFc linke tıkladıktan sonra verify yapın ve validatör test rolünü seçin. Faucet görünür olacaktır.



Faucet kanalına $request WalletAddress şeklinde mesaj atın, kısa sürede tokenler gelecektir. (WalletAddress kısmına kendi cüzdan adresinizi yazın)


Tokenler geldikten ve ağla senkronize olduktan sonra validatör oluşturabiliriz. Senkron durumunu alttaki kodla kontrol edelim. Çıktımız "false" olmalı
```
arkeod status 2>&1 | jq .SyncInfo
```


#### Validatör Oluşturalım

Not: MonikerName ve WalletName kısımlarını kendimize göre düzenleyelim!!!
```
arkeod tx staking create-validator \
--chain-id arkeo \
--commission-rate 0.05 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.1 \
--min-self-delegation "1" \
--amount 1000000uarkeo \
--details "" \
--website "" \
--pubkey $(arkeod tendermint show-validator) \
--moniker "MonikerName" \
--from WalletName \
--fees="5000uarkeo" \
--yes
```


#### Valoper Adresinizi Alıp Doğrulayıcıya Yetki Verin
```
arkeod keys show wallet --bech val –a
```

Aldığınız valoper adresini koddaki ValoperAddress ile düzenleyin, WalletName de aynı şekilde düzenlenecek!!!
```
arkeod tx staking delegate ValoperAddress 999000500uarkeo --from WalletName --chain-id arkeo --fees="500uarkeo"
```

Nodunuzun durumunu explorerdan takip edebilirsiniz:


https://testnet.arkeo.explorers.guru/


