---
title: 使用 Python Requests 爬取并下载小说 <br/> 以 耳根 的小说 《一念永恒》 为例
date: 2018-11-22 14:04:49
updated: 2018-11-23 14:04:49
categories:
    Python
tags:
    - python
---

![一念永恒](/images/python-download-novel/一念永恒_耳根.png)

# 使用 Python Requests 爬取 [笔趣看](http://www.biqukan.com/) 小说网站

<!-- more -->

## 使用工具：
> * Python 3.6 +
> * Requests 2.20 +

## 需要爬取的小说主站地址：
> * http://www.biqukan.com/1_1094


# 页面分析

## 1. 分析列表页面布局

使用浏览器审查元素，查找章节列表所在位置的 dom、class、id 等属性，可以看到所有的章节列表都在一个 class 为 "listmain" 的 div 中包含着，由此可以得出结论，该 div 即是我们需要定位的章节列表所在位置。

![章节列表](/images/python-download-novel/list.png)

## 2、分析章节名称、链接、第一章起始位置
根据列表布局，可以看到，每个章节的章节名称、链接，都在 a 标签中，而小说的第一章，是在第 16 个 a 标签中。分析章节链接、名称、第一章起始位置

## 3、分析章节内容页

进入第一章节，利用浏览器审查元素，可以看到文章的内容都在一个 id 为 content，class 为 showtxt 的 div 中，由此我们可以获取到章节内容

![章节内容](/images/python-download-novel/content.png)

# 代码实例

## 创建下载类，声明存放章节名称、章节链接、章节数的数组

```python
class download(object):
    def __init__(self):
        self.server = 'https://www.biqukan.com'
        self.target = self.server + '/1_1094'
        # 存放章节名称
        self.names = []
        # 存放章节链接
        self.urls = []
        # 存放章节数
        self.nums = 0
```

## 在下载类中新建获取章节列表的方法

```python
    # 获取下载链接
    def get_download_url(self):
        # verify 绕过 https，self.target 即为：https://www.biqukan.com/1_1094
        req = requests.get(url=self.target, verify=False)
        # 获取请求到的文本内容
        html = req.text
        # 将文本内容转为流
        div_bf = BeautifulSoup(html)
        # 获取所有 class 为 listmain 的 div
        div = div_bf.find_all('div', class_='listmain')
        # 取第一个 div 转为流
        a_bf = BeautifulSoup(str(div[0]))
        # 获取所有 a 标签
        a = a_bf.find_all('a')
        # 由第二步得知，第一章节为第 16 个 a 标签，所以去除不必要的前 15 个章节
        self.nums = len(a[15:])
        # 转换 a 标签内容
        for href in a[15:]:
            # 获取 a 标签中的文本，放入章节名称数组
            self.names.append(href.string)
            # 获取 a 标签中的列表，放入章节链接数组（由于链接没有域名，需要拼接域名）
            self.urls.append(self.server + href.get('href'))
```

## 获取章节内容

```python
    # 获取章节内容，target 为章节内容链接
    def get_content(self, target):
        # verify 绕过 https
        req = requests.get(url=target, verify=False)
        # 获取请求到的内容
        html = req.text
        bf = BeautifulSoup(html)
        # 获取 class 为 showtxt 的 div 的内容（即小说章节内容）
        texts = bf.find_all('div', class_='showtxt')
        # 将 &nbsp; 转换为空字符串
        texts = texts[0].text.replace('\xa0' * 8, '')
        # 压缩一下，把两个换行转换为一个换行
        texts = texts.replace('\n\n','\n')
        return texts
```

## 将内容写入到 txt 文件中

```python
    """
        name：文章名称
        path：文章保存路径
        text：文章内容
    """
    def write(self, name, path, text):
        write_flag = True
        with open(path, 'a', encoding='utf-8') as f:
            f.write(name + '\n')
            f.writelines(text)
            f.write('\n')
```

## 开始下载

```python
if __name__ == '__main__':

    """
        压制由于忽略 HTTPS，导致的运行时警告
        from requests.packages.urllib3.exceptions import InsecureRequestWarning
        requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
    """
    from requests.packages.urllib3.exceptions import InsecureRequestWarning
    requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

    dl = download()
    dl.get_download_url()
    print('开始下载')
    for i in range(dl.nums):
        dl.write(dl.names[i], '一念永恒.txt', dl.get_content(dl.urls[i]))
        # 计算下载百分比
        sys.stdout.write('正在下载 %s，已下载：%.3f%% \r' % (dl.names[i], float(i / dl.nums)))
        # 刷新打印消息
        sys.stdout.flush()
    print('下载完成')
```