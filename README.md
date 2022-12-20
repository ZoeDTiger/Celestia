## Celestia Mocha 验证节点部署

    系统要求：Ubuntu Linux 20.04 （LTS） x64
    内存：8 GB
    CPU：四核
    磁盘：250 GB固态硬盘
    带宽：下载 1 Gbps/上传 100 Mbps

### 第一步 环境准备

#### 1、系统依赖安装：安装用到的依赖包

    sudo apt update && sudo apt upgrade -y
    sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential git make ncdu -y

#### 2、golang编译环境安装

##### a、下载Go压缩包

    cd $HOME
    wget "https://go.dev/dl/go1.19.4.linux-amd64.tar.gz"
    sudo rm -rf /usr/local/go
    sudo tar -C /usr/local -zxvf go1.19.4.linux-amd64.tar.gz

##### b、设置golang的环境变量

    echo "export GOROOT=/usr/local/go" |  sudo tee -a /etc/profile
    echo "export GOPATH=$HOME/go" |  sudo tee -a /etc/profile
    echo "export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin" |  sudo tee -a /etc/profile
    echo "export GO111MODULE=on" |  sudo tee -a /etc/profile
    echo "export GOPROXY=https://goproxy.cn" |  sudo tee -a /etc/profile

##### c、使用环境生效

    source /etc/profile

#### 3、设置进程运行的文件句柄数量：防止进程莫名其秒的自动退出

    echo "ulimit -SHn 1048576" |  sudo tee -a /etc/profile
    echo "* hard nofile 1048576" |  sudo tee -a /etc/security/limits.conf
    echo "* soft nofile 1048576" |  sudo tee -a /etc/security/limits.conf
    echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/user.conf
    echo "DefaultLimitNOFILE=1048576" |  sudo tee -a /etc/systemd/system.conf
    echo "session required pam_limits.so" |  sudo tee -a /etc/pam.d/common-session
    source /etc/profile

### 第二部分 部署celestia-app

#### 1、编译celestia-appd二进制应用

    cd $HOME
    rm -rf celestia-app
    git clone https://github.com/celestiaorg/celestia-app.git
    cd celestia-app/
    APP_VERSION=v0.11.0
    git checkout tags/$APP_VERSION -b $APP_VERSION
    make install
    sudo cp $HOME/go/bin/celestia-appd /usr/local/bin/
    celestia-appd version

#### 2、准备网络创世文件
下载新的项目代码仓库目录, 目的是为了获取到里面的genesis.json创世文件

    cd $HOME
    rm -rf networks
    git clone https://github.com/celestiaorg/networks.git
设置验证节点的名称为MONIKER

    MONIKER="your_moniker"
生成~/.celestia-app/配置目录

    celestia-appd init $MONIKER --chain-id mocha
拷贝genesis.json创世文件到~/.celestia-app/配置目录中

    cp $HOME/networks/mocha/genesis.json $HOME/.celestia-app/config
配置p2p参数，获取到官方的启动节点列表，并修改配置文件, 将官方的启动节点添加到里面

    BOOTSTRAP_PEERS=$(curl -sL https://raw.githubusercontent.com/celestiaorg/networks/master/mamaki/bootstrap-peers.txt | tr -d '\n')
    echo $BOOTSTRAP_PEERS
    sed -i.bak -e "s/^bootstrap-peers *=.*/bootstrap-peers = \"$BOOTSTRAP_PEERS\"/" $HOME/.celestia-app/config/config.toml

设置共识配置选项（可选）

    sed -i 's/timeout-commit = ".*/timeout-commit = "25s"/g' $HOME/.celestia-app/config/config.toml
    sed -i 's/peer-gossip-sleep-duration *=.*/peer-gossip-sleep-duration = "2ms"/g' $HOME/.celestia-app/config/config.toml
    
    max_num_inbound_peers=40
    max_num_outbound_peers=10
    max_connections=50
    
    sed -i -e "s/^use-legacy *=.*/use-legacy = false/;\
    s/^max-num-inbound-peers *=.*/max-num-inbound-peers = $max_num_inbound_peers/;\
    s/^max-num-outbound-peers *=.*/max-num-outbound-peers = $max_num_outbound_peers/;\
    s/^max-connections *=.*/max-connections = $max_connections/" $HOME/.celestia-app/config/config.toml

#### 3、配置节点为修剪模式（节省存储空间）

为了降低磁盘空间使用率，下面的命令主要是修改celestia-appd配置文件app.toml里面的部分内容

    PRUNING="custom"
    PRUNING_KEEP_RECENT="100"
    PRUNING_INTERVAL="10"
    sed -i -e "s/^pruning *=.*/pruning = \"$PRUNING\"/" $HOME/.celestia-app/config/app.toml
    sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$PRUNING_KEEP_RECENT\"/" $HOME/.celestia-app/config/app.toml
    sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$PRUNING_INTERVAL\"/" $HOME/.celestia-app/config/app.toml

#### 4、配置节点为validator运行模式

下面的命令主要是修改celestia配置文件config.toml里面的部分内容, 修改为 mode = "validator"

    sed -i.bak -e "s/^mode *=.*/mode = \"validator\"/" $HOME/.celestia-app/config/config.toml

#### 5、复位网络区块数据，并下载新数据导入

复位网络区块数据, 将会清空$HOME/.celestia-app/data 目录下的区块数据文件

    cd $HOME
    celestia-appd tendermint unsafe-reset-all --home $HOME/.celestia-app
获取将要下载的区块数据文件名称，并解压下载的区块数据到 $HOME/.celestia-app/data/ 目录中

    rm -rf ~/.celestia-app/data
    mkdir -p ~/.celestia-app/data
    SNAP_NAME=$(curl -s https://snaps.qubelabs.io/celestia/ | egrep -o ">mamaki.*tar" | tr -d ">")
    wget -O - https://snaps.qubelabs.io/celestia/${SNAP_NAME} | tar xf - -C ~/.celestia-app/data/

#### 6、创建钱包：接水教程往下看

在~/.celestia-app/下生成钱包信息

    celestia-appd config chain-id mamaki
    celestia-appd config keyring-backend test
创建名为 youe_wallet_name 的钱包并生成地址(请保管好钱包的助记词, 私钥等信息)

    celestia-appd keys add youe_wallet_name
查看钱包地址

    celestia-appd keys list

#### 7、启动节点

第5步快速同步区块快照也可使用如下方式：screen另起窗口下载： 

    wget https://snaps.qubelabs.io/celestia/mamaki_2022-12-14.tar

要确保$HOME/.celestia-app/data目录下有priv_validator_state.json，否则无法启动

    cd $HOME
    screen -S celestia
    celestia-appd start

#### 8、运行验证节点并质押

通过钱包地址查看验证地址

    celestia-appd keys show 第6步生成的钱包地址 --bech val -a
查询钱包余额

    celestia-appd query bank balances 第6步生成的钱包地址
创建验证人

    MONIKER="your_moniker"
    VALIDATOR_WALLET="youe_wallet_name"
    celestia-appd tx staking create-validator \
    --amount=1000000utia \
    --pubkey=$(celestia-appd tendermint show-validator) \
    --moniker=$MONIKER \
    --chain-id=mamaki \
    --commission-rate=0.1 \
    --commission-max-rate=0.2 \
    --commission-max-change-rate=0.01 \
    --min-self-delegation=1000000 \
    --from=$VALIDATOR_WALLET \
    --keyring-backend=test
质押

    VALIDATOR_WALLET=celestia1dxxxxxxxxxxxxxxxxxxxxxx
    AMOUNT=1000000utia
    OP_WALLET=celestiavaloper1dxxxxxxxxxxxxxxxxxxxxxx
    celestia-appd tx staking delegate $OP_WALLET $AMOUNT --from=$VALIDATOR_WALLET --chain-id=mamaki



## celestia mamaki 测试网接水教程

### 第一步：准备钱包地址

方式一：Keplr钱包

    1、安装Keplr钱包, https://www.keplr.app/download
    2、将 Celestia 做为网络添加到 Keplr钱包中, http://aviaone.com/celestia-connect-wallet-keplr-testnet-mamaki.html/

方式二：安装节点，通过命令生成钱包，轻节点与验证节点都支持

### 第二步：加入官方DC开始接水

    https://discord.com/invite/YsnTPcSfWQ
    完成基本认证后进入频道，输入接水命令，即可。图中1为频道，2为接水命令（换成自己的钱包地址），3为接水成功后机器人的回答。成功后获得10TIA。
![image](https://user-images.githubusercontent.com/100336530/204741398-bcebe555-718b-4c69-b337-7a54bebf1dc9.png)

**注意：同一DC帐号、同一钱包；不同DC帐号、同一钱包接水都有7天的间隔时间**

![image](https://user-images.githubusercontent.com/100336530/204741497-1eab1b82-5a3f-459b-8d58-d5b2735364bd.png)
