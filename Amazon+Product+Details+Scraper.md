
### This script scrapes the product name, price, and rating for all items of a given brand available on Amazon. In this example we are scraping Brawny products. 


```python
import certifi
import urllib3
http = urllib3.PoolManager(
cert_reqs='CERT_REQUIRED',
ca_certs=certifi.where())
```


```python
from lxml import html  
from lxml.html import fromstring
import requests
from itertools import cycle
import re
from time import sleep
import pandas as pd
import math
from bs4 import BeautifulSoup
import numpy
from urllib.request import Request, urlopen
from urllib.error import URLError
import ssl
from fake_useragent import UserAgent
import csv
import random
from random import shuffle
import cfscrape
```

A common roadblock to large-scale web scraping is getting blocked. A website can block your IP address if it can tell you are a single 'bot' hitting the site over and over again in a small amount of time. One way to avoid getting blocked is to make it look like your requests are coming from different browsers. We accomplish this by using a different user agent in the header of each request. 


```python
ua = UserAgent()
```

Rotating through a list of proxies is also an option to avoid getting blocked. However, proxies aren't foolproof and many of the free proxies won't be recognized, which will raise a connection error. So free proxies (you can pay for real proxies) are only marginally useful compared to rotating user agents, which get us most of the way there. 


```python
def get_proxies():
    url = 'https://free-proxy-list.net/'
    response = requests.get(url)
    parser = fromstring(response.text)
    proxies = set()
    for i in parser.xpath('//tbody/tr')[:100]:
        if i.xpath('.//td[7][contains(text(),"yes")]'):
            proxy = ":".join([i.xpath('.//td[1]/text()')[0], i.xpath('.//td[2]/text()')[0]])
            proxies.add(proxy)
    return proxies

proxies = get_proxies()
proxy_pool = cycle(proxies)
```

Since we will be scraping all of the pages associated with a particular brand, and the number of pages and products will vary among brands, we first need to find out the number of pages we will be scraping. For this exercise we will be scraping all Brawny product data. We specify Brawny by using its individual srs code, located at the end of the url. You will have to know these ahead of time. 


```python
basePage1 = 'http://www.amazon.com/s?i=specialty-aps&srs='
#default number of results per page
resultsPerPage = 16
#srs code
Brawny = '3019634011'
master = [Brawny]
```

Now for the scraping part. We are making one request to a single page, so we are not rotating user agents yet.


```python
for i in master:
    
    basePage = basePage1+i
    
    s = requests.Session()
    s.headers['User-Agent'] = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/34.0.1847.131 Safari/537.36'
    page1 = s.get(basePage)
    
    tree = html.fromstring(page1.content)
    
    ratingCount = tree.xpath('//*[@id="s-result-count"]//text()')
    
    ratingCount = ratingCount[0].replace('1-16 of ','')
    
    ratingCount = ratingCount.replace(' results for','')
    
    pages1 = math.ceil(int(ratingCount) / resultsPerPage)
    
    pages = [str(i) for i in range(1,pages1+1)]
```


```python
basePage
```




    'http://www.amazon.com/s?i=specialty-aps&srs=3019634011'




```python
len(pages)
```




    22



Using cfscrape module along with Requests to scrape each page. See https://github.com/Anorov/cloudflare-scrape for more info.


```python
scraper = cfscrape.create_scraper()
```

Shuffling the order in which we scrape each page is another way we avoid detection. Accessing pages in seemingly random order makes our bot seem more 'human',giving it less chance of being recognized as a robot. So we shuffle our pages, take one proxy from the pool (hoping it works), and then choose a random user agent for each page requested. We then scrape the html text from each page and store the text in [html_list].  


```python
random.shuffle(pages)
```


```python
html_list=[]
proxy = next(proxy_pool)
for i in pages:
    try:
        headers = {'User-Agent':str(ua.random)}
        htmltext = scraper.get(basePage+'&page='+str(i),headers = headers,proxies={"http": proxy, "https": proxy}).content
        sleep(numpy.random.randint(1,4)) 
        htmltext = str(htmltext)
        html_list.append(htmltext)        
        print(i)
    except:
        for i in range(20):
            sleep(numpy.random.randint(1,4))
```

    18
    12
    14
    2
    11
    4
    9
    7
    20
    22
    17
    13
    16
    5
    21
    10
    6
    3
    

Check a few pages' html text to make sure the data is there. The length of each should be in the hundreds of thousands. If the text contains a message like "Sorry! Something went wrong on our end" or length is around 2000, you were blocked. Re-shuffle your pages. Try a new proxy. Test again.


```python
html_list[1]
len(html_list[1])
```




    6611



The purpose of [html_list] is to store the ASIN's from each page. These are Amazon's unique product identifiers that we then use to access the individual product pages. We use regular expressions to pull each ASIN from our [html_list]. 


```python
asin_list=[]
for i in range(0,len(html_list)):
    pattern = re.compile("data-asin=\"([A-Z0-9]{10})\"")
    list2 = re.findall(pattern, html_list[i])
    asin_list.append(list(list2))       
```


```python
asin_list2 = [item for sublist in asin_list for item in sublist]
asin_list2 = list(asin_list2)
```


```python
#deduplicate
asin3 = []
for i in asin_list2:
    if i not in asin3:
        asin3.append(i)
```


```python
len(asin3)
```




    27



Now we use our ASIN list to access each product page and pull product name, price, and overall rating. We will use BeautifulSoup for this part.


```python
def get_soup(url):
            
            context = ssl._create_unverified_context()
            retries = 2
    
            wait_time = 5
            read_url = None
            soup = ""
            tries = 1
            req = Request(url)
            req.add_header('User-agent', str(ua.random))
    
            # access url, with exponential back-off in event of failure
            while read_url is None:
                try:
                    read_url = urlopen(req, context=context).read()
                    soup = BeautifulSoup(read_url, "lxml")
                except URLError:
                    for i in range(20):
                        # Generating random delays
                        sleep(numpy.random.randint(1,3))
                        # Adding verify=False to avold ssl related issues
                    if tries == retries:
                        soup = ""
    
            return soup
```


```python
product_list = []
for i in range(1,len(asin3)-1):
        reviewPage = 'https://www.amazon.com/dp/product-reviews/'+asin3[i]
        base = get_soup(reviewPage)
        print(i)
        name_raw = base.findAll('div',{'class':'a-row product-title'})
        name = re.findall(r'ie=UTF8">(.*?)</a>',str(name_raw))
        rating_raw = base.findAll('span',{'data-hook':'rating-out-of-text'})
        rating = re.findall(r'data-hook="rating-out-of-text">(.*?)</span>',str(rating_raw))
        price_raw = base.findAll('span',{'class':'a-color-price arp-price'})
        price = re.findall(r'"a-color-price arp-price">(.*?)</span>',str(price_raw))
        #handle currently unavailable items- they won't have price or rating
        if len(price) == 0:
            price = 'NA'
        else:
            price = price[0]
        if len(name) == 0:
            name = name
        else:
            name = name[0]  
        if rating[0] == '0.0 out of 5 stars':
            rating = 'NA'
        else:
            rating = rating[0]
        product_dict = {'Product Name': name,
                        'Price': price,
                        'Rating': rating}
        product_list.append(product_dict)
```

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10
    11
    12
    13
    14
    15
    16
    17
    18
    19
    20
    21
    22
    23
    24
    25
    


```python
#store as data frame
df1 = pd.DataFrame(product_list)
```


```python
#rearrange columns
cols = df1.columns.tolist()
cols
```




    ['Price', 'Product Name', 'Rating']




```python
cols = cols[-2:] + cols[:-2]
cols
```




    ['Product Name', 'Rating', 'Price']




```python
df1 = df1[cols]
```


```python
df1.head() #looks good!
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Product Name</th>
      <th>Rating</th>
      <th>Price</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Heavy-Duty Quarter fold Shop Towels in White</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Brawny Pick-a-Size Giant Plus Roll Paper Towel...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Brawny Select-a-size Paper Towels 6 Rolls 78 2...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Brawny Big Roll (1.25X Regular), 2 Ply, Prints</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Brawny Pick-A-Size Paper Towels, 8 XL Rolls - ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
  </tbody>
</table>
</div>




```python
df1.to_csv('out.csv')
```


```python
from IPython.core.interactiveshell import InteractiveShell
InteractiveShell.ast_node_interactivity = "all"
```


```python
df1.head()
```


```python
#pd.set_option('display.height', 1000)
pd.set_option('display.max_rows', 500)
pd.set_option('display.max_columns', 500)
pd.set_option('display.width', 1000)
```


```python
df1.head()
```


```python
from IPython.display import display
display(df1)
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Product Name</th>
      <th>Rating</th>
      <th>Price</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Heavy-Duty Quarter fold Shop Towels in White</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Brawny Pick-a-Size Giant Plus Roll Paper Towel...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Brawny Select-a-size Paper Towels 6 Rolls 78 2...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Brawny Big Roll (1.25X Regular), 2 Ply, Prints</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Brawny Pick-A-Size Paper Towels, 8 XL Rolls - ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Brawny Pick-A-Size Paper Towels, 8 XL Rolls - ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Wholesale Brawny Paper Towels 1pk 48ct 2-Ply</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Brawny HQHEMQ Pick-a-Size Paper Towels, 12 Big...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Brawny pHXdkF Pick-a-Size Paper Towels, 16XL R...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Brawny OmPInC Pick-a-Size Paper Towels, 16XL R...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Wet Shop Towels in Blue</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Brawny Pick A Size Regular Roll, 2 Ply-3pk</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>12</th>
      <td>Brawny Dine-A-Wipe Foodservice Busing Towel, 1...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Brawny Paper Towels, Tear-A-Square Sheets Stro...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Brawny Pick-a-Size Paper Towels bhVJVx, 80 XL ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>15</th>
      <td>Brawny Pick-a-Size Paper Towels cuMTav, 32 XL ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>16</th>
      <td>Brawny Pick-a-Size Paper Towels RyxBnc, 48 XL ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>17</th>
      <td>Brawny Pick-a-Size Paper Towels BYEeaS, 80 XL ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>18</th>
      <td>Brawny lnyGqB Pick-a-Size Paper Towels, 12 Big...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>19</th>
      <td>Brawny Pick-a-Size Paper Towels 24 XL Rolls (2...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>20</th>
      <td>Brawny Pick-a-Size Paper Towels LrvneW, 64 XL ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>21</th>
      <td>Brawny Pick a Size KrMsBw Paper Towels, 24 Cou...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>22</th>
      <td>Brawny Pick a Size jIlElT Paper Towels, 24 Cou...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>23</th>
      <td>Brawny Pick-a-Size Paper Towels ssbpKh, 80 XL ...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
    <tr>
      <th>24</th>
      <td>Brawny Pick a Size eLWTjL Paper Towels, 24 Cou...</td>
      <td>NA</td>
      <td>NA</td>
    </tr>
  </tbody>
</table>
</div>

