#### Section4：DMLのハンズオン
## 1. DMLの説明 (INSERT, SELECT, UPDATE, DELETE)
import sqlite3
import pandas as pd

# DB接続
conn = sqlite3.connect('factory.db')
cursor = conn.cursor()

# --- Step 1: データの登録 (INSERT) ---
print("--- Step 1: INSERT ---")
cursor.execute("""
    INSERT INTO t_inventory (item_code, location_code, quantity)
    VALUES ('LED-RED', 'WH-01', 100)
""")
conn.commit() # 確定

# --- Step 2: データの参照 (SELECT) ---
print("\n--- Step 2: SELECT ---")
df = pd.read_sql("SELECT * FROM t_inventory WHERE item_code = 'LED-RED'", conn)
display(df)

# --- Step 3: データの更新 (UPDATE) ---
print("\n--- Step 3: UPDATE ---")
cursor.execute("""
    UPDATE t_inventory
    SET quantity = 95
    WHERE item_code = 'LED-RED' AND location_code = 'WH-01'
""")
conn.commit()

# 確認
df = pd.read_sql("SELECT * FROM t_inventory WHERE item_code = 'LED-RED'", conn)
display(df)

# --- Step 4: データの削除 (DELETE) ---
print("\n--- Step 4: DELETE ---")
cursor.execute("""
    DELETE FROM t_inventory
    WHERE item_code = 'LED-RED' AND location_code = 'WH-01'
""")
conn.commit()

# 確認（0件のはず）
df = pd.read_sql("SELECT * FROM t_inventory WHERE item_code = 'LED-RED'", conn)
display(df)

conn.close()

## 2. JOIN 操作 (Inner Join vs Left Join)
conn = sqlite3.connect('factory.db')

# --- Step 1: データの状況確認 ---
print("--- Step 1: データの状況確認 ---")
print("1. マスタにはあるが...")
display(pd.read_sql("SELECT item_code, item_name FROM m_item WHERE item_code = 'SEN-IR-01'", conn))
print("2. 在庫にはない")
display(pd.read_sql("SELECT * FROM t_inventory WHERE item_code = 'SEN-IR-01'", conn))

# --- Step 2: 内部結合 (Inner Join) ---
print("\n--- Step 2: Inner Join (消えてしまう) ---")
sql_inner = """
SELECT i.item_code, i.item_name, inv.quantity
FROM m_item i
INNER JOIN t_inventory inv ON i.item_code = inv.item_code
"""
display(pd.read_sql(sql_inner, conn).tail()) # 件数が多いので末尾だけ表示

# --- Step 3: 左外部結合 (Left Join) ---
print("\n--- Step 3: Left Join (残るがNULLになる) ---")
sql_left = """
SELECT i.item_code, i.item_name, inv.quantity
FROM m_item i
LEFT JOIN t_inventory inv ON i.item_code = inv.item_code
"""
display(pd.read_sql(sql_left, conn).tail())

# --- Step 4: NULLの処理 (COALESCE) ---
print("\n--- Step 4: COALESCE (0になる) ---")
sql_coalesce = """
SELECT i.item_code, i.item_name, 
       COALESCE(inv.quantity, 0) as quantity_safe
FROM m_item i
LEFT JOIN t_inventory inv ON i.item_code = inv.item_code
"""
display(pd.read_sql(sql_coalesce, conn).tail())

conn.close()

## 3. 再帰クエリ (BOM展開)
conn = sqlite3.connect('factory.db')

# --- Step 1: データの親子関係を確認 ---
print("--- Step 1: 直接の親子関係 ---")
display(pd.read_sql("SELECT * FROM m_bom WHERE parent_item_code = 'ROBOT-X'", conn))

# --- Step 2: 再帰クエリの作成 ---
print("\n--- Step 2: BOM全展開 (再帰クエリ) ---")
sql_recursive = """
WITH RECURSIVE bom_tree AS (
    -- Anchor Member
    SELECT 
        parent_item_code as root_item,
        child_item_code,
        quantity,
        1 AS level
    FROM m_bom
    WHERE parent_item_code = 'ROBOT-X'
    
    UNION ALL
    
    -- Recursive Member
    SELECT 
        parent.root_item,
        child.child_item_code,
        parent.quantity * child.quantity,
        parent.level + 1
    FROM bom_tree parent
    JOIN m_bom child ON parent.child_item_code = child.parent_item_code
)
SELECT * FROM bom_tree ORDER BY level, child_item_code
"""
display(pd.read_sql(sql_recursive, conn))

conn.close()

## 4. 集計関数 (GROUP BY, HAVING)
conn = sqlite3.connect('factory.db')

# --- Step 1: スカラー関数 (ROUND) ---
print("--- Step 1: スカラー関数 ---")
display(pd.read_sql("SELECT item_code, unit_price, ROUND(unit_price, 0) AS rounded_price FROM m_purchase_price LIMIT 5", conn))

# --- Step 2: 集計関数 (GROUP BY) ---
print("\n--- Step 2: 集計関数 ---")
sql_agg = """
SELECT 
    item_code,
    COUNT(supplier_code) AS supplier_count,
    MIN(unit_price) AS min_price,
    MAX(unit_price) AS max_price,
    AVG(unit_price) AS avg_price
FROM m_purchase_price
GROUP BY item_code
"""
display(pd.read_sql(sql_agg, conn))

# --- Step 3: GROUP BY なし (意図しない結果) ---
print("\n--- Step 3: GROUP BY なし (テーブル全体を集計対象とみなす) ---")
display(pd.read_sql("SELECT item_code, MAX(unit_price) FROM m_purchase_price", conn))

# --- Step 4: WHERE と HAVING ---
print("\n--- Step 4: WHERE vs HAVING ---")
sql_having = """
SELECT item_code, SUM(quantity) AS total_qty
FROM t_inventory
WHERE location_code = 'WH-01'
GROUP BY item_code
HAVING SUM(quantity) >= 10
"""
display(pd.read_sql(sql_having, conn))

conn.close()

# --- Step 5: HAVING のみ ---
print("\n--- Step 5: Only HAVING ---")
sql_having = """
SELECT item_code,location_code, SUM(quantity) AS total_qty
FROM t_inventory
GROUP BY item_code
HAVING SUM(quantity) >= 10
"""
display(pd.read_sql(sql_having, conn))

# --- Step 6: WHERE のみ ---
print("\n--- Step 6: Only WHERE ---")
sql_where = """
SELECT item_code,location_code, SUM(quantity) AS total_qty
FROM t_inventory
WHERE location_code = 'WH-01'

"""
display(pd.read_sql(sql_where, conn))

## 5. 在庫計算 (ROBOT-X)
conn = sqlite3.connect('factory.db')

print("--- 在庫引当計算の結果 ---")
sql_allocation = """
-- 【Block 1: BOM Explosion】
WITH RECURSIVE bom_tree AS (
    SELECT 'ROBOT-X' as root_item, child_item_code, quantity * 10 as required_qty
    FROM m_bom WHERE parent_item_code = 'ROBOT-X'
    UNION ALL
    SELECT parent.root_item, child.child_item_code, parent.required_qty * child.quantity
    FROM bom_tree parent JOIN m_bom child ON parent.child_item_code = child.parent_item_code
),
-- 【Block 2: Aggregation】
grouped_req AS (
    SELECT child_item_code, SUM(required_qty) as total_req_qty
    FROM bom_tree GROUP BY child_item_code
)
-- 【Block 3: Inventory Allocation】
SELECT 
    req.child_item_code,
    i.item_name,
    req.total_req_qty AS required_qty,
    COALESCE(inv.quantity, 0) AS current_stock,
    MAX(req.total_req_qty - COALESCE(inv.quantity, 0), 0) AS shortage_qty
FROM grouped_req req
LEFT JOIN m_item i ON req.child_item_code = i.item_code
LEFT JOIN t_inventory inv ON req.child_item_code = inv.item_code
ORDER BY shortage_qty DESC
"""
display(pd.read_sql(sql_allocation, conn))

conn.close()

## 6. 価格履歴 (最新単価)
conn = sqlite3.connect('factory.db')

# --- Step 1: 現状データの確認 ---
print("--- Step 1: 履歴データ確認 ---")
display(pd.read_sql("SELECT item_code, start_date, unit_price FROM m_purchase_price WHERE item_code = 'MTR-DC-001'", conn))

# --- Step 2: 順位付け (ROW_NUMBER) ---
print("\n--- Step 2: 順位付け ---")
sql_rank = """
SELECT 
    item_code, start_date, unit_price,
    ROW_NUMBER() OVER (PARTITION BY item_code, supplier_code ORDER BY start_date DESC) as rn
FROM m_purchase_price
"""
display(pd.read_sql(sql_rank, conn).head())

# --- Step 3: 最新価格の確定 ---
print("\n--- Step 3: 最新価格のみ抽出 ---")
sql_latest = """
WITH ranked_price AS (
    SELECT 
        item_code, supplier_code, start_date, unit_price, moq,
        ROW_NUMBER() OVER (PARTITION BY item_code, supplier_code ORDER BY start_date DESC) as rn
    FROM m_purchase_price
)
SELECT * FROM ranked_price WHERE rn = 1
"""
display(pd.read_sql(sql_latest, conn).head())

conn.close()







