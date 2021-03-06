# Storekeeper

__IMPORTANT__: This is mainly an experiment, and you must think twice before using this in production!

Storekeeper is a Steam bot powered by [node-steam](https://github.com/seishun/node-steam) and its plugins. While Storekeeper writen in JavaScript, you don't need to write any JS code to use it, since you can control the Storekeeper bot via [JSON-RPC](http://www.jsonrpc.org/) protocol using any implementation of JSON-RPC 2.0 you like on any programming language you like (PHP, for example).

_In fact, if you can code in JavaScript, .NET or Java, I strongly advise you to NOT use Storekeeper, since you can achieve better performance and stability using [SteamKit2](https://github.com/SteamRE/SteamKit) and its ports directly._

# Installation

First, you need [io.js](https://iojs.org/) or [Node.js](http://nodejs.org/) installed. Then just do:

```
npm install -g steam-storekeeper
```

# Getting started

You are ready to start your first Storekeeper bot!
Run it in any directory where you have write permissions:

```
storekeeper <config_name>
```

Where `<config_name>` is a name of a config file for Storekeeper. It will be created as `<config_name>.json` file on its first run.

For example, I like to use a Steam account username as config filename:

```
storekeeper misaki58
```

This will create `misaki58.json` file in the directory where you run Storekeeper.

Next, Storekeeper will guide you through configuration process and ask you several questions.

# What's next?

Code!

Check out the [list of JSON-RPC implementations](https://en.wikipedia.org/wiki/JSON-RPC#Implementations).

Storekeeper allows you to call almost all methods of [node-steam](https://github.com/seishun/node-steam), [node-steam-trade](https://github.com/seishun/node-steam-trade), and [node-steam-tradeoffers](https://github.com/Alex7Kom/node-steam-tradeoffers). Also, Storekeeper for each of said libraries exposes `getProperty` method which gives you access to documented properties.

Storekeeper acts as JSON-RPC server when your want to execute any method.

For example, you want to call `setPersonaName` method of `node-steam` you make a request:

```json
{
  "jsonrpc": "2.0",
  "method": "steam.setPersonaName",
  "params": ["Misaki"],
  "id": 0
}
```

`params` should be an array of arguments described in the documentation. If a method accepts a callback or returns something, expect results returned in a response.

`node-steam` and `node-steam-trade` can emit events for which Storekeeper listens. In that case Storekeeper acts as JSON-RPC client and makes requests to your server that contain information about an occured event. For example:

```json
{
  "jsonrpc": "2.0",
  "method": "steam.tradeOffers",
  "params": {
    "steamID": "76561198090583785",
    "arguments": [1]
  },
  "id": 0
}
```

This says that `tradeOffers` event of `node-steam` emitted, and `arguments` array contains a number of pending trade offers. Storekeeper does not care about response on events.

Methods are prefixed with `steam` for `node-steam`, `trade` for `node-steam-trade`, and `tradeOffers` for `node-steam-tradeoffers`.

# Examples

Examples here use my [Callchedan](https://github.com/Alex7Kom/callchedan) JSON-RPC library for PHP.

`pingpong.php` answers `pong` on your `ping` messages in Steam Friend Chat:

 ```php
require '../callchedan/lib/server.php';
require '../callchedan/lib/client.php';

$client = new Callchedan\Client('http://127.0.0.1:5080/'); // storekeeper url

$methods = array(

  // answer 'pong' on message 'ping'
  'steam.friendMsg' => function ($params) use ($client) {
    if ($params['arguments'][1] == 'ping') {
      $client->call('steam.sendMessage', array($params['arguments'][0], 'pong'));
    }
  }

);

$server = new Callchedan\Server($methods);
echo $server->handle();
```

`storehouse.php` auto-accepts your trade offers:

```php
require '../callchedan/lib/server.php';
require '../callchedan/lib/client.php';

$client = new Callchedan\Client('http://127.0.0.1:5080/'); // storekeeper url
$admin = '76561197981406440'; // put your steamid here so the bot can accept your offers

$methods = array(

  // auto-accept trade offers from admin
  'steam.tradeOffers' => function ($params) use ($client, $admin) {
    if ($params['arguments'][0] == 0) {
      return;
    }

    $offers = $client->call('tradeOffers.getOffers', array(
      array(
        'get_received_offers' => 1,
        'active_only' => 1,
        'time_historical_cutoff' => time()
      )
    ));

    if (!isset($offers[1]['response']['trade_offers_received'])) {
      return;
    }

    $offers_received = $offers[1]['response']['trade_offers_received'];

    if (count($offers_received) == 0) {
      return;
    }

    foreach ($offers_received as $offer) {
      if ($offer['trade_offer_state'] == 2){
        if($offer['steamid_other'] == $admin) {
          $client->call('tradeOffers.acceptOffer', array(
            array(
              'tradeOfferId' => $offer['tradeofferid']
            )
          ));
        } else {
          $client->call('tradeOffers.declineOffer', array(
            array(
              'tradeOfferId' => $offer['tradeofferid']
            )
          ));
        }
      }
    }
  }

);

$server = new Callchedan\Server($methods);
echo $server->handle();
```

# Advanced configuration

Storekeeper config is a normal JSON file.

```json
{
  "steamUsername": "misaki58",
  "steamPassword": "password",
  "steamGuard": [
    115,
    ...
    232
  ],
  "server": {
    "port": 5080,
    "host": "127.0.0.1",
    "path": "/",
    "strict": true
  },
  "client": {
    "port": 80,
    "host": "example.com",
    "path": "/path/to/your/rpc/server.php",
    "strict": true
  }
}
```

What you absolutely MUST understand, that config file contains all credentials of a Steam account, including authenticated Steam Guard property, and you absolutely must not handle it to anyone nor post it anywhere, including an issue tracker of this project.

## Blacklisting/whitelisting of methods and events

You can whitelist methods so only listed methods will be allowed to be called. For example:

```json
  "methodsWhitelist": ["steam.sendMessage", "steam.joinChat"]
```

Storekeeper will be allowed to handle only `steam.sendMessage` and `steam.joinChat` calls.

Blacklist allows you to specify methods you don't want to be called. For example:

```json
  "methodsBlacklist": ["steam.sendMessage", "steam.joinChat"]
```

Storekeeper will not be allowed to handle `steam.sendMessage` and `steam.joinChat` calls.

Note that blacklist is completely ignored if whitelist exists.

Listing events is very similar:

```json
  "eventsWhitelist": ["steam.friendMsg", "steam.chatMsg"]
```

Storekeeper will listen and notify you about `steam.friendMsg` and `steam.chatMsg` only.

```json
  "eventsBlacklist": ["steam.friendMsg", "steam.chatMsg"]
```

Storekeeper will not listen for these events.

Again, blacklist will be ignored if whitelist is present.

## Security

If your Storekeeper listens for external connections, you must take additional security measures.

You shouldn't rely on [security through obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity) to secure your Storekeeper. Simply keeping JSON-RPC entry points in secrecy is probably not a good idea.

TODO: Info about SSL.

### Basic HTTP Auth

Storekeeper supports [basic access HTTP authentication](https://en.wikipedia.org/wiki/Basic_access_authentication) via [node-json-rpc](https://github.com/NemoPersona/node-json-rpc) module.

To configure Storekeeper to require authentication, include `auth` block as shown below:

```json
  "server": {
    "port": 5080,
    "host": "127.0.0.1",
    "path": "/",
    "strict": true",
    "auth": {
      "users": [
        {
          "login": "user",
          "hash": "password"
        }
      ]
    }
  }
```

Then you'll need to configure your JSON-RPC client according its docs to enable basic auth.

If you have basic HTTP auth enabled on your JSON-RPC server, you can configure Storekeeper like so:

```json
  "client": {
    "port": 80,
    "host": "example.com",
    "path": "/path/to/your/rpc/server.php",
    "strict": true,
    "login": "user",
    "hash": "password"
  }
```

# License

The MIT License (MIT)

Copyright (c) 2014-2015 Alexey Komarov <alex7kom@gmail.com>

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.