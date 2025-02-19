# docker-syslog

Docker image to aggregate syslog with fluentd

## 参考資料

[Dockerとrsyslogとfluentdでログ収集をする \#Ubuntu \- Qiita](https://qiita.com/mmmmmmmmmmmmm/items/3a17b7e6a5bbfa9272e9)

2020年の記事なのでそのままでは動作しない。


### rsyslogd を動かしてみる


```txt
# Send log messages to Fluentd
*.* @fluentd:5140
```

の代わりに `/var/log/syslogd.log` に書き込むように

```txt
# Save the logs to a file
*.* /var/log/syslog.log
```

に書き換えて、 VOLUME の記載もいくつか書き換えて `docker build -t rsyslog-img` でビルドし、以下で起動する。

```sh
docker run -d -it --rm -p 514:514 -p 514:514/udp -v ./files/etc/rsyslog.conf:/etc/rsyslog.conf.d rsyslog-img
```

`syslogd.log` にログが書き込まれていることを確認。


### fluentd を動かしてみる


#### WSL2 の Ubuntu 20.04 に手動でインストールしてみる

deb package でのインストールは以下のライブラリの依存関係のエラーが解決できなかった。

```sh
The following packages have unmet dependencies:
 fluent-package : Depends: libc6 (>= 2.35) but 2.31-0ubuntu9.16 is to be installed
                  Depends: libffi8 (>= 3.4) but it is not installable
                  Depends: libgmp10 (>= 2:6.2.1+dfsg) but 2:6.2.0+dfsg-4ubuntu0.1 is to be installed
                  Depends: libssl3 (>= 3.0.0~~alpha1) but it is not installable
E: Unable to correct problems, you have held broken packages.
```

gem でインストールする

[Install by Ruby Gem \| Fluentd](https://docs.fluentd.org/installation/install-by-gem)

`fluentd --setup ./fluent` して作成される `fluent.conf` の有効部分は以下

```txt
<source>
  @type forward
  @id forward_input
</source>
<source>
  @type http
  @id http_input
  port 8888
</source>
<source>
  @type monitor_agent
  @id monitor_agent_input
  port 24220
</source>
<source>
  @type debug_agent
  @id debug_agent_input
  bind "127.0.0.1"
  port 24230
</source>
<match debug.**>
  @type stdout
  @id stdout_output
</match>
```

全文は以下

```txt
# In v1 configuration, type and id are @ prefix parameters.
# @type and @id are recommended. type and id are still available for backward co
mpatibility

## built-in TCP input
## $ echo <json> | fluent-cat <tag>
<source>
  @type forward
  @id forward_input
</source>

## built-in UNIX socket input
#<source>
#  @type unix
#</source>

# HTTP input
# http://localhost:8888/<tag>?json=<json>
<source>
  @type http
  @id http_input

  port 8888
</source>

## File input
## read apache logs with tag=apache.access
#<source>
#  @type tail
#  format apache
#  path /var/log/httpd-access.log
#  tag apache.access
#</source>

## Mutating event filter
## Add hostname and tag fields to apache.access tag events
#<filter apache.access>
#  @type record_transformer
#  <record>
#    hostname ${hostname}
#    tag ${tag}
#  </record>
#</filter>

## Selecting event filter
## Remove unnecessary events from apache prefixed tag events
#<filter apache.**>
#  @type grep
#  include1 method GET # pass only GET in 'method' field
#  exclude1 message debug # remove debug event
#</filter>

# Listen HTTP for monitoring
# http://localhost:24220/api/plugins
# http://localhost:24220/api/plugins?type=TYPE
# http://localhost:24220/api/plugins?tag=MYTAG
<source>
  @type monitor_agent
  @id monitor_agent_input

  port 24220
</source>

# Listen DRb for debug
<source>
  @type debug_agent
  @id debug_agent_input

  bind 127.0.0.1
  port 24230
</source>

## match tag=apache.access and write to file
#<match apache.access>
#  @type file
#  path /var/log/fluent/access
#</match>

## match tag=debug.** and dump to console
<match debug.**>
  @type stdout
  @id stdout_output
</match>

## match tag=system.** and forward to another fluent server
#<match system.**>
#  @type forward
#  @id forward_output
#
#  <server>
#    host 192.168.0.11
#  </server>
#  <secondary>
#    <server>
#      host 192.168.0.12
#    </server>
#  </secondary>
#</match>

## match tag=myapp.** and forward and write to file
#<match myapp.**>
#  @type copy
#  <store>
#    @type forward
#    buffer_type file
#    buffer_path /var/log/fluent/myapp-forward
#    retry_limit 50
#    flush_interval 10s
#    <server>
#      host 192.168.0.13
#    </server>
#  </store>
#  <store>
#    @type file
#    path /var/log/fluent/myapp
#  </store>
#</match>

## match fluent's internal events
#<match fluent.**>
#  @type null
#</match>

## match not matched logs and write to file
#<match **>
#  @type file
#  path /var/log/fluent/else
#  compress gz
#</match>

## Label: For handling complex event routing
#<label @STAGING>
#  <match system.**>
#    @type forward
#    @id staging_forward_output
#    <server>
#      host 192.168.0.101
#    </server>
#  </match>
#</label>
```

```sh
fluentd -c ./fluent.conf -vv &
```

で実行する。

8888 で HTTP 待ち受けをしているので、以下を送ってみる。

```sh
#POST
curl -X POST -d 'json={"json":"message"}' http://localhost:8888/debug.post
```

```txt
2025-02-19 11:54:29.515811955 +0900 debug.post: {"json":"message"}
```

が stdout に出力された。




#### Docker で動かしてみる

```txt
RUN gem install fluent-plugin-rewrite-tag-filter
```

でエラーが出る。

```txt
fluent/fluentd:v1.17-debian-1 で ERROR: Failed to build gem native extension が出る

fluent/fluentd:v1.17-debian-1 イメージで gem install コマンドを実行するときに ERROR: Failed to build gem native extension エラーが発生する場合、ビルドに必要な依存関係が不足している可能性があります。これを解決するために、必要なビルドツールをインストールする必要があります。

以下のように Dockerfile を更新して、ビルドツールをインストールしてください。
```

とのこと。

gem で fluent-plugin-rewrite-tag-filter をインストールするにはビルドなどをしないといけないので、ツールが必要になると思われる。

build-essential の代わりに ruby-dev ではだめ。 g++ make をインストールしたら 50MB 軽くなった。

```sh
docker run --rm -it -v ./files/etc/:/fluentd/etc fluentd-img:latest
```

で実行したら動作した。

syslog の収集が出来ていないようなので、以下を参考に fluentd 自身で http のログを収集する。

- [Fluentdはデータ収集ソフトウェア最強のツールである](https://zenn.dev/ryoatsuta/articles/a0dea1dc377000)
- [rewrite\_tag\_filter \| Fluentd](https://docs.fluentd.org/0.12/output/rewrite_tag_filter)

```xml
<match mytag.**>
  @type rewrite_tag_filter
  <rule>
    key json
    pattern ^\[(\w+)\].+$
    tag $1.${tag}
  </rule>
</match>
```

で "json" オブジェクト内の `[XXX]` によって tag に XXX を追加する設定になるはず。


```sh
docker run --rm -it -p 9880:9880 -v ./files/etc/:/fluentd/etc fluentd-img:latest
```

localhost に対して以下を送ってみる

```sh
curl -X POST -d 'json={"json":"[info]: message"}' http://127.0.0.1:9880/mytag.post
```

```txt
2025-02-19 04:28:09.124114374 +0000 info.mytag.echo: {"json":"[info]: echo message"}
2025-02-19 04:28:56.140307745 +0000 info.mytag.post: {"json":"[info]: message"}
```

が帰ってきたので、動作している。

rewrite_tag_filter が動作していることが分かったので、syslog の設定をしていく。

取得できるログのどの部分でフィルタを掛けるか？だが、

```txt
2025-02-19 05:04:00.000000000 +0000 rsyslog.local0.info: {"host":"172.19.0.1","ident":"","message":"03:59\" fw=219.106.251.58 pri=6 c=262144 m=98 msg=\"Connection Opened\" n=454655 src=192.168.2.138:54380:X0 dst=140.82.112.22:443:X1 proto=tcp/https sent=52 dpi=0 fw_action=\"NA\""}
```

fw だと message の途中なので `pattern /^.+fw=219\.106\.251\.58.+$/` で指定すれば良い。

