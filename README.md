

# Charbel-SaGa-AI-
# -*- coding: utf-8 -*-"""AI Agent Charbel (チャーベル) - Core Engine機能: 1. 6つのGmailアカウントおよびSNS連携データの集約 2. 単一Googleカレンダーへの一元管理とダブルブッキング記録・記憶保存 3. 「①やらなければいけない事」「②お金の為」「③好きな事/大物連携」の自動分類 4. AIシーナ(厳格タスク管理) &amp; 猫チャメ(伴走・癒やし)の会話応答"""

import os
import datetime
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from google.auth.transport.requests import Request
from googleapiclient.discovery import build

# カレンダー・スプレッドシート権限
SCOPES = [
    'https://www.googleapis.com/auth/calendar',
    'https://www.googleapis.com/auth/spreadsheets',
    'https://www.googleapis.com/auth/gmail.readonly'
]

# 監視対象のアカウントリスト
TARGET_EMAILS = [
    "phoenixaim001@gmail.com",
    "phoenixaim000@gmail.com",
    "phoenixaim7777777@gmail.com",
    "phoenixaim77777@gmail.com",
    "enumaamano@gmail.com",
    "enumaamano8@gmail.com"
]

class CharbelAgent:
    def __init__(self):
        self.creds = None
        self.calendar_service = None
        self.sheets_service = None
        self.init_google_services()

    def init_google_services(self):
        """Google API認証処理"""
        if os.path.exists('token.json'):
            self.creds = Credentials.from_authorized_user_file('token.json', SCOPES)
        if not self.creds or not self.creds.valid:
            if self.creds and self.creds.expired and self.creds.refresh_token:
                self.creds.refresh(Request())
            else:
                if os.path.exists('credentials.json'):
                    flow = InstalledAppFlow.from_client_secrets_file('credentials.json', SCOPES)
                    self.creds = flow.run_local_server(port=0)
                    with open('token.json', 'w') as token:
                        token.write(self.creds.to_json())
                else:
                    print("⚠️ 'credentials.json' が見つかりません。GCPより取得して配置してください。")
                    return

        if self.creds:
            self.calendar_service = build('calendar', 'v3', credentials=self.creds)
            self.sheets_service = build('sheets', 'v4', credentials=self.creds)
            print("✅ チャーベル: Google Calendar & Sheets API 接続成功！")

    def categorize_task(self, title, description=""):
        """【優先順位判定エンジン】"""
        text = f"{title} {description}".lower()
        if any(keyword in text for keyword in ["急ぎ", "至急", "提出", "契約", "必達", "クエスト"]):
            return "①やらなければいけない事"
        elif any(keyword in text for keyword in ["スカウト", "案件", "時給", "報酬", "面談", "お金", "副業"]):
            return "②今必要なお金の為"
        else:
            return "③好きな事・大物と繋がる"

    def add_to_single_calendar(self, title, start_iso, end_iso, priority_cat, description="", source_account=""):
        """単一カレンダー統合登録 ＆ ダブルブッキング記憶保存"""
        if not self.calendar_service:
            print("⚠️ カレンダーサービスが初期化されていません。")
            return

        calendar_id = 'primary'  # 必ず単一の主カレンダーへ集約

        # 重複チェック
        events_result = self.calendar_service.events().list(
            calendarId=calendar_id,
            timeMin=start_iso,
            timeMax=end_iso,
            singleEvents=True,
            orderBy='startTime'
        ).execute()

        existing_events = events_result.get('items', [])
        conflict_log = ""
        if existing_events:
            conflicts = [e.get('summary', '無題') for e in existing_events]
            conflict_log = f"\n⚠️【ダブルブッキング記憶ログ】\n重複した既存予定: {', '.join(conflicts)}\n"

        # 案内プロンプト/キャラクターメッセージ生成
        character_msg = ""
        if priority_cat == "①やらなければいけない事":
            character_msg = "\n[AIシーナ]: ご主人様！これは世界と未来のための最重要ミッションよ。後回しにせずすぐやるのよ！"
        else:
            character_msg = "\n[猫チャメ]: ご主人様、お金と未来のための準備にゃ〜！一緒に落ち着いて進めるニャ。"

        full_description = f"優先度: {priority_cat}\n対象アカウント: {source_account}\n{description}{conflict_log}{character_msg}"

        event_body = {
            'summary': f"[{priority_cat.split('')[0]}] {title}",
            'description': full_description,
            'start': {'dateTime': start_iso, 'timeZone': 'Asia/Tokyo'},
            'end': {'dateTime': end_iso, 'timeZone': 'Asia/Tokyo'},
            'reminders': {
                'useDefault': False,
                'overrides': [
                    {'method': 'popup', 'minutes': 24 * 60}, # 1日前アラーム
                    {'method': 'popup', 'minutes': 60},      # 1時間前アラーム
                ],
            },
        }

        created_event = self.calendar_service.events().insert(calendarId=calendar_id, body=event_body).execute()
        print(f"✅ [登録完了] カレンダーイベント追加: {created_event.get('summary')}")
        print(f"🔗 URL: {created_event.get('htmlLink')}")
        return created_event

if __name__ == '__main__':
    print("==========================================")
    print("🤖 AIエージェント「チャーベル」起動スクリプト")
    print("==========================================")
    
    agent = CharbelAgent()
    
    # 即時動作テスト (即時実行例)
    now = datetime.datetime.now(datetime.timezone.utc)
    test_start = (now + datetime.timedelta(days=1)).replace(hour=6, minute=0, second=0).isoformat()
    test_end = (now + datetime.timedelta(days=1)).replace(hour=7, minute=0, second=0).isoformat()
    
    # 優先度①: やらなければいけない事のテスト登録
    agent.add_to_single_calendar(
        title="【CACAO屋RECORD】最重要世界ミッション手続き",
        start_iso=test_start,
        end_iso=test_end,
        priority_cat="①やらなければいけない事",
        description="世界と自分の未来のための必須手続クエスト。",
        source_account="enumaamano@gmail.com"
    )

