# Bitshares HUG API

![BTS HUG API Banner](https://i.imgur.com/secsyPh.png "BTS HUG API Banner")

## About

The intention of this Bitshares HUG REST API repo is to provide the Bitshares network an open-source high performance interface to the Bitshares network through simple GET requests.

By following the readme, you can easily recreate the API in your own control. Do remember to change the API key and the Bitshares FULL/API node you're connecting to. If you're creating a service which will produce a large amount of traffic, alert the node operator or consider running your own Bitshares node.

This HUG REST API makes heavy use of the [python-bitshares]()

## TODO

    Improve the NGINX & Gunicorn configurations
    Implement additional HUG functions.
    Work on ./WIP.py - Difficult broken code staging file.
    Use [websocketrpc](http://docs.pybitshares.com/en/latest/websocketrpc.html) to expose additional functionality (worker proposal, item search (committee info & worker proposal lists)) to HUG.
    Correctly handle 'datetime' user input for ranges of data requests (E.g. TX history between Feb & Mar 2017).

## Future usage plans

Once this API is completed, I'll be looking into creating an open source (MIT Licensed) [Google Assistant](https://developers.google.com/actions/) for Bitshares.

## License

The contents of this entire repo should be considered MIT licenced.

## How to contribute

It's difficult to debug development issues whilst running behind Gunicorn & NGINX, you're best running the HUG REST API directly with HUG during development "hug -f bts_api.py". Note that running HUG directly in this manner should only be performed during development, it is not suitable for exposing directly to the public as a production ready API.

If you want to donate: [Customminer](http://cryptofresh.com/u/customminer)

### About : Python-Bitshares

https://github.com/xeroc/python-bitshares/tree/develop

Created by xeroc, it's a thorough Bitshares python library which will be extensively used throughout this API. We won't be using it for any serious wallet control, purely the read-only blockchain/account/asset monitoring functionality.

Web Docs: http://docs.pybitshares.com/en/latest/

PDF Docs: https://media.readthedocs.org/pdf/python-bitshares/latest/python-bitshares.pdf

### About: HUG

> ##### Embrace the APIs of the future
> Drastically simplify API development over multiple interfaces. With hug, design and develop your API once, then expose it however your clients need to consume it. Be it locally, over HTTP, or through the command line - hug is the fastest and most modern way to create APIs on Python3.
> ##### Unparalleled performance
> hug has been built from the ground up with performance in mind. It is built to consume resources only when necessary and is then compiled with Cython to achieve amazing performance. As a result, hug consistently benchmarks as one of the fastest Python frameworks and without question takes the crown as the fastest high-level framework for Python 3.
>
> Source: [Official website](http://www.hug.rest/).

### About: Extensibility

If your HUG functions takes a long time to compute, then you must account for NGINX & Gunicorn worker timeouts (both the systemctl service file & the 'default' NGINX sites-available file). If you fail to account for this, the user will experience unhandled timeouts.

Since HUG utilizes Python, any Python library can be used to process/manipulate Bitshares data.

Ideally, rather than over-scraping data we should cache it or limit scraping functions to large batches (1000 instead of 500k etc).

---

## Install guide

This is an install guide for Ubuntu 17.10, it uses Python3+, HUG, Gunicorn & NGINX. If you change the OS or server components then the following guide will be less applicable, if you succeed please do provide a separate readme for alternative implementation solutions.

### Setup dependencies & Python environment

NOTE: We use the develop branch of python-bitshares because it uses the pycryptodome package (required since pycrypto is depreciated).

We create the 'btsapi' user, however you could rename this to whatever you want, just remember to change the NGINX & Gunicorn configuration files.

#### Setup a dedicated user

    adduser btsapi
    <ENTER NEW PASSWORD>
    <CONFIRM NEW PASSWORD>
    usermod -aG sudo btsapi
    sudo usermod -a -G www-data btsapi
    su - btsapi

#### Install required applications

    sudo apt-get install libffi-dev libssl-dev python3-pip python3-dev build-essential git nginx python3-setuptools virtualenv

#### Create Python virtual environment

    mkdir HUG
    virtualenv -p python3 HUG
    echo "source ./HUG/bin/activate" > access_env.sh
    chmod +x access_env.sh
    source access_env.sh

#### Install Python packages

    pip3 install --upgrade pip
    pip3 install --upgrade setuptools
    pip3 install --upgrade wheel
    pip3 install hug
    pip3 install gunicorn
    git clone https://github.com/xeroc/python-bitshares.git -b develop
    pip3 install -e python-bitshares/

### Configure NGINX

NGINX serves as a reverse web proxy to Gunicorn & uses an UNIX socket instead of an IP address for referencing Gunicorn.

    Copy the nginx.conf file to /etc/nginx/
    Reset nginx (sudo service nginx restart)

    sudo mv default /etc/nginx/sites-available/default

### Implement SSL Cert

You aught to implement a free LetsEncrypt SSL certificate, this requires a domain name (they don't sign IP addresses) and it needs to be renewed every few months by running certbot again.

https://certbot.eff.org/

    sudo add-apt-repository ppa:certbot/certbot
    sudo apt-get update
    sudo apt-get install python-certbot-nginx
    sudo certbot --nginx -d api.domain.tld

### Configure Gunicorn

Official website: http://gunicorn.org/

Documentation: http://docs.gunicorn.org/en/stable/

Gunicorn is used to provide scalable worker process management and task buffering for the HUG REST API. Gunicorn's documentation states that each CPU can provide roughly 2-3+ Gunicorn workers, however it may be able to achieve a higher quantity (worth testing).

    cp gunicorn.service /etc/systemd/system/gunicorn.service
    sudo systemctl start gunicorn
    sudo systemctl enable gunicorn

### MISC

If you make changes to the service or the hug script:

    sudo systemctl daemon-reload
    sudo systemctl restart gunicorn

If you want to monitor Gunicorn:

    tail -f gunicorn_access_log
    tail -f gunicorn_error_log
    sudo systemctl status gunicorn

---

# Available HUG REST API functionality

This section will detail the functionality which will be available to the public through GET requests.

The functions are currently all read-only functions, enabling the public to request data from the network without the risk of exposing critical wallet controls.

## Blockchain functions

More info: [python-bitshares docs](http://docs.pybitshares.com/en/latest/blockchain.html)

### chain_info

A high level overview of the Bitshares chain information.

#### [Run Command](https://btsapi.grcnode.co.uk/chain_info?&api_key=123abc)

Usage: `https://subdomain.domain.tld/chain_info?&api_key=API_KEY`

#### Example JSON output

```
{
 "id": "2.1.0",
 "head_block_number": 22901637,
 "head_block_id": "015d73858661412b31201155f6f77c772d044a95",
 "time": "2017-12-23T14:04:03",
 "current_witness": "1.6.16",
 "next_maintenance_time": "2017-12-23T15:00:00",
 "last_budget_time": "2017-12-23T14:00:00",
 "witness_budget": 112200000,
 "accounts_registered_this_interval": 10,
 "recently_missed_count": 0,
 "current_aslot": 23041161,
 "recent_slots_filled": "340282366920938463463374607431768211455",
 "dynamic_flags": 0,
 "last_irreversible_block_num": 22901622,
 "valid_key": true,
 "took": 0.12855
}
```

###  get_chain_properties

Get chain properties, return in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/get_chain_properties?api_key=123abc)

Usage: `https://subdomain.domain.tld/get_chain_properties?api_key=API_KEY`

#### Example JSON output

```
{
  "chain_properties": {
    "id": "2.11.0",
    "chain_id": "4018d7844c78f6a6c41c6a552b898022310fc5dec06da467ee7905a8dad512c8",
    "immutable_parameters": {
      "min_committee_member_count": 11,
      "min_witness_count": 11,
      "num_special_accounts": 100,
      "num_special_assets": 100
    }
  },
  "valid_key": true,
  "took": 0.01332
}
```

###  get_network

Return BTS network information in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/get_network?api_key=123abc)

Usage: `https://subdomain.domain.tld/get_network?api_key=API_KEY`

#### Example JSON output

```
{
  "get_network": {
    "chain_id": "4018d7844c78f6a6c41c6a552b898022310fc5dec06da467ee7905a8dad512c8",
    "core_symbol": "BTS",
    "prefix": "BTS"
  },
  "valid_key": true,
  "took": 0.12271
}
```

### get_info

This call returns the dynamic global properties in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/get_info?api_key=123abc)

Usage: `https://subdomain.domain.tld/get_info?api_key=API_KEY`

#### Example JSON output

```
{
  "chain_info": {
    "id": "2.1.0",
    "head_block_number": 22988592,
    "head_block_id": "015ec73077ef571e12193ae0ae512f7f55b971fb",
    "time": "2017-12-26T14:42:48",
    "current_witness": "1.6.75",
    "next_maintenance_time": "2017-12-26T15:00:00",
    "last_budget_time": "2017-12-26T14:00:00",
    "witness_budget": 34700000,
    "accounts_registered_this_interval": 118,
    "recently_missed_count": 0,
    "current_aslot": 23128120,
    "recent_slots_filled": "340282366920938463463374607431768211455",
    "dynamic_flags": 0,
    "last_irreversible_block_num": 22988579
  },
  "valid_key": true,
  "took": 0.13397
}
```

### get_config

Returns object 2.0.0 in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/get_config?api_key=123abc)

Usage: `https://subdomain.domain.tld/get_config?api_key=API_KEY`

#### [Example JSON output](./example_json/get_config.json)

### get_block_details

Retrieve the specified block's date/time details, return in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/get_block_details?api_key=123abc)

Usage: `https://subdomain.domain.tld/get_block_details?api_key=API_KEY`

#### Example JSON output

```
{
  "block_date": "2015-10-13T14:13:00",
  "block_timestamp": 1444745580,
  "block_number": 10,
  "valid_block_number": true,
  "valid_key": true,
  "took": 0.05997
}
```

### get_latest_block

Retrieve the details of the latest block, return in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/get_latest_block?api_key=123abc)

Usage: `https://subdomain.domain.tld/get_latest_block?api_key=API_KEY`

#### Example JSON Output

```
{
  "block_date": "2017-12-26T23:54:03",
  "block_timestamp": 1514332443,
  "block_number": 22999590,
  "valid_block_number": true,
  "valid_key": true,
  "took": 0.14057
}
```

### get_all_accounts

Retrieve all Bitshares account names. Takes a while!

#### [Run Command](https://btsapi.grcnode.co.uk/get_all_accounts?api_key=123abc)

Usage: `https://subdomain.domain.tld/get_all_accounts?api_key=API_KEY`

#### Example JSON output

```
{
  'accounts': [{acc1, acc2, ...}],
  'num_accounts': 500000,
  'valid_key': True,
  'took': 20.5
}
```

## Account information functions

More info: [python-bitshares docs](http://docs.pybitshares.com/en/latest/account.html)

### account_balances

Given a valid account name, output the user's balances in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/account_balances?account=example_usera&api_key=123abc)

Usage: `https://subdomain.domain.tld/account_balances?account=example_usera&api_key=API_KEY`

#### Example JSON output

Note: 'balances' has been concatenated to save space in this example.

```
{
  "balances": [{"BTS": 780.3515},
              {"ROSE": 999.0},
              {"FASTCASS": 8.0},
              {"DELETIP": 12760.0}],
 "account_has_balances": true,
 "valid_account": true,
 "valid_key": true,
 "took": 0.50434
}
```

### account_open_orders

Given a valid account name, output the user's open orders in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/account_open_orders?account=example_usera&api_key=123abc)

Usage: `https://subdomain.domain.tld/account_open_orders?account=example_usera&api_key=API_KEY`

#### Example JSON output

```
{
  "open_orders": [{"Sell: USD": "558.8355", "Buy: OPEN.PPY": "553.30247", "USD/OPEN.PPY": 1.0100000095788477},
                 {"Sell: BEYONDBIT": "500", "Buy: WHALESHARE": "12500", "BEYONDBIT/WHALESHARE": 0.04}],
  "account_has_open_orders": true,
  "valid_account": true,
  "valid_key": true,
  "took": 0.24319
}
```

### account_callpositions

Given a valid account name, output the user's call positions in JSON.

Note: Highly verbose! Example contains 26k lines of JSON!

#### [Run Command](https://btsapi.grcnode.co.uk/account_callpositions?account=example_usera&api_key=123abc)

Usage: `https://subdomain.domain.tld/account_callpositions?account=example_usera&api_key=API_KEY`

#### [Example JSON output](./example_json/account_callposition.json)

### account_history

Given a valid account name and transaction history limit (int), output the user's transaction history in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/account_history?account=example_user&tx_limit=10&api_key=123abc)

Usage: `https://subdomain.domain.tld/account_history?account=example_user&tx_limit=10&api_key=API_KEY`

#### [Example JSON output](./example_json/account_history.json)

### account_is_ltm

Given a valid account name, check if the user has LTM.

#### [Run Command](https://btsapi.grcnode.co.uk/account_is_ltm?account=example_user&api_key=123abc`

Usage: `https://subdomain.domain.tld/account_is_ltm?account=example_user&api_key=API_KEY`

#### Example JSON output

```
{
  "account_is_ltm": true,
  "account": "xeroc",
  "valid_account": true,
  "valid_key": true,
  "took": 0.14034
}
```

## DEX functions

More info: [python-bitshares docs](http://docs.pybitshares.com/en/latest/dex.html)

### list_fees

Retrieve the currently implemented fees in JSON format.

#### [Run Command](https://btsapi.grcnode.co.uk/list_fees?api_key=123abc)

Usage: `https://subdomain.domain.tld/list_fees?api_key=API_KEY`

#### [Example JSON output](./example_json/list_fees.json)

## Market information functions

More info: [python-bitshares docs](http://docs.pybitshares.com/en/latest/market.html)

### market_ticker

Given a valid market pair (e.g. USD:BTS), output the market pair's ticker information in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/market_ticker?market_pair=USD:BTS&api_key=123abc)

Usage: `https://subdomain.domain.tld/market_ticker?market_pair=USD:BTS&api_key=API_KEY`

#### [Example JSON output](./example_json/market_ticker.json)

### market_orderbook

Given a valid market pair (e.g. USD:BTS) and your desired orderbook size limit, output the market pair's orderbook (buy/sell order) information in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/market_orderbook?market_pair=USD:BTS&orderbook_limit=25&api_key=123abc)

Usage: `https://subdomain.domain.tld/market_orderbook?market_pair=USD:BTS&orderbook_limit=25&api_key=API_KEY`

#### [Example JSON output](./example_json/market_orderbook.json)

### market_24hr_vol

Given a valid market_pair (e.g. USD:BTS), output their 24hr market volume in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/market_24hr_vol?market_pair=USD:BTS&api_key=123abc)

Usage: `https://subdomain.domain.tld/market_24hr_vol?market_pair=USD:BTS&api_key=API_KEY`

#### [Example JSON output](./example_json/market_24hr_vol.json)

### market_trade_history

Given a valid market_pair (e.g. USD:BTS) & a TX limit, output the market's trade history in JSON.

#### [Run Command](https://btsapi.grcnode.co.uk/market_trade_history?market_pair=USD:BTS&tx_limit=10&api_key=123abc)

Usage: `https://subdomain.domain.tld/market_trade_history?market_pair=USD:BTS&tx_limit=10&api_key=API_KEY`

#### Example JSON output

```
{
  "market_trade_history": [
    {
      "date": "2017-12-24",
      "time": "15:54:36",
      "bought": "55.7395 USD",
      "sold": "107.13165 BTS",
      "rate ": "1.922005938 BTS/USD"
    },
    {
      "date": "2017-12-24",
      "time": "15:54:36",
      "bought": "100.0000 USD",
      "sold": "192.19944 BTS",
      "rate ": "1.921994400 BTS/USD"
    }
  ],
  "market": "USD:BTS",
  "valid_market": true,
  "valid_tx_limit": true,
  "valid_key": true,
  "took": 0.17249
}
```

## Witness functions

More info: [python-bitshares docs](http://docs.pybitshares.com/en/latest/witness.html)

### find_witness

Find details about a specific witness.

#### [Run Command](https://btsapi.grcnode.co.uk/find_witness?witness_name=blockchained&api_key=123abc)

Usage: `https://subdomain.domain.tld/find_witness?witness_name=blockchained&api_key=API_KEY`

#### [Example JSON output](./example_json/find_witness.json)

### list_of_witnesses

Retrieve a list of available witnesses.

#### [Run Command](https://btsapi.grcnode.co.uk/list_of_witnesses?api_key=123abc)

Usage: `https://subdomain.domain.tld/list_of_witnesses?api_key=API_KEY`

#### [Example JSON output](./example_json/list_of_witnesses.json)

---

# TODO

## Worker Proposal functions

More info: [python-bitshares docs](http://docs.pybitshares.com/en/latest/proposal.html)

May not be possible - tx proposals, not worker proposals?

### list_worker_proposals

Retrieve a list of worker proposals.

### find_worker_proposal

Find a specific worker proposal.
