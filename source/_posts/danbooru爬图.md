---
title: danbooru爬图
date: 2023/08/13
categories: 
- ai
tags:
- python
---

#### 简介
* 最近ai炼丹嫌找炼丹素材麻烦, 就写了个按`danbooru tag`爬图的脚本, 没什么第三方依赖, 想用直接copy去跑就行

#### 使用方法
1. 把脚本里面的`your_username`和`your_password`替换成你的danbooru账号密码(账号在https://danbooru.donmai.us注册)
2. main里面, 在`tag`填上要爬取的tag, `save_dir`为爬取的图片的保存路径
3. 如果需要通过代理爬取, 把`use_proxy`置为`True`, 然后改下`proxies`变量的代理地址即可


#### 脚本
```python
# -*- coding: utf-8 -*-
import os
import urllib.parse
import requests
from requests.auth import HTTPBasicAuth

use_proxy = True

headers = {
    'User-Agent': 'db/v1.0.0',
    'Content-Type': 'application/json; charset=utf-8',
}

proxies = {"http": "socks5://127.0.0.1:1080", 'https': 'socks5://127.0.0.1:1080'}

auth = HTTPBasicAuth('your_username', 'your_password')

session = requests.session()


def save_file(file_url, path):
    response = request_get(file_url)
    with open(path, 'wb') as f:
        f.write(response.content)
        f.flush()


def request_get(url):
    if use_proxy:
        return session.get(url, headers=headers, proxies=proxies, auth=auth)
    else:
        return session.get(url, headers=headers, auth=auth)


def get_all_danbooru_items(tag_list):
    """
    :param tag_list:
    :return: list of danbooru item
    """
    items = []
    cur_page = 1
    while True:
        page_items = get_danbooru_items(tag_list, cur_page)
        if len(page_items) == 0:
            break
        items.extend(page_items)
        cur_page += 1
    return items


def get_danbooru_items(tag_list, page=1, limit=100):
    """
    API文档 https://danbooru.donmai.us/wiki_pages/api%3Aposts
    :param tag_list:
    :param page: 从1开始
    :param limit:
    :return: list of danbooru item
    """
    clean_tag_list = list(map(lambda x: x.replace(' ', '_'), tag_list))
    query = {
        'tags': ' '.join(clean_tag_list),
        'page': page,
        'limit': limit,
    }
    posts_url = 'https://danbooru.donmai.us/posts.json?%s' % (urllib.parse.urlencode(query))
    print(posts_url)
    resp = request_get(posts_url)
    return resp.json()


def count_danbooru_items(tag_list):
    """
    :param tag_list:
    :return: int
    """
    clean_tag_list = list(map(lambda x: x.replace(' ', '_'), tag_list))
    query = {
        'tags': ' '.join(clean_tag_list),
    }
    posts_url = 'https://danbooru.donmai.us/counts/posts.json?%s' % (urllib.parse.urlencode(query))
    print(posts_url)
    resp = request_get(posts_url)
    return resp.json()['counts']['posts']


def download_imgs_by_tags(tags, save_dir):
    items = get_all_danbooru_items(tags)
    sub_folder_name = '_'.join(tags)
    folder_path = os.path.join(save_dir, sub_folder_name)
    if not os.path.exists(folder_path):
        os.mkdir(folder_path)
    total = len(items)
    cnt = 0
    print("downloading", sub_folder_name, 'total:', total)
    for item in items:
        img_id, img_url = item['id'], item.get('file_url')
        if img_url is None:
            print('emtpy file_url:', item)
            continue
        img_suffix = img_url[img_url.rindex('.'):]
        img_file_name = '%d%s' % (img_id, img_suffix)
        img_file_path = os.path.join(folder_path, img_file_name)
        if os.path.exists(img_file_path):
            print(img_file_name, 'exists! pass...')
            continue
        save_file(img_url, img_file_path)
        cnt += 1
        print('(%d/%d)%s download ok!' % (cnt, total, img_file_name))
    print('all done!')


# https://danbooru.donmai.us/
if __name__ == '__main__':
    use_proxy = False
    save_dir = './danbooru_imgs'
    tag = 'mari_(blue_archive)'
    cnt = count_danbooru_items([tag])
    print(cnt)
    if cnt > 0:
        download_imgs_by_tags([tag], save_dir)
```