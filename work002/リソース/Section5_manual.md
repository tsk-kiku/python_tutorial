#### ETL処理
## 
## Step1：発注数と最新単価
import pandas as pd
import sqlite3

# データベースへの接続を開く
conn = sqlite3.connect('factory.db')

# --- SQL 1: 発注数（不足数）の抽出 ---
# "Triple Quote" (""") を使うと、改行を含む長い文字列を書けます
sql_shortage = """
WITH RECURSIVE bom_tree AS (
    SELECT 'ROBOT-X' as root_item, child_item_code, quantity * 10 as required_qty 
    FROM m_bom WHERE parent_item_code = 'ROBOT-X'
    UNION ALL
    SELECT parent.root_item, child.child_item_code, parent.required_qty * child.quantity
    FROM bom_tree parent JOIN m_bom child ON parent.child_item_code = child.parent_item_code
),
grouped_req AS (
    SELECT child_item_code, SUM(required_qty) as total_req_qty 
    FROM bom_tree GROUP BY child_item_code
)
SELECT 
    req.child_item_code AS item_code,
    MAX(req.total_req_qty - COALESCE(inv.quantity, 0), 0) AS shortage_qty
FROM grouped_req req
LEFT JOIN t_inventory inv ON req.child_item_code = inv.item_code
WHERE MAX(req.total_req_qty - COALESCE(inv.quantity, 0), 0) > 0;
"""

# --- SQL 2: 最新単価リストの抽出 (Lecture 18の完成形) ---
sql_price = """
WITH ranked_price AS (
    SELECT 
        item_code, supplier_code, unit_price, moq, lead_time_days,
        ROW_NUMBER() OVER (PARTITION BY item_code, supplier_code ORDER BY start_date DESC) as rn
    FROM m_purchase_price
)
SELECT item_code, supplier_code, unit_price, moq, lead_time_days
FROM ranked_price WHERE rn = 1;
"""

## Step2：Extract（抽出）
# 1. 不足リストをロード
df_shortage = pd.read_sql(sql_shortage, conn)

# 2. 単価リストをロード
df_price = pd.read_sql(sql_price, conn)

# 3. ちゃんと読み込めたか確認（最初の5行を表示）
print("--- Shortage List (df_shortage) ---")
display(df_shortage.head())

print("--- Price List (df_price) ---")
display(df_price.head())

## データクレンジング
## Step1：汚いデータの確認
# moq列だけを表示して確認
print("--- Cleaning 前 ---")
print(df_price[['item_code', 'moq']])

## Step2：数字以外を除去する (Transform)
# 1. 正規表現置換: 数字以外([^0-9]) を 空文字('') にする
# regex=True を指定することで正規表現モードになります
df_price['moq_cleaned'] = df_price['moq'].astype(str).str.replace(r'[^0-9]', '', regex=True)

print("--- 置換直後 (まだ文字列型) ---")
print(df_price[['item_code', 'moq', 'moq_cleaned']].head())

## Step2：（参考）カッコ内のデータを抽出
def extract_moq(text):
    import re
    # カッコの中の数字を探すパターン
    match = re.search(r'\((\d+)\)', str(text))
    if match:
        return match.group(1) # カッコの中身を返す (500)
    else:
        # カッコがなければ、単純に数字以外を消す
        return re.sub(r'[^0-9]', '', str(text))

df_price['moq_cleaned'] = df_price['moq'].apply(extract_moq)

## Step3：数値型への変換
# 2. 空文字対策（もし数字が一つもないデータがあったら 1 にする）
# pd.to_numericで変換できないものはNaN(欠損)にし、fillna(1)で埋める
df_price['moq_num'] = pd.to_numeric(df_price['moq_cleaned'], errors='coerce').fillna(1)

# 3. 整数型(int)に変換
df_price['moq_num'] = df_price['moq_num'].astype(int)

print("--- Cleaning 完了 (計算可能) ---")
display(df_price[['item_code', 'moq', 'moq_num']].head())

## ビジネスロジック
## Step1：データの結合（Merge）
# df_shortage: 不足数が入っている (SQLの計算結果)
# df_price:    単価、MOQ、LTが入っている (マスタデータ)

# item_code をキーにして、横に結合する
df_order = pd.merge(df_shortage, df_price, on='item_code', how='left')

# unit_price が NaN の行を捨てる
df_order = df_order.dropna(subset=['unit_price'])

# 必要な列だけ表示して確認
cols = ['item_code', 'shortage_qty', 'moq_num', 'lead_time_days']
display(df_order[cols].head())

## Step2：ロット丸め計算 (MOQ Logic)
import numpy as np

# ロジックの分解解説:
# 例: 不足 73個, MOQ 100個 の場合

# 1. 何箱必要か？ (73 / 100 = 0.73)
units_float = df_order['shortage_qty'] / df_order['moq_num']

# 2. 切り上げをする (0.73 -> 1.0)
units_ceil = np.ceil(units_float)

# 3. 発注数に戻す (1.0 * 100 = 100個)
df_order['order_qty'] = units_ceil * df_order['moq_num']

# 4. 整数にする (100.0 -> 100)
df_order['order_qty'] = df_order['order_qty'].astype(int)

print("--- 丸め計算の結果 ---")
display(df_order[['item_code', 'shortage_qty', 'moq_num', 'order_qty']].head())

## Step3：納期計算 (Date Logic)
from datetime import datetime, timedelta

# 1. 起点となる「今日」を取得
today = datetime.now()
print(f"発注日: {today.strftime('%Y-%m-%d')}")

# 2. 日付の足し算 (今日 + リードタイム)
# pd.to_timedelta: 整数を「日数」データに変換する関数
# 例: 3 -> "3 days"
df_order['delivery_date'] = today + pd.to_timedelta(df_order['lead_time_days'], unit='D')

# 3. 見やすく整形 (YYYY-MM-DD)
df_order['delivery_date'] = df_order['delivery_date'].dt.strftime('%Y-%m-%d')

print("--- 納期計算の結果 ---")
display(df_order[['item_code', 'lead_time_days', 'delivery_date']].head())

## データの書き込み（Load）
## Step1：データベースへの保存 (Load)
# データベース接続 (conn) は前のレクチャーから引き継いでいる前提

# 保存するテーブル名: t_purchase_order
# if_exists='replace': 既存テーブルがあれば削除して作り直す
# index=False: DataFrameの行番号(0, 1, 2...)はDBに保存しない
df_order.to_sql('t_purchase_order', conn, if_exists='replace', index=False)

print("✅ 発注データのDB保存が完了しました！")

## Step2：保存結果の検証
# 確認用SQL
verify_sql = "SELECT * FROM t_purchase_order"

# 読み込んで表示
df_verify = pd.read_sql(verify_sql, conn)

print("--- DBに保存されたデータ ---")
display(df_verify)

## Step3：CSV出力
# CSVファイルとして出力
# encoding='utf-8-sig': Excelで開いた時に文字化けしないおまじない
df_order.to_csv('purchase_order_20231001.csv', index=False, encoding='utf-8-sig')

print("✅ CSVファイルの出力も完了しました！フォルダを確認してください。")
