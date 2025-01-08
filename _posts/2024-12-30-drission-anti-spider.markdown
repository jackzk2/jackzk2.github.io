---
layout:     post
title:      "DrissionPgae 自动化解决高并发复杂反爬"
date:       2024-12-30 16:53:00
author:     "zhangk"
catalog: true
tags:
    - Scrape
---



因为业务需要，我持续几年跟踪了国外某上市公司官网的公开数据，但是最近一段时间该网站进行了更新。不但使用了coludflare的五秒盾，同时加上了指纹验证和前端带参数的加密验证。

我觉得很有代表性，因此把解决的思路整理出来：



# 1、解决五秒盾

逆向破解coludflare turnstile难度太大，而且他会一直更新。

可行的办法一个是使用三方的付费api（比较好的是https://2captcha.com/的付费api），另一种是自动化，我决定使用第二种。

## 1.1、 解决robot check

我尝试使用selenium打开网页，然后手动点击，发现会被coludflare检测到，即使手动点击也无法通过验证。

尝试使用undetected chrome driver，还是会遇到同样的问题。

我又试了一下使用drission打开网页，手动点击，验证通过。



## 1.2、解决点击验证

我尝试用click方式点击单选框，发现drissionpage根本无法定位到单选框。

因为cloudflre的面板单独在一个iframe中，我尝试切换到iframe中再去点击

![image-20241230114920724](image/image-20241230114920724.png)

```python
iframe = tab.get_frame('css: iframe[src^="XXXXX"]')
iframe = tab.get_frame(1)
```

这样还是会被检测到。

这时我在github上找到了这个库https://github.com/seleniumbase/SeleniumBase，他提供了一个uc_gui_click_cf()方法，他提供了selenium + pyautogui 点击的方案。

我使用drissionpage 模仿他的代码，不过他的代码写的过于复杂。实际上只需要确定cloudflare面板的坐标位置，然后就可以大体确定选择框的坐标，使用pyautogui点击就可以了

```python
cf_loc = dp.ele('#MasterGC_ContentBlockHolder_divCaptcha').rect.screen_location
x = cf_loc[0] + 22 
y = cf_loc[1] + 29
pyautogui.moveTo(x, y, 0.5)
pyautogui.click(x=x, y=y)
```

这个方法简单易用，缺点是pyautofgui需要使用屏幕，而且自动化的稳定性稍差，需要做好容错。



# 2、解决其他验证

## 2.1、指纹

解决了五秒盾之后，以为带着cf_clearance参数就可以一路绿灯了，发现网站还使用了ja3指纹。

于是又用curl_cffi解决了指纹问题

```python
from curl_cffi import requests
requests.post(url, proxies=proxies, headers=headers, cookies=cookies, data=post_data, timeout=timeout, impersonate="chrome119", verify=False)
```



## 2.2、js参数

这时还有一些请求无法使用，抓包分析之后发现除了cf_clearance之外还有一个参数data_dun，分析数据包之后可以确定，这个参数是前端js生成的,要破解这个参数就需要对前端代码进行逆向。

跟值找到代码逻辑,发现data_dun是带着请求参数加密的，我通过逆向得到了data_dun的代码，但始终无法成功请求。

## 2.3、自动化

最后我只能选择将无法请求的页面用drissionpage自动化实现。

因为之前的代码使用的是requests, 如果直接使用` drissionpage.page_source `获取网页源码，对于数据的提取需要做很大修改。（因为前者是网站返回的json数据，后者是经过浏览器引擎处理的html数据）

为了直接使用之前的代码，就需要使用`dp.listen`抓取到请求包：

```python
dp.listen.start(targets='www.xxx.com/xxx',method=('POST'))
dp.ele('xxx').click()
target_block = listen.steps(timeout=20).__next__()
```



然后提取出响应的数据：

```python
state_code = target_block.response.extra_info.all_info.get('statusCode')
if state_code != 200:
	raise Exception
resp = target_block.response.raw_body
```



同时项目还涉及到文件下载，使用内置的to_download方法非常简单：

```python
mission = file_ele.click.to_download(save_path=save_path, rename=file_name)
mission.wait(60)
```

另外这个方法下载pdf时会直接打开，需要修改chrome的设置

![image-20241224145517301](image/image-20241230145517301.png)



# 3、多进程

因为项目涉及到的访问量比较大，我把程序改成了多进程。这也遇到了很多问题，自动化的坑还是比较多。

## 3.1、错误的将self作为参数传入

一开始我的代码是这样的，我发现当我想把self作为参数传入task 函数时会出问题：

```python
    def create_task_queue(self, task_list):
        task_queue = multiprocessing.Queue()
        for task in task_list:
            task_queue.put(task)
        return task_queue
        
    def mian():
        task_queue = self.create_task_queue(goods_list)
		for _ in range(self.num_workers):
            p = multiprocessing.Process(target=self.get_pages, args=(task_queue,))
            p.start()
            self.processes.append(p)
            
    def get_pages(xxx):
        pass
```



我把多进程结构放到Action类的外面，问题解决：

```python
def main_work(queue, other_kwargs):
    i = other_kwargs.get('i')
    lock = other_kwargs.get('lock')
	run_date = other_kwargs.get('run_date')
    a = Action()
    res = a.main(tender_block=tender_block, lock=lock, i=i, run_date=run_date)

task_queue = create_task_queue(tender_list)
workers = [multiprocessing.Process(target=main_work, 
                                   args=(task_queue, {'lock': lock, 'i': i, 'run_date': run_date})) 
           for i in range(num_workers)]
for w in workers:
	w.start()

task_queue.join()

for w in workers:
	w.terminate()
	w.join()
```



## 3.2、drissionpage多进程

drissionpage使用多进程时，要为每一个chrome设置不同的端口，于是我使用auto_port模式自动分配端口。

我发现之前对chrome的设置无法同步。

`co = ChromiumOptions().set_browser_path(r'xxx').auto_port()`

查询官方文档，发现set_browser_path 和 auto_port 会相互覆盖，而且auto_port不支持多进程。

 ![image-20241229153233774](image/image-20241230153233774.png)

因此我使用set_local_port为每个进程设置chrome端口号

`ChromiumOptions().set_browser_path(r'xxx').set_local_port(9600+mult_index)`

然后把每个端口下的chrome重新设置一遍，后续就会保持初始的设置。



## 3.3、使用屏幕

因为我的程序需要使用pyautogui点击屏幕，因此当需要点击屏幕时需要让当前浏览器保持在最顶层，

并且在执行pyautogui操作时使用线程锁锁定，这样其他进程就不会打开浏览器而占用屏幕，或者出现多个进程同时进行点击操作：

```PYTHON
with lock:
   cf_loc = dp.ele('#MasterGC_ContentBlockHolder_divCaptcha').rect.screen_location
   x = cf_loc[0] + 22 
   y = cf_loc[1] + 29
   pyautogui.moveTo(x, y, 0.5)
   pyautogui.click(x=x, y=y)
```



# 4、减少流量

我发现我在打开页面时，网站会重复请求一些资源文件，我希望把这些资源文件重定向，以减少流量的消耗。

drission的listen只能抓取请求包，并不能对请求的内容进行修改，因此我使用mitmproxy对请求的url修改。

但是修改之后发现网页会一直加载中，正常的请求会一直报403，猜测可能是开启了跨域的检测。



于是把不必要的请求直接杀死，问题解决，减少了部分流量的消耗。

