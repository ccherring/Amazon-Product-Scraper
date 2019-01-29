
conda install -c anaconda beautifulsoup4


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
```
