---
title: DoubanScraper豆瓣数据爬取及其衍生功能-2
date: 2025-04-10 20:42:14
tags: 
cover: images/Pasted%20image%2020250413003659.png
"repo:":
---


# 环境要求
基于chrome的chromedriver 豆瓣账号 python pip相关库

# 是否需要 --user-data-dir
其实简单理解就是代码全部使用的无痕目录，但cookie可疑的，来禁用复用登录来增加兼容性

| 目的                   | 是否需要 `--user-data-dir`              |
| -------------------- | ----------------------------------- |
| 加 cookie（用代码）        | ❌ 不需要                               |
| 自动复用登录状态（懒得加 cookie） | ✅ 需要，但注意路径唯一                        |
| 多开、多实例               | ❌ 不建议用 `user-data-dir`，或者每个实例用不同的路径 |
# cookie使用
去douban.com拿取
![|350](images/Pasted%20image%2020250415004909.png)
由于面向用户，让用户去修改成一个标准的格式还是不太友好，于是对脚本们cookie进行了处理，现在可以实现如下功能

| 目标                | 是否实现 |
| ----------------- | ---- |
| 直接复制浏览器一整段 Cookie | ✅ 支持 |
| 用户不需要手动格式化        | ✅ 简化 |
| 保持代码优雅            | ✅ 干净 |
# 自动设置UA
UA 是 User-Agent，全称“用户代理”，是浏览器告诉网站“我是谁”的方式。
比如：
```
Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36
```
在 无头模式（--headless） 下，浏览器的默认 UA 会暴露你是“机器人”，有些网站（比如豆瓣）可能会返回空页面或者错误提示。
所以我们手动加个正常的 UA，让网站以为我们是正常用户打开浏览器的，提升稳定性。

# 启用配置
```
chrome_options.add_argument("--headless")  # 如需可见浏览器，可注释这一行
chrome_options.add_argument("--disable-gpu")
chrome_options.add_argument("--no-sandbox")
```
进行了更好的调试和禁用gpu加速防止异常和容器化，禁用Chrome的沙盒机制 **解决权限问题**：当以root用户运行Chrome时，默认情况下会因为安全原因而失败。
# 异常情况
## 异常ip请求 登录使用豆瓣
![](images/Pasted%20image%2020250411113147.png)
被封禁ip了，导入cookie可以结局，如果降低速度或者什么的话会非常非常非常慢，所以我的解法是代理池和adb配合手机流量刷飞行


## 搜索访问太频繁
![|475](images/Pasted%20image%2020250411011435.png)
出现这种错误的解决方法是配合cookie去做访问 ，登录了就大大增加了搜索限制 ， 不过也是有阈值的
![|450](images/Pasted%20image%2020250411011456.png)
## 异常ip 点击继续
![](images/Pasted%20image%2020250411012649.png)
写针对性脚本来对抗此 作为自检程序，见附件0_0_rao.py
## 百条数据无权限
 
![|450](images/Pasted%20image%2020250411040026.png)
载入cookie后明显可以更多

## 短评400条为上限
![|275](images/Pasted%20image%2020250411160812.png)
所以短评那里
## 1600条数据封禁ip
在多次测试后都是到400的阈值封禁的ip，经检查代码判断停止标准为小于20条，但实际上豆瓣存在折叠不显示评论，实际某些页面少于20条，将限制评论改为
0 class 停止
![|450](images/Pasted%20image%2020250411160742.png)
## 豆瓣对反爬虫机制
-请求频率限制：每个 IP 地址每分钟最多只能请求 40 次；
-验证码：当请求次数超过阈值时，会弹出验证码；
-代理检测：检测是否使用代理服务器进行访问。


# 附件代码
## 豆瓣cookie示例
![|575](images/Pasted%20image%2020250412155146.png)
## 0_0_cookie_test.py
用来验证cookie是否可用的，需要外部文件cookie.txt
```
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
import time

def load_cookies(file_path):
    """从给定路径加载cookie"""
    cookies = []
    with open(file_path, 'r') as file:
        for line in file:
            # 去除注释行或空行
            if not line.strip().startswith("#") and "=" in line:
                # 分割name=value形式的字符串
                name, value = line.strip().split('=', 1)
                cookies.append({'name': name, 'value': value})
    return cookies

# 设置ChromeDriver的路径
chrome_driver_path = "C:/Users/13519/.cache/selenium/chromedriver/win64/135.0.7049.84/chromedriver.exe"  # 替换为你的chromedriver的实际路径

# 创建一个Chrome浏览器实例
service = Service(chrome_driver_path)
driver = webdriver.Chrome(service=service)

# 打开目标网站
driver.get('https://www.douban.com')

# 等待页面加载
time.sleep(3)

# 加载cookies
cookie_file_path = "cookie.txt"  # 替换为你的cookie.txt文件的实际路径
cookies = load_cookies(cookie_file_path)

for cookie in cookies:
    driver.add_cookie(cookie)

# 刷新页面使Cookie生效
driver.refresh()

# 您可以在这里进行进一步的操作或等待一段时间观察结果
time.sleep(10)

# 关闭浏览器
driver.quit()
```
## 0_0_rao.py
类似于自检功能，有时候豆瓣会有随机验证
用来屏蔽异常ip 点击续集按钮的，这个可以列为自检程序去运行
```
from selenium import webdriver
from selenium.webdriver.chrome.service import Service as ChromeService
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
import time

def handle_continue_button(driver):
    """
    检查并处理"点我继续浏览"按钮。
    如果按钮存在则进行点击操作，并再次检查确保按钮消失；
    如果按钮不存在，则直接返回。
    """
    try:
        # 尝试寻找按钮元素
        continue_button = driver.find_element(By.XPATH, "//button[@id='sub' and @name='btnsubmit']")
        if continue_button.is_displayed():
            print("发现'点我继续浏览'按钮，正在点击...")
            continue_button.click()
            print("已点击'点我继续浏览'按钮，等待页面加载...")
            time.sleep(3)  # 等待页面响应，可根据实际情况调整时间
            
            # 刷新页面验证按钮是否仍然存在
            driver.refresh()
            time.sleep(5)  # 页面刷新后的加载时间
            
            # 再次尝试查找按钮，以确认是否成功处理
            try:
                continue_button = driver.find_element(By.XPATH, "//button[@id='sub' and @name='btnsubmit']")
                if not continue_button.is_displayed():
                    print("按钮已成功处理，不再显示。")
                else:
                    print("警告：按钮依然存在，可能需要进一步处理。")
            except Exception:
                print("按钮已成功处理，不再显示。")
        else:
            print("未显示'点我继续浏览'按钮，无需处理。")
    except Exception:
        print("未找到'点我继续浏览'按钮，无需处理。")

# 设置Chrome选项
chrome_options = Options()
chrome_options.add_argument('--ignore-certificate-errors')
chrome_options.add_argument(f'user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36')

# 创建WebDriver实例
service = ChromeService(executable_path=r'C:\Users\13519\.cache\selenium\chromedriver\win64\135.0.7049.84\chromedriver.exe')
driver = webdriver.Chrome(service=service, options=chrome_options)

try:
    # 访问豆瓣主页
    driver.get('https://www.douban.com')
    
    # 处理“点我继续浏览”按钮
    handle_continue_button(driver)
    
    # 页面停留
    time.sleep(10)  # 停留10秒，可根据需求调整

except Exception as e:
    print(f"发生错误: {e}")
finally:
    # 关闭浏览器
    driver.quit()
```
## 0_saved_page.py
存储页面
由于豆瓣的js和反爬为中级水平，所以建议的方法还是获得一个完整的源代码再去做过滤
```
from selenium import webdriver
from selenium.webdriver.chrome.service import Service  # 导入Service类
import time

def load_cookies(file_path):
    """从给定路径加载cookie"""
    cookies = []
    with open(file_path, 'r') as file:
        for line in file:
            # 去除注释行或空行
            if not line.strip().startswith("#") and "=" in line:
                # 分割name=value形式的字符串
                name, value = line.strip().split('=', 1)
                cookies.append({'name': name, 'value': value})
    return cookies

def read_search_query(file_path):
    """从指定路径读取搜索关键词"""
    with open(file_path, 'r', encoding='utf-8') as file:
        query = file.readline().strip()  # 仅读取第一行作为查询关键词
    return query

def scroll_and_collect_data(driver):
    # 导航到目标搜索页面
    bian_liang1 = read_search_query("0_yonghu_select.txt")  # 修改这里以读取文件内容
    url = f'https://search.douban.com/movie/subject_search?search_text={bian_liang1}'
    driver.get(url)

    # 等待页面加载
    time.sleep(3)

    # 加载cookies
    cookie_file_path = "cookie.txt"  # 替换为你的cookie.txt文件的实际路径
    cookies = load_cookies(cookie_file_path)

    for cookie in cookies:
        driver.add_cookie(cookie)

    # 刷新页面使Cookie生效
    driver.get(url)

    # 模拟多次滚动加载内容（每次滚动后暂停一会）
    for _ in range(5):
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        time.sleep(3)  # 每次滚动后等待内容加载

    # 等加载完再保存 HTML
    with open('0_saved_page.html', 'w', encoding='utf-8') as f:
        f.write(driver.page_source)

    # 截图保存
    driver.save_screenshot('0_screenshot.png')

options = webdriver.ChromeOptions()
options.add_argument('--start-maximized')

# 设置ChromeDriver的路径
chrome_driver_path = "C:/Users/13519/.cache/selenium/chromedriver/win64/135.0.7049.84/chromedriver.exe"  # 替换为你的chromedriver的实际路径

driver = webdriver.Chrome(service=Service(chrome_driver_path), options=options)

scroll_and_collect_data(driver)

driver.quit()
```
## 1_guolv.py
过滤数据为html
这个的过滤是将采集到的html过滤为一个简化的html 使其能够有页面处理甚至面向用户的功能
```
from bs4 import BeautifulSoup

# 定义输入和输出文件名
input_file = '0_saved_page.html'
output_file = '1_guolv.html'

# 读取HTML文件
with open(input_file, 'r', encoding='utf-8') as file:
    contents = file.read()

# 使用BeautifulSoup解析HTML
soup = BeautifulSoup(contents, 'lxml')

# 查找id为"root"且class为"root"的div元素
root_div = soup.find("div", {"id": "root", "class": "root"})

# 检查是否找到了目标div
if root_div is not None:
    # 将找到的内容写入到新文件中
    with open(output_file, 'w', encoding='utf-8') as output:
        # 写入过滤后的内容
        output.write(str(root_div))
    print(f"已成功将结果保存到 {output_file}")
else:
    print("未找到id为'root'且class为'root'的div元素")
```
## 2_guolv.py
过滤数据为csv
这个是为了做可视化以及让用户选择的功能的 
比如手动选取要爬取具体搜索的哪个相关匹配度搞得数据 且附件id和跳转链接更方便给下面爬取脚本去识别
![](images/Pasted%20image%2020250412161706.png)
```
from bs4 import BeautifulSoup
import csv
import re

# 读取HTML文件
with open('1_guolv.html', 'r', encoding='utf-8') as file:
    html_content = file.read()

# 解析HTML
soup = BeautifulSoup(html_content, 'lxml')

# 准备CSV写入，注意这里使用了 utf-8-sig 编码来包含 BOM
with open('2guolv.csv', mode='w', newline='', encoding='utf-8-sig') as csv_file:
    writer = csv.writer(csv_file)

    # 写入表头
    writer.writerow(['名称', '年份', '类型', '评分', '评价人数', '国家/地区', '时长', '导演', '主演', 'ID', '链接'])

    # 查找所有项目
    items = soup.find_all(class_='item-root')
    for item in items:
        title_tag = item.find(class_='title-text')
        title = title_tag.get_text(strip=True) if title_tag else ''

        # 用正则提取 (年份)
        match_year = re.search(r'\((\d{4})\)', title)
        year = match_year.group(1) if match_year else ''
        title_clean = title.replace(f'({year})', '').strip() if year else title

        rating_tag = item.find(class_='rating_nums')
        rating = rating_tag.text if rating_tag else '暂无评分'

        votes_tag = item.find(class_='pl')
        votes = votes_tag.text.strip('()') if votes_tag and '人评价' in votes_tag.text else ''

        meta_tags = item.find_all(class_='meta')
        region_genre_duration = meta_tags[0].text if len(meta_tags) > 0 else ''
        director_cast = meta_tags[1].text if len(meta_tags) > 1 else ''

        # 获取链接和ID
        href = title_tag.get('href') if title_tag else ''
        match = re.search(r'/subject/(\d+)/', href)
        subject_id = match.group(1) if match else '未知'

        # 简化处理：直接分割字符串获取地区、类型和时长
        region_genre_duration_parts = region_genre_duration.split('/')
        region = region_genre_duration_parts[0].strip() if len(region_genre_duration_parts) > 0 else ''
        genre = '/'.join(region_genre_duration_parts[1:-1]).strip() if len(region_genre_duration_parts) > 2 else ''
        duration = region_genre_duration_parts[-1].strip() if len(region_genre_duration_parts) > 0 else ''

        # 分离导演和演员信息
        director_cast_parts = director_cast.split('/')
        director = director_cast_parts[0].replace('导演', '').strip() if len(director_cast_parts) > 0 else ''
        cast = ','.join(director_cast_parts[1:]).strip() if len(director_cast_parts) > 1 else ''

        # 写入数据行
        writer.writerow([title_clean, year, genre, rating, votes, region, duration, director, cast, subject_id, href])

print("CSV文件已成功生成")
```

## 3comment_duan.py
针对短评去采集的 但是有一点短评400条几乎就到上限了，豆瓣就不显示了，会显示没有数据了 且存在三种status可以调节 对应 P N F  
对应看过 在看 想看 所以这个短评的数据采集1200条就是上限了，我就设置为1500条阈值来让完整采集。
其它：
内存型写入，程序执行完，才对csv进行写入,id的选取取决于当前文件夹下的3id.txt 这个的存在方便和其他脚本对接
![|400](images/Pasted%20image%2020250412163247.png)
![|375](images/Pasted%20image%2020250412163237.png)

![](images/Pasted%20image%2020250412163342.png)
```
import time
import random
from selenium import webdriver
from selenium.webdriver.chrome.service import Service as ChromeService
from selenium.webdriver.common.by import By
from bs4 import BeautifulSoup
import pandas as pd


# 读取本地文件进行 变量
# 定义函数从文件中读取电影ID
def read_id_from_file(file_path):
    with open(file_path, 'r') as file:
        # 读取第一行并去除首尾空格
        id_search = file.readline().strip()
    return id_search
# 使用该函数读取ID
id_search = read_id_from_file('3id.txt')
# 打印出来确认是否正确读取
print(f"正在使用的电影ID是: {id_search}")


# 初始化全局数据列表
all_data = []

# 设置ChromeDriver路径
chrome_driver_path = r"C:\Users\13519\.cache\selenium\chromedriver\win64\135.0.7049.84\chromedriver.exe"

# 创建一个新的Chrome浏览器实例
service = ChromeService(executable_path=chrome_driver_path)
driver = webdriver.Chrome(service=service)

def load_cookies(file_path):
    """从给定路径加载cookie"""
    cookies = []
    with open(file_path, 'r') as file:
        for line in file:
            if not line.strip().startswith("#") and "=" in line:
                name, value = line.strip().split('=', 1)
                cookies.append({'name': name, 'value': value})
    return cookies

def fetch_comments(start, status):
    url = f'https://movie.douban.com/subject/{id_search}/comments?start={start}&limit=20&status={status}&sort=new_score'
    driver.get(url)
    
    # 加载cookies
    cookies = load_cookies('cookie.txt')
    for cookie in cookies:
        driver.add_cookie(cookie)
    
    # 刷新页面以应用cookies
    driver.refresh()
    
    # 等待页面加载完成
    time.sleep(5)
    
    # 使用BeautifulSoup解析页面内容
    soup = BeautifulSoup(driver.page_source, 'html.parser')
    comments = soup.find_all('div', class_='comment-item')
    
    return comments

def get_normal_distribution_delay(mean=7.5, sigma=1.25):
    min_delay = 5
    max_delay = 10
    delay = random.gauss(mean, sigma)
    if delay < min_delay:
        return min_delay
    elif delay > max_delay:
        return max_delay
    else:
        return delay

# 新增status列表
statuses = ['P', 'N', 'F']

for status in statuses:
    print(f"正在处理status={status}的评论...")
    start = 0
    
    while True:
        print(f"正在处理第 {start} 条开始的数据...")
        comments = fetch_comments(start, status)
        
        if len(comments) == 0 or start >= 300:  
            break

        for comment in comments:
            user = comment.find('a').get('title') if comment.find('a') else ''
            location = comment.find('span', class_='comment-location').text if comment.find('span', class_='comment-location') else ''
            content = comment.find('span', class_='short').text if comment.find('span', class_='short') else ''
            time_tag = comment.find('span', class_='comment-time').get('title') if comment.find('span', class_='comment-time') else ''
            
            all_data.append([user, location, content, time_tag, status])
        
        start += 20
        delay = get_normal_distribution_delay()
        print(f"等待 {delay:.2f} 秒后继续...")
        time.sleep(delay)

# 关闭浏览器
driver.quit()

# 将所有数据整合并导出为csv
df = pd.DataFrame(all_data, columns=['用户名', '位置', '评论内容', '时间', '状态'])
df.to_csv('3_comment_duan.csv', index=False, encoding='utf-8-sig')
print("所有数据已成功保存到3_comment_duan.csv")
```


## 3comment_chang.py
针对长评区做采集 不过这个不咋用
```
import time
import random
from selenium import webdriver
from selenium.webdriver.chrome.service import Service as ChromeService
from selenium.webdriver.common.by import By
from bs4 import BeautifulSoup
import pandas as pd


# 定义函数从文件中读取电影ID
def read_id_from_file(file_path):
    with open(file_path, 'r') as file:
        # 读取第一行并去除首尾空格
        id_search = file.readline().strip()
    return id_search
# 使用该函数读取ID
id_search = read_id_from_file('3id.txt')
# 打印出来确认是否正确读取
print(f"正在使用的电影ID是: {id_search}")

# 初始化全局数据列表
all_data = []

# 设置ChromeDriver路径
chrome_driver_path = r"C:\Users\13519\.cache\selenium\chromedriver\win64\135.0.7049.84\chromedriver.exe"

# 创建一个新的Chrome浏览器实例
service = ChromeService(executable_path=chrome_driver_path)
driver = webdriver.Chrome(service=service)

def load_cookies(file_path):
    """从给定路径加载cookie"""
    cookies = []
    with open(file_path, 'r') as file:
        for line in file:
            # 去除注释行或空行
            if not line.strip().startswith("#") and "=" in line:
                # 分割name=value形式的字符串
                name, value = line.strip().split('=', 1)
                cookies.append({'name': name, 'value': value})
    return cookies

def fetch_reviews(start):
    url = f'https://movie.douban.com/subject/{id_search}/reviews?sort=hotest&start={start}'
    driver.get(url)
    
    # 加载cookies
    cookies = load_cookies('cookie.txt')
    for cookie in cookies:
        driver.add_cookie(cookie)
    
    # 刷新页面以应用cookies
    driver.refresh()
    
    # 等待页面加载完成，可能需要根据网络情况调整等待时间
    time.sleep(5)
    
    # 使用BeautifulSoup解析页面内容
    soup = BeautifulSoup(driver.page_source, 'html.parser')
    reviews = soup.find_all('div', class_='review-item')  # 根据新页面结构调整
    
    return reviews

def get_normal_distribution_delay(mean=20, sigma=3):  # 可以根据需要调整sigma值
    """生成一个符合正态分布的延迟时间，确保其在min_delay和max_delay之间"""
    min_delay = 15 # 设置最小延迟
    max_delay = 25  # 设置最大延迟
    delay = random.gauss(mean, sigma)
    if delay < min_delay:
        return min_delay
    elif delay > max_delay:
        return max_delay
    else:
        return delay

start = 0
while True:
    print(f"正在处理第 {start} 条开始的数据...")
    reviews = fetch_reviews(start)

    # 修改为仅在没有获取到评论（即数量为0）或者超过特定数量时结束循环
    if len(reviews) == 0 or start >= 1500:  # 仅在没有获取到评论或达到设定的上限时停止循环
        break
    
    for review in reviews:
        user = review.find('a', class_='name').text if review.find('a', class_='name') else ''
        content = review.find('div', class_='short-content').text if review.find('div', class_='short-content') else ''
        time_tag = review.find('span', class_='main-meta').text if review.find('span', class_='main-meta') else ''
        
        all_data.append([user, content.strip(), time_tag])
    
    # 更新下一次开始的位置，并等待一段时间
    start += 20
    delay = get_normal_distribution_delay()
    print(f"等待 {delay:.2f} 秒后继续...")
    time.sleep(delay)
# 截图保存
driver.save_screenshot('execution_result.png')

# 关闭浏览器
driver.quit()

# 将所有数据整合并导出为csv
df = pd.DataFrame(all_data, columns=['用户名', '评论内容', '时间'])
df.to_csv('3_comment_chang.csv', index=False, encoding='utf-8-sig')
print("所有数据已成功保存到3_comment_chang.csv")
```

## app_meihua.py
用来监听web的
![|375](images/Pasted%20image%2020250413000023.png)
```
from flask import Flask, request, jsonify, render_template_string, send_file
import os
import subprocess

app = Flask(__name__)

# 获取最新的词云图文件（在 ci_yun_tu 文件夹下）
def get_latest_wordcloud_file():
    wordcloud_folder = os.path.join(os.getcwd(), 'ci_yun_tu')  # 当前目录下的 ci_yun_tu 文件夹
    if not os.path.exists(wordcloud_folder):
        return None
    wordcloud_files = [f for f in os.listdir(wordcloud_folder) if f.endswith('.png')]  # 假设词云图是 .png 格式
    if not wordcloud_files:
        return None
    wordcloud_files.sort(key=lambda f: os.path.getmtime(os.path.join(wordcloud_folder, f)), reverse=True)
    return os.path.join(wordcloud_folder, wordcloud_files[0])

@app.route('/download1_csv')
def download1_csv():
    file_path = '2guolv.csv'  # 查找结果的CSV
    if os.path.exists(file_path):
        return send_file(file_path, as_attachment=True)
    else:
        return jsonify({"status": "error", "message": "未找到查找CSV文件"}), 404
    
@app.route('/download_comment_duan_csv')
def download_comment_duan_csv():
    file_path = '3_comment_duan.csv'  # 需要下载的 CSV 文件路径
    if os.path.exists(file_path):
        return send_file(file_path, as_attachment=True)
    else:
        return jsonify({"status": "error", "message": "未找到 comment duan CSV 文件"}), 404

@app.route('/download_wordcloud')
def download_wordcloud():
    latest_wordcloud_file = get_latest_wordcloud_file()
    if latest_wordcloud_file is None:
        return jsonify({"status": "error", "message": "没有找到词云图文件"}), 404
    return send_file(latest_wordcloud_file, as_attachment=True)

@app.route('/execute', methods=['POST'])
def execute():
    user_input = request.json.get('userInput')
    if not user_input:
        return jsonify({"status": "error", "message": "请输入内容后再进行查找。"}), 400
    try:
        with open('0_yonghu_select.txt', 'w', encoding='utf-8') as f:
            f.write(user_input)
        result = subprocess.run(['python', '0_start.py'], capture_output=True, text=True)
        if result.returncode == 0:
            return jsonify({"status": "success", "output": result.stdout})
        else:
            return jsonify({"status": "error", "message": result.stderr}), 500
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500
    
    
@app.route('/execute2', methods=['POST'])
def execute2():
    user_input = request.json.get('userInput')
    if not user_input:
        return jsonify({"status": "error", "message": "请输入内容后再进行查找。"}), 400
    try:
        # 将用户输入写入 3id.txt
        with open('3id.txt', 'w', encoding='utf-8') as f:
            f.write(user_input)
        
        # 执行 0_0start.py 脚本
        result = subprocess.run(['python', '0_0start.py'], capture_output=True, text=True)
        if result.returncode == 0:
            return jsonify({"status": "success", "output": result.stdout})
        else:
            return jsonify({"status": "error", "message": result.stderr}), 500
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500


@app.route('/')
def home():
    return render_template_string('''
<html>
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1">
<style>
    body {
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen;
        margin: 0;
        padding: 0;
        background-color: #f4f4f4;
    }

    .container {
        max-width: 600px;
        margin: 30px auto;
        padding: 20px;
        background: white;
        border-radius: 10px;
        box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }

    .form-group {
        margin-bottom: 10px;
    }

    input[type="text"] {
        width: 100%;
        padding: 12px;
        font-size: 16px;
        margin-bottom: 10px;
        border: 1px solid #ccc;
        border-radius: 6px;
        box-sizing: border-box;
    }

    button {
        width: 100%;
        padding: 12px;
        font-size: 16px;
        border: none;
        border-radius: 6px;
        background-color: #28a745;
        color: white;
        cursor: pointer;
        margin-bottom: 5px;
    }

    button:hover {
        background-color: #218838;
    }

    button:disabled {
        background-color: #ccc;
        cursor: not-allowed;
    }

    #result {
        margin-top: 20px;
        padding: 15px;
        background-color: #e9f7ef;
        border-bottom: 5px solid #005f40;
        border-radius: 6px;
        white-space: pre-wrap;
        word-break: break-word;
    }

    table {
        width: 100%;
        border-collapse: collapse;
        margin-top: 15px;
        font-size: 14px;
    }

    th, td {
        border: 1px solid #ddd;
        padding: 8px;
    }

    th {
        background-color: #4CAF50;
        color: white;
    }

    #usageInstructions {
        background-color: #f8f9fa;
        padding: 10px;
        border-radius: 5px;
        margin-top: 20px;
        display: none;
    }
</style>

</head>
<body>
<div class="container">
    <!-- 使用说明按钮 -->
    <button onclick="toggleUsageInstructions()">查看使用说明</button>

    <div id="usageInstructions">
        <h3>使用说明</h3>
        <p><strong>0. 页面是干什么的: </strong>针对豆瓣影视短评进行爬取并选择性生成词云遮罩，当前页面主要是用于查找电影或电视剧相关信息并生成词云。</p>
        <p><strong>1. 查找功能：</strong>输入电影或电视剧名称，点击“查找”按钮，在等待一段时间后，返回豆瓣搜索的CSV 文件。</p>
        <p><strong>2. 词云功能: </strong>下载词云图，该图会根据当前爬取的数据和用户上传的词云图片生成。</p>
        <p><strong>3. 隐私说明: </strong>本页面数据属于用户隐私，注意数据保护。因此暂时不开放上传功能</p>
    </div>

    <div id="result"></div>

    <div class="form-group">
        <input type="text" id="userInput" placeholder="输入需要查找电视剧/电影的名字（待按钮重新亮起后下载查找csv）">
        <button onclick="handleExecute()" id="executeButton">查找</button>
        <button onclick="handleDownloadCsv1()">下载查找的CSV</button>
    </div>

    <div class="form-group">
    <input type="text" id="userInput2" placeholder="输入需要爬取的内容">
    <button onclick="handleExecute2()" id="executeButton2">爬取</button>
    </div>
    
    <div class="form-group">
    <button onclick="handleDownloadCommentDuanCsv()">下载评论短评CSV</button>
    </div>

    <div class="form-group">
        <button onclick="handleDownloadWordCloud()" id="wordcloudButton">下载词云图【暂未开放上传图片】</button>
    </div>

</div>

<script>
    // 切换使用说明显示/隐藏
    function toggleUsageInstructions() {
        const instructions = document.getElementById('usageInstructions');
        instructions.style.display = instructions.style.display === 'none' || instructions.style.display === '' ? 'block' : 'none';
    }

    async function handleExecute() {
        if (localStorage.getItem('isExecutingExecute') === 'true') return;
        localStorage.setItem('isExecutingExecute', 'true');
        updateButtonState('button[onclick="handleExecute()"]', true);

        const userInput = document.getElementById('userInput').value;
        if (!userInput) {
            alert("内容不能为空");
            localStorage.setItem('isExecutingExecute', 'false');
            updateButtonState('button[onclick="handleExecute()"]', false);
            return;
        }

        try {
            const response = await fetch('/execute', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({userInput})
            });
            const result = await response.json();
            document.getElementById('result').innerText = result.message || result.output;
        } finally {
            localStorage.setItem('isExecutingExecute', 'false');
            updateButtonState('button[onclick="handleExecute()"]', false);
        }
    }
    
    async function handleExecute2() {
    if (localStorage.getItem('isExecutingExecute2') === 'true') return;
    localStorage.setItem('isExecutingExecute2', 'true');
    updateButtonState('button[onclick="handleExecute2()"]', true);

    const userInput = document.getElementById('userInput2').value;
    if (!userInput) {
        alert("内容不能为空");
        localStorage.setItem('isExecutingExecute2', 'false');
        updateButtonState('button[onclick="handleExecute2()"]', false);
        return;
    }

    try {
        const response = await fetch('/execute2', {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({userInput})
        });
        const result = await response.json();
        document.getElementById('result').innerText = result.message || result.output;
    } finally {
        localStorage.setItem('isExecutingExecute2', 'false');
        updateButtonState('button[onclick="handleExecute2()"]', false);
    }
}
    

    function handleDownloadCsv1() {
        window.location.href = '/download1_csv';
    }
    
    function handleDownloadCommentDuanCsv() {
        window.location.href = '/download_comment_duan_csv';
    }

    function handleDownloadWordCloud() {
        window.location.href = '/download_wordcloud';
    }

    function updateButtonState(buttonSelector, disable) {
        const button = document.querySelector(buttonSelector);
        if (disable) {
            button.disabled = true;
        } else {
            button.disabled = false;
        }
    }
</script>
</body>
</html>
''')

from flask import Flask, request, jsonify, render_template_string, send_file
import os
import subprocess

app = Flask(__name__)

# 获取最新的词云图文件（在 ci_yun_tu 文件夹下）
def get_latest_wordcloud_file():
    wordcloud_folder = os.path.join(os.getcwd(), 'ci_yun_tu')  # 当前目录下的 ci_yun_tu 文件夹
    if not os.path.exists(wordcloud_folder):
        return None
    wordcloud_files = [f for f in os.listdir(wordcloud_folder) if f.endswith('.png')]  # 假设词云图是 .png 格式
    if not wordcloud_files:
        return None
    wordcloud_files.sort(key=lambda f: os.path.getmtime(os.path.join(wordcloud_folder, f)), reverse=True)
    return os.path.join(wordcloud_folder, wordcloud_files[0])

@app.route('/download1_csv')
def download1_csv():
    file_path = '2guolv.csv'  # 查找结果的CSV
    if os.path.exists(file_path):
        return send_file(file_path, as_attachment=True)
    else:
        return jsonify({"status": "error", "message": "未找到查找CSV文件"}), 404
    
@app.route('/download_comment_duan_csv')
def download_comment_duan_csv():
    file_path = '3_comment_duan.csv'  # 需要下载的 CSV 文件路径
    if os.path.exists(file_path):
        return send_file(file_path, as_attachment=True)
    else:
        return jsonify({"status": "error", "message": "未找到 comment duan CSV 文件"}), 404

@app.route('/download_wordcloud')
def download_wordcloud():
    latest_wordcloud_file = get_latest_wordcloud_file()
    if latest_wordcloud_file is None:
        return jsonify({"status": "error", "message": "没有找到词云图文件"}), 404
    return send_file(latest_wordcloud_file, as_attachment=True)

@app.route('/execute', methods=['POST'])
def execute():
    user_input = request.json.get('userInput')
    if not user_input:
        return jsonify({"status": "error", "message": "请输入内容后再进行查找。"}), 400
    try:
        with open('0_yonghu_select.txt', 'w', encoding='utf-8') as f:
            f.write(user_input)
        result = subprocess.run(['python', '0_start.py'], capture_output=True, text=True)
        if result.returncode == 0:
            return jsonify({"status": "success", "output": result.stdout})
        else:
            return jsonify({"status": "error", "message": result.stderr}), 500
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500
    
    
@app.route('/execute2', methods=['POST'])
def execute2():
    user_input = request.json.get('userInput')
    if not user_input:
        return jsonify({"status": "error", "message": "请输入内容后再进行查找。"}), 400
    try:
        # 将用户输入写入 3id.txt
        with open('3id.txt', 'w', encoding='utf-8') as f:
            f.write(user_input)
        
        # 执行 0_0start.py 脚本
        result = subprocess.run(['python', '0_0start.py'], capture_output=True, text=True)
        if result.returncode == 0:
            return jsonify({"status": "success", "output": result.stdout})
        else:
            return jsonify({"status": "error", "message": result.stderr}), 500
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500


@app.route('/')
def home():
    return render_template_string('''
<html>
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1">
<style>
    body {
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen;
        margin: 0;
        padding: 0;
        background-color: #f4f4f4;
    }

    .container {
        max-width: 600px;
        margin: 30px auto;
        padding: 20px;
        background: white;
        border-radius: 10px;
        box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }

    .form-group {
        margin-bottom: 10px;
    }

    input[type="text"] {
        width: 100%;
        padding: 12px;
        font-size: 16px;
        margin-bottom: 10px;
        border: 1px solid #ccc;
        border-radius: 6px;
        box-sizing: border-box;
    }

    button {
        width: 100%;
        padding: 12px;
        font-size: 16px;
        border: none;
        border-radius: 6px;
        background-color: #28a745;
        color: white;
        cursor: pointer;
        margin-bottom: 5px;
    }

    button:hover {
        background-color: #218838;
    }

    button:disabled {
        background-color: #ccc;
        cursor: not-allowed;
    }

    #result {
        margin-top: 20px;
        padding: 15px;
        background-color: #e9f7ef;
        border-bottom: 5px solid #005f40;
        border-radius: 6px;
        white-space: pre-wrap;
        word-break: break-word;
    }

    table {
        width: 100%;
        border-collapse: collapse;
        margin-top: 15px;
        font-size: 14px;
    }

    th, td {
        border: 1px solid #ddd;
        padding: 8px;
    }

    th {
        background-color: #4CAF50;
        color: white;
    }

    #usageInstructions {
        background-color: #f8f9fa;
        padding: 10px;
        border-radius: 5px;
        margin-top: 20px;
        display: none;
    }
</style>

</head>
<body>
<div class="container">
    <!-- 使用说明按钮 -->
    <button onclick="toggleUsageInstructions()">查看使用说明</button>

    <div id="usageInstructions">
        <h3>使用说明</h3>
        <p><strong>0. 页面是干什么的: </strong>针对豆瓣影视短评进行爬取并选择性生成词云遮罩，当前页面主要是用于查找电影或电视剧相关信息并生成词云。</p>
        <p><strong>1. 查找功能：</strong>输入电影或电视剧名称，点击“查找”按钮，在等待一段时间后，返回豆瓣搜索的CSV 文件。</p>
        <p><strong>2. 词云功能: </strong>下载词云图，该图会根据当前爬取的数据和用户上传的词云图片生成。</p>
        <p><strong>3. 隐私说明: </strong>本页面数据属于用户隐私，注意数据保护。因此暂时不开放上传功能</p>
    </div>

    <div id="result"></div>

    <div class="form-group">
        <input type="text" id="userInput" placeholder="输入需要查找电视剧/电影的名字（待按钮重新亮起后下载查找csv）">
        <button onclick="handleExecute()" id="executeButton">查找</button>
        <button onclick="handleDownloadCsv1()">下载查找的CSV</button>
    </div>

    <div class="form-group">
    <input type="text" id="userInput2" placeholder="输入需要爬取的内容">
    <button onclick="handleExecute2()" id="executeButton2">爬取</button>
    </div>
    
    <div class="form-group">
    <button onclick="handleDownloadCommentDuanCsv()">下载评论短评CSV</button>
    </div>

    <div class="form-group">
        <button onclick="handleDownloadWordCloud()" id="wordcloudButton">下载词云图【暂未开放上传图片】</button>
    </div>

</div>

<script>
    // 切换使用说明显示/隐藏
    function toggleUsageInstructions() {
        const instructions = document.getElementById('usageInstructions');
        instructions.style.display = instructions.style.display === 'none' || instructions.style.display === '' ? 'block' : 'none';
    }

    async function handleExecute() {
        if (localStorage.getItem('isExecutingExecute') === 'true') return;
        localStorage.setItem('isExecutingExecute', 'true');
        updateButtonState('button[onclick="handleExecute()"]', true);

        const userInput = document.getElementById('userInput').value;
        if (!userInput) {
            alert("内容不能为空");
            localStorage.setItem('isExecutingExecute', 'false');
            updateButtonState('button[onclick="handleExecute()"]', false);
            return;
        }

        try {
            const response = await fetch('/execute', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({userInput})
            });
            const result = await response.json();
            document.getElementById('result').innerText = result.message || result.output;
        } finally {
            localStorage.setItem('isExecutingExecute', 'false');
            updateButtonState('button[onclick="handleExecute()"]', false);
        }
    }
    
    async function handleExecute2() {
    if (localStorage.getItem('isExecutingExecute2') === 'true') return;
    localStorage.setItem('isExecutingExecute2', 'true');
    updateButtonState('button[onclick="handleExecute2()"]', true);

    const userInput = document.getElementById('userInput2').value;
    if (!userInput) {
        alert("内容不能为空");
        localStorage.setItem('isExecutingExecute2', 'false');
        updateButtonState('button[onclick="handleExecute2()"]', false);
        return;
    }

    try {
        const response = await fetch('/execute2', {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({userInput})
        });
        const result = await response.json();
        document.getElementById('result').innerText = result.message || result.output;
    } finally {
        localStorage.setItem('isExecutingExecute2', 'false');
        updateButtonState('button[onclick="handleExecute2()"]', false);
    }
}
    

    function handleDownloadCsv1() {
        window.location.href = '/download1_csv';
    }
    
    function handleDownloadCommentDuanCsv() {
        window.location.href = '/download_comment_duan_csv';
    }

    function handleDownloadWordCloud() {
        window.location.href = '/download_wordcloud';
    }

    function updateButtonState(buttonSelector, disable) {
        const button = document.querySelector(buttonSelector);
        if (disable) {
            button.disabled = true;
        } else {
            button.disabled = false;
        }
    }
</script>
</body>
</html>
''')

if __name__ == '__main__':
    app.run(debug=True, host='::', port=4002)



```

## 0_start.py
用来对web按钮执行操作的，这个是符合多个py来执行的 其中包括了3个子脚本
```
import subprocess

def execute_script(script_path):
    """
    使用Python解释器执行指定路径的脚本，并尝试以不同编码解码输出。
    
    参数:
        script_path (str): 要执行的Python脚本的文件路径。
    """
    result = subprocess.run(['python', script_path], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    
    def safe_decode(data):
        try:
            return data.decode('utf-8')
        except UnicodeDecodeError:
            try:
                return data.decode('gbk')
            except UnicodeDecodeError:
                return data.decode('utf-8', errors='ignore')
    
    if result.returncode == 0:
        print(f"执行成功,点击下载查找的 csv可获取数据")
    else:
        print(f"执行失败：\n{safe_decode(result.stderr)}")

if __name__ == "__main__":
    # 将这里的路径替换为你的目标脚本路径
    scripts_to_run = [
        '0_saved_page.py',
        '1_guolv.py',
        '2_guolv.py'
    ]
    
    for script in scripts_to_run:
        execute_script(script)
```
## 0_0start.py
针对第二个按钮进行，其中包括了两个脚本
```
import subprocess

def execute_script(script_path):
    """
    使用Python解释器执行指定路径的脚本，并尝试以不同编码解码输出。
    
    参数:
        script_path (str): 要执行的Python脚本的文件路径。
    """
    result = subprocess.run(['python', script_path], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    
    def safe_decode(data):
        try:
            return data.decode('utf-8')
        except UnicodeDecodeError:
            try:
                return data.decode('gbk')
            except UnicodeDecodeError:
                return data.decode('utf-8', errors='ignore')
    
    if result.returncode == 0:
        print(f"执行成功,点击下载爬取的 csv可获取数据")
    else:
        print(f"执行失败：\n{safe_decode(result.stderr)}")

if __name__ == "__main__":
    # 将这里的路径替换为你的目标脚本路径
    scripts_to_run = [
        '3comment_duan.py',
        'Z_ciyun_start.py',
    ]
    
    for script in scripts_to_run:
        execute_script(script)
```
## Z_ciyun_start.py
词云处理的 处理下面的两个词云子脚本
```
import subprocess

def execute_script(script_path):
    """
    使用Python解释器执行指定路径的脚本，并尝试以不同编码解码输出。
    
    参数:
        script_path (str): 要执行的Python脚本的文件路径。
    """
    result = subprocess.run(['python', script_path], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    
    def safe_decode(data):
        try:
            return data.decode('utf-8')
        except UnicodeDecodeError:
            try:
                return data.decode('gbk')
            except UnicodeDecodeError:
                return data.decode('utf-8', errors='ignore')
    
    if result.returncode == 0:
        print(f"执行成功")
    else:
        print(f"执行失败：\n{safe_decode(result.stderr)}")

if __name__ == "__main__":
    # 将这里的路径替换为你的目标脚本路径
    scripts_to_run = [
        'Z_ciyun_zhuan.py',
        'Z_ciyun_shengcheng.py',
    ]
    
    for script in scripts_to_run:
        execute_script(script)
```

## Z_ciyun_zhuan.py
```
import csv

# 定义输出文件名
position_output_file = 'Z_weizhi.txt'
comment_content_output_file = 'Z_pinglun_neirong.txt'

# 打开CSV文件进行读取
with open('3_comment_duan.csv', mode='r', encoding='utf-8') as file:
    reader = csv.reader(file)
    
    # 打开输出文件准备写入
    with open(position_output_file, mode='w', encoding='utf-8') as pos_file, \
         open(comment_content_output_file, mode='w', encoding='utf-8') as comment_file:
        
        # 遍历每一行数据
        for row in reader:
            if len(row) > 3:  # 确保该行至少有四列数据
                # 写入位置信息（第三列，索引为2）到Z_weizhi.txt
                pos_file.write(row[1] + '\n')
                # 写入评论内容（第四列，索引为3）到Z_pinglun_neirong.txt
                comment_file.write(row[2] + '\n')

print("处理完成，文件已生成。")
```

## Z_ciyun_shengcheng.py
这个是我比较喜欢的版本（保留原图大小 + 背景扩展模糊）
```
from wordcloud import WordCloud, STOPWORDS
from PIL import Image, ImageFilter
import numpy as np
from datetime import datetime
import os

def read_file(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        return file.read()

def generate_wordcloud(text, image_path):
    # 打开原始图像
    original_rgba = Image.open(image_path).convert("RGBA")
    w, h = original_rgba.size  # 保持输出为这个尺寸

    # 1. 扩大原图并模糊作为背景源
    scale = 3  # 放大3倍再模糊，以便裁剪中央部分
    large_bg = original_rgba.resize((w * scale, h * scale), Image.LANCZOS)
    blurred_large_bg = large_bg.filter(ImageFilter.GaussianBlur(radius=60)).convert("RGB")

    # 2. 裁剪中心区域作为最终背景（尺寸保持 w x h）
    left = (w * scale - w) // 2
    upper = (h * scale - h) // 2
    cropped_bg = blurred_large_bg.crop((left, upper, left + w, upper + h)).convert("RGBA")

    # 3. 生成词云遮罩（来自 alpha 通道）
    mask_array = np.array(original_rgba)
    alpha_channel = mask_array[:, :, 3]
    mask = np.where(alpha_channel > 0, 255, 0)

    # 4. 设置停用词
    stopwords = set(STOPWORDS)
    stopwords.add("一些")

    # 5. 生成词云图像
    font_path = 'SmileySans-Oblique.ttf'
    wordcloud = WordCloud(
        font_path=font_path,
        width=w, height=h,
        background_color=None,
        mode="RGBA",
        mask=mask,
        stopwords=stopwords,
        max_font_size=150,
        contour_width=0,
        colormap='tab20'
    ).generate(text)
    wordcloud_rgba = wordcloud.to_image().convert("RGBA")

    # 6. 叠加原图和词云到背景图
    final_image = cropped_bg.copy()
    final_image.alpha_composite(original_rgba)
    final_image.alpha_composite(wordcloud_rgba)

    # 7. 转为RGB并保存
    final_rgb = final_image.convert("RGB")
    current_time = datetime.now().strftime("%Y%m%d_%H%M")
    save_dir = "ci_yun_tu"
    os.makedirs(save_dir, exist_ok=True)
    save_path = os.path.join(save_dir, f"ci_yun_tu_{current_time}.png")
    final_rgb.save(save_path, format="PNG")
    print(f"词云图已保存至: {save_path}")

if __name__ == '__main__':
    file_path = 'Z_pinglun_neirong.txt'
    image_path = 'Z_ciyun_daoru.png'
    text = read_file(file_path)
    if not text.strip():
        print("提供的文件为空或者没有可读取的内容。")
    else:
        generate_wordcloud(text, image_path)
```
## Z_ciyun_shengcheng_no_mohu.py
```
from wordcloud import WordCloud, STOPWORDS
import matplotlib.pyplot as plt
from PIL import Image, ImageFilter
import numpy as np
from datetime import datetime
import os

# 读取文本文件内容
def read_file(file_path):
    with open(file_path, 'r', encoding='utf-8') as file:
        text = file.read()
    return text

# 生成词云并叠加到图像上然后保存
def generate_wordcloud(text, image_path):
    # 打开原始图像，确保有RGBA通道
    original_image = Image.open(image_path).convert("RGBA")

    # 提取图像形状并构造词云遮罩
    mask_image = np.array(original_image)
    alpha_channel = mask_image[:, :, 3]
    mask = np.where(alpha_channel > 0, 255, 0)

    # 设置字体路径
    font_path = 'SmileySans-Oblique.ttf'  # 替换为你本地支持中文的字体路径

    # 设置停用词
    stopwords = set(STOPWORDS)
    stopwords.add("一些")  # 可添加更多停用词

    # 生成词云对象
    wordcloud = WordCloud(
        font_path=font_path,
        width=original_image.width,
        height=original_image.height,
        background_color=None,
        mode="RGBA",
        mask=mask,
        stopwords=stopwords,
        max_font_size=150,
        contour_width=0,
        colormap='tab20'
    ).generate(text)

    # 转换为图像格式
    wordcloud_image = wordcloud.to_image().convert("RGBA")

    # 第一个图层：高斯模糊背景
    blurred_background = original_image.copy().filter(ImageFilter.GaussianBlur(radius=30))

    # 第二个图层：原图（保留主图形状和内容）
    combined = Image.alpha_composite(blurred_background, original_image)

    # 第三个图层：词云图层（透明背景）
    final = Image.alpha_composite(combined, wordcloud_image)

    # 构建保存路径
    current_time = datetime.now().strftime("%Y%m%d_%H%M")
    save_dir = "ci_yun_tu"
    os.makedirs(save_dir, exist_ok=True)
    save_path = os.path.join(save_dir, f"ci_yun_tu_{current_time}.png")

    # 保存最终图像
    final.save(save_path, format="PNG")
    print(f"词云图已保存至: {save_path}")

# 主程序入口
if __name__ == '__main__':
    file_path = 'Z_pinglun_neirong.txt'  # 文本文件路径
    image_path = 'Z_ciyun_daoru.png'     # 带透明背景的图像路径

    # 读取文本
    text = read_file(file_path)

    if not text.strip():
        print("提供的文件为空或者没有可读取的内容。")
    else:
        generate_wordcloud(text, image_path)
```


# 成品
不建议数据过多，那样的话一堆小字，挺丑的
![|325](images/Pasted%20image%2020250413003659.png)
![|300](images/Pasted%20image%2020250413003707.png)
![|300](images/Pasted%20image%2020250413003720.png)
这个明显数据太多了
![|293](images/Pasted%20image%2020250413004030.png)
![|368](images/Pasted%20image%2020250413004213.png)
我觉得小舞这一张带alpha的更好看
![|368](images/Pasted%20image%2020250413004245.png)

