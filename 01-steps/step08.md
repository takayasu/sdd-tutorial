# Step 8: PBT導入（在庫ロットの状態遷移）

## 目的

### これは何か

Property-Based Testing（PBT）を導入する。具体的なテストケースを手書きする代わりに、「任意の入力に対して成り立つべき性質」をコードで記述し、ライブラリが自動的に反例を探す。

- Python: **hypothesis**（`@given` デコレータでランダム入力を生成）
- TypeScript: **fast-check**（`fc.property` でランダム入力を生成）

### なぜやるのか

- AIが生成したコードは「用意されたテストケース」をパスするが、境界値や組み合わせ爆発で壊れることが多い
- 「`LotNumber` の `year` が 2100 のとき何が起きるか」を人間が書かなくても自動で発見できる
- 状態遷移の「全パターン」を書くより「不変条件（invariant）」を書く方がテストが堅牢

### 何がうれしいのか

- バグが「見つけにくい入力」で発覚する
- 反例が見つかると最小化（shrinking）されて人間が読める形で出力される
- 仕様の抜け漏れが「テスト失敗」として可視化される

## 完了条件

```bash
# Python PBT テスト通過
$ cd backend && pytest tests/test_lot_properties.py -v
PASSED tests/test_lot_properties.py::test_lot_number_str_contains_year
PASSED tests/test_lot_properties.py::test_lot_number_roundtrip
PASSED tests/test_lot_properties.py::test_manufacturing_lot_status_is_always_manufacturing
PASSED tests/test_lot_properties.py::test_invalid_year_raises
$ echo $?
0

# TypeScript PBT テスト通過
$ cd frontend && npx vitest run tests/lot.property.test.ts
✓ tests/lot.property.test.ts (3 tests)
$ echo $?
0
```

---

## Python（hypothesis）

### 1. 依存確認（pyproject.toml）

```toml
[project.optional-dependencies]
dev = [
    ...
    "hypothesis>=6",
]
```

```bash
cd backend && uv sync
```

### 2. tests/test_lot_properties.py

```python
from __future__ import annotations

import pytest
from hypothesis import given
from hypothesis import strategies as st
from hypothesis.strategies import SearchStrategy

from src.domain.lot import LotCommon, LotNumber, ManufacturedLot, ManufacturingLot


# ---------- カスタムストラテジー ----------

lot_number_strategy: SearchStrategy[LotNumber] = st.builds(
    LotNumber,
    year=st.integers(min_value=2000, max_value=2099),
    location=st.text(
        min_size=1,
        max_size=10,
        alphabet=st.characters(whitelist_categories=("Lu", "Ll", "Nd")),
    ),
    seq=st.integers(min_value=1, max_value=9999),
)

lot_common_strategy: SearchStrategy[LotCommon] = st.builds(
    LotCommon,
    lot_number=lot_number_strategy,
    division_code=st.integers(min_value=1, max_value=99),
    department_code=st.integers(min_value=1, max_value=999),
    section_code=st.integers(min_value=1, max_value=9999),
    process_category=st.integers(min_value=1, max_value=9),
    inspection_category=st.integers(min_value=1, max_value=9),
    manufacturing_category=st.integers(min_value=1, max_value=9),
)


# ---------- プロパティ ----------

@given(lot_number_strategy)
def test_lot_number_str_contains_year(lot_number: LotNumber) -> None:
    """LotNumber の文字列表現は必ず year を含む"""
    assert str(lot_number.year) in str(lot_number)


@given(lot_number_strategy)
def test_lot_number_roundtrip(lot_number: LotNumber) -> None:
    """同じ値で作った LotNumber は等価（frozen dataclass の等値性）"""
    other = LotNumber(
        year=lot_number.year,
        location=lot_number.location,
        seq=lot_number.seq,
    )
    assert lot_number == other


@given(lot_common_strategy)
def test_manufacturing_lot_status_is_always_manufacturing(common: LotCommon) -> None:
    """ManufacturingLot の status は任意の入力に対して必ず 'manufacturing'"""
    lot = ManufacturingLot(common=common)
    assert lot.status == "manufacturing"


@given(st.integers())
def test_invalid_year_raises(year: int) -> None:
    """2000–2099 以外の year は ValueError を投げる"""
    if 2000 <= year <= 2099:
        return  # 有効な値はスキップ
    with pytest.raises(ValueError):
        LotNumber(year=year, location="TK", seq=1)


@given(lot_common_strategy, st.dates())
def test_manufactured_lot_preserves_lot_number(common: LotCommon, completed: ...) -> None:
    """状態遷移後もロット番号は変わらない"""
    manufactured = ManufacturedLot(
        common=common,
        manufacturing_completed_date=completed,
    )
    assert manufactured.common.lot_number == common.lot_number
```

### 3. よくある反例の読み方

```
Falsifying example: test_lot_number_str_contains_year(
    lot_number=LotNumber(year=2000, location='', seq=1)
)
```

- `location=''` が空文字 → `__post_init__` で弾かれるはずなのに通った → バグ発見
- hypothesis が最小の反例（shrinking）を自動生成する

---

## TypeScript（fast-check）

### 1. 依存追加

```bash
cd frontend
pnpm add -D fast-check
```

### 2. tests/lot.property.test.ts

```typescript
import { describe, it, expect } from 'vitest'
import * as fc from 'fast-check'
import { formatLotNumber } from '@/types/lot'
import type { LotNumber, ManufacturingLot } from '@/types/lot'

// ---------- Arbitrary（カスタム生成器） ----------

const lotNumberArb: fc.Arbitrary<LotNumber> = fc.record({
  year: fc.integer({ min: 2000, max: 2099 }),
  location: fc.stringOf(fc.alphaNumericChar(), { minLength: 1, maxLength: 10 }),
  seq: fc.integer({ min: 1, max: 9999 }),
})

// ---------- プロパティ ----------

describe('LotNumber properties', () => {
  it('formatLotNumber contains year', () => {
    fc.assert(
      fc.property(lotNumberArb, (lot) => {
        return formatLotNumber(lot).includes(String(lot.year))
      }),
    )
  })

  it('seq is zero-padded to 4 digits', () => {
    fc.assert(
      fc.property(lotNumberArb, (lot) => {
        const formatted = formatLotNumber(lot)
        const parts = formatted.split('-')
        const seqPart = parts[parts.length - 1]
        return seqPart.length === 4
      }),
    )
  })
})

describe('ManufacturingLot properties', () => {
  it('status is always manufacturing', () => {
    fc.assert(
      fc.property(lotNumberArb, (lotNumber) => {
        const lot: ManufacturingLot = {
          status: 'manufacturing',
          lotNumber,
          divisionCode: 1,
          departmentCode: 10,
          sectionCode: 100,
          processCategory: 1,
          inspectionCategory: 1,
          manufacturingCategory: 1,
        }
        return lot.status === 'manufacturing'
      }),
    )
  })
})
```

---

## hypothesis の設定（pyproject.toml）

```toml
[tool.hypothesis]
max_examples = 100
```

---

## 次のステップ

Step 8が完了したら [Step 9: テストカバレッジ](./step09.md) へ進む。
