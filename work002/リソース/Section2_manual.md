#### Section 2: 開発環境の構築
## Google Colab
# 1. 動作確認
print("Hello, Manufacturing Data Engineering!")

# 2. 必要なライブラリの確認（Colabには最初から入っていますが、念のため）
import pandas as pd
import sqlite3
import sqlalchemy

print(f"Pandas version: {pd.__version__}")
# 出力例: Pandas version: 1.5.3 など

# --- 講座用初期化スクリプト ---
import pandas as pd
import sqlite3
import os
import glob
from google.colab import files

# データベースファイル名
DB_NAME = "factory.db"

# 1. 環境のクリーニング
extensions_to_remove = ['*.db', '*.csv']

print("🧹 環境をクリーニングしています...")
for ext in extensions_to_remove:
    for file in glob.glob(ext):
        try:
            os.remove(file)
            print(f"  - 削除しました: {file}")
        except Exception as e:
            print(f"  - 削除失敗: {file} ({e})")

print("✨ クリーニング完了 \n")

# 2. CSVファイルのアップロード
print("⬇️ 以下の4つのCSVファイルを選択してアップロードしてください ⬇️")
print("m_item.csv, m_bom.csv, m_purchase_price.csv, t_inventory.csv")

uploaded = files.upload()

# 3. データベースへの登録
required_files = {
    'm_item.csv': 'm_item',
    'm_bom.csv': 'm_bom',
    'm_purchase_price.csv': 'm_purchase_price',
    't_inventory.csv': 't_inventory'
}

try:
    conn = sqlite3.connect(DB_NAME)
    
    for filename in uploaded.keys():       
        if filename in required_files:
            table_name = required_files[filename]
            
            # CSV読み込み
            df = pd.read_csv(filename)
            
            # DB保存
            df.to_sql(table_name, conn, if_exists='replace', index=False)
            print(f"✅ テーブル作成成功: {table_name} ({len(df)}行)")
            
        else:
            # 想定外のファイル名ができた場合の警告
            print(f"⚠️ スキップしました（ファイル名）: {filename}")
            print("   ※もう一度このセルを最初から実行しなおしてください。")

    conn.close()
    print(f"\n🎉 環境構築完了！ データベース {DB_NAME} が作成されました。")

except Exception as e:
    print(f"❌ エラーが発生しました: {e}")

## SQLクライアント
# 1. SQLite データベースに接続
# factory.db という SQLiteファイルを開いています
conn = sqlite3.connect('factory.db')

# 2. SQL を書いて実行
query = "SELECT * FROM m_item"

# 3. 結果を表示
df = pd.read_sql(query, conn)
display(df)
