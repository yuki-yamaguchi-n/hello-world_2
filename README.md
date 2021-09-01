#Prerequisites
Azure CLIがversionxxがインストール済である
ローカルのUbuntu環境で実行することを想定

#Preparation
##Login in azure on ubuntu
az login

##subscription set to KMR-Run-nisssan
subscription=KMR-Run-nissan
az account set --subscription $subscription

#Create resource group
##Create ResourceGroup
RGname=DB-patch-proxy
az group create --name $RGname --location japaneast

##Create Virtual NW
Vnetname=DB-patch-proxy-Vnet
SBnetname=DB-patch-proxy-subnet
az network vnet create 
--name $Vnetname 
--resource-group $RGname 
--subnet-name $SBnetname 
--address-prefix 10.37.0.0/24 
--subnet-prefix 10.37.0.0/26

#Create Linux jump server
##resource group:DB-patch-proxyに、Ubuntu18.04(imageにはUbuntuLTSを指定)を構築
VMname=Linux-jump-server
az vm create 
--resource-group $RGname 
--name $VMname 
--image UbuntuLTS 
--admin-username azureuser 
--generate-ssh-keys
--vnet-name $Vnetname 
--subnet $SBnetname 
--assign-identity

#Configuration Linux jump server after creating
##AzureAD extention
az vm extension set 
--publisher Microsoft.Azure.ActiveDirectory 
--name AADSSHLoginForLinux 
--resource-group $RGname 
--vm-name $VMname

##(GUIで実施)Linux VMにKMR VPNのインバウンド許可をするnsgを追加
仮想マシンlinux-jump-serverのNetworkingタグからKMR VPNである『52.156.52.223』を許容するよう設定


###NSG作成
nsgname=KMRInboundallow
az network nsg create -g $RGname -n $nsgname -l japaneast

###NSG作成の確認
az network nsg list -g $RGname | grep $nsgname

###NSGのルールを作成(KMRのVPN52.156.52.223からのインバウンドを許す)
TBU(下記テスト中も成功せずサポート問合せ予定)

az network nsg rule create 
--name KMRallow 
--nsg-name $nsgname 
--priority 150 
--resource-group $RGname 
--access Allow 
--description "allow Inbound access from KMRVPN" 
--destination-port-ranges ' ' 
--direction Inbound 
--protocol TCP 
--source-address-prefixes 52.156.52.223 
--source-port-ranges ' '  
--subscription $subscriptionname

az network nsg rule create 
--name KMRallow 
--nsg-name $nsgname 
--priority 150 
--resource-group $RGname 
--access Allow 
--description "allow Inbound access from KMRVPN" 
--direction Inbound 
--protocol TCP 
--source-address-prefixes 52.156.52.223 
--subscription $subscriptionname

☞Portが規定値の80のままで、ServiceがHTTPになっている。それぞれ、PoCでは22,SSHになっていた。
☞右記オプションを追記して実行すると、--destination-address-prefixes 'SSH' 
invalid Address prefix. Value provided: SSHと怒られる
☞対応策不明のためサポート

###Linux NICにnsgを追加して更新(未実施)
az network nic update 
--resource-group myResourceGroup 
--name myNic 
--network-security-group $nsgname

##(GUIで実施)Tags (Env/prod)
仮想マシンのTagタグからenv:prodを追加


##(GUIで実施)Addition new roles(Devから受領するアカウントをuser.nameに入れて変数に代入する)
仮想マシンlinux-jump-serverのAccess Control (IAM)タグで、”Virtual Machine User Login”権限を付与
※PostgreSQLをインストールする際はAdmin権限が必要なため”Virtual Machine Administrator Login”権限を付与


 

TBU(下記実行もエラーが出てしまい確認中))

username=username1
az role assignment create 
--role "Virtual Machine User Login" 
--assignee $username 
--scope "/subscriptions/87e400dc-091c-4a03-8e8e-2e05a02c3f6d/resourceGroups/$RGname",

#Install PostgreSQL version9.6 on the linux jump server
##LinuxにSSH接続し、Linuxのターミナル上で操作しパッケージのダウンロード一覧にpostgreを追加
sudo sh -c "echo 'deb Index of /pub/repos/apt/  bionic-pgdg main' > /etc/apt/sources.list.d/pgdg.list"
##公開鍵を信頼キーリストに追加
wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
##公開鍵の確認(B97B 0AFC AA1A 47F0 44F2  44A0 7FCC 7D46 ACCC 4CF8があることを確認)
apt-key list
##パッケージを更新
sudo apt update
##Postgre v9.6をインストール
sudo apt install postgresql-9.6
##psqlがパスのbinディレクトリはいかにあることを確認(/usr/bin/psqlが返ってくることを確認)
which psql

#(GUIで実施)logging Configuration
##Creating workspace
DB-patch-proxy-workspace という名前のワークスペースを、下記場所に作成
サブスクリプション:KMR-Run-Nissan
Resource Group:db-patch-proxy

Microsoft Azure 

TBU(下記実行もエラーが出てしまい確認中)
az deployment group create --resource-group $RGname --name WS-DB-patch-proxy --template-file deploylaworkspacetemplate.json

##Agent Install and connection to workspace
仮想マシンlinux-jump-serverのLogタグからEnableを押下


az vm extension set 
--resource-group myResourceGroup 
--vm-name myVM 
--name OmsAgentForLinux 
--publisher Microsoft.EnterpriseCloud.Monitoring 
--protected-settings '{"workspaceKey":"myWorkspaceKey"}' 
--settings '{"workspaceId":"myWorkspaceId"}'

##Workspace's Agents configuration(Syslog addition including debug)
ワークスペースDB-patch-proxy-workspaceのAgents configurationタグで、
LinuxのSyslogタブを開き、Facilityとしてauthとauthprivを追加し、全ての重大度レベルにチェックを入れて
Applyを押下する (ログインがauthprivに、ログアウトがauthのファシリティで記録されているため)


 

###/etc/rsyslog.d/95-omsagent.confのファイルで"warning"を"debug"に書き換える(未実施)
cd /etc/rsyslog.d
vim 95-omsagent.conf

#Allow Linux server's IP on PostgreSQL's FW
要相談★
アカウントの切替が必要なためどう纏めて処理するべきか？

##(GUIで実施)get Linux servers's IP

##Postgres FW追加 (まずはjp-prd-sa-vnextproxy-replicaで実施しそのあと全体に対してFW設定)
###サブスクリプションの切替 (jp-prd-sa-vnextproxy-replicaのサブスクリプションはKMR-Run)
subscription=KMR-Run
az account set --subscription $subscription

###FWの追加
LinuxIP=20.89.143.200
DBname=jp-prd-sa-vnextproxy-replica
RGname=jekplatprod
FWname=linuxjump-inbound
az postgres server firewall-rule create --name $FWname --resource-group $RGname --server-name $DBname --start-ip-address $LinuxIP --end-ip-address $LinuxIP

###Postgres server FW情報の確認 ($FWnameの登録があることを確認)
az postgres server firewall-rule list --resource-group $RGname --server-name $DBname | grep $FWname

#Test login
要相談★
対象DBのPWが取れない
（Apps Managerで探せない、探せたとしても全て対応できない）

#Close old IP on each DB
TBU
