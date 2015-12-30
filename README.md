# Magento2 (Varnish + PHP7 + Redis) cluster ready docker-compose infrastructure

## Infrastructure overview
* Container 1: MariaDB
* Container 2: Redis (we'll use it for Magento's cache)
* Container 3: Apache 2.4 + PHP 7 (modphp)
* Container 4: Cron
* Container 5: Varnish 4

###Why a separate cron container?
First of all containers should be (as far as possible) single process, but the most important thing is that (if someday we'll be able to deploy this infrastructure in production) we may need a cluster of apache+php containers but a single cron container running.

Plus, with this separation, in the context of a docker swarm, you may be able in the future to separare resources allocated to the cron container from the rest of the infrastructure.

## Downloading Magento2
```
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition magento2
```

## Do you want sample data?
Execute (from the host machine):
```
cd magento2
php bin/magento sampledata:deploy
composer update
```
You can lunch the same commmands from within the container, it's actually the same thing

## Starting all docker containers
```
docker-compose up -d
```
The fist time you run this command it's gonna take some time to download all the required images from docker hub.

## Install Magento2

open your browser to the address:
```
http://magento2.docker/
```
and use the wizard to install Magento2.

## Deploy static files
```
docker exec -it dockermagento2_apache_1 bash
php bin/magento dev:source-theme:deploy
php bin/magento setup:static-content:deploy
```

## Enable Redis for Magento's cache
open magento2/app/etc/env.php and add these lines:
```php
'cache' => array(
  'frontend' => array(
    'default' => array(
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => array(
        'server' => 'redis_cache',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '0',
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'read_timeout' => '10', // Set read timeout duration
        'automatic_cleaning_factor' => '0', // Disabled by default
        'compress_data' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_tags' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_threshold' => '20480', // Strings below this size will not be compressed
        'compression_lib' => 'gzip', // Supports gzip, lzf and snappy,
        'use_lua' => '0' // Lua scripts should be used for some operations
      )
    ),
    'page_cache' => array(
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => array(
        'server' => 'redis_cache',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '1', // Separate database 1 to keep FPC separately
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'lifetimelimit' => '57600', // 16 hours of lifetime for cache record
        'compress_data' => '0' // DISABLE compression for EE FPC since it already uses compression
      )
    )
  )
),
```
and delete all Magento's cache with
```
rm -rf magento2/var/cache/*
```
from now on the var/cache directory should stay empty cause all the caches should be stored in Redis.

## Enable Varnish
Varnish Full Page Cache should already be enabled out of the box (we startup Varnish with the default VCL file generated by Magento2) but you could anyway go to "stores -> configuration -> advanced -> system -> full page cache" and:
* select Varnish in the "caching application" combobox
* type "apache" in both "access list" and "backend host" fields
* type 80 in the "backend port" field
* save

# Cross platform and performance

At the moment the apache/php container has been created in order to work with the default Docker Machine, to be able to run on both windows and mac (check the usermod 1000 in the Dockerfile).

Performarce are not ok on a good hardware but everybody knows about vbox shared folders slowness...

On mac is surely better to use Dinghy as a replacement for the default Docker Machine but the problem is that I would have need to generate the apache/php image with "usermod 501" making it not compatible with the default Docker Machine and thus windows devs.

What's the better choice? Please share your ideas with me.

# Scaling apache containers
If you need more horsepower you can
```
docker-compose scale apache=X
```
where X is the number of apache containers you want to start.

The cron container will check how many apache containers we have and update Varnish's VCL.
Unfortunately at the moment there's no service autodiscovery, you've to start your infrastructure already with multiple apache containers if you need them. I hope to be able to add real autodiscovery soon.

# TODO
* Support for autodiscovery new/dead apache containers
* Add a SSL terminator image
* DB clustering?
