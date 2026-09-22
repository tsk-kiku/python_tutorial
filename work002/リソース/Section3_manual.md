#### Step3:データベース設計 (DDL)
## ER図（実体関連図）
# 以下のコードをテキストセルに貼ると図になります
from graphviz import Digraph

# 1. インスタンス作成
dot = Digraph(comment='Factory ER Diagram')
dot.attr(rankdir='LR')
dot.attr('node', shape='box', style='filled', fillcolor='lightyellow')

# 2. テーブル（ノード）の定義
# 品目マスタ
dot.node('m_item', '''<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0">
  <TR><TD BGCOLOR="lightblue"><B>m_item</B></TD></TR>
  <TR><TD ALIGN="LEFT">PK item_code</TD></TR>
  <TR><TD ALIGN="LEFT">   item_name</TD></TR>
  <TR><TD ALIGN="LEFT">   category</TD></TR>
</TABLE>>''')

# 在庫テーブル
dot.node('t_inventory', '''<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0">
  <TR><TD BGCOLOR="lightgrey"><B>t_inventory</B></TD></TR>
  <TR><TD ALIGN="LEFT">PK,FK item_code</TD></TR>
  <TR><TD ALIGN="LEFT">PK    location_code</TD></TR>
  <TR><TD ALIGN="LEFT">      quantity</TD></TR>
</TABLE>>''')

# 購買単価マスタ
dot.node('m_purchase_price', '''<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0">
  <TR><TD BGCOLOR="lightgrey"><B>m_purchase_price</B></TD></TR>
  <TR><TD ALIGN="LEFT">PK,FK item_code</TD></TR>
  <TR><TD ALIGN="LEFT">PK    supplier_code</TD></TR>
  <TR><TD ALIGN="LEFT">PK    start_date</TD></TR>
  <TR><TD ALIGN="LEFT">      end_date</TD></TR>
  <TR><TD ALIGN="LEFT">      unit_price</TD></TR>
  <TR><TD ALIGN="LEFT">      moq (Dirty)</TD></TR>
  <TR><TD ALIGN="LEFT">      lead_time_days</TD></TR>
</TABLE>>''')

# BOM（部品表）
dot.node('m_bom', '''<
<TABLE BORDER="0" CELLBORDER="1" CELLSPACING="0">
  <TR><TD BGCOLOR="lightgrey"><B>m_bom</B></TD></TR>
  <TR><TD ALIGN="LEFT">PK,FK parent_item_code</TD></TR>
  <TR><TD ALIGN="LEFT">PK,FK child_item_code</TD></TR>
  <TR><TD ALIGN="LEFT">      quantity</TD></TR>
</TABLE>>''')

# 3. リレーション
dot.edge('m_item', 't_inventory', label='1:N')
dot.edge('m_item', 'm_purchase_price', label='1:N')
dot.edge('m_item', 'm_bom', label='Parent')
dot.edge('m_item', 'm_bom', label='Child')

# 描画
dot

## 正規化
# Step1：品目と在庫が混ざった表を作成
data = {
    'item_code':     ['ROBOT-X', 'ROBOT-X', 'SCR-M4-10'],
    'item_name':     ['Cleaning Robot', 'Cleaning Robot', 'Screw M4'], # 重複
    'category':      ['Product', 'Product', 'Component'],              # 重複
    'location_code': ['WH-01', 'WH-02', 'WH-01'],
    'quantity':      [5, 3, 500]
}
df_dirty = pd.DataFrame(data)

print("--- 正規化前のデータ ---")
display(df_dirty)

# Step2：item_codeをキーにして、属性(name, category)を抜き出す
df_item_master = df_dirty[['item_code', 'item_name', 'category']].drop_duplicates()

print("\n--- 正規化後: m_item ---")
display(df_item_master)

## SQLによるテーブル作成 (DDL)
import sqlite3

# 1. DBに接続
conn = sqlite3.connect('factory.db')
cursor = conn.cursor()

# 2. SQL定義（長いので三重引用符で囲みます）
sql_create_tables = """
-- 既存テーブルの削除
DROP TABLE IF EXISTS m_item;
DROP TABLE IF EXISTS t_inventory;

-- 1. 品目マスタ
CREATE TABLE m_item (
    item_code VARCHAR(20) NOT NULL,
    item_name VARCHAR(100),
    category  VARCHAR(20),
    PRIMARY KEY (item_code)
);

-- 2. 在庫テーブル
CREATE TABLE t_inventory (
    item_code     VARCHAR(20) NOT NULL,
    location_code VARCHAR(20) NOT NULL,
    quantity      INTEGER DEFAULT 0,
    PRIMARY KEY (item_code, location_code),
    FOREIGN KEY (item_code) REFERENCES m_item(item_code)
);
"""

# 3. 実行 (executescriptを使うと複数のSQLをまとめて実行できます)
try:
    cursor.executescript(sql_create_tables)
    print("✅ m_item, t_inventory テーブルを作成しました。")
except Exception as e:
    print(f"❌ エラー: {e}")

# 4. 変更を確定
conn.commit()
conn.close()

## 複合プライマリーキーを設定 (m_purchase_price)
import sqlite3

conn = sqlite3.connect('factory.db')
cursor = conn.cursor()

sql_create_price = """
DROP TABLE IF EXISTS m_purchase_price;

CREATE TABLE m_purchase_price (
    item_code      VARCHAR(20) NOT NULL,
    supplier_code  VARCHAR(20) NOT NULL,
    start_date     DATE        NOT NULL,
    end_date       DATE,
    unit_price     REAL        NOT NULL,
    moq            VARCHAR(20),
    lead_time_days INTEGER,
    
    PRIMARY KEY (item_code, supplier_code, start_date),
    FOREIGN KEY (item_code) REFERENCES m_item(item_code)
);
"""

try:
    cursor.executescript(sql_create_price)
    print("✅ m_purchase_price テーブルを作成しました（複合キー設定済み）。")
except Exception as e:
    print(f"❌ エラー: {e}")

conn.commit()
conn.close()

## 重複エラーの確認 (制約テスト)
import sqlite3

conn = sqlite3.connect('factory.db')
cursor = conn.cursor()

try:
    # 1. 正常データ登録
    cursor.execute("""
        INSERT INTO m_purchase_price (item_code, supplier_code, start_date, unit_price, moq)
        VALUES ('SCR-M4-10', 'SUP-A', '2023-01-01', 10.5, '100 pcs')
    """)
    print("✅ 1件目: 正常登録成功")

    # 2. 日付違いならOK
    cursor.execute("""
        INSERT INTO m_purchase_price (item_code, supplier_code, start_date, unit_price, moq)
        VALUES ('SCR-M4-10', 'SUP-A', '2024-01-01', 12.0, '100 pcs')
    """)
    print("✅ 2件目: 正常登録成功（日付違い）")

    # 3. 完全重複データ（エラーになるべき）
    print("👉 3件目: 重複データの登録を試みます...")
    cursor.execute("""
        INSERT INTO m_purchase_price (item_code, supplier_code, start_date, unit_price, moq)
        VALUES ('SCR-M4-10', 'SUP-A', '2023-01-01', 999.9, 'Err')
    """)
    
    # ここに来たらおかしい（エラーにならなかった場合）
    print("❌ エラーが発生しませんでした（テスト失敗）")

except sqlite3.IntegrityError as e:
    # 期待通りエラーが出た場合
    print(f"✅ 期待通りのエラーが発生しました: {e}")
    print("   → 複合プライマリキーが正しく機能しています！")

except Exception as e:
    print(f"❌ 予期せぬエラー: {e}")

conn.commit()
conn.close()

## BOMテーブルの作成
import sqlite3

conn = sqlite3.connect('factory.db')
cursor = conn.cursor()

sql_create_bom = """
DROP TABLE IF EXISTS m_bom;

CREATE TABLE m_bom (
    parent_item_code VARCHAR(20) NOT NULL,
    child_item_code  VARCHAR(20) NOT NULL,
    quantity         INTEGER,
    
    PRIMARY KEY (parent_item_code, child_item_code),
    FOREIGN KEY (parent_item_code) REFERENCES m_item(item_code),
    FOREIGN KEY (child_item_code)  REFERENCES m_item(item_code)
);
"""

try:
    cursor.executescript(sql_create_bom)
    print("✅ m_bom テーブルを作成しました。")
except Exception as e:
    print(f"❌ エラー: {e}")

conn.commit()
conn.close()

## データのローディング（Pandas活用）
import sqlite3
import pandas as pd

# 事前にアップロードしたCSVをDataFrameに読み込む
# (ファイルがない場合は Section 2 の初期化スクリプトを再実行してください)
try:
    df_item = pd.read_csv('m_item.csv')
    df_inventory = pd.read_csv('t_inventory.csv')
    df_price = pd.read_csv('m_purchase_price.csv')
    df_bom = pd.read_csv('m_bom.csv')

    # DB接続
    conn = sqlite3.connect('factory.db')

    # 既存データがあれば消して入れ直す (replace)
    df_item.to_sql('m_item', conn, if_exists='replace', index=False)
    df_inventory.to_sql('t_inventory', conn, if_exists='replace', index=False)
    df_price.to_sql('m_purchase_price', conn, if_exists='replace', index=False)
    df_bom.to_sql('m_bom', conn, if_exists='replace', index=False)

    print("✅ 全データのロードが完了しました。")
    conn.close()

except Exception as e:
    print(f"❌ エラー: CSVファイルが見つかりません。初期化スクリプトを実行してください。\n詳細: {e}")

## データの確認 (SELECT)
import sqlite3
import pandas as pd

conn = sqlite3.connect('factory.db')

# SQL実行
query = "SELECT * FROM m_purchase_price LIMIT 5"
df_result = pd.read_sql(query, conn)

print("--- m_purchase_price の中身 ---")
display(df_result)

conn.close()

