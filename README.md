# ansible-sensu

# Setup

Install `cmacrae.sensu` from ansible-galaxy if needed.

```
$ ansible-galaxy install cmacrae.sensu
```

Added `ssh_config` file and edit it as you like.
You should make the file when you want to use bastin server.

# Edit config

Make `hosts` file and Edit it.

Example
```
[rabbitmq_servers]
mq.cmacrae.sensu.com

[redis_servers]
redis.cmacrae.sensu.com

[sensu_masters]
cmacrae.sensu.com

[zones]
client.domain.com

[nginx]
web.domain.com

[postfix]
mail.domain.com

[mysql]
db.domain.com

[supervisord]
supervisord.domain.com
```

@see `site.yml` , `zones.yml` and check `ssh login-user`

## Set some parameters

@see `group_vars/sensu_masters.yml`

You may need to set these environment variables.

- `export SENSU_API_HOST=""`
- `export SENSU_SLACK_WEBHOOK_URL=""`
- `export SENSU_SLACK_CHANNEL="#"`
- `export SENSU_HANDLER_MAIL_FROM=""`
- `export SENSU_HANDLER_MAIL_TO=""`
- `export SENSU_INFRA_EMAIL_TO=""`
- `export SENSU_INFRA_EMAIL_TO2=""`
- `export SENSU_RABBITMQ_HOST=""`
- `export SENSU_REDIS_HOST=""`

## Create custom client hosts

When you want to use different parameters by hosts, create custom client.

`data/static/sensu/custom_client/host-name.client.json.j2`

Example

```
{
    "client": {
        "name": "{{ sensu_client_name }}",
        "address": "{{ ansible_default_ipv4['address'] }}",
        "subscriptions": {{ sensu_client_subscriptions | to_nice_json }},
        "mysql": {
            "hostname": "localhost",
            "username": "root",
            "password": "password",
            "database": "dbname",
            "socket": "/var/lib/mysql/mysql.sock"
        }
    }
}
```

And you also have to write down the file path.

`host_vars/host-name.com.yml`

```
sensu_client_config: data/static/sensu/custom_client/host-name.client.json.j2
```

How to use custom client `key-value` definitions.

```
:::mysql.hostname:::
:::mysql.username:::

...

```


# Install sensu_server, graphite and grafana

## set grafana from domain

Modify the file if you want to change from domain.

```
roles/grafana/tasks/main.yml

    45      - { key: "host", value: "localhost:25" }
    46      - { key: "from_address", value: "admin@grafana.localhost" }
    47      - { key: "from_name", value: "grafana" }

```

Run

```
ansible-playbook -v -i hosts site.yml --private-key="~/.ssh/priv-key.pem"
```

- open uchiwa port 3000
- open api port 4567
- open mq port 5671

# Install sensu_client
```
ansible-playbook -v -i hosts zones.yml --private-key="~/.ssh/priv-key.pem"
```

# Browse

Uchiwa Dashboard

`http://[ip address]:3000/`

Grafana Dashboard

`https://[ip address]/`

## Play Grafana

- Access to Grafana
    - Launch a web browser
    - Access to url: ```https://xxx.xxx.xxx.xxx/``` (Your public IP address)
    - Accept untrusted certification
    - Log in with username=`admin`, password=`admin`
- Click the top-left menu icon
- Data Sources -> Add new
    - Name: `Graphite`
    - Default: On
    - Type: Graphite
    - Url: `http://localhost:9001`
    - Access: proxy
    - Basic Auth: Enable: Off
- Dashboards -> Home -> New
- Row menu (gree button) -> Add Panel -> Graph -> Click on the title -> edit
- Select the metrics (e.g. `carbon.agents.ip-xxx-xxx-xxx-xxx-a.memUsage`)

# Plugins

@see `group_vars/sensu_masters.yml`
@see `group_vars/zones.yml`
@see `data/static/sensu/definitions/*`


# Bug Fix

Ansible-sensu uses `cmacrae.sensu` from ansible-galaxy.
If you want to run this script under the AmazonLinux, you should fix some points.

```
diff --git a/tasks/Amazon/main.yml b/tasks/Amazon/main.yml
index 7e3857b..42db7ab 100644
--- a/tasks/Amazon/main.yml
+++ b/tasks/Amazon/main.yml
@@ -10,7 +10,7 @@
       content: |
         [sensu]
         name=sensu
-        baseurl=https://sensu.global.ssl.fastly.net/yum/$releasevar/$basearch/
+        baseurl=https://sensu.global.ssl.fastly.net/yum/6/$basearch/
         gpgcheck=0
         enabled=1
       owner: root
diff --git a/tasks/ssl_generate.yml b/tasks/ssl_generate.yml
index 80e8892..57d5f7c 100644
--- a/tasks/ssl_generate.yml
+++ b/tasks/ssl_generate.yml
@@ -23,7 +23,7 @@
         copy: no

     - name: Generate SSL certs
-      command: "_ {{ sensu_config_path }}/ssl_generation/sensu_ssl_tool/ssl_certs.sh generate"
+      command: "{{ sensu_config_path }}/ssl_generation/sensu_ssl_tool/ssl_certs.sh generate"
       args:
         chdir: "{{ sensu_config_path }}/ssl_generation/sensu_ssl_tool"
         creates: "{{ sensu_config_path }}/ssl_generation/sensu_ssl_tool/server"
```

# Trouble Shooting

Plugins Not Found Error

```
{"timestamp":"2019-02-14T10:08:17.474295+0900","level":"info","message":"publishing check result","payload":{"client":"prod-bbm","check":{"command":"check-disk-usage.rb -w 80 -c 90","refresh":1800,"handlers":["mailer","slack"],"contact":"infra","name":"check-disk-usage","issued":1550106497,"executed":1550106497,"duration":0.002,"output":"sh: check-disk-usage.rb: コマンドが見つかりません\n","status":127}}}
```

Check plugins PATH is corrrect and then restart sensu-client

```
# Check PATH
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/opt/aws/bin:/root/bin:/opt/sensu/embedded/bin

# Restart
$ /opt/sensu/embedded/bin/ruby /opt/sensu/bin/sensu-client -b -c /etc/sensu/config.json -d /etc/sensu/conf.d -e /etc/sensu/extensions -p /var/run/sensu/sensu-client.pid -l /var/log/sensu/sensu-client.log -L info
```
