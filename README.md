# fabric-dev-server-single-node-setup
Fabric single node setup with composer rest server, angular-app, and sample bna
### Sample application

```
index.html
package.json
```

### Diagram sources:

* http://usblogs.pwc.com/emerging-technology/a-primer-on-blockchain-infographic/
* http://www.think-foundry.com/wp-content/uploads/2017/11/fabricDeployment.png
* https://fabrictestdocs.readthedocs.io/en/latest/glossary.html

### Launch AWS Ubuntu 18.04 

```

ubuntu-minimal/images-testing/hvm-ssd/ubuntu-bionic-daily-amd64-minimal-20180810 - ami-032ac327c5850744f
Canonical, Ubuntu, 18.04 LTS Minimal, UNSUPPORTED daily amd64 bionic minimal image built on 2018-08-10
Root Device Type: ebs Virtualization type: hvm

Community AMI: ami-032ac327c5850744f
Instance type: t2.xlarge 16GB

ssh -i "blockchain.pem" ubuntu@IP1

Copying fabric-dev to IP2
a. On Local: scp -i "blockchain.pem" blockchain.pem ubuntu@IP1:/home/ubuntu
b. On IP1: scp -i "blockchain.pem" -r fabric-dev-servers-multipeer/ ubuntu@IP2:/home/ubuntu
c. Enable the SG group on AWS to allow all.

```

#### Here are the full terminal instructions starting from a basic Ubuntu 18.04 install 

```
sudo apt update -y && sudo apt upgrade -y

Update only:
sudo apt update -y
sudo apt install nano git make gcc g++ libltdl-dev curl python pkg-config vim -y
```

### Install Docker/ Docker Compose
```
curl -fsSL test.docker.com | sh
sudo usermod -aG docker $USER
exec sudo su -l $USER
sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Install nodejs / Composer

```
cd ~
wget https://nodejs.org/dist/v8.11.2/node-v8.11.2-linux-x64.tar.xz
tar -xf node-v8.11.2-linux-x64.tar.xz 
mv node-v8.11.2-linux-x64/ node/
echo 'export PATH=~/node/bin:$PATH' >> ~/.profile
source ~/.profile
npm install -g npm 
npm install -g grpc
npm install -g composer-cli@0.20 composer-playground@0.20 generator-hyperledger-composer@0.20
npm install -g composer-rest-server@0.20
npm install -g yo
```

### Install Go / Hyperledger Binaries
```
wget https://dl.google.com/go/go1.10.2.linux-amd64.tar.gz
sudo tar -C /usr/local -xvf go1.10.2.linux-amd64.tar.gz
echo 'export GOPATH=/opt/go' >> ~/.profile
echo 'export GOBIN=/opt/go/bin' >> ~/.profile
echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.profile
source ~/.profile
sudo mkdir -p /opt/go/bin
sudo mkdir /opt/go/src
cd $GOPATH
sudo chown -R $USER /opt/go
cd src
mkdir -p github.com/hyperledger
cd github.com/hyperledger
git clone https://gerrit.hyperledger.org/r/fabric
cd fabric
make release
```

### Download Fabric - https://hyperledger.github.io/composer/latest/installing/development-tools.html
```
cd ~
mkdir ~/fabric-dev-servers && cd ~/fabric-dev-servers
curl -O https://raw.githubusercontent.com/hyperledger/composer-tools/master/packages/fabric-dev-servers/fabric-dev-servers.tar.gz
tar -xvf fabric-dev-servers.tar.gz
cd ~/fabric-dev-servers
export FABRIC_VERSION=hlfv12
./downloadFabric.sh

```

### Starting/stopping 
```
cd ~/fabric-dev-servers
export FABRIC_VERSION=hlfv12
./startFabric.sh
./createPeerAdminCard.sh
```

### Start Composer Plaground 
```
nohup composer-playground -p 8181 &
http://IP1:8181/

```

### Create Sample business network archive 1 - collections-network

```

mkdir ~/sample-network-bna
cd ~/sample-network-bna

yo hyperledger-composer:businessnetwork

? Business network name: collections-network
? Description: Collections Network
? Author name:  John Doe
? Author email: johndoe@johndoe.com
? License: Apache-2.0
? Namespace: org.example.mynetwork

Do you want to generate an empty template network? 
> No: generate a populated sample network

   create package.json
   create README.md
   create models/org.example.mynetwork.cto
   create permissions.acl
   create .eslintrc.yml
   create features/sample.feature
   create features/support/index.js
   create test/logic.js
   create lib/logic.js

cd collections-network

Remove and replace with below data

$ nano models/org.example.mynetwork.cto

/**
 * My collections network
 */
namespace org.example.mynetwork
asset Account identified by accountNumber {
    o String accountNumber
    o Double balance
    o String status
    o String dueDate
    o Double minPay
    --> Customer customer
}
participant Customer identified by customerId {
    o String customerId
    o String dateOfBirth
}
transaction makePayment {
    --> Account account
    --> Customer customer
    o Double paymentAmt
}

Remove and replace with below data

$ nano lib/logic.js

/**
 * Make payment
 * @param {org.example.mynetwork.makePayment} tx - the customer to be processed
 * @transaction
 */
async function makePayment(tx) {
    let assetRegistry = await getAssetRegistry('org.example.mynetwork.Account');
    tx.account.customer.accountNumber = tx.customer.customerId;
    tx.account.balance = tx.account.balance - tx.paymentAmt;
    tx.account.status = 'PAID';
    tx.account.minPay = 0;
    await assetRegistry.update(tx.account);
}

$ nano permissions.acl

/**
 * Access control rules for collections-network
 */
rule Default {
    description: "Allow all participants access to all resources"
    participant: "ANY"
    operation: ALL
    resource: "org.example.mynetwork.*"
    action: ALLOW
}

rule SystemACL {
  description:  "System ACL to permit all access"
  participant: "ANY"
  operation: ALL
  resource: "org.hyperledger.composer.system.**"
  action: ALLOW
}

Build BNA

composer archive create -t dir -n .
composer network install --card PeerAdmin@hlfv1 --archiveFile collections-network@0.0.1.bna
composer network start --networkName collections-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card
composer card import --file networkadmin.card
composer network ping --card admin@collections-network 

Refresh the Composer UI in the browser
http://IP:8181

Enable the REST Server
nohup composer-rest-server -c admin@collections-network  -n never -w true & 

http://IP:3000/explorer/

Sample API request:

{
  "$class": "org.example.mynetwork.makePayment",
  "account": "ACT1111",
  "customer": "JohnDoe",
  "paymentAmt": 1500,
  "transactionId": "",
  "timestamp": "2018-08-20T22:42:37.263Z"
}

```


### Start angular-app - https://hyperledger.github.io/composer/latest/tutorials/developer-tutorial.html
```

mkdir ~/sample-network-bna/collections-network-app
cd ~/sample-network-bna/collections-network-app
cp ~/sample-network-bna/collections-network/collections-network\@0.0.1.bna .

yo hyperledger-composer:angular


Welcome to the Hyperledger Composer Angular project generator
? Do you want to connect to a running Business Network? No
? Project name: collections-network-app
? Description: Collections Network App
? Author name: John Doe
? Author email: johndoe@johndoe.com
? License: Apache-2.0
? Business network archive file (Path from the current working directory): collections-network@0.0.1.bna
? REST server address: http://YOUR_IP
? REST server port: 3000
? Are namespaces used in the generated REST API:  Namespaces are not used

Allow access to YOUR_IP - Change from false to return true;

nano ~/sample-network-bna/collections-network-app/collections-network-app/node_modules/webpack-dev-server/lib/Server.js (line 425 nano - Ctrl _):

Start it
cd ~/sample-network-bna/collections-network-app/collections-network-app
nohup npm start &
cat nohup.out
On Browser -> http://IP:4200/

```

## Sample REST-Client Application (Javascript)

```

mkdir ~/sample-network-bna/collections-network-ui
cd ~/sample-network-bna/collections-network-ui

$ nano index.html 

<!DOCTYPE html>
<html lang="en">
<head>
  <title>Collections Network UI - List Of Accounts</title>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
  <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js"></script>

<script>

var service = 'http://34.230.1.116:3000/api';

function makePayment(inputData) {
  console.log(inputData);
  var inputDataAr = inputData.split("#");
  var amount = prompt("Please specify amount to pay for:\nAccount# " + inputDataAr[0] + "\nCustomer Name: " + inputDataAr[1] +"\nMinimum Pay: "+ inputDataAr[2], inputDataAr[2]);
  if (amount != null) {
      makePaymentAjax(inputDataAr[0], inputDataAr[1], amount);
  }
  
};

function makePaymentAjax(account, customer, paymentAmt) {

   jQuery.support.cors = true;

    $.ajax(
    {
        type: "POST",
        url: service + '/makePayment',
        data: "{\"account\":\""+account+"\", \"customer\":\""+customer+"\", \"paymentAmt\":\""+paymentAmt+"\"}",
        contentType: "application/json",
        dataType: "json",
        cache: false,
        success: function (data) {  
          console.log(data);
          $("#accounts tbody").empty();
          loadAccount();
        },
        
        error: function (msg) {
            alert(msg);
        }
    });

    console.log('Payments success.');

};

function loadAccount() {

   jQuery.support.cors = true;

    $.ajax(
    {
        type: "GET",
        url: service + '/Account',
        data: "{}",
        contentType: "application/json; charset=utf-8",
        dataType: "json",
        cache: false,
        success: function (data) {
            
        var trHTML = '';
                
        $.each(data, function (i, item) {
            
            var custName = item.customer;
            var custName1 = custName.split("#");
            trHTML += '<tr><td>' + (i+1) + '.</td><td>' + item.accountNumber + '</td><td>' + custName1[1] + '</td><td>' + item.balance + '</td><td>' + item.status + '</td><td>' + item.dueDate + '</td><td>' + item.minPay + '</td><td><button onclick="makePayment( \''+item.accountNumber+'#'+custName1[1]+'#' +item.minPay+'\')" class="btn btn-primary">Pay</button></td></tr>';

        });
        
        $('#accounts').append(trHTML);
        
        },
        
        error: function (msg) {
            alert(msg);
        }


    });

    console.log('Accounts loaded.');

};

$(document).ready(function(){

  loadAccount();

})

</script>


</head>
<body>

<div class="container">
  <br/><br/><br/>
  <h2>Collections Network UI</h2><h3>List Of Accounts in Hyperledger Blockchain</h3>
  <p>List of account numbers in Card Network</p>            
  <table id="accounts" class="table table-hover">
    <thead>
      <tr>
        <th>No.</th>
        <th>Acct#</th>
        <th>Customer</th>
        <th>Balance</th>
        <th>Status</th>
        <th>Date Due</th>
        <th>Minimum Pay</th>
        <th>Action</th>
      </tr>
    </thead>
    <tbody>
    </tbody>
  </table>
</div>

</body>
</html>

$ nano package.json 
{
  "name": "collectionsnetworkui",
  "version": "1.0.0",
  "description": "Collections Network UI",
  "local-web-server": {
    "port": 8100,
    "forbid": "*.json"
  },
  "scripts": {
    "start": "ws",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "John Doe",
  "license": "MIT",
  "devDependencies": {
    "local-web-server": "^1.1.0"
  },
  "repository": {
   "type": "git",
   "url": "https://github.com/blockchainfellow/fabric-dev-server-single-node-setup/edit/master/README.md"
  },
  "dependencies": {}
}

Starting it.

npm install
nohup npm start &
http://YOUR_IP:8100/

```

## Old Examples

### Create business network 2 - https://hyperledger.github.io/composer/latest/tutorials/developer-tutorial.html

```
yo hyperledger-composer:businessnetwork

? Business network name: tutorial-network
? Description: Sample Network
? Author name:  John Doe
? Author email: johndoe@chain.com
? License: Apache-2.0
? Namespace: org.example.mynetwork

Do you want to generate an empty template network? 
No: generate a populated sample network

   create package.json
   create README.md
   create models/org.example.mynetwork.cto
   create permissions.acl
   create .eslintrc.yml
   create features/sample.feature
   create features/support/index.js
   create test/logic.js
   create lib/logic.js

cd tutorial-network

vi models/org.example.mynetwork.cto
/**
 * My commodity trading network
 */
namespace org.example.mynetwork
asset Commodity identified by tradingSymbol {
    o String tradingSymbol
    o String description
    o String mainExchange
    o Double quantity
    --> Trader owner
}
participant Trader identified by tradeId {
    o String tradeId
    o String firstName
    o String lastName
}
transaction Trade {
    --> Commodity commodity
    --> Trader newOwner
}

vi lib/logic.js
/**
 * Track the trade of a commodity from one trader to another
 * @param {org.example.mynetwork.Trade} trade - the trade to be processed
 * @transaction
 */
async function tradeCommodity(trade) {
    trade.commodity.owner = trade.newOwner;
    let assetRegistry = await getAssetRegistry('org.example.mynetwork.Commodity');
    await assetRegistry.update(trade.commodity);
}

vi permissions.acl
/**
 * Access control rules for tutorial-network
 */
rule Default {
    description: "Allow all participants access to all resources"
    participant: "ANY"
    operation: ALL
    resource: "org.example.mynetwork.*"
    action: ALLOW
}

rule SystemACL {
  description:  "System ACL to permit all access"
  participant: "ANY"
  operation: ALL
  resource: "org.hyperledger.composer.system.**"
  action: ALLOW
}

Generate a business network archive
composer archive create -t dir -n .

Install the business network
composer network install --card PeerAdmin@hlfv1 --archiveFile tutorial-network@0.0.1.bna

Start the business network
composer network start --networkName tutorial-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card

Import the network administrator identity as a usable business network card
composer card import --file networkadmin.card

Check that the business network has been deployed successfully,
composer network ping --card admin@tutorial-network

Generating REST server - https://hyperledger.github.io/composer/latest/tutorials/developer-tutorial.html

composer-rest-server

? Enter the name of the business network card to use: admin@tutorial-network
? Specify if you want namespaces in the generated REST API: never use namespaces
? Specify if you want to use an API key to secure the REST API: No
? Specify if you want to enable authentication for the REST API using Passport: No
? Specify if you want to enable event publication over WebSockets: Yes
? Specify if you want to enable TLS security for the REST API: No

composer-rest-server -c admin@tutorial-network -n never -w true

http://IP1:3000/explorer/#/

Sample request:

{
  "$class": "org.example.mynetwork.Trade",
  "commodity": "IBM",
  "newOwner": "JohnDoe",
  "timestamp": "2018-08-20T07:04:25.893Z"
}

```

### Start composer in Docker

```

cd ~/fabric-dev-servers
export FABRIC_VERSION=hlfv12
./startFabric.sh
./createPeerAdminCard.sh

sudo apt install telnet
sudo apt install iputils-ping

Replace IP1 with your public/private IP

PeerAdmin

sed -e 's/localhost:7051/IP1:7051/' -e 's/localhost:7053/IP1:7053/' -e 's/localhost:7054/IP1:7054/'  -e 's/localhost:7050/IP1:7050/' < $HOME/.composer/cards/PeerAdmin@hlfv1/connection.json  > /tmp/connection.json && cp -p /tmp/connection.json $HOME/.composer/cards/PeerAdmin@hlfv1/

Collections Network

sed -e 's/localhost:7051/IP1:7051/' -e 's/localhost:7053/IP1:7053/' -e 's/localhost:7054/IP1:7054/'  -e 's/localhost:7050/IP1:7050/' < $HOME/.composer/cards/admin@collections-network/connection.json  > /tmp/connection.json && cp -p /tmp/connection.json $HOME/.composer/cards/admin@collections-network/

composer network install --card PeerAdmin@hlfv1 --archiveFile collections-network@0.0.1.bna
composer network start --networkName collections-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card
composer card import --file networkadmin.card
composer network ping --card admin@collections-network 

composer network list -c admin@collections-network
composer network ping -c admin@collections-network

Tutorial Network

sed -e 's/localhost:7051/IP1:7051/' -e 's/localhost:7053/IP1:7053/' -e 's/localhost:7054/IP1:7054/'  -e 's/localhost:7050/IP1:7050/' < $HOME/.composer/cards/admin@tutorial-network/connection.json  > /tmp/connection.json && cp -p /tmp/connection.json $HOME/.composer/cards/admin@tutorial-network/

composer network install --card PeerAdmin@hlfv1 --archiveFile tutorial-network@0.0.1.bna
composer network start --networkName tutorial-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card
composer card import --file networkadmin.card
composer network ping --card admin@tutorial-network 

docker rm composer-playground
docker run --name composer-playground -v ~/.composer:/home/composer/.composer --publish 80:8080 hyperledger/composer-playground

Run on background:
docker run --name composer-playground -v ~/.composer:/home/composer/.composer --publish 80:8080 --detach hyperledger/composer-playground

References:
https://medium.com/coinmonks/installing-hyperledger-composer-playground-58ad359d4a2f
https://hyperledger.github.io/composer/latest/tutorials/google_oauth2_rest
https://stackoverflow.com/questions/50739551/running-composer-playground-in-docker-container-doesnt-connect-to-fabric-networ?newreg=f237a94fc58b40e8bfbebd8dd817a2e0

```

### Create the Composer profile on the First Machine and start Composer Playground and Blockchain Explorer
```
cd ~
git clone https://github.com/InflatibleYoshi/blockchain-explorer
sudo apt install postgresql postgresql-contrib
cd blockchain-explorer
git checkout release-3
```
Edit config.json with the correct tlscerts path. You do not need them functionally but they are there because there have been reported issues when not including the tls certs.
```
config.json

{
        "network-config": {
                "org1": {
                        "name": "org1",
                        "mspid" : "Org1MSP",
                        "peer0": {
                                "requests": "grpc://localhost:7051",
                                "events": "grpc://localhost:7053",
                                "server-hostname": "peer0.org1.example.com",
                                "tls_cacerts": "/home/ubuntu/fabric-dev-servers/fabric-scripts/hlfv12/composer/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt"
                        },
                        "admin": {
                                "key": "/home/ubuntu/fabric-dev-servers/fabric-scripts/hlfv12/composer/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore",
                                "cert": "/home/ubuntu/fabric-dev-servers/fabric-scripts/hlfv12/composer/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts"
                        }
                }

        },
        "host": "localhost",
        "port": "8080",
        "channel": "composerchannel",
        "keyValueStore": "/tmp/fabric-client-kvs",
        "eventWaitTime": "30000",
        "users":[
                {
                   "username":"admin",
                   "secret":"adminpw"
                }
         ],
        "pg": {
                "host": "127.0.0.1",
                "port": "5432",
                "database": "fabricexplorer",
                "username": "hppoc",
                "passwd": "password"
        },
        "license": "Apache-2.0"
}


sudo -u postgres psql
\i app/db/explorerpg.sql
\q
npm install
cd app/test
npm install
npm run test
cd ../../client
npm install
npm test -- -u –coverage
npm run build
cd ..
./start.sh
```

At this point you should be able to navigate a browser to http:/{HOST1-DOMAIN/IP}:8080 and connect to either alice or bob's trade network instances.



