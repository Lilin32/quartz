---
title: sqlmap 源码解读：controller.py
tags:
  - 渗透测试
  - web安全
  - python
  - SQL注入
  - sqlmap
creation date: 2024-05-05
done:
---
# 1. start 函数  
---
> [!note]
> 该函数检测 URL 的稳定性，并检测所有的 GET、POST、Cookie 和 User-Agent 参数是否为动态并可能受到 SQL 注入攻击的影响。  

## 1.1 使用 -d 命令
```python
if conf.direct:  
    initTargetEnv()  
    setupTargetEnv()  
    action()  
    return True
```
初始化目标环境，更新并保存目标环境。这部分内容都会涉及到更新全局变量：conf 和 kb
action()函数，执行 SQL 注入攻击  

## 1.2 遍历 kb.targets 内容并执行注入攻击  
```python
for targetUrl, targetMethod, targetData, targetCookie, targetHeaders in kb.targets:  
    targetCount += 1  
    ……
    try:  
        if conf.checkInternet:  ……

```
### 1.2.1 使用 --check-internet 命令  
检测是否连接到互联网。若没有也会尝试重连，超出配置次数后会报错并退出程序。  

### 1.2.2 执行注入的预处理 
1. 对 conf 的 data 部分执行 URL 编码并删除请求头中重复内容  
	```python
	if conf.data:   
	    conf.data = re.sub(r"\b(__\w+)=([^&]+)", lambda match: "%s=%s" % (match.group(1), urlencode(match.group(2), safe='%')), conf.data)  
  
	conf.httpHeaders = [conf.httpHeaders[i] for i in xrange(len(conf.httpHeaders)) if conf.httpHeaders[i][0].upper() not in (__[0].upper() for __ in conf.httpHeaders[i + 1:])]
	```
2. 初始化目标环境，并对解析目标 url 地址  
	```python
	initTargetEnv()  
	parseTargetUrl()
	```
	parseTargetUrl 函数：检查 URL 是否合法，处理自定义注入标记，解析 URL 的各个组成部分（协议、主机名、端口、路径等），根据配置强制转换为 HTTPS，校验主机名和端口的合法性，并构造最终的 URL。
3. 判断是否需要测试 SQL 注入漏洞    
	```python
	# 判断是否需要测试SQL注入
	testSqlInj = False  
	# 检测每个参数是否未被测试过SQL注入
	if PLACE.GET in conf.parameters and not any((conf.data, conf.testParameter)):  
	    for parameter in re.findall(r"([^=]+)=([^%s]+%s?|\Z)" % (re.escape(conf.paramDel or "") or DEFAULT_GET_POST_DELIMITER, re.escape(conf.paramDel or "") or DEFAULT_GET_POST_DELIMITER), conf.parameters[PLACE.GET]):  
	        paramKey = (conf.hostname, conf.path, PLACE.GET, parameter[0])  
			
			# 如果参数未被测试过，则设置testSqlInj为True并跳出循环
	        if paramKey not in kb.testedParams:  
	            testSqlInj = True  
	            break
		            
	else:  
	    paramKey = (conf.hostname, conf.path, None, None)  
	    if paramKey not in kb.testedParams:  
	        testSqlInj = True  

	# 如果需要测试SQL注入且当前主机已被标记为存在漏洞，询问是否跳过此主机的后续测试
	if testSqlInj and conf.hostname in kb.vulnHosts:  
	    if kb.skipVulnHost is None:  
	        message = "SQL injection vulnerability has already been detected "  
	        message += "against '%s'. Do you want to skip " % conf.hostname  
	        message += "further tests involving it? [Y/n]"  
			# 读取用户输入，决定是否跳过此主机的测试，保存到环境变量 kb
	        kb.skipVulnHost = readInput(message, default='Y', boolean=True)  
		# 根据用户选择决定是否继续测试此主机
	    testSqlInj = not kb.skipVulnHost  
	    
	# 如果不需要测试SQL注入，则记录信息并跳过当前目标URL的测试
	if not testSqlInj:  
	    infoMsg = "skipping '%s'" % targetUrl  
	    logger.info(infoMsg)  
	    continue
	```
4. 询问用户是否测试表单，可由用户输入请求包中的 data 信息，并重新解析目标 URL ，设置目标环境  
5. 根据原始页面的 `<select>`  标签，更新 kb 的随机池
	```python
	# 如果conf.rParam为真且原始页面已知，进行以下操作
    if conf.rParam and kb.originalPage:
        # 确保randomPool中的每个值都是列表类型
        kb.randomPool = dict([_ for _ in kb.randomPool.items() if isinstance(_[1], list)])

        # 遍历原始页面中所有<select>元素，提取name和options
        for match in re.finditer(r"(?si)<select[^>]+\bname\s*=\s*[\"']([^\"']+)(.+?)</select>", kb.originalPage):
            name, _ = match.groups()
            # 提取每个<select>元素中的<option>值
            options = tuple(re.findall(r"<option[^>]+\bvalue\s*=\s*[\"']([^\"']+)", _))
            # 如果找到选项值，则更新randomPool
            if options:
                kb.randomPool[name] = options
	```
6. 检测目标网站是否使用 waf 产品，详细内容可以查看： [[sqlmap 源码解读：checks.py#1. checkWaf 函数]]  
	```python
	checkWaf()
	```
7. 检测参数是否存在 SQL 注入漏洞，并且对待检测的参数进行排序，如果发现漏洞，将保存在在 kb.injections 中
	- 当 level 大于等于 3，检测请求头中的 User-Agent 和 Referer 字段；  
	- 当 level 大于等于 5，检测请求头中的 Host 字段；  
	- 当 level 大于等于 2，检测请求头中的 Cookie 字段；
	- 检测是否使用 `--param-filter` 命令；  
	- 检测参数是否存在 SQL 注入，使用到  [[sqlmap 源码解读：checks.py#2. checkSqlInjection 函数]]  
8. 若经过检测存在 SQL 注入，执行以下函数：  
	```python
	kb.testMode = False  
  
	_saveToResultsFile()  
	_saveToHashDB()  
	_showInjections()  
	_selectInjection()
	```

### 1.2.3 执行 SQL 注入攻击  
根据上述的扫描结果，以及用户的选择，来决定是否进行 SQL 注入攻击  
```python
if kb.injection.place is not None and kb.injection.parameter is not None:  
if conf.multipleTargets:  
    message = "do you want to exploit this SQL injection? [Y/n] "  
    condition = readInput(message, default='Y', boolean=True)  
else:  
    condition = True  
  
if condition:  
	# 执行SQL注入攻击
    action()
```

action 函数详解：[[sqlmap 源码解读：action.py]]

### 1.2.4 异常处理  

## 1.3 输出日志  
输出扫描结果的保存路径  
