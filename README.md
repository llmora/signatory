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
improved in any way, just submit a pull request please.

**Signatory key functionalities**

  - HTTP-based API, for ease of integration with client applications
  - Key password is not stored on disk, requires manual input on startup
  - Configurable maximum amount of ETH to accept transfers for
  - Configurable list of destinations, so that ETH cannot be transferred to unknown addresses

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

  `/sign`: Accessible over POST, signs a transaction with the configured key and returns it.

    Request: Accepts three parameters in JSON: "destination" is the address that the transfer will go to, "amount" is the value in wei that you want to transfer and "nonce" is the nonce to use for the transaction.

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

`signatory` attempts to be as secure as possible, the following risks have been identified that may impact the security of the application (pull requests welcome to identify additional threats and improve the current controls):

**The wallet private key is stolen after a break-in to the server running signatory**

`signatory` checks that the key is encrypted on disk (R1) and refuses to start otherwise.

Controls not yet implemented:

   - Memory dump can extract decrypted private key
   - Debugger can extract decrypted private key

**The wallet passphrase is stolen after a break-in on the server running signatory**

The key must be fed using a TTY on start-up, the passphrase will only be read from a TTY so if the user attempts to fed it through stdin the application will refuse to start (R6).

In addition, just in case the user fakes a TTY, a further check is conducted to ensure that there is a delay between application startup and the key being entered (R5) to avoid the key being store on disk.

After the key is unlocked the passphrase is wiped from memory as soon as possible, to avoid a memory dump from showing the passphrase (R2).

We considered using a HTTP request to unlock the key, but the amount of copies of data that would be laying around memory put us off.

**An attacker abuses the signing interface to transfer money to their wallet**

The configuration file (which can only be remotely modified through the application) holds a list of allowed destination wallet addresses (parameter *destination*), so that only trusted wallets can receive the transactions (R3).

The amount of currency that can be transferred is capped through the *transfer_limit_wei* configuration setting (R4).

**The application gets replaced by a fake application that mimics the TTY prompt and tricks the user into entering the passphrase on start-up**

Controls not yet implemented:

`signatory` must demonstrate that it has access to something the attacker does not - but because at this stage the key is still locked we cannot use a key signature to demonstrate the application is legitimate. There is no other property that we can use, so it is not possible to have the application authenticate itself - we must fall back to external verification methods that the attacker would not have subverted.

Also if the attacker is able to modify the application we need to consider the private key as compromised and alert the user.

Controls not yet implementes:

  - Trusted execution enclaves (e.g. intel GSX) that validate the application and protect the key

**An attacker (or a failed disk) wipes the private key**

Controls not yet implemented:

   - Regularly remind the user to backup the key
   - Use secure enclaves to distribute copies of the key

**Signatory is unavailable due to a DoS attack on the HTTP interface**

Controls not yet implemented:

   - Use Rack::Security to limit the number of requests
   - Filter source IPs
   - Authenticate user - mutual authentication

**Transaction parameters are altered in transit between client application and signatory**

Controls not yet implemented:

   - Protect the access to application with HTTPS
   - Audit log of all transactions kept and signed
   - Verify on client that the signed transaction has not been altered, before submitting it

License
-------

signatory is released as open source under an [MIT License](http://opensource.org/licenses/MIT), check the LICENSE.txt file for more details.
