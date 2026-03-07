```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 10808,
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "settings": {
        "auth": "noauth",
        "udp": true
      }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy-reality",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "146.190.42.44",
            "port": 443,
            "users": [
              {
                "id": "3901bb11-c1f8-4fae-ab3b-36ff03c20c1b",
                "email": "t@t.tt",
                "security": "auto",
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "www.icloud.com",
          "fingerprint": "chrome",
          "show": false,
          "publicKey": "fhHbFC70HLCTU0bbys0S-CTOZ1J6zoEDExZrChaTXUc",
          "shortId": "ee",
          "spiderX": "/",
          "mldsa65Verify": ""
        }
      },
      "mux": {
        "enabled": false,
        "concurrency": -1
      }
    },
    {
      "tag": "proxy-iplc",
      "protocol": "shadowsocks",
      "settings": {
        "servers": [
          {
            "address": "ncgdyd.gdsxrk.com",
            "method": "chacha20-ietf-poly1305",
            "ota": false,
            "password": "87474136-214b-4146-ad13-f64da40762f4",
            "port": 21320,
            "level": 1
          }
        ]
      },
      "streamSettings": {
        "network": "tcp"
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom"
    }
  ],
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private", "geoip:cn"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": ["geosite:cn"],
        "outboundTag": "direct"
      }
    ]
  }
}
```

> [!tip] 改造方法
> 将出站规则(`outbounds`)里面的
> `tag`，`protocol`，`settings`，`streamSetting`进行替换即可