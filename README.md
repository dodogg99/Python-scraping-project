## 591_scraping
#### 591_scraping使用HTTP request及Selenium兩種方式進行591租屋網出租資訊的爬取。先輸入想要搜尋的城市、再輸入要搜尋的頁數後，便會爬取所有該城市一般刊登出租的資料，但不包含置頂的資料。取得資料後使用pandas進行整理，將車位出租刪除及各個租屋tag、鄰近交通站、樓層等資訊整理成行的形式後，儲存為城市名稱的CSV檔案。

## 主要使用Module
  - requests、BeautifulSoup、Selenium、pandas

## 步驟說明
#### 1.安裝主要使用Module
#### 2.執行requests_591_scrapying.py或selenium_591_scrapying.py
#### 3.輸入要搜尋的城市在以下對照表的region代碼及頁數便開始抓取該城市的出租資訊，並且print目前抓取的頁碼
  - 爬取1萬筆資料花費約20分鐘

![城市代表對照表](https://github.com/dodogg99/Python-project/blob/main/591%E7%A7%9F%E5%B1%8B%E7%B6%B2%E5%9F%8E%E5%B8%82%E4%BB%A3%E7%A2%BC%E5%B0%8D%E7%85%A7%E8%A1%A8.JPG)

## HTTP requests程式碼
```python
import requests
from bs4 import BeautifulSoup 
import pandas as pd
from random import randint
import time

#網頁資料爬取

region_code = {'1': '台北市', '2': '基隆市', '3': '新北市', '4': '新竹市', '5': '新竹縣', '6': '桃園市', '7': '苗栗縣', '8': '台中市',
             '10': '彰化縣', '11': '南投縣', '12': '嘉義市', '13': '嘉義縣', '14': '雲林縣', '15': '台南市', '17': '高雄市',
             '19': '屏東縣', '21': '宜蘭縣', '22': '台東縣', '23': '花蓮縣', '24': '澎湖縣', '25': '金門縣', '26': '連江縣'}


#輸入搜尋的城市
def search_region():
    while True:
        region = input('請輸入要搜尋的城市代碼')
        if region in region_code:
            print(f'你搜尋的城市為{region_code[region]}')
            break
        else:
            print('無效的城市代碼，請確認城市代碼後再輸入一次')
    return region

def search_page():
    while True:
        try:
            page = int(input('請輸入要搜尋的頁數'))
            if page < 1:
                print('請輸入大於0的整數')
            else:
                return page
        except:
            print('請輸入整數')

#獲得原始網頁訊息
def get_original_url_message(original_url, header, region):
    rq = requests.Session()
    res = rq.get(original_url, headers=header)
    soup = BeautifulSoup(res.text, 'html.parser')
    token_item = soup.select_one('meta[name="csrf-token"]').get("content")
    header['X-CSRF-TOKEN'] = token_item
    return rq


#對ajax的url發出request
def ajax_url_request(search_url, row, rq, header):
    parameter = 'is_format_data=1&is_new_list=1&type=1&firstRow={}'
    res = rq.get(search_url, params=parameter.format(row), headers=header)
    return res


region = search_region()
original_url = 'https://rent.591.com.tw/'
header = {'User-Agent':'Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36'}
url_domain = '.591.com.tw'

rq = get_original_url_message(original_url, header, region)

# 城市搜尋變數被cookie中的urlJumpIp影響，修改為要搜尋的城市
rq.cookies.set('urlJumpIp', region, domain=url_domain)

search_url = 'https://rent.591.com.tw/home/search/rsList'
row = 0
#先發出request確認總資料筆數
res = ajax_url_request(search_url, row, rq, header)
if res.status_code != requests.codes.ok:
    print('請求失敗', res.status_code)
else:
    print('請求成功')

#確認是否有租屋資料
if res.json()['records'] == 0:
    print(f'{region_code[region]}沒有租屋')
else:
    #將千分位數字字串改成一般整數
    total_row = int(res.json()['records'].replace(',', ''))

    total_data = []
    #每個頁面有30筆data，從0開始
    final_page_row = int(total_row/30)*30
    search_page_row = (search_page() - 1) * 30

    while row <= search_page_row:
        page = int(row/30+1)
        print(f'抓取第{page}頁')
        res = ajax_url_request(search_url, row, rq, header)
        #只儲存一般data，排除top data
        total_data.extend(res.json()['data']['data'])
        row += 30
        if row > final_page_row:
            print(f'已是最後一頁，該城市的資料為{page}頁')
            break
        #隨機停止1-4秒，避免頻繁發出request
        time.sleep(randint(1, 4))
    print('抓取完成')
    #資料格式整理

    df = pd.DataFrame(total_data)
    #刪除重複的租屋
    df = df.drop_duplicates(subset='post_id')
    needed_data = df.iloc[:, [0, 2, 3, 4, 5, 7, 9, 10, 11, 13, 14, 15, 18, 23]]
    #刪除車位出租
    needed_data = needed_data[needed_data.kind_name != '車位']
    total_rent_infor = []
    for i in range(0, len(needed_data)):
        rent_infor = [needed_data.iat[i, 1]]
        #將包含文字的tag轉換成只有數字的lsit
        try:
            needed_data.iat[i, 9] = pd.DataFrame(needed_data.iat[i, 9]).iloc[:, 0].tolist()
            for j in range(1, 18):
                if j in needed_data.iat[i, 9]:
                    rent_infor.append(1)
                else:
                    rent_infor.append(0)
        except:
            #如果沒有rent_tag就輸入0
            rent_infor.extend([0]*17)
        #拆分鄰近交通站內容
        try:
            rent_infor.extend([needed_data.iat[i, 13]['type'],
                               needed_data.iat[i, 13]['desc'],
                               needed_data.iat[i, 13]['distance'][0:-2]])
        except:
            #如果沒有鄰近交通站就輸入0
            rent_infor.extend([0]*3)
        #拆分樓層跟總樓層
        try:
            backslash = needed_data.iat[i, 4].find("/")
            rent_infor.extend([needed_data.iat[i, 4][0:backslash],
                               needed_data.iat[i, 4][backslash+1:]])
        except:
            #如果沒有樓層就輸入0
            rent_infor.extend([0]*2)
        total_rent_infor.append(rent_infor)
    title = ['post_id', '屋主直租', '近捷運', '拎包入住', '近商圈', '隨時可遷入', '可開伙', '可養寵物', '有車位', '有陽台',
             '有電梯', '押一付一', '免服務費', '南北通透', '免管理費', '可短租', '新上架', '影片房屋', '鄰近交通站', '站名', '距離/m',
             '樓層', '總樓層']
    pd.DataFrame(total_rent_infor, columns=title)
    #將整理資料與原始資料合併
    tag_df = pd.DataFrame(total_rent_infor, columns=title)
    final_data = needed_data.merge(tag_df, how='inner', on='post_id').drop(['floor_str', 'rent_tag', 'surrounding'],
                                                                           axis='columns')
    final_data.to_csv(f"{region_code[region]}.csv",
                      index=False,
                      encoding='utf_8_sig')
```
## Selenium程式碼
```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import NoSuchElementException, TimeoutException
import pandas as pd
import re
import time
from random import randint
#網頁資料爬取

region_code = {'1': '台北市', '2': '基隆市', '3': '新北市', '4': '新竹市', '5': '新竹縣', '6': '桃園市', '7': '苗栗縣', '8': '台中市',
               '10': '彰化縣', '11': '南投縣', '12': '嘉義市', '13': '嘉義縣', '14': '雲林縣', '15': '台南市', '17': '高雄市',
               '19': '屏東縣', '21': '宜蘭縣', '22': '台東縣', '23': '花蓮縣', '24': '澎湖縣', '25': '金門縣', '26': '連江縣'}

rent_tag_code = {'屋主直租': 0, '近捷運': 1, '拎包入住': 2, '近商圈': 3, '隨時可遷入': 4,'可開伙': 5, '可養寵物': 6, '有車位': 7,
                 '有陽台': 8, '有電梯': 9, '押一付一': 10, '免服務費': 11, '南北通透': 12, '免管理費': 13, '可短租': 14,
                 '新上架': 15, '影片房屋': 16}


#輸入搜尋的城市
def search_region():
    while True:
        region = input('請輸入要搜尋的城市代碼')
        if region in region_code:
            print(f'你搜尋的城市為{region_code[region]}')
            return region
        else:
            print('無效的城市代碼，請確認城市代碼後再輸入一次')


def search_page():
    while True:
        try:
            page = int(input('請輸入要搜尋的頁數'))
            if page < 1:
                print('請輸入大於0的整數')
            else:
                return page
        except:
            print('請輸入整數')


driver_path = 'C:\\Users\\user\\Downloads\\chromedriver.exe'
region = search_region()
url = 'https://rent.591.com.tw/?region={}'.format(region)

#headless mode
ops = Options()
ops.add_argument('window-size=1920x1080')
ops.add_argument('--headless=new')
driver = webdriver.Chrome(options=ops)
driver.get(url)

sr_page = search_page()
page = 1
last_page = 0
total_df = pd.DataFrame()
extra_tag_title = []
rent_content = None

while page <= sr_page:
    while True:
        #確認上一頁的element已被刪除，再Locate目前頁面element
        try:
            driver.find_element(By.XPATH, f"//span[@class='pageCurrent' and text()={last_page}]")
        except NoSuchElementException:
            try:
                rent_content = WebDriverWait(driver, 5).until(EC.presence_of_all_elements_located((By.XPATH, "//section[@class='vue-list-rent-item']")))
            except TimeoutException:
                print(f'{region_code[region]}沒有租屋')
            break
    if not rent_content:
        break
    #爬取頁面內容        
    for item in rent_content:
        rent_title = item.find_element(By.XPATH, ".//div[@class='item-title']").text
        post_id = item.get_attribute("data-bind")
        rent_detail_url = item.find_element(By.XPATH, "./a").get_attribute("href")
        rent_style = item.find_element(By.XPATH, ".//ul[@class='item-style']").text.split(' ')
        rent_kind_name = rent_style[0]
        #當有租屋格局時，rent_style會有4個element
        if len(rent_style) == 4:
            rent_room_structure = rent_style[1]
            rent_area = rent_style[2][:-1]
            rent_floor = rent_style[3].split('/')[0]
            rent_total_floor = rent_style[3].split('/')[1]
        else:
            rent_area = rent_style[1][:-1]
            rent_room_structure = ""
            rent_floor = rent_style[2].split('/')[0]
            rent_total_floor = rent_style[2].split('/')[0]
            
        #租屋位置若有另外說明或tag時，一樣只記錄區域及街道名稱
        rent_position = item.find_element(By.XPATH, ".//div[@class='item-area']").text.split(' ')[-1]
        rent_section_name = rent_position.split('-')[0]
        rent_street_name = rent_position.split('-')[1]
        
        rent_price = item.find_element(By.XPATH, ".//div[@class='item-price-text']/span").text
        rent_role_man = item.find_element(By.XPATH, ".//div[@class='item-msg']").text.split(' ')[0]
        #只儲存昨日點擊數字
        rent_yesterday_hit = re.search(f'\d+', item.find_element(By.XPATH, ".//div[@class='item-msg']").text.split(' ')[-1]).group()

        #該筆租屋有的tag值為1，其餘值為0
        rent_tag = [0]*17
        rent_tag_name = item.find_element(By.XPATH, ".//ul[@class='item-tags']").text.split(' ')
        #沒有rent_tag或tag不在預設的rent_tag_code時跳過
        if '' in rent_tag_name:
            pass
        else:
            for tag in rent_tag_name:
                #確認是否有未被記錄的rent_tag
                if tag in rent_tag_code:
                    rent_tag[rent_tag_code[tag]] = 1
                else:
                    extra_tag_title.append(rent_title)

        #從item-tip中確認鄰近地標類型，沒有地標時填入空格
        try:
            rent_tip_type = item.find_element(By.XPATH, ".//div[contains(@class,'item-tip')]").get_attribute("class").split(' ')[1]
            rent_tip_name = ''.join(item.find_element(By.XPATH, ".//div[contains(@class,'item-tip')]").text.split(' ')[:-1])
            rent_tip_distance = item.find_element(By.XPATH, ".//div[contains(@class,'item-tip')]").text.split(' ')[-1][:-2]
        except:
            rent_tip_type = ""
            rent_tip_name = ""
            rent_tip_distance = ""

        #儲存租屋資訊
        head_one = ['title', 'post_id', 'detail_url', 'kind_name', 'room_str', 'price元/月', 'section_name',
                    'street_name', 'area', 'role_name', 'yesterday_hit']
        head_two = ['屋主直租', '近捷運', '拎包入住', '近商圈', '隨時可遷入', '可開伙', '可養寵物', '有車位', '有陽台', '有電梯',
                    '押一付一', '免服務費', '南北通透', '免管理費', '可短租', '新上架', '影片房屋']
        head_three = ['鄰近地標', '地標名稱', '地標距離m', '樓層', '總樓層']
        df = pd.concat([pd.DataFrame([[rent_title, post_id, rent_detail_url, rent_kind_name, rent_room_structure, rent_price,
                                       rent_section_name, rent_street_name, rent_area, rent_role_man, rent_yesterday_hit]], columns=head_one),
                        pd.DataFrame([rent_tag], columns=head_two),
                        pd.DataFrame([[rent_tip_type, rent_tip_name, rent_tip_distance, rent_floor, rent_total_floor]], columns=head_three)],
                       axis=1)

        total_df = pd.concat([total_df, df], axis=0)
    print(f'已爬取第{page}頁,')

    # 當已經是最後一頁時會找不到下一頁的element
    try:
        next_page = WebDriverWait(driver, 5).until(EC.element_to_be_clickable((By.XPATH, "//a[@class='pageNext']")))
        next_page.click()
    except TimeoutException:
        print(f'已是最後一頁，該城市的資料為{page}頁')
        break
    last_page = page
    page += 1
driver.quit()
#確認是否有爬取資料
if not total_df.empty:
    print('已完成爬取')

    #如果有未記錄的rent_tag，print tag title
    if extra_tag_title:
        print(f'未紀錄rent_tag的title = {extra_tag_title}')

    #刪除車位出租及重複租屋資訊
    final_data = total_df[total_df.kind_name != '車位']
    final_data = final_data.drop_duplicates(subset='post_id')
    final_data.to_csv(f"{region_code[region]}_selenium.csv", index=False, encoding='utf_8_sig')
```
## 輸出CSV檔案格式範例
![資料格式](https://github.com/dodogg99/Python-project/blob/main/591%E7%A7%9F%E5%B1%8B%E8%B3%87%E8%A8%8A%E6%95%B4%E7%90%86%E6%A0%BC%E5%BC%8F.JPG)
