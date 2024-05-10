## 简单使用

```
 ./hqCenter -h
  -initCodesFile string
    	启动即订阅的code (default "./data/initCodes.json")
  -listen string
    	http监听地址 (default ":31800")
  -saveHqFile string
    	行情写入文件,自动加日期后缀。为空则不写入文件。 (default "./data/hq")
  -token string
    	jvQuant平台的访问token
```


## 控制API
动态增加或删除订阅可以通过 自定义指令API 实现。

```
接收实时行情 
    http://127.0.0.1:31800/hq
接收回放行情 
    http://127.0.0.1:31800/hq?file=hq.2024-04-10.txt
发送自定义指令 
    http://127.0.0.1:31800/cmd?cmd=add=lv2_600519
接收实时日志 
    http://127.0.0.1:31800/log
控制退出 
    http://127.0.0.1:31800/ctl?op=exit
```

/hq接口file参数为空则为实时行情模式，指定file则为回放模式

## 起因

之前写的自动交易策略,把行情/策略/交易都写在了一个包里，牵一发动全身，调试很头疼。

基于软件工程解耦的思想，决定拆分模块。

行情模块专注接收行情，暂时完善了订阅/推送/存储/回放/控制这几个功能。

用Http SSE推送模式，实时性相差无几，本地起个服务，支持多个订阅者，Python/PHP/JS接入也方便，不用再受tcp连接控制的苦。

Golang有跨平台直接生成可执行文件的优点，支持Mac/Linux/Windows。

可执行文件放在hqCenter/bin下,后缀区分不同平台。

没有Go环境的可以直接下载bin里的可执行文件，都是最新编译再推上来的。

发送自定义指令cmd 细节请参看jvQuant官方的[wiki](https://jvquant.com/wiki.html#--10)

这个行情模块放出来给有需要的人,有bug或建议可以提Issue。没有账号的可以通过[jvQuant邀请码链接](https://www.jvquant.com/register.html?yqm=VQ9993) 注册, 感谢支持。

执行示例:
> cd hqCenter/
>
> ./bin/hqCenter.exe -listen=:31800 -saveHqFile=./data/hq -initCodesFile=./data/initCodes.json -token=token

initCodesFile 格式为json Map,数组内分别为启动立即订阅的code。

lv1代表level1行情，lv2代表level2行情。 格式示例:

```json
{
  "lv1": [
    "600519"
  ],
  "lv2": [
    "000700",
    "123089"
  ]
}
```

##SSE接收示例:
- Shell
> curl -N 'http://127.0.0.1:31800/hq'

- Python
```python
import requests

url = 'http://127.0.0.1:31800/hq'
r = requests.get(url=url, stream=True)

for line in r.iter_lines(decode_unicode=True):
    print(line)
```

- PHP
```PHP
<?php
$url = 'http://127.0.0.1:31800/hq';
$ch = curl_init($url);
curl_setopt($ch, CURLOPT_WRITEFUNCTION, function($ch, $data) {
    print_r($data);
    return strlen($data);
});
curl_exec($ch);
curl_close($ch);
```

- Golang
```go
func SseGet(url string, ch chan []byte) (err error) {
	var rb []byte
	var resp *http.Response
	resp, err = http.Get(url)
	if err != nil {
		return
	}
	defer resp.Body.Close()
	br := bufio.NewReader(resp.Body)
	for {
		rb, err = br.ReadBytes('\n')
		if err != nil {
			return
		}
		ch <- rb
	}
	return
}
```
