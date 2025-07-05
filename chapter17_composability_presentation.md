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
    def cook_bacon(self): pass
    def prepare_lettuce(self): pass
    def slice_tomato(self): pass

class SandwichAssembler:
    def assemble(self, *ingredients): pass

class BLTRecipe:
    def __init__(self, preparator, assembler):
        self.preparator = preparator
        self.assembler = assembler
    
    def make(self):
        ingredients = [
            self.preparator.cook_bacon(),
            self.preparator.prepare_lettuce(),
            self.preparator.slice_tomato()
        ]
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
# 基本的な関数
def add(x, y):
    return x + y

def multiply(x, y):
    return x * y

def square(x):
    return multiply(x, x)

# 高階関数による組み合わせ
def compose(f, g):
    return lambda x: f(g(x))

# 使用例
def add_five(x):
    return add(x, 5)

def square_then_add_five(x):
    return compose(add_five, square)(x)

# 関数型スタイル
from functools import reduce
from operator import add

def sum_of_squares(numbers):
    return reduce(add, map(square, numbers))
```

### 純粋関数の利点
```python
# 純粋関数: 副作用なし、同じ入力に対して同じ出力
def calculate_tax(amount, rate):
    return amount * rate

# 不純な関数: 副作用あり
class TaxCalculator:
    def __init__(self):
        self.total_collected = 0
    
    def calculate_tax(self, amount, rate):
        tax = amount * rate
        self.total_collected += tax  # 副作用
        return tax

# 組み合わせ可能な設計
def calculate_total_with_tax(amount, tax_rate, discount_rate=0):
    discounted_amount = amount * (1 - discount_rate)
    tax = calculate_tax(discounted_amount, tax_rate)
    return discounted_amount + tax
```

---

## 17.3.2 コンポーザブルなアルゴリズム

### アルゴリズムの分解
```python
# 悪い例: 単一の大きな関数
def process_orders(orders):
    result = []
    for order in orders:
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
    return order.status == 'pending' and order.total > 100

def apply_discount(total, discount_rate=0.1):
    return total * (1 - discount_rate)

def calculate_tax(amount, tax_rate=0.1):
    return amount * tax_rate

def meets_minimum_threshold(amount, threshold=50):
    return amount > threshold

def process_single_order(order):
    if not is_eligible_order(order):
        return None
    
    discounted_total = apply_discount(order.total)
    
    if not meets_minimum_threshold(discounted_total):
        return None
    
    tax = calculate_tax(discounted_total)
    final_total = discounted_total + tax
    
    return {
        'id': order.id,
        'total': final_total,
        'processed': True
    }

def process_orders(orders):
    return [
        result for result in 
        (process_single_order(order) for order in orders)
        if result is not None
    ]
```

### イテレータパターンの活用
```python
from typing import Iterator, Callable, TypeVar

T = TypeVar('T')
U = TypeVar('U')

def filter_items(items: Iterator[T], predicate: Callable[[T], bool]) -> Iterator[T]:
    return filter(predicate, items)

def transform_items(items: Iterator[T], transformer: Callable[[T], U]) -> Iterator[U]:
    return map(transformer, items)

def take(items: Iterator[T], n: int) -> Iterator[T]:
    for i, item in enumerate(items):
        if i >= n:
            break
        yield item

# 組み合わせ例
def process_data_pipeline(data):
    return take(
        transform_items(
            filter_items(data, lambda x: x > 0),
            lambda x: x ** 2
        ),
        10
    )
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
    # バリデーションロジック
    pass

def transform_data(data):
    # 変換ロジック
    pass

def save_data(data):
    # 保存ロジック
    pass

def process_data(raw_data):
    validated_data = validate_input(raw_data)
    transformed_data = transform_data(validated_data)
    save_data(transformed_data)
    return transformed_data

# 2. 設定可能なパラメータ
def create_processor(validator, transformer, saver):
    def process(data):
        return saver(transformer(validator(data)))
    return process

# 3. 関数型プログラミングの活用
from functools import partial

def process_with_config(validator_config, transformer_config):
    return partial(
        create_processor,
        validator=create_validator(validator_config),
        transformer=create_transformer(transformer_config)
    )
```

### 次のステップ
- より大きなシステムでのコンポーザビリティ
- マイクロサービスアーキテクチャ
- 関数型プログラミングパラダイム
- デザインパターンとの組み合わせ

---

## 質疑応答

**コンポーザビリティを意識した設計で、未来の開発者が喜ぶコードを書きましょう！**