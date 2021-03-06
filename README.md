# Chef-Rails

Kitchen to setup an Ubuntu Server ready to roll with Nginx, PostgreSQL and Rails.

## Requirements

* Ubuntu 16.04+

## Usage

To cook with this kitchen you must follow four easy steps.

### 0. Create server deploy user (Optional)

We create our deploy user in deploy server adding our SSH keys:
```bash
sudo adduser deploy --disabled-password
# Add your SSH keys to deploy authorized_keys
sudo mkdir /home/deploy/.ssh/
sudo vim /home/deploy/.ssh/authorized_keys
```

### 1. Prepare your local working copy

```bash
git clone <url>
cd chef-rails
bundle install
bundle exec librarian-chef install
```

### 2. Prepare the servers you want to configure

We need to copy chef-solo to any server we’re going to setup. For each server, execute

```bash
bundle exec knife solo prepare [user]@[host] -p [port]
```

where

* *user* is a user in the server with sudo and an authorized key.
* *host* is the ip or host of the server.
* *port* is the port in which ssh is listening on the server. Defaul port: 22.

### 3. Define the specs for each server

If you take a look at the *nodes* folder, you’re going to see files called [host].json, corresponding to the hosts or IPs of the servers we previously prepared, plus a file called *localhost.json.example* which is, as its name suggests, and example. Just enter your configuration in `< >` fields.

The specs for each server needs to be defined in those files, and the structure is exactly the same as in the example.

For the very same reason, we’re going to explain the example for you to ride on your own pony later on.

```json
{
  // This is the list of the recipes that are going to be cooked.
  "run_list": [
    "recipe[apt]",
    "recipe[sudo]",
    "recipe[build-essential]",
    "recipe[ohai]",
    "recipe[runit]",
    "recipe[git]",
    "recipe[postgresql]",
    "recipe[postgresql::contrib]",
    "recipe[postgresql::server]",
    "recipe[nginx]",
    "recipe[nginx::apps]",
    "recipe[monit]",
    "recipe[monit::ssh]",
    "recipe[monit::nginx]",
    "recipe[monit::postgresql]",
    "recipe[rvm::user]",
    "recipe[chef-rails]"
  ],

  "automatic": {
    "ipaddress": "<host_ip>"
  },

  // You must define who’s going to be the user(s) you’re going to use for deploy.
  "authorization": {
    "sudo": {
      "groups"      : ["<group_name>"],
      "users"       : ["<user_name>"],
      "passwordless": true
    }
  },

  // You must define the password for postgres user.
  // Leave config block commented untill next cook.
  "postgresql": {
    "contrib": {
      "extensions": ["pg_stat_statements"]
    },
    // "config": {
    //   "shared_buffers": "125MB",
    //   "shared_preload_libraries": "pg_stat_statements"
    // },
    "password"      : {
      "postgres": "<psql_passwd>"
    }
  },

  // You must specify the ubuntu distribution by it’s name to configure the proper version
  // of nginx, otherwise it’s going to fail.
  "nginx": {
    "user"          : "<user_name>",
    "distribution"  : "<linux_distribution>",
    "components"    : ["main"],
    "worker_rlimit_nofile": 30000,

    // Here you should define all the apps you want nginx to serve for you in the server.
    "apps": {
      // Example for an application served by Unicorn server
      "example.com": {
        "listen"     : [80],
        "server_name": "example.com www.example.com",
        "public_path": "/home/deploy/production.example.com/current/public",
        "upstreams"  : [
          {
            "name"    : "example.com",
            "servers" : [
              "unix:/home/deploy/production.example.com/shared/pids/example.com.sock max_fails=3 fail_timeout=1s"
            ]
          }
        ],
        "locations": [
          {
            "path": "/",
            "directives": [
              "proxy_set_header X-Forwarded-Proto $scheme;",
              "proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;",
              "proxy_set_header X-Real-IP $remote_addr;",
              "proxy_set_header Host $host;",
              "proxy_redirect off;",
              "proxy_http_version 1.1;",
              "proxy_set_header Connection '';",
              "proxy_pass http://example.com;"
            ]
          },
          {
            "path": "~ ^/(assets|fonts|system)/|favicon.ico|robots.txt",
            "directives": [
              "gzip_static on;",
              "expires max;",
              "add_header Cache-Control public;"
            ]
          }
        ]
      },

      // Example for an application served by Thin server
      "example2.com": {
        "listen"     : [80],
        "server_name": "example2.com www.example2.com",
        "public_path": "/home/deploy/production.example2.com/current/public",
        "upstreams"  : [
          {
            "name"    : "example2.com",
            "servers" : [
              "0.0.0.0:3000 max_fails=3 fail_timeout=1s",
              "0.0.0.0:3001 max_fails=3 fail_timeout=1s"
            ]
          }
        ],
        "locations": [
          {
            "path": "/",
            "directives": [
              "proxy_set_header X-Forwarded-Proto $scheme;",
              "proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;",
              "proxy_set_header X-Real-IP $remote_addr;",
              "proxy_set_header Host $host;",
              "proxy_redirect off;",
              "proxy_http_version 1.1;",
              "proxy_set_header Connection '';",
              "proxy_pass http://example2.com;"
            ]
          },
          {
            "path": "~ ^/(assets|fonts|system)/|favicon.ico|robots.txt",
            "directives": [
              "gzip_static on;",
              "expires max;",
              "add_header Cache-Control public;"
            ]
          }
        ]
      }
    }
  },

  // The ruby version you’re going to use and rvm user.
  "rvm" : {
    "user_installs": [
      {
        "user"         : "<user_name>",
        "default_ruby" : "ruby-2.3.3"
      }
    ]
  },

  // Monit configuration. Sets email, check period and delay since monit service start
  "monit" : {
    "poll_period"      : "60",
    "poll_start_delay" : "120"
  },

  // Finally, declare all the system packages required by the services and gems you’re using in your apps.
  // To give you an example: If you’re using paperclip, the native extensions compilation will fail unless you have installed imagemagick declared below.
  "chef-rails": {
    "packages": ["imagemagick", "nodejs-dev"]
  }
}
```

### 4. Happy cooking

We’re now ready to cook. For each server you want to setup, execute

```bash
bundle exec knife solo cook [user]@[host] -p [port]
```

following the same criteria we defined in step **2**.

Remember to clean your kitchen after cook

```bash
bundle exec knife solo clean [user]@[host] -p [port]
```

### 5. Create PostgreSQL user for deploy

```bash
sudo -u postgres psql
CREATE USER deploy SUPERUSER ENCRYPTED PASSWORD '<deploy_user_password>';
\q
```

At last add
```ruby
name 'cookbook_name'
```
to metadata.rb files in cookbooks folder

Have a nice cooking.

### 6. For Vagrant

Download vagrant from [here](https://www.vagrantup.com/downloads.html) and install

Make sure you have these vagrant plugins:
- vagrant-berkshelf
- vagrant-omnibus
- vagrant-share
- vagrant-vbguest

run `vagrant up --provision` to provision your VM with chef

and then when finished `vagrant ssh` into your VM

you are ready to try your Vagrant.
