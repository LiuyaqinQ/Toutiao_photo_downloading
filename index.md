#python -3.5
import  requests
from urllib.parse import urlencode
from requests.exceptions import RequestException
import json
from bs4 import BeautifulSoup
import re
import pymongo
import os
from hashlib import md5
import time
from multiprocessing import Pool
MONGO_URL='localhost'
MONGO_DB='toutiao'
MONGO_TABLE='meinv'
#设定pymongo存储
client=pymongo.MongoClient(MONGO_URL,27017)
db=client[MONGO_DB]
def save_to_mongo(result):
    if db[MONGO_TABLE].insert_one(result):
        print('成功存储到pymongo',result)
        return True
    else:
        return False

#对XHR类型url进行网页解析，构造urlencode，采用requests.get方式
def get_page_index(offset,keyword):
    data={
        'offset':offset,
        'format':'json',
        'keyword':keyword,
        'autoload':'true',
        'count':20,
        'cur_tab':1
    }
    url='http://www.toutiao.com/search_content/?'+urlencode(data)
    print (url)
    try:
        response=requests.get(url)
        if response.status_code==200:
            return response.text
        return None
    except RequestException:
        print('请求索引页失败')
        return None

#从解析后的XHR网页的json进行解析，获取图册url地址
def parse_page_index(html):
    data=json.loads(html)
    if 'data' in data.keys() and data:
        for item in data.get('data'):
            yield item.get('article_url')
#对每个图册进行网页解析，其实可以符合成一个函数进行调用——
def get_page_detail(url):
    try:
        response = requests.get(url)
        if response.status_code == 200:
            return response.text
        return None
    except RequestException:
        print('请求索引页失败',url)
        return None
#对图册的解析后的页面信息获得图片链接
def parse_page_detail(html,url):
    #如果解析页面非空则进行分析处理
    if html:
        soup=BeautifulSoup(html,'lxml')
        #获得图片标题
        title=soup.select('title')[0].get_text()
        #采用正则方式，获取图片url所在区域
        image_pattern = re.compile(r'articleInfo: (.*?)}', re.S)
        result = re.search(image_pattern, html)
        #将图片区域分解为列表形式进行存储
        image_pattern1 = re.compile(r'src=&quot;(.*?)&quot; img_width', re.S)
        if result:
            result_phtotos = re.findall(image_pattern1, result.group(0))
            data={
                'title':title,
                'images':result_phtotos,
                'url':url
            }
            #如果存在图片，则保存到当地目录
            if result_phtotos:
                for url in result_phtotos:
                    download_image(url,title)
            return data
        else:
            return False
    else:
        return False
#解析图片ulr，返回conten，并存储
def download_image(url,title):
    try:
        response = requests.get(url)
        if response.status_code == 200:
            save_image(response.content,title)
            print('saved '+url)
        return None
    except RequestException:
        print('请求索引页失败',url)
        return None
#图片存储名称和图片写入
def save_image(content,title):
    file_path='{0}/{1}{2}.{3}'.format(os.getcwd(),title,md5(content).hexdigest(),'jpg')
    # print(file_path)
    if not os.path.exists(file_path):
        with open(file_path,'wb') as f:
            f.write(content)

#主函数，根据offset和keyword进行扫描搜索，存储和下载。
def main(offset=0):
    html=get_page_index(offset,title)
    # print(html)
    for url in parse_page_index(html):
        time.sleep(1)
        print(url)
        if url:
            html=get_page_detail(url)
            if html:
                try:
                    result=parse_page_detail(html,url)
                    save_to_mongo(result)
                except:
                    pass


def mkdir( path):
    path = path.strip()
    # 判断路径是否存在
    # 存在     True
    # 不存在   False
    isExists = os.path.exists(path)
    # 判断结果
    if not isExists:
        # 如果不存在则创建目录
        print (u"新建了名字叫做", path, u'的文件夹')
        # 创建目录操作函数
        os.makedirs(path)
        return True
    else:
        # 如果目录存在则不创建，并提示目录已存在
        print (u"名为", path, '的文件夹已经创建成功')
        return False

title='韩国女主播'
if __name__=='__main__':
    currentpath = os.getcwd()
    mkdir(title)
    os.chdir(title)
    col=[i*20 for i in range(0,500)]
    print (col)
    #简单的多线程处理方式
    pool=Pool()
    pool.map(main,col)
    os.chdir(currentpath)

# title='韩国美女'
# currentpath = os.getcwd()
# mkdir(title)
# os.chdir(title)
# col=[{i*20,title} for i in range(0,1000)]
# print (col)
# #简单的多线程处理方式
# pool=Pool()
# pool.map(main,col)
# os.chdir(currentpath)


# if __name__ == '__main__':
#     url='http://toutiao.com/group/6470629259114185230/'
#     html=get_page_detail(url)
#     result=parse_page_detail(html,url)
#     print(result)

