# Python網站擷取
## 一.建構擷取程序
### 1. 你的第一個擷取程序
```python 
from urllib.request import urlopen
html = urlopen('http://pythonscraping.com/pages/page1.html')
print(html.read())
```
上述這段程式碼會輸出域名為http://pythonscraping.com 的伺服器目錄\<web root\>/pages裡的page1.html這個html檔案

* urllib是python的標準庫，包含要求網站資料、處理cookies，甚至是交換metadata等功能
* urlopen用來開啟並讀取網路的遠端物件
* BeautifulSoup不是預設的函式庫，它提供易於遍歷的python物件來代表XML結構，幫助我們格式化和組織凌亂的web
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup
html = urlopen('http://www.pythonscraping.com/pages/page1.html') 
bs = BeautifulSoup(html.read(), 'html.parser')
print(bs.h1)
```
上述程式碼會輸出**第一個**h1標籤的內容

除了字串，BS也可以直接使用檔案物件
```python 
bs = BeautifulSoup(html, 'html.parser')
```
這時BS的物件則會變成如下結構
```html
• html → <html><head>...</head><body>...</body></html> 
    — head → <head><title>A Useful Page<title></head>
      — title → <title>A Useful Page</title>
    — body → <body><h1>An Int...</h1><div>Lorem ip...</div></body>
      — h1 → <h1>An Interesting Title</h1>
      — div → <div>Lorem Ipsum dolor...</div>
```
#### Parser
建立BS物件時，要傳入的第二個參數為parser，在大部分情況下，選擇什麼parser並無差別
* html.parser：已經包含在python3中，不需另外安裝
* lxml：比html.parser更能處理格式凌亂或錯誤的代碼，速度也比較快，缺點是必須單獨安裝並依賴於第三方C庫，降低移植性與易用性
* html5lib：可以更主動地修正html，速度最慢，同樣依賴外部dependency
#### 例外處理
```python
from urllib.request import urlopen 
from urllib.error import HTTPError
from urllib.error import URLError
from bs4 import BeautifulSoup
def getTitle(url): 
    try:
        html = urlopen(url) 
    except HTTPError as e:
        return None 
    except URLError as e:
        print('The server could not be found!')
        return None
    try:
        bs = BeautifulSoup(html.read(), 'html.parser')
        title = bs.body.h1 
    except AttributeError as e:
        return None 
    return title

title = getTitle('http://www.pythonscraping.com/pages/page1.html')
if title == None:
    print('Title could not be found')
else:
    print(title)
```
* HTTPError處理找不到網頁或是擷取錯誤
* URLError處理網站掛掉或是網址輸入錯誤
* AttributeError處理對None物件的操作
### 2. 進階HTML解析
```python
bs.find_all('table')[4].find_all('tr')[2].find('td').find_all('div')[1].find('a')
```
上述程式碼非常難維護，即便網站做很輕微的改動也可能使我們的爬蟲失效

在這個[網站](http://www.pythonscraping.com/pages/warandpeace.html)中，人物說的話為紅色，人物的名字為綠色，範例如下
```html
<span class="red">Heavens! what a virulent attack!</span> replied 
<span class="green">the prince</span>, not in the least disconcerted by this reception.
```
為了找出所有的人名，我們可以使用find_all這個函數來找出所有span中class是green的內容
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup

html = urlopen('http:// www.pythonscraping.com/pages/warandpeace.html')
bs = BeautifulSoup(html.read(), 'html.parser')
nameList = bs.findAll('span', {'class':'green'})
for name in nameList:
    print(name.get_text())
```
#### find與find_all
BeautifulSoup的find()和find_all()非常相似，幾乎所有情況只會用到tag和attributes這兩個參數
```python
find_all(tag, attributes, recursive, text, limit, keywords)
find(tag, attributes, recursive, text, keywords)
```
* tag：標籤名稱或是包含標籤名稱的python list，ex: `.find_all(['h1','h2','h3','h4','h5','h6'])`
* attributes：屬性的python字典，配對包含這些屬性的標籤，ex:`.find_all('span', {'class':{'green', 'red'}})`
* recursive：預設為True，如果設為False，則只會找文件的最上層標籤
* text：配對文本內容而不是依據標籤屬性，例如要找"the prince"出現的次數可以使用以下代碼
    ```python
    nameList = bs.find_all(text='the prince') 
    print(len(nameList))
    ```
* limit：設定要找前limit個項目
* keyword：選擇包含特定屬性或屬性集的標籤，ex:`title = bs.find_all(id='title', class_='text')`，基本上id在一個頁面只能使用一次，所以上述代碼等同於`title = bs.find(id='title')`。此外，keyword參數技術上是多餘的，因為keyword能做的事情都可以用正規表示式來處理。另外，因為class在python中是保留字，所以不能用`bs.find_all(class='green')`而要用`bs.find_all(class_='green')`或是`bs.find_all('', {'class':'green'})`
#### BeautifulSoup的四個物件
1. BeautifulSoup：ex:`bs`
2. Tag：ex:`bs.div.h1`
3. NavigableString：表示標籤的文字而不是標籤本身
4. Comment：用來尋找html的註釋，ex:`<!--like this one-->`
#### Navigating Trees
選擇標籤時越具體越好，例如可以使用`bs.find('table',{'id':'giftList'}).tr`就不要用`bs.tr`，僅管結果相同
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup

html = urlopen('http://www.pythonscraping.com/pages/page3.html')
bs = BeautifulSoup(html, 'html.parser')
for child in bs.find('table',{'id':'giftList'}).children: 
    print(child)
for sibling in bs.find('table', {'id':'giftList'}).tr.next_siblings: 
    print(sibling)
print(bs.find('img',{'src':'../img/gifts/img1.jpg'}).parent.previous_sibling.get_text())
```
* 使用.children或不使用任何函數來抓取id為giftList的表格的子節點
* .descendants則會找每一個子孫節點
* .tr.next_siblings會從第一個tr的下一個兄弟節點開始找
* .parent會找父節點
#### 正規表示式
假設我們想取得網站上所有產品圖片的URL，使用.find_all("img")會有一些問題：
* 網站有時會有一些隱藏圖片用來間隔或對齊
* 網站的頁面佈局會變動，不能依賴頁面中圖片的位子來找標籤
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup 
import re

html = urlopen('http://www.pythonscraping.com/pages/page3.html')
bs = BeautifulSoup(html, 'html.parser')
images = bs.find_all('img',{'src':re.compile('\.\.\/img\/gifts\/img.*\.jpg')}) 
for image in images:
    print(image['src']) #或是 print(image.attrs['src'])
```
* \*：>=0次
* +：>=1次
* {m,n}：m到n次
* \[\]：符合brackets裡的字元
* \[^\]：不符合brackets裡的字元
* |：或
* .：任意單一字元
* ^：以...為開頭
* $：以...為結尾
* \：跳脫字元
* ?!：不包含。很難用，若要完全排除一個字元，在兩端使用^和$
#### Lambda表達式(匿名函式)
BeautifulSoup允許我們使用函數來當作在find_all函數裡面的參數，唯一的限制是這些函數必須將標籤對象作為參數並回傳布林值
```python
bs.find_all(lambda tag: len(tag.attrs) == 2)
```
上述代碼使用len(tag.attrs) == 2作為參數，如果為真則find_all回傳tag

Lambda函數可以用來代替BS函數，使用Lambda函數就不需要再記其他BeautifulSoup格式，例如下面兩段代碼可以做相同的事情
```python
bs.find_all(lambda tag: tag.get_text() == 'Or maybe he\'s only resting?')
```
```python
bs.find_all('', text='Or maybe he\'s only resting?')
```
### 3. 撰寫網站爬行程序
維基百科的六度理論：維基百科每篇條目連結到另一條目所需的次數<=6(包含頭尾)

接下來我們要試著找出從[Eric Idle](https://en.wikipedia.org/wiki/Eric_Idle)到[Kevin Bacon](https://en.wikipedia.org/wiki/Kevin_Bacon)的最短路徑
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup

html = urlopen('http://en.wikipedia.org/wiki/Kevin_Bacon') 
bs = BeautifulSoup(html, 'html.parser')
for link in bs.find_all('a'):
    if 'href' in link.attrs: 
        print(link.attrs['href'])
```
因為維基百科的頁面有很多不是條目的連結，所以上述代碼會得到一些我們不要的結果

維基百科的文章頁面有三個共通點
1. 在div中，id為bodyContent
2. URLs不包含冒號"："
3. URLs以/wiki/開頭
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup 
import re

html = urlopen('http://en.wikipedia.org/wiki/Kevin_Bacon') 
bs = BeautifulSoup(html, 'html.parser')
for link in bs.find('div', {'id':'bodyContent'}).find_all('a', href=re.compile('^(/wiki/)((?!:).)*$')): 
    if 'href' in link.attrs:
        print(link.attrs['href'])
```
使用這三個規則我們可以找出所有Kevin_Bacon頁面的文章連結，但這並沒有什麼用，我們真正需要的是：
* 一個回傳所有固定格式的文章URL的函數getLinks
* 對於getLinks回傳的每一連結再次呼叫getLinks直到程式結束或是沒有新的連結為止
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup 
import datetime
import random
import re
random.seed(datetime.datetime.now()) 
def getLinks(articleUrl):
    html = urlopen('http://en.wikipedia.org{}'.format(articleUrl)) 
    bs = BeautifulSoup(html, 'html.parser')
    return bs.find('div', {'id':'bodyContent'}).find_all('a',href=re.compile('^(/wiki/)((?!:).)*$'))

links = getLinks('/wiki/Barber_paradox') 
while len(links) > 0:
    newArticle = links[random.randint(0, len(links)-1)].attrs['href'] 
    print(newArticle)
    links = getLinks(newArticle)
```
上述代碼使用random使每次跑的結果隨機

為了解決維基百科六度問題，我們還需要儲存和分析資料，詳情見第六章
#### 爬整個網站
有時我們需要爬整個網站，例如建立網站地圖或是擷取特定資料。為了避免爬重複的連結，我們使用set來解決這個問題
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup 
import re

pages = set()
def getLinks(pageUrl):
    global pages
    html = urlopen('http://en.wikipedia.org{}'.format(pageUrl)) 
    bs = BeautifulSoup(html, 'html.parser')
    for link in bs.find_all('a', href=re.compile('^(/wiki/)')):
        if 'href' in link.attrs:
            if link.attrs['href'] not in pages:
                #We have encountered a new page
                newPage = link.attrs['href'] 
                print(newPage) 
                pages.add(newPage) 
                getLinks(newPage)
getLinks('')
```
python預設的遞迴限制為1000次，所以上述程式碼最終會達到遞迴限制而停止。此法適用於大部分少於1000個連結的網站。一個例外是：網頁連結透過當前網頁的URL來產生，這將造成無限重複的路徑

接著我們試著收集維基頁面中的三個項目
1. 標題：h1 → span
2. 內文第一段：div#mw-content-text → p
3. 編輯連結：li#ca-edit → span → a
```python
from urllib.request import urlopen 
from bs4 import BeautifulSoup 
import re

pages = set()
def getLinks(pageUrl):
    global pages
    html = urlopen('http://en.wikipedia.org{}'.format(pageUrl)) 
    bs = BeautifulSoup(html, 'html.parser')
    try:
        print(bs.h1.get_text())
        print(bs.find(id ='mw-content-text').find_all('p')[0]) 
        print(bs.find(id='ca-edit').find('span').find('a').attrs['href']) 
    except AttributeError:
        print('This page is missing something! Continuing.')
        
    for link in bs.find_all('a', href=re.compile('^(/wiki/)')): 
        if 'href' in link.attrs:
            if link.attrs['href'] not in pages: 
                #We have encountered a new page 
                newPage = link.attrs['href'] 
                print('-'*20)
                print(newPage) 
                pages.add(newPage) 
                getLinks(newPage)
getLinks('')
```
#### 處理重新導向
重新導向允許web伺服器將一個域名指向不同位址的一段內容，所以有時抓取的網址可能不是輸入的網址。有兩類重新導向
1. 伺服端重新導向：頁面載入前URL就改變
    * python3的urllib會自動處理重新導向，使用
    ```python
    r = requests.get('http://github.com', allow_redirects=True)
    ```
2. 客戶端重新導向：頁面載入後才重新導向，例如有時會看到"You will be redirected in 10 seconds"
    * 有關使用JavaScript或HTML執行的客戶端重新導向的更多信息，參考第12章
#### 爬網際網路
在爬網際網路之前，應該問自己以下幾個問題：
1. 想收集哪些資料？
2. 給定幾個網站還是自動找尋我們所不知道的網站？
3. DFS還是BFS？
4. 不希望爬的網站條件？
5. 需要非中英文網站嗎？
6. 如何保護自己免受法律訴訟？(見第18章)
```python
from urllib.request import urlopen 
from urllib.parse import urlparse 
from bs4 import BeautifulSoup 
import re
import datetime
import random

pages = set()
random.seed(datetime.datetime.now())

#Retrieves a list of all Internal links found on a page
def getInternalLinks(bs, includeUrl):
    includeUrl = '{}://{}'.format(urlparse(includeUrl).scheme,urlparse(includeUrl).netloc)
    internalLinks = []
    #Finds all links that begin with a "/" 
    for link in bs.find_all('a',href=re.compile('^(/|.*'+includeUrl+')')):
        if link.attrs['href'] is not None:
            if link.attrs['href'] not in internalLinks:
                if(link.attrs['href'].startswith('/')):
                    internalLinks.append(includeUrl+link.attrs['href'])
                else:
                    internalLinks.append(link.attrs['href'])
    return internalLinks

#Retrieves a list of all external links found on a page
def getExternalLinks(bs, excludeUrl):
    externalLinks = []
    #Finds all links that start with "http" that do not contain the current URL
    for link in bs.find_all('a',href=re.compile('^(http|www)((?!'+excludeUrl+').)*$')):
        if link.attrs['href'] is not None:
            if link.attrs['href'] not in externalLinks:
                externalLinks.append(link.attrs['href'])
    return externalLinks

def getRandomExternalLink(startingPage):
    html = urlopen(startingPage)
    bs = BeautifulSoup(html, 'html.parser')
    externalLinks = getExternalLinks(bs,urlparse(startingPage).netloc)
    if len(externalLinks) == 0:
        print('No external links, looking around the site for one')
        domain = '{}://{}'.format(urlparse(startingPage).scheme,urlparse(startingPage).netloc)
        internalLinks = getInternalLinks(bs, domain)
        return getRandomExternalLink(internalLinks[random.randint(0,len(internalLinks)-1)])
    else:
        return externalLinks[random.randint(0, len(externalLinks)-1)]
    
def followExternalOnly(startingSite):
    externalLink = getRandomExternalLink(startingSite)
    print('Random external link is: {}'.format(externalLink))
    followExternalOnly(externalLink)
    
followExternalOnly('http://oreilly.com')
```
上述程式碼流程：取得外部連結，有則回傳，沒有則進入內部連結找。

使用錯誤處理來改善上面的代碼，當遇到錯誤時選擇另一個URL
### 4. 網站爬行模型
### 5. Scrapy
### 6. 儲存資料
## 二.儲存資料
### 7. 讀取文件
### 8. 清理髒資料
### 9. 讀寫自然語言
### 10. 表單與登入
### 11. 與擷取相關的JavaScript
### 12. 透過API 爬行
### 13. 影像處理與文字辨識
### 14. 避開擷取陷阱
### 15. 以爬行程序測試你的網站
### 16. 平行擷取網站
### 17. 遠端擷取
### 18. 網站擷取的法規與道德
