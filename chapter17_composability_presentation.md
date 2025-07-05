# Robust Python 第17章: コンポーザビリティ
## 発表資料

---

## 目次
- 17.1 コンポーザビリティとは何か
- 17.2 ポリシーとメカニズム
- 17.3 小規模な部品のコンポーザビリティ
  - 17.3.1 コンポーザブルな関数
  - 17.3.2 コンポーザブルなアルゴリズム
- 17.4 まとめ

---

## 17.1 コンポーザビリティとは何か

### 定義
- **コンポーザビリティ**: 小さくて離散的で再利用可能なコンポーネントを作る設計原則
- 未来の開発者がシステムを変更する際の摩擦を減らす

### 目標
- 小さなコンポーネントを作成する
- 相互依存関係を最小限に抑える
- ビジネスロジックを内部に埋め込まない
- 読みやすく理解しやすいコードにする

### 課題
- 将来の開発者がシステムをどのように変更するかを予測するのは困難
- ビジネスは進化し、今日の前提が明日のレガシーシステムになる

---

## 17.2 ポリシーとメカニズム

### ポリシー vs メカニズム
- **ポリシー**: "何を" 行うか（ビジネスルール）
- **メカニズム**: "どのように" 行うか（実装方法）

### 分離の重要性
```python
# 悪い例: ポリシーとメカニズムが混在
class BLTMaker:
    def make_sandwich(self):
        # ポリシー: BLTの作り方
        # メカニズム: 具体的な調理手順
        bacon = self.cook_bacon()
        lettuce = self.prepare_lettuce()
        tomato = self.slice_tomato()
        return self.assemble_sandwich(bacon, lettuce, tomato)

# 良い例: ポリシーとメカニズムを分離
class IngredientPreparator:
    """食材の準備を担当するクラス（メカニズム）"""
    def cook_bacon(self): 
        """ベーコンを調理する"""
        pass
    
    def prepare_lettuce(self): 
        """レタスを準備する"""
        pass
    
    def slice_tomato(self): 
        """トマトをスライスする"""
        pass

class SandwichAssembler:
    """サンドイッチを組み立てるクラス（メカニズム）"""
    def assemble(self, *ingredients): 
        """材料を組み立ててサンドイッチを作る"""
        pass

class BLTRecipe:
    """BLTサンドイッチのレシピ（ポリシー）"""
    def __init__(self, preparator, assembler):
        # 依存性注入：具体的な実装に依存しない
        self.preparator = preparator
        self.assembler = assembler
    
    def make(self):
        """BLTサンドイッチを作る手順を定義"""
        # レシピに従って材料を準備
        ingredients = [
            self.preparator.cook_bacon(),      # ベーコン
            self.preparator.prepare_lettuce(), # レタス
            self.preparator.slice_tomato()     # トマト
        ]
        # 組み立てて完成
        return self.assembler.assemble(*ingredients)
```

---

## 17.3 小規模な部品のコンポーザビリティ

### 基本原則
- 小さなコンポーネントに分割する
- 各コンポーネントは単一の責任を持つ
- 組み合わせ可能な設計にする

### 実世界の例
#### Unixコマンドライン
```bash
# 専用プログラムではなく、小さなプログラムの組み合わせ
grep -i "ERROR" log.txt | cut -f 3,5 | sort -r
```

#### CI/CDパイプライン
```yaml
# GitHub Actionsの例
- name: Run tests
  run: pytest
- name: Security scan
  uses: securecodewarrior/github-action-add-sarif@v1
- name: Deploy
  uses: actions/deploy-pages@v1
```

---

## 17.3.1 コンポーザブルな関数

### 関数の組み合わせ
```python
# 基本的な関数（再利用可能な小さな単位）
def add(x, y):
    """2つの数値を加算する"""
    return x + y

def multiply(x, y):
    """2つの数値を乗算する"""
    return x * y

def square(x):
    """数値を2乗する（既存の関数を再利用）"""
    return multiply(x, x)

# 高階関数による組み合わせ
def compose(f, g):
    """関数合成：f(g(x))を実行する高階関数"""
    return lambda x: f(g(x))

# 使用例
def add_five(x):
    """5を加算する特化関数"""
    return add(x, 5)

def square_then_add_five(x):
    """2乗してから5を加算する合成関数"""
    return compose(add_five, square)(x)

# 関数型スタイル
from functools import reduce
from operator import add

def sum_of_squares(numbers):
    """数値リストの各要素を2乗して合計する"""
    # map(square, numbers): 各要素を2乗
    # reduce(add, ...): 全てを加算
    return reduce(add, map(square, numbers))
```

### 純粋関数の利点
```python
# 純粋関数: 副作用なし、同じ入力に対して同じ出力
def calculate_tax(amount, rate):
    """税金を計算する純粋関数"""
    return amount * rate

# 不純な関数: 副作用あり
class TaxCalculator:
    """状態を持つ不純な税金計算器"""
    def __init__(self):
        self.total_collected = 0  # 状態を保持
    
    def calculate_tax(self, amount, rate):
        """税金を計算し、同時に累計を更新（副作用）"""
        tax = amount * rate
        self.total_collected += tax  # 副作用：状態を変更
        return tax

# 組み合わせ可能な設計
def calculate_total_with_tax(amount, tax_rate, discount_rate=0):
    """割引と税金を適用した合計を計算"""
    # 段階的な計算：各ステップが純粋関数
    discounted_amount = amount * (1 - discount_rate)  # 割引適用
    tax = calculate_tax(discounted_amount, tax_rate)   # 税金計算
    return discounted_amount + tax                      # 合計計算
```

---

## 17.3.2 コンポーザブルなアルゴリズム

### アルゴリズムの分解
```python
# 悪い例: 単一の大きな関数（モノリシック）
def process_orders(orders):
    """注文を処理する単一の大きな関数"""
    result = []
    for order in orders:
        # 複数の条件とロジックが混在
        if order.status == 'pending' and order.total > 100:
            discounted_total = order.total * 0.9
            if discounted_total > 50:
                tax = discounted_total * 0.1
                final_total = discounted_total + tax
                result.append({
                    'id': order.id,
                    'total': final_total,
                    'processed': True
                })
    return result

# 良い例: 組み合わせ可能なアルゴリズム
def is_eligible_order(order):
    """注文が処理対象かどうかを判定"""
    return order.status == 'pending' and order.total > 100

def apply_discount(total, discount_rate=0.1):
    """割引を適用する"""
    return total * (1 - discount_rate)

def calculate_tax(amount, tax_rate=0.1):
    """税金を計算する"""
    return amount * tax_rate

def meets_minimum_threshold(amount, threshold=50):
    """最小閾値を満たしているかチェック"""
    return amount > threshold

def process_single_order(order):
    """単一の注文を処理する"""
    # 早期リターン：対象外の場合
    if not is_eligible_order(order):
        return None
    
    # 割引適用
    discounted_total = apply_discount(order.total)
    
    # 最小閾値チェック
    if not meets_minimum_threshold(discounted_total):
        return None
    
    # 税金計算と最終合計
    tax = calculate_tax(discounted_total)
    final_total = discounted_total + tax
    
    # 結果を構築
    return {
        'id': order.id,
        'total': final_total,
        'processed': True
    }

def process_orders(orders):
    """複数の注文を処理する"""
    # ジェネレータ式とリスト内包表記を使用
    return [
        result for result in 
        (process_single_order(order) for order in orders)
        if result is not None
    ]
```

### イテレータパターンの活用
```python
from typing import Iterator, Callable, TypeVar

T = TypeVar('T')  # 入力の型
U = TypeVar('U')  # 出力の型

def filter_items(items: Iterator[T], predicate: Callable[[T], bool]) -> Iterator[T]:
    """条件に合致するアイテムのみを返すフィルタ"""
    return filter(predicate, items)

def transform_items(items: Iterator[T], transformer: Callable[[T], U]) -> Iterator[U]:
    """各アイテムを変換する"""
    return map(transformer, items)

def take(items: Iterator[T], n: int) -> Iterator[T]:
    """最初のn個のアイテムのみを取得"""
    for i, item in enumerate(items):
        if i >= n:
            break
        yield item

# 組み合わせ例：データ処理パイプライン
def process_data_pipeline(data):
    """データ処理のパイプライン"""
    return take(                               # 最初の10個を取得
        transform_items(                       # 各値を2乗
            filter_items(                      # 正の値のみフィルタ
                data, 
                lambda x: x > 0               # 条件：正の値
            ),
            lambda x: x ** 2                  # 変換：2乗
        ),
        10                                    # 取得数：10個
    )

# 使用例
numbers = [1, -2, 3, -4, 5, 6, -7, 8, 9, 10, 11, 12]
result = list(process_data_pipeline(numbers))
# 結果: [1, 9, 25, 36, 64, 81, 100, 121, 144] (最初の10個の正の値の2乗)
```

---

## 17.4 まとめ

### コンポーザビリティの利点
1. **再利用性**: 小さなコンポーネントは異なる文脈で再利用できる
2. **テスト容易性**: 小さな単位でテストが可能
3. **保守性**: 変更が局所化される
4. **拡張性**: 新しい機能を既存コンポーネントの組み合わせで実現

### 設計原則
- **単一責任**: 各コンポーネントは一つのことをうまくやる
- **疎結合**: コンポーネント間の依存関係を最小限に抑える
- **高凝集**: 関連する機能を一つのコンポーネントにまとめる

### 実践のヒント
```python
# 1. 小さな関数に分割
def validate_input(data):
    """入力データのバリデーション"""
    # バリデーションロジック
    if not data:
        raise ValueError("データが空です")
    return data

def transform_data(data):
    """データの変換処理"""
    # 変換ロジック
    return [item.upper() for item in data]

def save_data(data):
    """データの保存処理"""
    # 保存ロジック
    print(f"保存されました: {data}")

def process_data(raw_data):
    """データ処理のメイン関数"""
    # 各ステップを組み合わせて処理
    validated_data = validate_input(raw_data)    # 検証
    transformed_data = transform_data(validated_data)  # 変換
    save_data(transformed_data)                  # 保存
    return transformed_data

# 2. 設定可能なパラメータ（高階関数）
def create_processor(validator, transformer, saver):
    """処理関数を組み合わせてカスタム処理器を作成"""
    def process(data):
        # 3つの処理を順番に実行
        return saver(transformer(validator(data)))
    return process

# 3. 関数型プログラミングの活用
from functools import partial

def process_with_config(validator_config, transformer_config):
    """設定に基づいて処理器を部分適用で作成"""
    return partial(
        create_processor,
        validator=create_validator(validator_config),     # 検証器を設定
        transformer=create_transformer(transformer_config)  # 変換器を設定
    )

# 使用例
def create_validator(config):
    """設定に基づいてバリデータを作成"""
    return lambda data: validate_input(data)

def create_transformer(config):
    """設定に基づいて変換器を作成"""
    return lambda data: transform_data(data)
```

### 次のステップ
- より大きなシステムでのコンポーザビリティ
- マイクロサービスアーキテクチャ
- 関数型プログラミングパラダイム
- デザインパターンとの組み合わせ

---

## 質疑応答

**コンポーザビリティを意識した設計で、未来の開発者が喜ぶコードを書きましょう！**