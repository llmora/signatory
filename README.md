signatory: online Ethereum signer
=================================

When implementing applications around Ethereum at some stage we need to
start sending transactions over the network, an operation that requires
access to the wallet private key in order to sign the transaction. Because
the private key controls all the expenditure of a wallet, if it is
compromised the attacker can empty the wallet without any chance of
recovering it.

Given the risk of a private key compromise, it seems unwise to give an
application access to it - however without this access we can't create
applications that send transactions without manual intervention.

The goal of signatory is to allow for the automated creation of transactions
while minimising the risks of a private key compromise. This is implemented
in the way of a small ruby-based application with a very limited attack
surface and that implements as many controls a possible to reduce the risk
of compromise.

We cannot promise that the private key will not be compromised, but *we can
assure you that the key will be as protected as it can be while still
allowing for online signatures*. The alternative is manual offline
transactions, which may make sense for large transactions but that just
hinders smaller transactions that happen very often.

The "Risk assessment" section at the end of this document explains the
threats  to the private key and the controls implemented by signatory to
reduce the risk of a compromise. If you think the application can be
improved in any way, just submit a PR please.

**Signatory key functionalities**

  - HTTP-based API, for ease of integration with client applications
  - Key password is not stored on disk, requires manual input on startup
  - Configurable maximum amount of ETH to accept transfers for
  - Configurable list of destinations, so that ETH cannot be transferred to
    unknown addresses

Installation
------------

The easiest way to install signatory is to clone the git repository and run bundler. This installs all required dependencies and leaves it ready to be configured and run:

```
$ git clone https://github.com/llmora/signatory
$ cd signatory
$ bundle install
```

Configuration
-------------

signatory supports configuration through a `settings.yml` on the `config` directory of the application. The settings file accepts a set of `key: value` parameters:

- *source*: source of the ETH transfers, must be the address associated with the key file

- *keyfile*: name of the file that contains the key, in web3 format (such as the ones generated by geth or [ruby-eth](https://github.com/se3000/ruby-eth))

- *destination*: list of destination ETH addresses that we support trasnfers to, requests to sign transactions with another destination will be rejected

- *transfer_limit_wei*: Maximum amount in WEI to transfer, requests to sign transactions above this amount will be rejected

- *gas_price*: (optional) Gas price in WEI we are willing to pay for the transaction to be added to a block (by default 41 Gwei)

- *etherscan_api_token*: (optional) EtherScan.io API token, in case you have an account. By default we use the anonymous interface, which has throttling restrictions - if you plan to make heavy use of the signer get an API token from them

- *network*: (optional) Choose which network to operate on: "mainnet" (default) or any of the test networks "ropsten", "kovan" or "rinkeby"

An example content of `settings.yml`:
```yaml
  source: "03969653c12...78563fd186dee62"
  keyfile: eth-key.json
  destinations: ["0xe3365D1a2...0CD181e0C22305", "0xfbc3DCf2567...8b2F7C1a2"]
  transfer_limit_wei: 1_000_000_000_000_000_000 # 1 ETH
```

Running
-------

The application is based around Sinatra, just run it through rackup:

```sh
$ bundle exec rackup config.ru -p <port>
```

This will start the API web service on the specified port
On startup signatory will ask you for the password of the private key, once unlocked the API web service will be started on the specified port - if not specified defaults to port 9292

Interacting with the API
------------------------

The API accepts requests and responds over JSON. The response format is as follows:

  { "status": "<status>", "message": "<message>", [ "data": <data> ] }

  - status: One of "ok" or "error". Use this value to verify the correctness of the response
  - message: A descriptive message of the response status. Use this value to learn more about errors
  - data: Only present in sucessful responses, JSON representation of the response content

signatory exposes two methods over the API:

  `/status`: Accessible over GET, call it to verify if the server is ready for signing.

    Request: Accepts no parameters

    Response: If the server is ready a status of "ok" is sent, otherwise an "error" along with an explanation in "message" is presented.

  `/sign`: Accessible over POST, it finds the next nonce for the transaction, signs it with the configured key and returns the content of the transaction.

    Request: Accepts two parameters in JSON: "destination" is the address that the transfer will go to and "amount" is the value in wei that you want to transfer. 

    Response: If the transfer is successful a status of "ok" is sent and the "data" portion of the response contains two keys: "transaction" with the hex representation of the transaction and "hash", the hash of the transaction, ready to be submitted to the nextwork through your local node or using one of the "pushTx" services.

Running in Docker
-----------------

signatory ships with a Dockerfile that allows it to run under
Docker without any modification, just drop a 'settings.yml' and your
ethereum keyfile in the `config` directory, then build the docker container:

```sh
$ docker build -t signatory .
```

The container will only be operational once you enter the key, so make sure
you run it with tty and stdin redirection:

```sh
$ docker run -it signatory
```

If you are using `docker-compose` to deploy a group of containers you need
to add 'tty: true' and 'stdin_open: true' so that it sets up a tty and waits
for stdin. After deploying the containers just use `docker attach` to
connect and enter the passphrase (there will be no prompt, just type in the
passphrase and hit enter).

Risk assessment
---------------

To be completed...

License
-------

signatory is released as open source under an [MIT License](http://opensource.org/licenses/MIT)
