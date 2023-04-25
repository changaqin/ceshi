import asyncio
import aiohttp
import re
import collections
import validators
import logging
import os
import concurrent.futures
from collections import defaultdict
class WebCrawler:
    def __init__(self, urls, keywords, headers, cache):
        self.urls = urls
        self.keywords = keywords
        self.headers = headers
        self.cache = cache
        self.status_code_dict = defaultdict(list)
    async def fetch(self, session, url):
        try:
            async with session.head(url, headers=self.headers, timeout=10) as head_response:
                status_code = head_response.status
                if status_code == 200:
                    if url in self.cache:
                        matched_keywords = self.cache[url]
                    else:
                        async with session.get(url, headers=self.headers) as get_response:
                            content = await get_response.text()
                            matched_keywords = ""
                            for keyword in self.keywords:
                                if keyword in content:
                                    matched_keywords += keyword + ","
                            if matched_keywords:
                                matched_keywords = matched_keywords.rstrip(",")
                                matched_keywords = "({})".format(matched_keywords)
                            self.cache[url] = matched_keywords
                    self.status_code_dict[status_code].append((url, matched_keywords))
                else:
                    logging.error("The status code of {} is {}".format(url, status_code))
                    return
        except (aiohttp.ClientError, asyncio.TimeoutError, ValueError) as e:
            logging.error(str(e))
            return
    async def worker(self):
        async with aiohttp.ClientSession(headers=self.headers) as session:
            tasks = []
            for url in self.urls:
                if validators.url(url):
                    tasks.append(asyncio.ensure_future(self.fetch(session, url)))
            await asyncio.gather(*tasks)
    def run(self):
        asyncio.run(self.worker())
if __name__ == '__main__':
    with open("text.txt", "r") as f:
        lines = f.readlines()
    KEYWORDS = ["电子书", "下载", "百度网盘"]
    HEADERS = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36'}
    urls = [re.search(r'([a-zA-z]+://[^\s]*)', line).group(1) for line in lines if re.search(r'([a-zA-z]+://[^\s]*)', line)]
    cache = {}
    web_crawler = WebCrawler(urls, KEYWORDS, HEADERS, cache)
    num_threads = os.cpu_count() * 2
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_threads) as executor:
        loop = asyncio.get_event_loop()
        loop.set_default_executor(executor)
        web_crawler.run()
    logging.basicConfig(filename='web_crawler.log', level=logging.ERROR, format='%(asctime)s:%(levelname)s:%(message)s')
    print("按照不同的状态码分类的网址有：")
    for status_code in sorted(web_crawler.status_code_dict.keys()):
        file_name = str(status_code) + ".txt"
        with open(file_name, "w") as f:
            for url, matched_keywords in sorted(web_crawler.status_code_dict[status_code], key=lambda x: x[0]):
                f.write(url + " " + matched_keywords + "\n")
                print("{:<60} {:<10} {:<20}".format(url, status_code, matched_keywords))
