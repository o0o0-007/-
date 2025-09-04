# 記帳 Line Bot 教學

## 前置準備

1. **註冊 LINE Developers 帳號**  
   - 前往 [LINE Developers](https://developers.line.biz/) 註冊帳號。  
   - 註冊完成後，點選 **稍後進行認證**。
     
-<img width="565" height="375" alt="image" src="https://github.com/user-attachments/assets/7cf5f46c-572a-4ff5-b555-cd900bcf63e2" />

2. **設定回應功能**  
   - 進入主頁，點選右上方 **設定**。  
   - 在 **回應設定** 中：  
     - 關閉「加入好友」、「歡迎訊息」、「自動回應訊息」。
       
-<img width="565" height="317" alt="image" src="https://github.com/user-attachments/assets/f24aa0fb-fc84-4bfc-a54c-6b016456ed82" />-

   - 將 **聊天回應方式** 改為「手動聊天 + 自動回應訊息」。
    
-<img width="565" height="208" alt="image" src="https://github.com/user-attachments/assets/9d0085c4-2b76-4b90-8ff0-dfb00074b7d1" />-

   - 點選 **回應功能的 Webhook**（若尚未開啟 Message API，會跳轉到 **Messaging API** 設定畫面，啟用即可）。  

3. **建立服務提供者**  
   - 按「同意」並建立服務提供者。  
   - 完成後與提供者連動。  

4. **取得 Channel Access Token**  
   - 前往 **LINE Developers Console**，點選剛設定的 Line Bot。
     
-<img width="565" height="274" alt="image" src="https://github.com/user-attachments/assets/8a8e478a-cb63-4c1e-a394-87d8b0a62bf5" />-

   - 進入 **Messaging API** 頁面，滑到最下方，點選 **Issue Channel Access Token**。  
   - 這就是你的 **Line Bot API Token**。
     
-<img width="565" height="298" alt="image" src="https://github.com/user-attachments/assets/ab768541-cff4-4795-a886-bc3bfdfec511" />-

---

## Google 雲端硬碟與試算表設定

1. 在 Google 雲端硬碟建立資料夾，命名為 **linebot**。  
2. 建立一份 Google 試算表，命名為 **linebot**。  
3. 在試算表中建立三個工作表：  
   - **控制變數**：在 A1 輸入 "0"（表示狀態）。  
   - **帳目資訊**：建立欄位 `名稱 / 金額 / 時間 / 類別 / 總金額 / 記帳數目`:  
     - 在「總金額」右邊輸入公式：  

       ```excel
       =SUM(B:B)
       ```

     - 在「記帳數目」右邊輸入公式：  

       ```excel
       =COUNTA(A3:A)
       ```
 -<img width="565" height="636" alt="image" src="https://github.com/user-attachments/assets/95553a4b-4a02-4b73-abc8-fcda3b92956e" />-

   - **帳目類別**：輸入自訂分類，例如：  

     ```
     1. 飲食
     2. 交通
     3. 購物
     ...
     ```  
-<img width="565" height="848" alt="image" src="https://github.com/user-attachments/assets/a94f54e7-6a75-4683-a4d7-56ef50e94541" />-

---

## Google Cloud 設定

1. 註冊 [Google Cloud](https://cloud.google.com/) 帳號。  
2. 進入 **IAM 與管理 > 服務帳號**，新增服務帳號。  
3. 在「金鑰」中，點選 **新增金鑰 > 建立新的金鑰**，選擇 JSON。  
4. 將下載的 JSON 檔命名為 **linebot.json**，上傳至 Google 雲端硬碟中的 **linebot** 資料夾。

-<img width="565" height="255" alt="image" src="https://github.com/user-attachments/assets/c78fa5b5-f2b2-40e5-8022-9b18058eb253" /> -

5. 開啟 Google 試算表的共享設定，將服務帳號的電子郵件加到共用，並給予 **編輯權限**。

-<img width="565" height="241" alt="image" src="https://github.com/user-attachments/assets/a78f1bb4-f25e-4642-b9f8-b98c517c051f" />-

---

## 程式碼設定

**開啟 Google Colab**

-先設定通道：
```python
!nohup ssh -p 443 -o StrictHostKeyChecking=no -R0:localhost:5000 a.pinggy.io -T &
```
-透過 ngrok 或 pinggy.io 產生一個公開網址：
```python
!cat nohup.out
```
-安裝必要套件
```python
!pip install flask
!pip install line-bot-sdk
!pip install pydrive
!pip install oauth2client
!pip install gspread
!pip install pygsheets
```
-在程式碼中需要設定以下變數：
```python
from flask import Flask, request
from linebot import LineBotApi, WebhookHandler
from linebot.models import *
from datetime import datetime, timezone, timedelta
from google.colab import drive
from oauth2client.service_account import ServiceAccountCredentials as SAC
from pydrive.auth import GoogleAuth
from pydrive.drive import GoogleDrive
import json
import random
import sys
import pygsheets
import gspread
import os

line_bot_api = LineBotApi('你的 Channel Access Token')
handler = WebhookHandler('你的 Channel Secret ') #在 Basic Settings 找到

drive.mount('/content/drive', force_remount=True)
GDriveJSON = 'linebot.json'(你的json檔)
GSpreadSheet = 'linebot'(google sheet 名稱)
GsheetKey = 'google sheet 網址(d/後直到/edit)' #圖放程式下參考
sheet_url='https://docs.google.com/spreadsheets/d/(same as GsheetKey)/'

gauth = GoogleAuth()
scope_gd = ["https://www.googleapis.com/auth/drive"]
gauth.credentials = SAC.from_json_keyfile_name('/content/drive/MyDrive/linebot/linebot.json', scope_gd) #你的linebot.json位置
mygd = GoogleDrive(gauth)

try:
  scope_gs = ['https://spreadsheets.google.com/feeds']
  credentials = SAC.from_json_keyfile_name('/content/drive/MyDrive/linebot/linebot.json', scope_gs) #你的linebot.json位置
  gc = gspread.authorize(credentials)
  sheet = gc.open_by_url(sheet_url)
  Sheets_stage=sheet.get_worksheet(0)
  Sheets_money=sheet.get_worksheet(1)
  Sheets_category=sheet.get_worksheet(2)


except Exception as ex:
  print('無法連線Google試算表', ex)
  sys.exit(1)

event_num = int(Sheets_stage.cell(1,1).value)

os.chdir('/content/drive/MyDrive/linebot')  # Colab 換路徑使用 #你的linebot資料夾

app = Flask(__name__)

@app.route("/", methods=['POST'])
def linebot():
    body = request.get_data(as_text=True)
    json_data = json.loads(body)
    print(json_data)
    Time_now = datetime.now(timezone(timedelta(hours=+8),'CST'))
    event_num = int(Sheets_stage.cell(1,1).value)
    print(Time_now)
    try:
        signature = request.headers['X-Line-Signature']
        handler.handle(body, signature)
        tk = json_data['events'][0]['replyToken']      # 取得 reply token
        msg_type = json_data['events'][0]['message']['type'] # 取得 LINE 收到的訊息類型
        if msg_type == 'text':
          msg = json_data['events'][0]['message']['text']   # 取得使用者發送的訊息
        msgID = json_data['events'][0]['message']['id']
        print(msgID)
        print(msg_type)
        if '重設' in msg:
          text_message = TextSendMessage(text='已重設事件代碼')
          line_bot_api.reply_message(tk,text_message)
          event_num = 0
          Sheets_stage.update_cell(1,1,event_num)

        while (event_num == 0):
          if '記帳' in msg:
            text_message = TextSendMessage(text='進入記帳系統')
            line_bot_api.reply_message(tk,[text_message, TextSendMessage(text="1 新增帳目"+"\n"+"2 新增類別"+"\n"+"3 進行結算"+"\n"+"4 退出系統")])
            event_num = 10
            Sheets_stage.update_cell(1,1,event_num)
            break

        while (event_num == 11):
          if " " in msg:
            msg_s = msg.split(' ')
            item_name = msg_s[0]
            item_amount = int(msg_s[1])
            today = str(datetime.strftime(Time_now,'%Y-%m-%d %H:%M:%S'))
            item_data=[item_name,item_amount,today]
            Sheets_money.append_row(item_data,1)
            text_message = TextSendMessage(text='記帳資料已新增')
            text_message_1 = TextSendMessage(text="請輸入所屬類別")
            category_list = Sheets_category.get_all_values()
            print(category_list)
            category_number = len(category_list)
            print(category_number)
            category_list_show = ""
            for list_num in range(0,category_number,1):
              category_num, category_name = category_list [list_num]
              category_list_name = category_num+"\t"+category_name+"\n"
              category_list_show = category_list_show + category_list_name
            print(category_list_show)
            text_message_2 = TextSendMessage(text=category_list_show)
            line_bot_api.reply_message(tk,[TextSendMessage(text=msg+"\n"),text_message,text_message_1,text_message_2])
            event_num = 12
            Sheets_stage.update_cell(1,1,event_num)
            break
          if 'exit' in msg:
            text_message = TextSendMessage(text='返回記帳系統選單')
            line_bot_api.reply_message(tk,[text_message, TextSendMessage(text="1 新增帳目"+"\n"+"2 新增類別"+"\n"+"3 進行結算"+"\n"+"4 退出系統")])
            event_num = 10
            Sheets_stage.update_cell(1,1,event_num)
            break

        while (event_num == 12):
          category_list = Sheets_category.get_all_values()
          category_number = len(category_list)
          money_list = Sheets_money.get_all_values()
          money_number = len(money_list)
          msg_num = int(msg)
          if msg_num > category_number:
            text_message = TextSendMessage(text='請輸入正確類別編號')
            line_bot_api.reply_message(tk,text_message)
            event_num = 12
            Sheets_stage.update_cell(1,1,event_num)
            break
          if msg_num <= category_number:
            category = Sheets_category.cell(msg_num,2).value
            Sheets_money.update_cell(money_number,4,category)
            text_message = TextSendMessage(text="已寫入類別"+"\t"+category)
            line_bot_api.reply_message(tk,[text_message, TextSendMessage(text="1 新增帳目"+"\n"+"2 新增類別"+"\n"+"3 進行結算"+"\n"+"4 退出系統")])
            event_num = 10
            Sheets_stage.update_cell(1,1,event_num)
            break

        while (event_num == 13):
          if 'exit' in msg:
            text_message = TextSendMessage(text='返回記帳系統選單')
            line_bot_api.reply_message(tk,[text_message, TextSendMessage(text="1 新增帳目"+"\n"+"2 新增類別"+"\n"+"3 進行結算"+"\n"+"4 退出系統")])
            event_num = 10
            Sheets_stage.update_cell(1,1,event_num)
            break
          category_name = msg
          category_list = Sheets_category.get_all_values()
          category_number = len(category_list)
          category_data = [category_number+1, category_name]
          Sheets_category.append_row(category_data,1)
          text_message = TextSendMessage(text="類別"+"\t"+category_name+"\t"+"已新增")
          line_bot_api.reply_message(tk,[text_message, TextSendMessage(text="1 新增帳目"+"\n"+"2 新增類別"+"\n"+"3 進行結算"+"\n"+"4 退出系統")])
          event_num = 10
          Sheets_stage.update_cell(1,1,event_num)
          break

        while (event_num == 10):
          if '1' in msg:
            text_message = TextSendMessage(text='請輸入項目名稱與金額，並使用空白區隔，例如：早餐 50。或輸入exit回到上一層')
            line_bot_api.reply_message(tk,text_message)
            event_num = 11
            Sheets_stage.update_cell(1,1,event_num)
            break
          if '2' in msg:
            text_message = TextSendMessage(text='請輸入類別名稱。或輸入exit回到上一層')
            line_bot_api.reply_message(tk,text_message)
            event_num = 13
            Sheets_stage.update_cell(1,1,event_num)
            break
          if '3' in msg:
            total_amount = str(Sheets_money.cell(1,6).value)
            print(total_amount)
            text_message = TextSendMessage(text="目前總共花費"+"\t"+total_amount+"\t"+"元")
            money_list = Sheets_money.get_all_values()
            money_number = len(money_list)
            print(money_number)
            money_write=""
            for listnumber in range(money_number):
              money_name = money_list[listnumber][0]
              money_amount = money_list[listnumber][1]
              money_date = money_list[listnumber][2]
              money_category = money_list[listnumber][3]
              money_list_write = money_name+'\t'+money_amount+'\t'+money_date+'\t'+money_category+'\n'
              money_write = money_write + money_list_write
            line_bot_api.reply_message(tk,[TextSendMessage(text=money_write),text_message,TextSendMessage(text="1 新增帳目"+"\n"+"2 新增類別"+"\n"+"3 進行結算"+"\n"+"4 退出系統")])
            break
          if '4' in msg:
            text_message = TextSendMessage(text='已退出記帳系統')
            line_bot_api.reply_message(tk,text_message)
            event_num = 0
            Sheets_stage.update_cell(1,1,event_num)
            break
          text_message = TextSendMessage(text='輸入錯誤，請重新輸入')
          line_bot_api.reply_message(tk,text_message)
          break
    except:
        print('error')
    return 'OK'

if __name__ == "__main__":
    app.run()
```

**google sheet 網址：**

-<img width="865" height="93" alt="image" src="https://github.com/user-attachments/assets/44003c09-6294-4501-9205-4e85cf3157c7" />-

## Webhook 設定
1. 執行程式後，會產生一個網址（例如 `https://xxxxx`）。
   
-<img width="878" height="149" alt="image" src="https://github.com/user-attachments/assets/7845cdb7-9f52-41d8-a19d-f8aac5bad425" />-

2. 複製該網址，貼到 **Line Messaging API > Webhook URL**。  
   - 點選 **Edit** → 貼上網址 → **Update**。  
3. 按下 **Verify**，如果顯示 Success 就代表設定成功。
   
-<img width="565" height="283" alt="image" src="https://github.com/user-attachments/assets/77868ae9-c864-4a90-9cae-b976b3edad8f" />-
 
4. 回到 **回應功能**，開啟 **Webhook**。

-<img width="565" height="231" alt="image" src="https://github.com/user-attachments/assets/416b302d-bacb-473c-ac9c-32645ed6edd6" />-

5. 完成後，你就可以到 LINE 聊天室測試功能！ 🎉

## 完成介面

**line 介面：**

<img width="856" height="1454" alt="image" src="https://github.com/user-attachments/assets/f6c86061-e99a-4e23-a243-c79369e7a2c1" />

**google sheet 顯示畫面**

-<img width="222" height="322" alt="image" src="https://github.com/user-attachments/assets/a627940b-466b-4f31-b955-47c6452ca3e9" />-


-<img width="465" height="207" alt="image" src="https://github.com/user-attachments/assets/2f412e43-fe1a-46c0-a26e-9c896221a116" />-
