```markdown
# Step 2: Chuyển Đổi Cấu Trúc Dữ Liệu Lõi Sang Khung Tĩnh (ScriptableObjects)
## Cursor Instructions — TCG Shop Simulator (Unity Port)

**Phiên bản tài liệu:** 1.0  
**Giai đoạn:** Data Foundation Layer  
**Yêu cầu tiên quyết:** Bước 1 đã hoàn thành, `GameManager.cs` đang hoạt động, Console in `[GameManager] Ready.` không lỗi.

---

## 1. Mục Tiêu Của Bước Này

Hệ thống cũ (Vue/Phaser) lưu trữ dữ liệu thẻ bài trong SQLite, truy vấn runtime qua Web Worker, và tính giá động theo công thức JavaScript. Kiến trúc đó phù hợp với web nhưng không phù hợp với Unity vì:

- SQLite không có trong Unity build mặc định, cần thư viện ngoài
- Truy vấn async từ Web Worker không map tốt sang Unity coroutine
- Dữ liệu phân tán trong nhiều store (apiStore, inventoryStore) khó debug

**Giải pháp Unity:** ScriptableObject — asset tĩnh, serialize bởi Unity Editor, load nhanh từ disk, không cần runtime query, kiểm tra được trong Inspector.

Bước này thiết lập ba trụ cột dữ liệu:

1. **`CardData` ScriptableObject** — Đơn vị dữ liệu tối thiểu của một lá bài
2. **`PackData` ScriptableObject** — Định nghĩa một gói booster với bảng tỷ lệ rơi (Drop Table)
3. **`InventoryManager`** — Singleton quản lý số lượng thẻ và pack trong kho với O(1) lookup
4. **Gacha RNG Engine** — Thuật toán Cumulative Weighted Probability thay thế `ORDER BY RANDOM()` của SQLite cũ

**Kết quả mong đợi:** Mở 10 pack tự động trong `Start()`, Console in thống kê tỷ lệ rớt thẻ chứng minh RNG phân phối đúng theo bảng cấu hình.

---

## 2. Danh Sách Files Cần Tạo

Cursor phải tạo đúng các file sau, đặt đúng thư mục. Không được tự ý thay đổi đường dẫn.

```
Assets/
├── Scripts/
│   ├── Data/
│   │   ├── CardData.cs                  ← ScriptableObject định nghĩa lá bài
│   │   ├── PackData.cs                  ← ScriptableObject định nghĩa gói booster
│   │   ├── RarityDefinition.cs          ← ScriptableObject định nghĩa bậc hiếm
│   │   └── CardDatabase.cs              ← ScriptableObject tổng hợp toàn bộ catalogue
│   ├── Inventory/
│   │   ├── InventoryManager.cs          ← Singleton quản lý kho đồ
│   │   └── InventoryEntry.cs            ← Data class cho một slot kho
│   ├── Gacha/
│   │   ├── GachaEngine.cs               ← Thuật toán Cumulative Weighted Probability
│   │   └── GachaResult.cs               ← Data class kết quả một lần mở pack
│   └── Debug/
│       └── GachaDebugTester.cs          ← Test tự động 10 pack, in thống kê
│
├── ScriptableObjects/
│   ├── Rarities/                        ← Chứa các RarityDefinition assets
│   │   ├── Common.asset
│   │   ├── Uncommon.asset
│   │   ├── Rare.asset
│   │   ├── DoubleRare.asset
│   │   ├── UltraRare.asset
│   │   ├── IllustrationRare.asset
│   │   ├── SpecialIllustrationRare.asset
│   │   ├── SecretRare.asset
│   │   └── GhostRare.asset
│   ├── Cards/                           ← Chứa CardData assets (tạo mẫu 5 thẻ)
│   │   ├── Card_Charizard_001.asset
│   │   ├── Card_Pikachu_002.asset
│   │   ├── Card_Mewtwo_003.asset
│   │   ├── Card_Bulbasaur_004.asset
│   │   └── Card_Squirtle_005.asset
│   ├── Packs/                           ← Chứa PackData assets (tạo mẫu 1 pack)
│   │   └── Pack_BaseSet_001.asset
│   └── Database/
│       └── CardDatabase_Main.asset      ← Database tổng hợp
│
└── Editor/
    └── CardDataEditor.cs                ← Custom Inspector cho CardData
```

---

## 3. Lý Thuyết Nền: Tại Sao Không Dùng Random Thô

### 3.1 Vấn Đề Của Hệ Thống Cũ

Hệ thống Vue/Phaser dùng:
```sql
SELECT * FROM cards WHERE set_id = ? ORDER BY RANDOM() LIMIT 6
```

Cách này có ba vấn đề nghiêm trọng:

**Vấn đề 1 — Không kiểm soát được tỷ lệ:** Nếu database có 100 Common và 5 Ultra Rare, xác suất ra Ultra Rare là đúng 5%, không thể điều chỉnh theo game design.

**Vấn đề 2 — Không có guarantee:** Có thể mở 50 pack liên tiếp không ra Rare nào (chuỗi xui xéo dài vô hạn).

**Vấn đề 3 — Phụ thuộc database:** Thêm card vào database thay đổi tỷ lệ rơi — side effect nguy hiểm.

### 3.2 Giải Pháp: Cumulative Weighted Probability

Thay vì random từ pool, ta random từ **bảng tỷ lệ cố định**:

```
Bảng Drop Table của một Pack (6 cards):
  Slot 1-4: Guaranteed Common/Uncommon
  Slot 5:   70% Common, 25% Uncommon, 5% Rare
  Slot 6:   40% Rare, 35% Double Rare, 15% Ultra Rare, 10% Secret Rare+
```

**Cumulative Weighted Probability** hoạt động như sau:

```
Cho bảng weights: [Common=50, Uncommon=30, Rare=15, Ultra=5]
Tổng = 100

Cumulative thresholds:
  Common:   0  → 50   (nếu random [0,100) < 50  → Common)
  Uncommon: 50 → 80   (nếu random [0,100) < 80  → Uncommon)
  Rare:     80 → 95   (nếu random [0,100) < 95  → Rare)
  Ultra:    95 → 100  (nếu random [0,100) < 100 → Ultra)

Ví dụ: roll = 73.5
  73.5 >= 50  → không phải Common
  73.5 < 80   → là Uncommon ✓
```

Đây là cơ chế chuẩn công nghiệp (dùng trong Hearthstone, Pokemon TCG Online, v.v.).

---

## 4. Chi Tiết Kỹ Thuật Từng File

### 4.1 `RarityDefinition.cs`

**Vị trí:** `Assets/Scripts/Data/RarityDefinition.cs`  
**Mục đích:** ScriptableObject định nghĩa một bậc hiếm. Tách riêng để có thể thêm bậc hiếm mới mà không sửa code.

```csharp
// Assets/Scripts/Data/RarityDefinition.cs

using UnityEngine;

/// <summary>
/// ScriptableObject định nghĩa một bậc hiếm (Rarity Tier) trong game.
/// 
/// CÁCH TẠO ASSET:
///   Right-click trong Project > Create > TCGShop > Data > Rarity Definition
///
/// MAPPING TỪ HỆ THỐNG CŨ (Vue/Phaser rarityRegistry.ts):
///   'Common'                    → sortingRank = 0
///   'Uncommon'                  → sortingRank = 1
///   'Rare' / 'Rare Holo'        → sortingRank = 3
///   'Double Rare'               → sortingRank = 4
///   'Ultra Rare'                → sortingRank = 5
///   'Illustration Rare'         → sortingRank = 7
///   'Special Illustration Rare' → sortingRank = 8
///   'Secret Rare'               → sortingRank = 6
///   'Ghost Rare'                → sortingRank = 10
/// </summary>
[CreateAssetMenu(
    fileName = "Rarity_New",
    menuName = "TCGShop/Data/Rarity Definition",
    order = 0
)]
public class RarityDefinition : ScriptableObject
{
    // =========================================================================
    // THÔNG TIN CƠ BẢN
    // =========================================================================

    [Header("Identity")]
    [Tooltip("Tên hiển thị trong UI. Khớp với chuỗi rarity trong database cũ.")]
    public string displayName = "Common";

    [Tooltip("Màu sắc đại diện cho bậc hiếm này trong UI.")]
    public Color rarityColor = Color.white;

    [Tooltip("Icon đại diện (tùy chọn).")]
    public Sprite rarityIcon;

    // =========================================================================
    // THÔNG SỐ SẮP XẾP
    // =========================================================================

    [Header("Sorting")]
    [Tooltip("Thứ hạng để sort thẻ theo độ hiếm. Cao hơn = hiếm hơn. " +
             "Khớp với rarityRank trong inventoryStore.ts cũ.")]
    [Range(0, 10)]
    public int sortingRank = 0;

    [Tooltip("Nếu true, thẻ này được coi là High Rarity và có hiệu ứng holo. " +
             "Khớp với isHighRarity() trong rarityRegistry.ts cũ.")]
    public bool isHighRarity = false;

    // =========================================================================
    // THÔNG SỐ GACHA
    // =========================================================================

    [Header("XP Reward")]
    [Tooltip("XP người chơi nhận được khi mở được thẻ bậc này. " +
             "High rarity → 15 XP, Common/Uncommon → 2 XP (từ inventoryStore.ts cũ).")]
    public int xpReward = 2;

    // =========================================================================
    // TIỆN ÍCH
    // =========================================================================

    /// <summary>
    /// Kiểm tra nhanh xem đây có phải bậc hiếm cao không.
    /// Tương đương isHighRarity() trong rarityRegistry.ts cũ (rank >= 3).
    /// </summary>
    public bool IsHighRarity => sortingRank >= 3;

    /// <summary>
    /// Chuỗi hiển thị đầy đủ dùng cho logging và debug.
    /// </summary>
    public override string ToString() => $"Rarity[{displayName}, Rank={sortingRank}]";
}
```

---

### 4.2 `CardData.cs`

**Vị trí:** `Assets/Scripts/Data/CardData.cs`  
**Mục đích:** ScriptableObject lưu trữ toàn bộ thông tin của một lá bài. Tương đương một row trong bảng `cards` của SQLite cũ.

**Mapping từ schema cũ:**
```
SQLite cards table    →   CardData ScriptableObject
─────────────────────────────────────────────────────
id (TEXT)             →   cardId (string)
name (TEXT)           →   cardName (string)
hp (TEXT)             →   baseHp (int) — đã parse từ string
rarity (TEXT)         →   rarity (RarityDefinition)
types (JSON array)    →   cardTypes (EnergyType[])
attacks (JSON)        →   attacks (AttackData[])
weaknesses (JSON)     →   weaknesses (TypeModifier[])
resistances (JSON)    →   resistances (TypeModifier[])
retreatCost (INTEGER) →   retreatCost (int)
pricing (JSON)        →   marketValue (float) — đã extract
image (TEXT URL)      →   cardSprite (Sprite)
set_id (TEXT)         →   setId (string)
series_id (TEXT)      →   seriesId (string)
```

```csharp
// Assets/Scripts/Data/CardData.cs

using UnityEngine;

/// <summary>
/// ScriptableObject lưu trữ toàn bộ dữ liệu tĩnh của một lá bài TCG.
///
/// THIẾT KẾ:
/// - Dữ liệu TĨnh (không thay đổi trong gameplay): lưu ở đây.
/// - Dữ liệu ĐỘNG (số lượng trong kho, giá đang bán): lưu trong InventoryManager.
///
/// CÁCH TẠO ASSET:
///   Right-click trong Project > Create > TCGShop > Data > Card Data
///
/// CÁCH TẠO HÀNG LOẠT:
///   Dùng CardDataEditor (custom Inspector) hoặc
///   Dùng CardDataFactory (Editor script) để import từ CSV/JSON.
/// </summary>
[CreateAssetMenu(
    fileName = "Card_NewCard_000",
    menuName = "TCGShop/Data/Card Data",
    order = 1
)]
public class CardData : ScriptableObject
{
    // =========================================================================
    // IDENTITY — Tương đương cột id, name, set_id, series_id trong SQLite cũ
    // =========================================================================

    [Header("Identity")]
    [Tooltip("ID duy nhất của thẻ. Format: {setId}-{number}. Vd: 'sv01-001'. " +
             "Phải khớp với id trong database cũ để migration đúng.")]
    public string cardId;

    [Tooltip("Tên hiển thị của thẻ. Vd: 'Charizard ex'.")]
    public string cardName;

    [Tooltip("ID của set chứa thẻ này. Vd: 'sv01'. Dùng để group thẻ theo pack.")]
    public string setId;

    [Tooltip("ID của series. Vd: 'sv'. Dùng để tính requiredLevel của pack.")]
    public string seriesId;

    [Tooltip("Số thứ tự trong set. Vd: '001'. Dùng để sort và display.")]
    public string cardNumber;

    // =========================================================================
    // VISUAL — Thay thế URL trong SQLite cũ bằng Sprite reference Unity
    // =========================================================================

    [Header("Visual")]
    [Tooltip("Sprite mặt trước của thẻ. Thay thế image URL trong database cũ. " +
             "Import texture với settings: Sprite (2D and UI), Max Size 512.")]
    public Sprite cardSprite;

    [Tooltip("Sprite mặt sau thẻ bài (dùng chung cho mọi thẻ cùng set).")]
    public Sprite cardBackSprite;

    // =========================================================================
    // RARITY — Thay thế chuỗi rarity string trong SQLite cũ
    // =========================================================================

    [Header("Rarity")]
    [Tooltip("Bậc hiếm của thẻ. Reference đến RarityDefinition ScriptableObject. " +
             "Thay thế chuỗi string 'Ultra Rare' trong database cũ.")]
    public RarityDefinition rarity;

    // =========================================================================
    // BATTLE STATS — Dùng cho hệ thống Battle (Bước 5)
    // =========================================================================

    [Header("Battle Stats")]
    [Tooltip("HP của thẻ. Trong SQLite cũ lưu dạng string '120', đã parse thành int.")]
    [Range(10, 340)]
    public int baseHp = 60;

    [Tooltip("Hệ năng lượng của thẻ. Tương đương mảng types trong SQLite cũ.")]
    public EnergyType[] cardTypes;

    [Tooltip("Chi phí rút lui. Tương đương retreatCost INTEGER trong SQLite cũ.")]
    [Range(0, 5)]
    public int retreatCost = 1;

    [Tooltip("Danh sách đòn tấn công.")]
    public AttackData[] attacks;

    [Tooltip("Điểm yếu của thẻ (×2 damage).")]
    public TypeModifier[] weaknesses;

    [Tooltip("Kháng cự của thẻ (-30 damage).")]
    public TypeModifier[] resistances;

    // =========================================================================
    // ECONOMY — Tương đương pricing JSON trong SQLite cũ
    // =========================================================================

    [Header("Economy")]
    [Tooltip("Giá thị trường tham chiếu (USD). " +
             "Tương đương tcgplayer.normal.marketPrice trong pricing JSON cũ. " +
             "Dùng để tính giá mua/bán mặc định của pack.")]
    [Min(0f)]
    public float marketValue = 0.99f;

    [Tooltip("Họa sĩ vẽ thẻ. Dùng cho UI chi tiết.")]
    public string artistName;

    // =========================================================================
    // TIỆN ÍCH
    // =========================================================================

    /// <summary>
    /// Kiểm tra nhanh thẻ có phải High Rarity không.
    /// Tương đương isHighRarity() trong rarityRegistry.ts cũ.
    /// </summary>
    public bool IsHighRarity => rarity != null && rarity.IsHighRarity;

    /// <summary>
    /// Sorting rank của thẻ. Dùng để sort kết quả gacha (thẻ hiếm nhất ở cuối).
    /// Tương đương getRarityRank() trong inventoryStore.ts cũ.
    /// </summary>
    public int RarityRank => rarity != null ? rarity.sortingRank : 0;

    /// <summary>
    /// XP người chơi nhận khi mở được thẻ này.
    /// Tương đương logic gainExp trong tearPack() của inventoryStore.ts cũ.
    /// </summary>
    public int XpReward => rarity != null ? rarity.xpReward : 2;

    /// <summary>
    /// Validation: Kiểm tra tất cả field bắt buộc đã được điền.
    /// </summary>
    public bool IsValid()
    {
        if (string.IsNullOrEmpty(cardId)) return false;
        if (string.IsNullOrEmpty(cardName)) return false;
        if (rarity == null) return false;
        return true;
    }

    public override string ToString() =>
        $"Card[{cardId}|{cardName}|{rarity?.displayName ?? "NoRarity"}|${marketValue:F2}]";
}

// =============================================================================
// SUPPORTING DATA STRUCTURES
// =============================================================================

/// <summary>
/// Enum các hệ năng lượng trong Pokémon TCG.
/// Tương đương EnergyType type trong battle/types/index.ts cũ.
/// </summary>
public enum EnergyType
{
    Colorless,
    Fire,
    Water,
    Grass,
    Lightning,
    Psychic,
    Fighting,
    Darkness,
    Metal,
    Dragon,
    Fairy
}

/// <summary>
/// Dữ liệu một đòn tấn công.
/// Tương đương ParsedAttack interface trong battle/types/index.ts cũ.
/// </summary>
[System.Serializable]
public class AttackData
{
    [Tooltip("Tên đòn đánh.")]
    public string attackName;

    [Tooltip("Sát thương cơ bản. 0 nếu đòn không gây damage trực tiếp.")]
    [Min(0)]
    public int baseDamage;

    [Tooltip("Chi phí năng lượng để dùng đòn này.")]
    public EnergyType[] energyCost;

    [Tooltip("Mô tả hiệu ứng đặc biệt của đòn.")]
    [TextArea(2, 4)]
    public string effectDescription;
}

/// <summary>
/// Modifier cho Weakness hoặc Resistance.
/// Tương đương {type, value} object trong weaknesses/resistances JSON cũ.
/// </summary>
[System.Serializable]
public class TypeModifier
{
    public EnergyType energyType;

    [Tooltip("Weakness: nhập '×2'. Resistance: nhập '-30'.")]
    public string value;
}
```

---

### 4.3 `PackData.cs`

**Vị trí:** `Assets/Scripts/Data/PackData.cs`  
**Mục đích:** ScriptableObject định nghĩa một gói booster pack với bảng Drop Table. Đây là nơi duy nhất chứa tỷ lệ rơi thẻ — không được hard-code ở bất kỳ đâu khác.

**Mapping từ hệ thống cũ:**
```
StockItemInfo (inventoryStore.ts cũ)  →  PackData ScriptableObject
──────────────────────────────────────────────────────────────────
id: "pack_sv01"                        →  packId
name: "SV01 Booster Pack"             →  packName
buyPrice: float                        →  buyCost
sellPrice: float                       →  defaultSellPrice
requiredLevel: int                     →  requiredShopLevel
sourceSetId: string                    →  sourceSetId
generation: string                     →  generationName
-- MỚI THÊM --
dropTable: DropTableSlot[]            ← KHÔNG CÓ trong hệ thống cũ
cardPool: CardData[]                  ← Thay thế query SQL
```

```csharp
// Assets/Scripts/Data/PackData.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// ScriptableObject định nghĩa một gói Booster Pack.
/// Chứa bảng Drop Table xác định TỶ LỆ RƠI chính xác cho từng slot.
///
/// KIẾN TRÚC DROP TABLE:
/// Mỗi pack có N slot (mặc định 6). Mỗi slot có bảng weight riêng.
/// Slot 1-4: Common/Uncommon guaranteed.
/// Slot 5:   Mix Common-Rare.
/// Slot 6:   Rare guaranteed slot (Ultra/Secret có thể xuất hiện).
///
/// CÁCH TẠO ASSET:
///   Right-click trong Project > Create > TCGShop > Data > Pack Data
/// </summary>
[CreateAssetMenu(
    fileName = "Pack_NewPack_001",
    menuName = "TCGShop/Data/Pack Data",
    order = 2
)]
public class PackData : ScriptableObject
{
    // =========================================================================
    // IDENTITY
    // =========================================================================

    [Header("Identity")]
    [Tooltip("ID duy nhất. Format: 'pack_{setId}'. Vd: 'pack_sv01'. " +
             "Phải khớp với id trong shopItems của hệ thống cũ để migration.")]
    public string packId;

    [Tooltip("Tên hiển thị. Vd: 'Scarlet & Violet Booster Pack'.")]
    public string packName;

    [Tooltip("ID của set mà pack này thuộc về. Vd: 'sv01'. " +
             "Dùng để tìm cards trong CardDatabase.")]
    public string sourceSetId;

    [Tooltip("Tên generation. Vd: 'GENERATION IX'. " +
             "Tương đương generation field trong StockItemInfo cũ.")]
    public string generationName;

    // =========================================================================
    // VISUAL
    // =========================================================================

    [Header("Visual")]
    [Tooltip("Sprite ảnh mặt trước của vỏ pack.")]
    public Sprite packFrontSprite;

    [Tooltip("Sprite ảnh mặt sau của vỏ pack.")]
    public Sprite packBackSprite;

    // =========================================================================
    // ECONOMY — Tương đương StockItemInfo pricing trong hệ thống cũ
    // =========================================================================

    [Header("Economy")]
    [Tooltip("Giá mua pack từ nhà cung cấp (USD). " +
             "Tương đương buyPrice trong StockItemInfo cũ.")]
    [Min(0.01f)]
    public float buyCost = 4.99f;

    [Tooltip("Giá bán mặc định cho khách (USD). Player có thể điều chỉnh. " +
             "Tương đương sellPrice trong StockItemInfo cũ. " +
             "Thường = buyCost × 1.6 (markup 60%).")]
    [Min(0.01f)]
    public float defaultSellPrice = 7.99f;

    [Tooltip("Level shop tối thiểu để mở khóa pack này. " +
             "Tương đương requiredLevel trong StockItemInfo cũ.")]
    [Range(1, 80)]
    public int requiredShopLevel = 1;

    // =========================================================================
    // CARD POOL — Thay thế SQL query trong hệ thống cũ
    // =========================================================================

    [Header("Card Pool")]
    [Tooltip("Danh sách tất cả thẻ có thể rơi ra từ pack này. " +
             "Thay thế query: SELECT * FROM cards WHERE set_id = ? trong hệ thống cũ. " +
             "Kéo thả CardData assets vào đây.")]
    public List<CardData> availableCards = new List<CardData>();

    // =========================================================================
    // DROP TABLE — KHÔNG TỒN TẠI trong hệ thống cũ (dùng ORDER BY RANDOM())
    // Đây là tính năng MỚI HOÀN TOÀN — xem GachaEngine.cs để hiểu cách dùng
    // =========================================================================

    [Header("Drop Table (NEW — Replaces ORDER BY RANDOM())")]
    [Tooltip("Số lượng thẻ rơi ra mỗi lần mở pack. Mặc định 6 (giống hệ thống cũ).")]
    [Range(1, 10)]
    public int cardsPerPack = 6;

    [Tooltip("Bảng Drop Table: mỗi phần tử = một slot trong pack, với bảng weight riêng. " +
             "Thứ tự slot quan trọng: slot đầu tiên = thẻ đầu tiên lật ra. " +
             "Slot cuối cùng = thẻ hiếm nhất (giống hệ thống cũ để thẻ hiếm ở cuối).")]
    public List<DropTableSlot> dropTable = new List<DropTableSlot>();

    // =========================================================================
    // COMPUTED PROPERTIES
    // =========================================================================

    /// <summary>
    /// Profit margin khi bán ở giá mặc định.
    /// Tương đương công thức profit = sellPrice - buyPrice trong SetPriceModal.vue cũ.
    /// </summary>
    public float DefaultProfitMargin => defaultSellPrice - buyCost;

    /// <summary>
    /// Profit margin theo phần trăm.
    /// Tương đương profitPct = (sellPrice/buyPrice - 1) × 100 trong SetPriceModal.vue cũ.
    /// </summary>
    public float DefaultProfitPercent =>
        buyCost > 0 ? (defaultSellPrice / buyCost - 1f) * 100f : 0f;

    /// <summary>
    /// Kiểm tra Pack có đủ dữ liệu để mở không.
    /// </summary>
    public bool IsValid()
    {
        if (string.IsNullOrEmpty(packId)) return false;
        if (availableCards == null || availableCards.Count == 0) return false;
        if (dropTable == null || dropTable.Count == 0) return false;
        return true;
    }

    /// <summary>
    /// Tìm tất cả cards trong pool có rarity khớp với một RarityDefinition.
    /// Dùng bởi GachaEngine để lấy pool thẻ cho từng slot.
    /// </summary>
    public List<CardData> GetCardsByRarity(RarityDefinition targetRarity)
    {
        var result = new List<CardData>();
        foreach (var card in availableCards)
        {
            if (card != null && card.rarity == targetRarity)
            {
                result.Add(card);
            }
        }
        return result;
    }

    public override string ToString() =>
        $"Pack[{packId}|{packName}|{cardsPerPack} cards|${buyCost:F2}→${defaultSellPrice:F2}]";
}

// =============================================================================
// DROP TABLE DATA STRUCTURES
// =============================================================================

/// <summary>
/// Một slot trong Drop Table của Pack.
/// Slot = một vị trí thẻ trong pack (pack 6 thẻ → 6 slots).
/// Mỗi slot có bảng weight riêng xác định xác suất ra từng bậc hiếm.
/// </summary>
[System.Serializable]
public class DropTableSlot
{
    [Tooltip("Tên mô tả slot này, dùng cho Inspector. Vd: 'Slot 1-4 (Common)', 'Slot 6 (Rare Guaranteed)'.")]
    public string slotLabel = "Slot";

    [Tooltip("Danh sách các rarity có thể xuất hiện ở slot này, kèm trọng số (weight). " +
             "Tổng weight KHÔNG cần bằng 100 — GachaEngine tự normalize. " +
             "Ví dụ: Common=70, Uncommon=25, Rare=5 → tổng=100, hoặc Common=7, Uncommon=2.5, Rare=0.5 → kết quả như nhau.")]
    public List<RarityWeight> rarityWeights = new List<RarityWeight>();
}

/// <summary>
/// Cặp (RarityDefinition, Weight) cho một entry trong Drop Table.
/// </summary>
[System.Serializable]
public class RarityWeight
{
    [Tooltip("Bậc hiếm này.")]
    public RarityDefinition rarity;

    [Tooltip("Trọng số tương đối. Cao hơn = xuất hiện nhiều hơn. " +
             "Ví dụ: Common=70, Uncommon=25, Rare=5 → Common xuất hiện 70% thời gian.")]
    [Min(0f)]
    public float weight = 1f;
}
```

---

### 4.4 `CardDatabase.cs`

**Vị trí:** `Assets/Scripts/Data/CardDatabase.cs`  
**Mục đích:** ScriptableObject tổng hợp — một điểm duy nhất để truy cập toàn bộ cards và packs. Thay thế apiStore.shopItems và apiStore.setCardsCache trong hệ thống cũ.

```csharp
// Assets/Scripts/Data/CardDatabase.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// ScriptableObject tổng hợp toàn bộ catalogue thẻ bài và pack.
/// Là điểm truy cập trung tâm thay thế apiStore trong hệ thống cũ.
///
/// MAPPING TỪ HỆ THỐNG CŨ:
///   apiStore.shopItems     → allPacks (list) + packLookup (dictionary)
///   apiStore.setCardsCache → cardsBySetId (dictionary)
///   dbService.query(...)   → Không còn cần thiết — data đã load sẵn
///
/// CÁCH TẠO ASSET:
///   Right-click > Create > TCGShop > Data > Card Database
///   Chỉ cần MỘT database duy nhất: CardDatabase_Main.asset
/// </summary>
[CreateAssetMenu(
    fileName = "CardDatabase_Main",
    menuName = "TCGShop/Data/Card Database",
    order = 3
)]
public class CardDatabase : ScriptableObject
{
    // =========================================================================
    // DATA LISTS — Điền trong Inspector bằng cách kéo thả assets
    // =========================================================================

    [Header("Content")]
    [Tooltip("Tất cả PackData assets. Kéo thả từ ScriptableObjects/Packs/.")]
    public List<PackData> allPacks = new List<PackData>();

    [Tooltip("Tất cả RarityDefinition assets. Kéo thả từ ScriptableObjects/Rarities/.")]
    public List<RarityDefinition> allRarities = new List<RarityDefinition>();

    // =========================================================================
    // RUNTIME LOOKUP DICTIONARIES — Build từ lists ở trên khi khởi tạo
    // Lý do dùng Dictionary: O(1) lookup thay vì O(n) List.Find()
    // Tương đương lý do dùng Record<string, T> trong Pinia stores cũ
    // =========================================================================

    // Không serialize — tự build lại từ lists mỗi khi game chạy
    private Dictionary<string, PackData> _packLookup;
    private Dictionary<string, CardData> _cardLookup;
    private Dictionary<string, List<CardData>> _cardsBySetId;
    private bool _isInitialized = false;

    // =========================================================================
    // KHỞI TẠO
    // =========================================================================

    /// <summary>
    /// Build tất cả lookup dictionaries từ lists.
    /// Phải gọi một lần trước khi dùng các hàm lookup.
    /// Gọi từ InventoryManager.Initialize().
    /// </summary>
    public void Initialize()
    {
        if (_isInitialized) return;

        _packLookup = new Dictionary<string, PackData>();
        _cardLookup = new Dictionary<string, CardData>();
        _cardsBySetId = new Dictionary<string, List<CardData>>();

        // Build pack lookup
        foreach (var pack in allPacks)
        {
            if (pack == null || string.IsNullOrEmpty(pack.packId))
            {
                Debug.LogWarning("[CardDatabase] Pack null hoặc thiếu packId. Bỏ qua.");
                continue;
            }

            if (_packLookup.ContainsKey(pack.packId))
            {
                Debug.LogWarning($"[CardDatabase] Duplicate packId: '{pack.packId}'. " +
                                 "Giữ lại entry đầu tiên.");
                continue;
            }

            _packLookup[pack.packId] = pack;

            // Build card lookups từ pool của pack
            foreach (var card in pack.availableCards)
            {
                if (card == null || string.IsNullOrEmpty(card.cardId)) continue;

                // Global card lookup
                if (!_cardLookup.ContainsKey(card.cardId))
                {
                    _cardLookup[card.cardId] = card;
                }

                // Set-based card lookup
                if (!_cardsBySetId.ContainsKey(card.setId))
                {
                    _cardsBySetId[card.setId] = new List<CardData>();
                }

                if (!_cardsBySetId[card.setId].Contains(card))
                {
                    _cardsBySetId[card.setId].Add(card);
                }
            }
        }

        _isInitialized = true;
        Debug.Log($"[CardDatabase] Initialized: {_packLookup.Count} packs, " +
                  $"{_cardLookup.Count} unique cards, " +
                  $"{_cardsBySetId.Count} sets.");
    }

    // =========================================================================
    // LOOKUP API — O(1) complexity
    // =========================================================================

    /// <summary>
    /// Tìm PackData theo packId. O(1).
    /// Thay thế: apiStore.shopItems[packId] trong hệ thống cũ.
    /// </summary>
    public bool TryGetPack(string packId, out PackData pack)
    {
        EnsureInitialized();
        return _packLookup.TryGetValue(packId, out pack);
    }

    /// <summary>
    /// Tìm CardData theo cardId. O(1).
    /// Thay thế: apiStore.setCardsCache lookup trong hệ thống cũ.
    /// </summary>
    public bool TryGetCard(string cardId, out CardData card)
    {
        EnsureInitialized();
        return _cardLookup.TryGetValue(cardId, out card);
    }

    /// <summary>
    /// Lấy tất cả cards thuộc một set. O(1).
    /// Thay thế: apiStore.setCardsCache[setId] trong hệ thống cũ.
    /// </summary>
    public List<CardData> GetCardsBySet(string setId)
    {
        EnsureInitialized();
        return _cardsBySetId.TryGetValue(setId, out var cards)
            ? cards
            : new List<CardData>();
    }

    /// <summary>
    /// Lấy tất cả packs người chơi có thể mua ở shop level hiện tại.
    /// Thay thế: sortedShopItems getter trong apiStore cũ.
    /// </summary>
    public List<PackData> GetAvailablePacks(int currentShopLevel)
    {
        EnsureInitialized();
        var result = new List<PackData>();
        foreach (var pack in allPacks)
        {
            if (pack != null && pack.requiredShopLevel <= currentShopLevel)
            {
                result.Add(pack);
            }
        }
        return result;
    }

    private void EnsureInitialized()
    {
        if (!_isInitialized)
        {
            Debug.LogWarning("[CardDatabase] Chưa được Initialize(). Gọi Initialize() trước.");
            Initialize();
        }
    }
}
```

---

### 4.5 `GachaResult.cs`

**Vị trí:** `Assets/Scripts/Gacha/GachaResult.cs`  
**Mục đích:** Data class thuần (không MonoBehaviour, không ScriptableObject) đóng gói kết quả một lần mở pack. Dùng để truyền data giữa GachaEngine và InventoryManager.

```csharp
// Assets/Scripts/Gacha/GachaResult.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Kết quả của một lần mở Pack.
/// Data class thuần — không kế thừa MonoBehaviour hay ScriptableObject.
///
/// TƯƠNG ĐƯƠNG trong hệ thống cũ:
///   sortedCards trong tearPack() của inventoryStore.ts
///   Thẻ được sort theo rarityRank ASC (thẻ hiếm nhất ở cuối)
///   để UI lật từ Common → Rare (reveal climax ở cuối)
/// </summary>
public class GachaResult
{
    // =========================================================================
    // DATA
    // =========================================================================

    /// <summary>
    /// Danh sách thẻ rơi ra, đã sort theo rarityRank ASC.
    /// Index 0 = thẻ thường nhất (lật đầu tiên).
    /// Index Count-1 = thẻ hiếm nhất (lật cuối cùng — climax reveal).
    /// Tương đương sortedCards trong tearPack() của inventoryStore.ts cũ.
    /// </summary>
    public List<CardData> DroppedCards { get; private set; }

    /// <summary>
    /// Pack nào đã được mở.
    /// </summary>
    public PackData SourcePack { get; private set; }

    /// <summary>
    /// Tổng XP nhận được từ lần mở này.
    /// Tương đương: gainExp tích lũy trong vòng for của tearPack() cũ.
    /// </summary>
    public int TotalXpGained { get; private set; }

    /// <summary>
    /// Timestamp khi mở pack (dùng cho logging và analytics).
    /// </summary>
    public float OpenedAtTime { get; private set; }

    // =========================================================================
    // COMPUTED PROPERTIES
    // =========================================================================

    /// <summary>
    /// Thẻ hiếm nhất trong kết quả (thẻ cuối cùng hiển thị).
    /// </summary>
    public CardData HighlightCard =>
        DroppedCards != null && DroppedCards.Count > 0
            ? DroppedCards[DroppedCards.Count - 1]
            : null;

    /// <summary>
    /// Có thẻ High Rarity không (để trigger animation đặc biệt).
    /// Tương đương logic check rarity tier >= 3 trong PackOpeningOverlay.vue cũ.
    /// </summary>
    public bool HasHighRarityCard
    {
        get
        {
            if (DroppedCards == null) return false;
            foreach (var card in DroppedCards)
            {
                if (card != null && card.IsHighRarity) return true;
            }
            return false;
        }
    }

    /// <summary>
    /// Tổng giá trị thị trường của tất cả thẻ trong kết quả.
    /// </summary>
    public float TotalMarketValue
    {
        get
        {
            float total = 0f;
            if (DroppedCards == null) return total;
            foreach (var card in DroppedCards)
            {
                if (card != null) total += card.marketValue;
            }
            return total;
        }
    }

    // =========================================================================
    // CONSTRUCTOR
    // =========================================================================

    public GachaResult(PackData sourcePack, List<CardData> droppedCards)
    {
        SourcePack = sourcePack;
        DroppedCards = droppedCards ?? new List<CardData>();
        OpenedAtTime = Time.time;

        // Tính tổng XP
        TotalXpGained = 0;
        foreach (var card in DroppedCards)
        {
            if (card != null)
            {
                TotalXpGained += card.XpReward;
            }
        }
    }

    // =========================================================================
    // TIỆN ÍCH DEBUG
    // =========================================================================

    /// <summary>
    /// Tạo chuỗi mô tả đầy đủ kết quả cho logging.
    /// </summary>
    public string ToDebugString()
    {
        var sb = new System.Text.StringBuilder();
        sb.AppendLine($"=== GachaResult [{SourcePack?.packName ?? "Unknown Pack"}] ===");
        sb.AppendLine($"  Cards ({DroppedCards?.Count ?? 0}), XP: +{TotalXpGained}, " +
                      $"Total Value: ${TotalMarketValue:F2}");

        if (DroppedCards != null)
        {
            for (int i = 0; i < DroppedCards.Count; i++)
            {
                var card = DroppedCards[i];
                string highlight = card.IsHighRarity ? " ★ HIGH RARITY!" : "";
                sb.AppendLine($"  [{i + 1}] {card.cardName} " +
                              $"({card.rarity?.displayName ?? "?"}) " +
                              $"${card.marketValue:F2}{highlight}");
            }
        }

        return sb.ToString();
    }
}
```

---

### 4.6 `GachaEngine.cs` ← FILE QUAN TRỌNG NHẤT

**Vị trí:** `Assets/Scripts/Gacha/GachaEngine.cs`  
**Mục đích:** Thuật toán Cumulative Weighted Probability. Đây là trái tim của hệ thống Gacha, thay thế hoàn toàn `ORDER BY RANDOM()` của SQLite cũ.

**Cursor phải implement chính xác thuật toán sau:**

```csharp
// Assets/Scripts/Gacha/GachaEngine.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Engine tính toán kết quả mở pack theo thuật toán Cumulative Weighted Probability.
///
/// ===========================================================================
/// TẠI SAO KHÔNG DÙNG ORDER BY RANDOM() (Hệ thống cũ)?
/// ===========================================================================
/// Hệ thống cũ (inventoryStore.ts) dùng:
///   SQL: SELECT * FROM cards WHERE set_id = ? ORDER BY RANDOM() LIMIT 6
///
/// Vấn đề: Xác suất ra thẻ hiếm hoàn toàn phụ thuộc vào tỷ lệ thẻ trong DB.
/// Nếu set sv01 có 100 Common, 30 Uncommon, 10 Rare, 2 Ultra → Ultra = 2/142 = 1.4%
/// Nhưng game design muốn Ultra = 5% → Không thể đạt được với random thô.
///
/// ===========================================================================
/// CUMULATIVE WEIGHTED PROBABILITY — Cách hoạt động
/// ===========================================================================
///
/// INPUT: Drop Table Slot với weights:
///   Common    = 60
///   Uncommon  = 30
///   Rare      = 8
///   Ultra     = 2
///   (Tổng = 100, nhưng engine tự normalize nên KHÔNG cần tổng = 100)
///
/// BƯỚC 1: Tính tổng tất cả weights
///   totalWeight = 60 + 30 + 8 + 2 = 100
///
/// BƯỚC 2: Build bảng tích lũy (cumulative thresholds)
///   Common:   upperBound = 60/100  = 0.60
///   Uncommon: upperBound = 90/100  = 0.90
///   Rare:     upperBound = 98/100  = 0.98
///   Ultra:    upperBound = 100/100 = 1.00
///
/// BƯỚC 3: Roll một số random trong [0, 1)
///   roll = UnityEngine.Random.value  (ví dụ: 0.734)
///
/// BƯỚC 4: Tìm rarity đầu tiên có upperBound > roll
///   roll = 0.734
///   Common:   0.60 > 0.734? NO
///   Uncommon: 0.90 > 0.734? YES → Chọn Uncommon ✓
///
/// KẾT QUẢ: Với bảng này, mỗi lần roll:
///   60% → Common, 30% → Uncommon, 8% → Rare, 2% → Ultra
///   BẤT KỂ có bao nhiêu thẻ trong pool.
///
/// ===========================================================================
/// THIẾT KẾ CLASS
/// ===========================================================================
/// - Static class: Không cần instance, không cần MonoBehaviour.
/// - Pure functions: Cùng input → phân phối đúng theo weight.
/// - Không có side effects: Không modify bất kỳ state nào bên ngoài.
/// </summary>
public static class GachaEngine
{
    // =========================================================================
    // PUBLIC API
    // =========================================================================

    /// <summary>
    /// Mở một pack và trả về kết quả.
    /// Đây là entry point chính của toàn bộ hệ thống Gacha.
    ///
    /// LUỒNG THỰC THI:
    ///   1. Validate pack data
    ///   2. Với mỗi slot trong dropTable:
    ///      a. Roll rarity bằng Cumulative Weighted Probability
    ///      b. Lấy pool thẻ có rarity đó từ pack
    ///      c. Chọn ngẫu nhiên đều từ pool (uniform random trong pool)
    ///   3. Sort kết quả: thẻ thường trước, thẻ hiếm sau (index cuối)
    ///   4. Trả về GachaResult
    ///
    /// TƯƠNG ĐƯƠNG TRONG HỆ THỐNG CŨ:
    ///   Hàm tearPack() trong inventoryStore.ts, phần:
    ///   - getWeightedRandomCardsFromSet() → Bước 2
    ///   - sortedCards sort ASC by rarityRank → Bước 3
    ///
    /// </summary>
    /// <param name="pack">Pack cần mở. Phải IsValid() == true.</param>
    /// <param name="seed">
    ///   Seed ngẫu nhiên tùy chọn. Dùng cho reproducible testing.
    ///   -1 = không dùng seed (fully random).
    ///   Dùng trong GachaDebugTester để đảm bảo test nhất quán.
    /// </param>
    /// <returns>GachaResult chứa danh sách thẻ đã sort.</returns>
    public static GachaResult OpenPack(PackData pack, int seed = -1)
    {
        // --- Validate ---
        if (pack == null)
        {
            Debug.LogError("[GachaEngine] pack là null!");
            return new GachaResult(null, new List<CardData>());
        }

        if (!pack.IsValid())
        {
            Debug.LogError($"[GachaEngine] Pack '{pack.packId}' không hợp lệ. " +
                           "Kiểm tra availableCards và dropTable đã được điền.");
            return new GachaResult(pack, new List<CardData>());
        }

        // --- Khởi tạo Random State ---
        // Lưu và khôi phục state của Random để seed không ảnh hưởng phần còn lại của game
        Random.State previousRandomState = Random.state;
        if (seed >= 0)
        {
            Random.InitState(seed);
        }

        // --- Thực hiện Roll từng slot ---
        var droppedCards = new List<CardData>();
        int slotCount = Mathf.Min(pack.dropTable.Count, pack.cardsPerPack);

        for (int slotIndex = 0; slotIndex < slotCount; slotIndex++)
        {
            var slot = pack.dropTable[slotIndex];

            // Bước A: Roll rarity cho slot này
            RarityDefinition rolledRarity = RollRarity(slot);

            if (rolledRarity == null)
            {
                Debug.LogWarning($"[GachaEngine] Slot {slotIndex} của pack '{pack.packId}' " +
                                 "không roll được rarity. Bỏ qua slot này.");
                continue;
            }

            // Bước B: Lấy pool thẻ có rarity đó
            List<CardData> rarityPool = pack.GetCardsByRarity(rolledRarity);

            if (rarityPool == null || rarityPool.Count == 0)
            {
                // Fallback: Nếu không có thẻ với rarity này, lấy thẻ có rarity gần nhất
                Debug.LogWarning($"[GachaEngine] Không tìm thấy card nào với rarity " +
                                 $"'{rolledRarity.displayName}' trong pack '{pack.packId}'. " +
                                 "Dùng toàn bộ pool làm fallback.");
                rarityPool = pack.availableCards;
            }

            // Bước C: Chọn đều từ pool (uniform random)
            CardData selectedCard = SelectRandomFromPool(rarityPool);

            if (selectedCard != null)
            {
                droppedCards.Add(selectedCard);
            }
        }

        // --- Restore Random State ---
        if (seed >= 0)
        {
            Random.state = previousRandomState;
        }

        // --- Bước 3: Sort ASC theo rarity rank ---
        // Thẻ thường (rank thấp) ở đầu, thẻ hiếm nhất ở cuối (index cuối)
        // Tương đương: sortedCards.sort((a, b) => getRarityRank(a) - getRarityRank(b))
        // trong tearPack() của inventoryStore.ts cũ
        droppedCards.Sort((a, b) =>
        {
            int rankA = a?.RarityRank ?? 0;
            int rankB = b?.RarityRank ?? 0;
            return rankA.CompareTo(rankB); // ASC: thấp trước, cao sau
        });

        return new GachaResult(pack, droppedCards);
    }

    // =========================================================================
    // THUẬT TOÁN LÕI: CUMULATIVE WEIGHTED PROBABILITY
    // =========================================================================

    /// <summary>
    /// THUẬT TOÁN CHÍNH: Chọn một RarityDefinition từ Drop Table Slot
    /// dựa trên Cumulative Weighted Probability.
    ///
    /// BƯỚC 1: Tính totalWeight = tổng tất cả weight trong slot
    /// BƯỚC 2: Roll một số random trong [0, totalWeight)
    /// BƯỚC 3: Duyệt qua từng rarity, cộng dồn weight
    ///         Khi cộng dồn >= roll → đó là rarity được chọn
    ///
    /// ĐỘ PHỨC TẠP: O(n) với n = số rarity trong slot.
    /// Với n <= 9 (số bậc hiếm tối đa), đây là đủ hiệu quả.
    ///
    /// KHÔNG DÙNG: Lookup table pre-built vì n nhỏ và thay đổi không thường xuyên.
    /// </summary>
    private static RarityDefinition RollRarity(DropTableSlot slot)
    {
        if (slot == null || slot.rarityWeights == null || slot.rarityWeights.Count == 0)
        {
            Debug.LogWarning("[GachaEngine] Slot trống hoặc không có rarityWeights.");
            return null;
        }

        // BƯỚC 1: Tính tổng weight
        float totalWeight = 0f;
        foreach (var entry in slot.rarityWeights)
        {
            if (entry != null && entry.rarity != null && entry.weight > 0f)
            {
                totalWeight += entry.weight;
            }
        }

        if (totalWeight <= 0f)
        {
            Debug.LogWarning($"[GachaEngine] Slot '{slot.slotLabel}' có totalWeight = 0. " +
                             "Kiểm tra lại các weight trong Inspector.");
            return null;
        }

        // BƯỚC 2: Roll số random trong khoảng [0, totalWeight)
        // Random.Range(min, max) với float: min inclusive, max exclusive
        float roll = Random.Range(0f, totalWeight);

        // BƯỚC 3: Duyệt tích lũy để tìm rarity
        float cumulativeWeight = 0f;

        foreach (var entry in slot.rarityWeights)
        {
            // Bỏ qua entry không hợp lệ
            if (entry == null || entry.rarity == null || entry.weight <= 0f) continue;

            cumulativeWeight += entry.weight;

            // Khi cộng dồn đã vượt qua roll → đây là rarity được chọn
            if (roll < cumulativeWeight)
            {
                return entry.rarity;
            }
        }

        // Fallback: Trường hợp floating point edge case (roll == totalWeight)
        // Chọn rarity cuối cùng hợp lệ
        for (int i = slot.rarityWeights.Count - 1; i >= 0; i--)
        {
            var lastEntry = slot.rarityWeights[i];
            if (lastEntry != null && lastEntry.rarity != null)
            {
                return lastEntry.rarity;
            }
        }

        Debug.LogError("[GachaEngine] Không thể chọn rarity. Kiểm tra Drop Table configuration.");
        return null;
    }

    /// <summary>
    /// Chọn một CardData ngẫu nhiên đều từ pool.
    /// Đây là uniform random (không có weight) — tất cả thẻ cùng rarity có cơ hội bằng nhau.
    /// Tương đương phần random từ pool sau khi đã biết rarity.
    /// </summary>
    private static CardData SelectRandomFromPool(List<CardData> pool)
    {
        if (pool == null || pool.Count == 0) return null;
        if (pool.Count == 1) return pool[0];

        int randomIndex = Random.Range(0, pool.Count); // [0, Count)
        return pool[randomIndex];
    }

    // =========================================================================
    // UTILITY API — Dùng cho Preview và Analytics
    // =========================================================================

    /// <summary>
    /// Tính xác suất lý thuyết của từng rarity trong một slot.
    /// Dùng để hiển thị "Rates" trong UI, không dùng trong gameplay.
    ///
    /// Trả về Dictionary[RarityDefinition → xác suất từ 0.0 đến 1.0]
    /// </summary>
    public static Dictionary<RarityDefinition, float> CalculateTheoreticalRates(DropTableSlot slot)
    {
        var rates = new Dictionary<RarityDefinition, float>();

        if (slot?.rarityWeights == null) return rates;

        float totalWeight = 0f;
        foreach (var entry in slot.rarityWeights)
        {
            if (entry?.rarity != null && entry.weight > 0f)
                totalWeight += entry.weight;
        }

        if (totalWeight <= 0f) return rates;

        foreach (var entry in slot.rarityWeights)
        {
            if (entry?.rarity == null || entry.weight <= 0f) continue;
            rates[entry.rarity] = entry.weight / totalWeight;
        }

        return rates;
    }

    /// <summary>
    /// Simulate mở N pack và trả về thống kê phân phối rarity.
    /// Dùng cho GachaDebugTester để verify RNG không bị thiên lệch.
    /// </summary>
    public static Dictionary<string, int> SimulateOpenPacks(PackData pack, int numberOfPacks)
    {
        var rarityCount = new Dictionary<string, int>();

        for (int i = 0; i < numberOfPacks; i++)
        {
            var result = OpenPack(pack);
            foreach (var card in result.DroppedCards)
            {
                string rarityName = card.rarity?.displayName ?? "Unknown";
                if (!rarityCount.ContainsKey(rarityName))
                    rarityCount[rarityName] = 0;
                rarityCount[rarityName]++;
            }
        }

        return rarityCount;
    }
}
```

---

### 4.7 `InventoryManager.cs`

**Vị trí:** `Assets/Scripts/Inventory/InventoryManager.cs`  
**Mục đích:** Singleton quản lý kho đồ với O(1) lookup. Tương đương inventoryStore.ts trong hệ thống cũ.

```csharp
// Assets/Scripts/Inventory/InventoryManager.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Singleton quản lý toàn bộ kho đồ của người chơi.
///
/// MAPPING TỪ HỆ THỐNG CŨ (inventoryStore.ts):
///   shopInventory: Record<itemId, quantity>   → _packInventory: Dictionary<string, int>
///   personalBinder: Record<cardId, quantity>  → _cardBinder: Dictionary<string, int>
///   shopItems: Record<itemId, StockItemInfo>  → cardDatabase (CardDatabase reference)
///
/// TẠI SAO DÙNG DICTIONARY:
///   Dictionary<string, int> cho phép lookup O(1) — bất kể có 10 hay 10,000 items.
///   List<T>.Find() là O(n) — chậm hơn khi inventory lớn.
///   Điều này giải thích lý do hệ thống cũ dùng Record<string, T> trong Pinia.
///
/// LƯU Ý: InventoryManager quản lý SỐ LƯỢNG.
///        Metadata (tên, giá, sprite) lấy từ CardDatabase.
/// </summary>
public class InventoryManager : MonoBehaviour
{
    // =========================================================================
    // SINGLETON
    // =========================================================================

    public static InventoryManager Instance { get; private set; }

    // =========================================================================
    // CONFIGURATION
    // =========================================================================

    [Header("Database Reference")]
    [Tooltip("CardDatabase ScriptableObject chứa toàn bộ catalogue. " +
             "Kéo thả CardDatabase_Main.asset vào đây.")]
    [SerializeField] private CardDatabase cardDatabase;

    // =========================================================================
    // INVENTORY STATE
    // Dùng Dictionary<string, int> thay vì List để đảm bảo O(1) lookup.
    // Tương đương Record<string, number> trong Pinia stores cũ.
    // =========================================================================

    /// <summary>
    /// Kho pack đang có trong shop (chờ xếp lên kệ hoặc mở).
    /// Key = packId (vd: "pack_sv01"), Value = số lượng.
    /// Tương đương shopInventory trong inventoryStore.ts cũ.
    /// </summary>
    private Dictionary<string, int> _packInventory = new Dictionary<string, int>();

    /// <summary>
    /// Bộ sưu tập thẻ cá nhân (đã mở ra từ pack).
    /// Key = cardId (vd: "sv01-001"), Value = số lượng.
    /// Tương đương personalBinder trong inventoryStore.ts cũ.
    /// </summary>
    private Dictionary<string, int> _cardBinder = new Dictionary<string, int>();

    // =========================================================================
    // KHỞI TẠO
    // =========================================================================

    private void Awake()
    {
        // Singleton setup
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        // Validate database reference
        if (cardDatabase == null)
        {
            Debug.LogError("[InventoryManager] cardDatabase chưa được assign trong Inspector! " +
                           "Kéo thả CardDatabase_Main.asset vào field 'Card Database'.");
            enabled = false;
            return;
        }

        // Khởi tạo database lookup tables
        cardDatabase.Initialize();

        Debug.Log("[InventoryManager] Initialized.");
    }

    // =========================================================================
    // PACK INVENTORY API
    // Tương đương các actions liên quan đến shopInventory trong inventoryStore.ts cũ
    // =========================================================================

    /// <summary>
    /// Thêm pack vào kho. O(1).
    /// Tương đương: shopInventory[packId] += amount trong inventoryStore.ts cũ.
    /// </summary>
    public void AddPack(string packId, int amount = 1)
    {
        if (string.IsNullOrEmpty(packId) || amount <= 0) return;

        if (!_packInventory.ContainsKey(packId))
            _packInventory[packId] = 0;

        _packInventory[packId] += amount;
        Debug.Log($"[InventoryManager] Added {amount}x '{packId}'. " +
                  $"Total: {_packInventory[packId]}");
    }

    /// <summary>
    /// Kiểm tra số lượng pack trong kho. O(1).
    /// Tương đương: shopInventory[packId] ?? 0 trong inventoryStore.ts cũ.
    /// </summary>
    public int GetPackCount(string packId)
    {
        return _packInventory.TryGetValue(packId, out int count) ? count : 0;
    }

    /// <summary>
    /// Mở một pack: Giảm kho, chạy Gacha, thêm thẻ vào binder.
    /// Tương đương hàm tearPack() trong inventoryStore.ts cũ.
    ///
    /// Trả về GachaResult để caller (UI) biết thẻ nào rơi ra.
    /// Trả về null nếu không đủ pack trong kho.
    /// </summary>
    public GachaResult OpenPack(string packId)
    {
        // Kiểm tra có pack không
        if (GetPackCount(packId) <= 0)
        {
            Debug.LogWarning($"[InventoryManager] Không đủ pack '{packId}' trong kho!");
            return null;
        }

        // Lấy PackData
        if (!cardDatabase.TryGetPack(packId, out PackData pack))
        {
            Debug.LogError($"[InventoryManager] Không tìm thấy PackData cho packId '{packId}'!");
            return null;
        }

        // Trừ kho trước khi mở (tương đương shopInventory[packId]-- trong cũ)
        _packInventory[packId]--;
        if (_packInventory[packId] <= 0)
            _packInventory.Remove(packId); // Xóa key nếu hết hàng

        // Chạy Gacha
        GachaResult result = GachaEngine.OpenPack(pack);

        // Thêm thẻ vào binder
        foreach (var card in result.DroppedCards)
        {
            if (card != null)
                AddCardToBinder(card.cardId);
        }

        // Thêm XP (sẽ gọi GameManager/StatsManager ở bước sau)
        Debug.Log($"[InventoryManager] Opened pack '{packId}'. " +
                  $"Got {result.DroppedCards.Count} cards, +{result.TotalXpGained} XP.");
        Debug.Log(result.ToDebugString());

        return result;
    }

    // =========================================================================
    // CARD BINDER API
    // Tương đương personalBinder trong inventoryStore.ts cũ
    // =========================================================================

    /// <summary>
    /// Thêm thẻ vào binder. O(1).
    /// Tương đương: personalBinder[cardId]++ trong inventoryStore.ts cũ.
    /// </summary>
    public void AddCardToBinder(string cardId, int amount = 1)
    {
        if (string.IsNullOrEmpty(cardId) || amount <= 0) return;

        if (!_cardBinder.ContainsKey(cardId))
            _cardBinder[cardId] = 0;

        _cardBinder[cardId] += amount;
    }

    /// <summary>
    /// Số lượng một thẻ trong binder. O(1).
    /// Tương đương: personalBinder[cardId] ?? 0 trong inventoryStore.ts cũ.
    /// </summary>
    public int GetCardCount(string cardId)
    {
        return _cardBinder.TryGetValue(cardId, out int count) ? count : 0;
    }

    /// <summary>
    /// Toàn bộ binder dưới dạng read-only.
    /// Dùng cho UI hiển thị danh sách thẻ.
    /// </summary>
    public IReadOnlyDictionary<string, int> GetFullBinder()
    {
        return _cardBinder;
    }

    /// <summary>
    /// Toàn bộ pack inventory dưới dạng read-only.
    /// </summary>
    public IReadOnlyDictionary<string, int> GetFullPackInventory()
    {
        return _packInventory;
    }

    // =========================================================================
    // TRUY CẬP DATABASE
    // =========================================================================

    /// <summary>
    /// Lấy CardDatabase. Dùng cho các hệ thống khác cần truy cập catalogue.
    /// </summary>
    public CardDatabase Database => cardDatabase;
}
```

---

### 4.8 `GachaDebugTester.cs` ← FILE TEST BẮT BUỘC

**Vị trí:** `Assets/Scripts/Debug/GachaDebugTester.cs`  
**Mục đích:** Tự động mở 10 pack trong `Start()` và in thống kê tỷ lệ rơi ra Console để chứng minh RNG phân phối đúng theo bảng cấu hình, không bị thiên lệch.

```csharp
// Assets/Scripts/Debug/GachaDebugTester.cs

using System.Collections.Generic;
using System.Text;
using UnityEngine;

/// <summary>
/// Debug tester tự động chạy khi game start.
/// Mở 10 pack liên tục và in thống kê tỷ lệ rơi ra Console.
///
/// MỤC ĐÍCH:
///   Chứng minh GachaEngine phân phối đúng theo Drop Table đã cấu hình.
///   So sánh tỷ lệ THỰC TẾ (từ simulation) với tỷ lệ LÝ THUYẾT (từ weights).
///
/// CÁCH DÙNG:
///   1. Gắn script này vào bất kỳ GameObject nào trong GameScene.
///   2. Assign packToTest trong Inspector.
///   3. Nhấn Play → Kiểm tra Console.
///   4. SAU KHI TEST XONG: Disable hoặc xóa component này.
///      Không để component này trong production build.
///
/// KẾT QUẢ MONG ĐỢI trong Console:
///   [GachaDebugTester] ════════════════════════════════════
///   [GachaDebugTester] PACK OPENING SIMULATION: 10 packs
///   [GachaDebugTester] Pack: "Base Set Booster Pack"
///   [GachaDebugTester] ════════════════════════════════════
///   [GachaDebugTester] INDIVIDUAL PACK RESULTS:
///   [GachaDebugTester]   Pack #1: Bulbasaur(C), Squirtle(C), Pikachu(U), ...
///   ...
///   [GachaDebugTester] ════════════════════════════════════
///   [GachaDebugTester] RARITY DISTRIBUTION (60 total cards from 10 packs):
///   [GachaDebugTester]   Common:    38 cards | 63.33% actual | ~60.00% theoretical | PASS ✓
///   [GachaDebugTester]   Uncommon:  15 cards | 25.00% actual | ~25.00% theoretical | PASS ✓
///   [GachaDebugTester]   Rare:       5 cards |  8.33% actual |  ~8.00% theoretical | PASS ✓
///   [GachaDebugTester]   Ultra Rare: 2 cards |  3.33% actual |  ~5.00% theoretical | WARN ⚠
///   [GachaDebugTester] ════════════════════════════════════
///   [GachaDebugTester] ECONOMY STATS:
///   [GachaDebugTester]   Total pack cost:   $49.90  (10 × $4.99)
///   [GachaDebugTester]   Total card value:  $67.43
///   [GachaDebugTester]   Total XP gained:   +147
///   [GachaDebugTester]   Return on invest:  +35.1%
///   [GachaDebugTester] ════════════════════════════════════
///   [GachaDebugTester] RNG BIAS CHECK: PASS — Distribution within tolerance.
/// </summary>
public class GachaDebugTester : MonoBehaviour
{
    // =========================================================================
    // CONFIGURATION
    // =========================================================================

    [Header("Test Configuration")]
    [Tooltip("Pack cần test. Phải có Drop Table được cấu hình đầy đủ.")]
    [SerializeField] private PackData packToTest;

    [Tooltip("Số pack mở trong test. Nhiều hơn → kết quả chính xác hơn. Mặc định 10.")]
    [SerializeField][Range(1, 100)] private int numberOfPacksToOpen = 10;

    [Tooltip("Sai số chấp nhận được giữa tỷ lệ thực tế và lý thuyết. " +
             "Mặc định 15%: nếu lý thuyết 5%, chấp nhận 4.25%-5.75%. " +
             "Với ít pack (10), biến động cao là bình thường.")]
    [SerializeField][Range(0.05f, 0.50f)] private float tolerancePercent = 0.15f;

    [Tooltip("Nếu true, in chi tiết từng pack. Nếu false, chỉ in tổng kết.")]
    [SerializeField] private bool printIndividualPacks = true;

    [Tooltip("Nếu true, tự disable component sau khi test xong.")]
    [SerializeField] private bool disableAfterTest = true;

    // =========================================================================
    // VÒNG ĐỜI
    // =========================================================================

    private void Start()
    {
        // Kiểm tra dependencies
        if (packToTest == null)
        {
            Debug.LogError("[GachaDebugTester] packToTest chưa được assign! " +
                           "Kéo thả một PackData asset vào field 'Pack To Test'.");
            return;
        }

        if (!packToTest.IsValid())
        {
            Debug.LogError($"[GachaDebugTester] Pack '{packToTest.packId}' không hợp lệ. " +
                           "Kiểm tra availableCards và dropTable trong Inspector.");
            return;
        }

        // Chạy test
        RunFullTest();

        // Tự disable sau khi test
        if (disableAfterTest)
        {
            enabled = false;
        }
    }

    // =========================================================================
    // LOGIC TEST
    // =========================================================================

    /// <summary>
    /// Chạy toàn bộ test suite và in kết quả ra Console.
    /// </summary>
    private void RunFullTest()
    {
        const string DIVIDER = "════════════════════════════════════════════════════";

        // --- Header ---
        Debug.Log($"[GachaDebugTester] {DIVIDER}");
        Debug.Log($"[GachaDebugTester] PACK OPENING SIMULATION: {numberOfPacksToOpen} packs");
        Debug.Log($"[GachaDebugTester] Pack: \"{packToTest.packName}\" (ID: {packToTest.packId})");
        Debug.Log($"[GachaDebugTester] Cards per pack: {packToTest.cardsPerPack}");
        Debug.Log($"[GachaDebugTester] Cards in pool: {packToTest.availableCards?.Count ?? 0}");
        Debug.Log($"[GachaDebugTester] {DIVIDER}");

        // --- Mở từng pack ---
        var allResults = new List<GachaResult>();
        var totalRarityCount = new Dictionary<string, int>();
        float totalCardValue = 0f;
        int totalXpGained = 0;

        if (printIndividualPacks)
        {
            Debug.Log($"[GachaDebugTester] INDIVIDUAL PACK RESULTS:");
        }

        for (int i = 0; i < numberOfPacksToOpen; i++)
        {
            // Mở pack (không dùng seed để test random thực)
            GachaResult result = GachaEngine.OpenPack(packToTest);
            allResults.Add(result);

            totalCardValue += result.TotalMarketValue;
            totalXpGained += result.TotalXpGained;

            // Đếm rarity
            foreach (var card in result.DroppedCards)
            {
                if (card == null) continue;
                string rarityName = card.rarity?.displayName ?? "Unknown";
                if (!totalRarityCount.ContainsKey(rarityName))
                    totalRarityCount[rarityName] = 0;
                totalRarityCount[rarityName]++;
            }

            // In chi tiết pack nếu được bật
            if (printIndividualPacks)
            {
                PrintIndividualPackResult(i + 1, result);
            }
        }

        // --- Tính tổng kết ---
        int totalCards = numberOfPacksToOpen * packToTest.cardsPerPack;
        float totalPackCost = packToTest.buyCost * numberOfPacksToOpen;

        Debug.Log($"[GachaDebugTester] {DIVIDER}");

        // --- In phân phối rarity ---
        PrintRarityDistribution(totalRarityCount, totalCards);

        Debug.Log($"[GachaDebugTester] {DIVIDER}");

        // --- In stats kinh tế ---
        PrintEconomyStats(totalPackCost, totalCardValue, totalXpGained, numberOfPacksToOpen);

        Debug.Log($"[GachaDebugTester] {DIVIDER}");

        // --- Bias check ---
        bool passed = RunBiasCheck(totalRarityCount, totalCards);
        if (passed)
        {
            Debug.Log($"[GachaDebugTester] ✅ RNG BIAS CHECK: PASS — " +
                      $"Distribution within {tolerancePercent * 100:F0}% tolerance.");
        }
        else
        {
            Debug.LogWarning($"[GachaDebugTester] ⚠️ RNG BIAS CHECK: WARN — " +
                             $"Some rarities outside {tolerancePercent * 100:F0}% tolerance. " +
                             $"Tăng numberOfPacksToOpen để có kết quả chính xác hơn.");
        }

        Debug.Log($"[GachaDebugTester] {DIVIDER}");
    }

    /// <summary>
    /// In kết quả của một pack cụ thể.
    /// Format: Pack #N: CardName(R), CardName(U), ...
    /// </summary>
    private void PrintIndividualPackResult(int packNumber, GachaResult result)
    {
        var sb = new StringBuilder();
        sb.Append($"[GachaDebugTester]   Pack #{packNumber:D2}: ");

        for (int i = 0; i < result.DroppedCards.Count; i++)
        {
            var card = result.DroppedCards[i];
            if (card == null) continue;

            // Ký hiệu rarity: C=Common, U=Uncommon, R=Rare, UR=Ultra, ★=Ghost
            string rarityCode = GetRarityCode(card.rarity);

            // Tô đỏ thẻ hiếm để dễ nhìn trong Console
            if (card.IsHighRarity)
                sb.Append($"★{card.cardName}({rarityCode})★");
            else
                sb.Append($"{card.cardName}({rarityCode})");

            if (i < result.DroppedCards.Count - 1)
                sb.Append(", ");
        }

        sb.Append($" | Value: ${result.TotalMarketValue:F2} | XP: +{result.TotalXpGained}");
        Debug.Log(sb.ToString());
    }

    /// <summary>
    /// In bảng phân phối rarity: thực tế vs lý thuyết.
    /// Đây là phần quan trọng nhất để verify RNG hoạt động đúng.
    /// </summary>
    private void PrintRarityDistribution(Dictionary<string, int> rarityCount, int totalCards)
    {
        Debug.Log($"[GachaDebugTester] RARITY DISTRIBUTION " +
                  $"({totalCards} total cards from {numberOfPacksToOpen} packs):");

        // Tính tỷ lệ lý thuyết từ Drop Table (slot đầu tiên làm tham chiếu)
        Dictionary<string, float> theoreticalRates = CalculateAverageTheoreticalRates();

        // In từng rarity theo thứ tự sortingRank
        var sortedRarities = new List<string>(rarityCount.Keys);
        sortedRarities.Sort(); // Alphabetical fallback nếu không có rank

        foreach (var rarityName in sortedRarities)
        {
            int count = rarityCount[rarityName];
            float actualPercent = totalCards > 0 ? (float)count / totalCards * 100f : 0f;

            string theoreticalStr = "N/A";
            string passStr = "";

            if (theoreticalRates.TryGetValue(rarityName, out float theoreticalRate))
            {
                float theoreticalPercent = theoreticalRate * 100f;
                theoreticalStr = $"~{theoreticalPercent:F2}%";

                // Kiểm tra có trong tolerance không
                float deviation = Mathf.Abs(actualPercent - theoreticalPercent) / theoreticalPercent;
                passStr = deviation <= tolerancePercent ? "  ✓" : "  ⚠";
            }

            Debug.Log($"[GachaDebugTester]   {rarityName,-30} " +
                      $"{count,4} cards | " +
                      $"{actualPercent,6:F2}% actual | " +
                      $"{theoreticalStr,12} theoretical{passStr}");
        }
    }

    /// <summary>
    /// In thống kê kinh tế của lần test.
    /// </summary>
    private void PrintEconomyStats(float totalCost, float totalValue,
                                    int totalXp, int packCount)
    {
        float roi = totalCost > 0 ? (totalValue - totalCost) / totalCost * 100f : 0f;
        string roiSign = roi >= 0 ? "+" : "";

        Debug.Log($"[GachaDebugTester] ECONOMY STATS:");
        Debug.Log($"[GachaDebugTester]   Total pack cost:    ${totalCost:F2}  " +
                  $"({packCount} × ${packToTest.buyCost:F2})");
        Debug.Log($"[GachaDebugTester]   Total card value:   ${totalValue:F2}");
        Debug.Log($"[GachaDebugTester]   Total XP gained:    +{totalXp}");
        Debug.Log($"[GachaDebugTester]   Return on invest:   {roiSign}{roi:F1}%");
    }

    /// <summary>
    /// Kiểm tra bias: So sánh tỷ lệ thực tế với lý thuyết.
    /// Trả về true nếu tất cả rarity trong tolerance.
    /// </summary>
    private bool RunBiasCheck(Dictionary<string, int> rarityCount, int totalCards)
    {
        if (totalCards <= 0) return false;

        Dictionary<string, float> theoreticalRates = CalculateAverageTheoreticalRates();
        bool allPassed = true;

        foreach (var kvp in rarityCount)
        {
            string rarityName = kvp.Key;
            float actualRate = (float)kvp.Value / totalCards;

            if (!theoreticalRates.TryGetValue(rarityName, out float theoreticalRate))
                continue;

            if (theoreticalRate <= 0f) continue;

            float deviation = Mathf.Abs(actualRate - theoreticalRate) / theoreticalRate;
            if (deviation > tolerancePercent)
            {
                allPassed = false;
                Debug.LogWarning($"[GachaDebugTester]   BIAS DETECTED: {rarityName} | " +
                                 $"Actual: {actualRate * 100:F2}% | " +
                                 $"Expected: {theoreticalRate * 100:F2}% | " +
                                 $"Deviation: {deviation * 100:F1}% (limit: {tolerancePercent * 100:F0}%)");
            }
        }

        return allPassed;
    }

    /// <summary>
    /// Tính tỷ lệ lý thuyết trung bình qua tất cả các slots của pack.
    /// Vì mỗi slot có Drop Table khác nhau, ta lấy trung bình có trọng số.
    /// </summary>
    private Dictionary<string, float> CalculateAverageTheoreticalRates()
    {
        var averageRates = new Dictionary<string, float>();
        int slotCount = packToTest.dropTable?.Count ?? 0;
        if (slotCount == 0) return averageRates;

        // Tích lũy tỷ lệ từ mỗi slot
        foreach (var slot in packToTest.dropTable)
        {
            var slotRates = GachaEngine.CalculateTheoreticalRates(slot);
            foreach (var kvp in slotRates)
            {
                string rarityName = kvp.Key.displayName;
                if (!averageRates.ContainsKey(rarityName))
                    averageRates[rarityName] = 0f;
                averageRates[rarityName] += kvp.Value;
            }
        }

        // Chia trung bình
        var keys = new List<string>(averageRates.Keys);
        foreach (var key in keys)
        {
            averageRates[key] /= slotCount;
        }

        return averageRates;
    }

    /// <summary>
    /// Chuyển RarityDefinition thành code ngắn để in trong Individual Pack Result.
    /// </summary>
    private string GetRarityCode(RarityDefinition rarity)
    {
        if (rarity == null) return "?";
        return rarity.sortingRank switch
        {
            0 => "C",    // Common
            1 => "U",    // Uncommon
            3 => "R",    // Rare
            4 => "2R",   // Double Rare
            5 => "UR",   // Ultra Rare
            6 => "SR",   // Secret Rare
            7 => "IR",   // Illustration Rare
            8 => "SIR",  // Special Illustration Rare
            9 => "HSR",  // Hyper Secret Rare
            10 => "GR",  // Ghost Rare
            _ => rarity.displayName[..Mathf.Min(3, rarity.displayName.Length)]
        };
    }
}
```

---

### 4.9 `CardDataEditor.cs` — Custom Inspector

**Vị trí:** `Assets/Editor/CardDataEditor.cs`  
**Mục đích:** Custom Inspector cho CardData ScriptableObject, giúp dễ dàng tạo và kiểm tra thẻ bài trong Unity Editor.

```csharp
// Assets/Editor/CardDataEditor.cs

#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;

/// <summary>
/// Custom Inspector cho CardData ScriptableObject.
/// Thêm các tính năng tiện ích:
///   - Preview thẻ trực quan trong Inspector
///   - Nút validate dữ liệu
///   - Hiển thị computed properties (IsHighRarity, XpReward)
///   - Nút tạo nhanh card mới từ template
/// </summary>
[CustomEditor(typeof(CardData))]
public class CardDataEditor : Editor
{
    private CardData _target;
    private bool _showBattleStats = true;
    private bool _showEconomy = true;
    private bool _showComputedProps = true;

    private void OnEnable()
    {
        _target = (CardData)target;
    }

    public override void OnInspectorGUI()
    {
        serializedObject.Update();

        // --- Header với card preview ---
        DrawCardPreviewHeader();

        EditorGUILayout.Space(10);

        // --- Default fields ---
        DrawDefaultInspector();

        EditorGUILayout.Space(10);

        // --- Computed Properties (readonly) ---
        DrawComputedProperties();

        EditorGUILayout.Space(10);

        // --- Action Buttons ---
        DrawActionButtons();

        serializedObject.ApplyModifiedProperties();
    }

    private void DrawCardPreviewHeader()
    {
        EditorGUILayout.BeginVertical(EditorStyles.helpBox);

        // Tên thẻ lớn
        GUIStyle titleStyle = new GUIStyle(EditorStyles.boldLabel)
        {
            fontSize = 16,
            alignment = TextAnchor.MiddleCenter
        };
        EditorGUILayout.LabelField(
            string.IsNullOrEmpty(_target.cardName) ? "(Chưa đặt tên)" : _target.cardName,
            titleStyle,
            GUILayout.Height(30)
        );

        // Card ID và Set ID
        EditorGUILayout.BeginHorizontal();
        EditorGUILayout.LabelField("ID:", GUILayout.Width(30));
        EditorGUILayout.LabelField(
            string.IsNullOrEmpty(_target.cardId) ? "(chưa có ID)" : _target.cardId,
            EditorStyles.miniLabel
        );
        EditorGUILayout.EndHorizontal();

        // Rarity với màu
        if (_target.rarity != null)
        {
            Color oldColor = GUI.color;
            GUI.color = _target.rarity.rarityColor;
            EditorGUILayout.LabelField(
                $"★ {_target.rarity.displayName}",
                EditorStyles.boldLabel
            );
            GUI.color = oldColor;
        }
        else
        {
            EditorGUILayout.LabelField("⚠ Chưa gán Rarity!", EditorStyles.miniLabel);
        }

        // Preview sprite
        if (_target.cardSprite != null)
        {
            Rect spriteRect = GUILayoutUtility.GetRect(100, 140, GUILayout.ExpandWidth(false));
            spriteRect.x = (EditorGUIUtility.currentViewWidth - 100) / 2;
            GUI.DrawTextureWithTexCoords(
                spriteRect,
                _target.cardSprite.texture,
                new Rect(0, 0, 1, 1)
            );
        }

        EditorGUILayout.EndVertical();
    }

    private void DrawComputedProperties()
    {
        _showComputedProps = EditorGUILayout.Foldout(_showComputedProps,
            "Computed Properties (Read Only)", true);

        if (!_showComputedProps) return;

        EditorGUI.BeginDisabledGroup(true);
        EditorGUILayout.BeginVertical(EditorStyles.helpBox);

        EditorGUILayout.Toggle("Is High Rarity", _target.IsHighRarity);
        EditorGUILayout.IntField("Rarity Rank", _target.RarityRank);
        EditorGUILayout.IntField("XP Reward", _target.XpReward);
        EditorGUILayout.Toggle("Is Valid", _target.IsValid());

        EditorGUILayout.EndVertical();
        EditorGUI.EndDisabledGroup();
    }

    private void DrawActionButtons()
    {
        EditorGUILayout.BeginHorizontal();

        // Nút Validate
        if (GUILayout.Button("✓ Validate Data", GUILayout.Height(30)))
        {
            if (_target.IsValid())
            {
                EditorUtility.DisplayDialog("Validation PASS",
                    $"CardData '{_target.cardName}' hợp lệ!\n\n" +
                    $"ID: {_target.cardId}\n" +
                    $"Rarity: {_target.rarity?.displayName}\n" +
                    $"HP: {_target.baseHp}\n" +
                    $"Market Value: ${_target.marketValue:F2}",
                    "OK");
            }
            else
            {
                EditorUtility.DisplayDialog("Validation FAIL",
                    "CardData không hợp lệ! Kiểm tra:\n" +
                    $"• cardId: {(string.IsNullOrEmpty(_target.cardId) ? "TRỐNG ❌" : "OK ✓")}\n" +
                    $"• cardName: {(string.IsNullOrEmpty(_target.cardName) ? "TRỐNG ❌" : "OK ✓")}\n" +
                    $"• rarity: {(_target.rarity == null ? "CHƯA GÁN ❌" : "OK ✓")}",
                    "OK");
            }
        }

        // Nút Copy ID vào Clipboard
        if (GUILayout.Button("📋 Copy ID", GUILayout.Height(30)))
        {
            EditorGUIUtility.systemCopyBuffer = _target.cardId;
            Debug.Log($"[CardDataEditor] Copied to clipboard: '{_target.cardId}'");
        }

        EditorGUILayout.EndHorizontal();

        // Nút tạo card mới từ template
        EditorGUILayout.Space(5);
        if (GUILayout.Button("+ Tạo Card Mới Từ Template Này", GUILayout.Height(25)))
        {
            CreateCardFromTemplate();
        }
    }

    private void CreateCardFromTemplate()
    {
        string path = EditorUtility.SaveFilePanelInProject(
            "Tạo Card Mới",
            $"Card_New_{System.DateTime.Now:yyyyMMdd_HHmmss}",
            "asset",
            "Chọn vị trí lưu card mới",
            "Assets/ScriptableObjects/Cards"
        );

        if (string.IsNullOrEmpty(path)) return;

        var newCard = CreateInstance<CardData>();
        newCard.rarity = _target.rarity; // Copy rarity từ template
        newCard.cardTypes = (EnergyType[])_target.cardTypes?.Clone();
        newCard.retreatCost = _target.retreatCost;
        newCard.baseHp = _target.baseHp;
        newCard.setId = _target.setId;
        newCard.seriesId = _target.seriesId;

        AssetDatabase.CreateAsset(newCard, path);
        AssetDatabase.SaveAssets();
        Selection.activeObject = newCard;

        Debug.Log($"[CardDataEditor] Đã tạo card mới từ template tại: {path}");
    }
}
#endif
```

---

## 5. Tạo Sample Assets (Cursor Phải Tạo Qua Code)

Cursor phải tạo một Editor script để auto-generate các sample assets cần thiết cho test. **Không** hướng dẫn người dùng tạo tay.

```csharp
// Assets/Editor/SampleDataGenerator.cs

#if UNITY_EDITOR
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

/// <summary>
/// Tạo tất cả sample ScriptableObject assets cần thiết để test Bước 2.
/// Chạy một lần qua menu TCGShop > Setup > Generate Sample Data.
/// </summary>
public static class SampleDataGenerator
{
    [MenuItem("TCGShop/Setup/Generate Sample Data for Step 2")]
    public static void GenerateAllSampleData()
    {
        // Đảm bảo thư mục tồn tại
        EnsureDirectoriesExist();

        // 1. Tạo Rarity Definitions
        var rarities = CreateRarityDefinitions();

        // 2. Tạo Sample Cards
        var cards = CreateSampleCards(rarities);

        // 3. Tạo Sample Pack với Drop Table
        var pack = CreateSamplePack(rarities, cards);

        // 4. Tạo Card Database
        CreateCardDatabase(pack);

        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();

        Debug.Log("[SampleDataGenerator] ✅ Tất cả sample assets đã được tạo thành công!");
        EditorUtility.DisplayDialog(
            "Sample Data Generated",
            "Đã tạo:\n" +
            "• 9 RarityDefinition assets\n" +
            "• 5 CardData assets (mẫu)\n" +
            "• 1 PackData asset với Drop Table đầy đủ\n" +
            "• 1 CardDatabase asset\n\n" +
            "Kiểm tra thư mục Assets/ScriptableObjects/",
            "OK"
        );
    }

    private static void EnsureDirectoriesExist()
    {
        string[] dirs = {
            "Assets/ScriptableObjects",
            "Assets/ScriptableObjects/Rarities",
            "Assets/ScriptableObjects/Cards",
            "Assets/ScriptableObjects/Packs",
            "Assets/ScriptableObjects/Database"
        };

        foreach (var dir in dirs)
        {
            if (!AssetDatabase.IsValidFolder(dir))
            {
                var parts = dir.Split('/');
                var parent = string.Join("/", parts[..^1]);
                AssetDatabase.CreateFolder(parent, parts[^1]);
            }
        }
    }

    private static Dictionary<string, RarityDefinition> CreateRarityDefinitions()
    {
        var rarities = new Dictionary<string, RarityDefinition>();

        var definitions = new[]
        {
            ("Common",                    0, false, Color.gray,                         2),
            ("Uncommon",                  1, false, Color.green,                        2),
            ("Rare",                      3, true,  new Color(0.8f, 0.7f, 0.1f),       15),
            ("Double Rare",               4, true,  new Color(1f, 0.85f, 0f),           15),
            ("Ultra Rare",                5, true,  new Color(0.5f, 0.1f, 0.9f),       15),
            ("Illustration Rare",         7, true,  new Color(0.2f, 0.7f, 1f),         15),
            ("Special Illustration Rare", 8, true,  new Color(1f, 0.4f, 0.8f),         15),
            ("Secret Rare",               6, true,  new Color(1f, 0.8f, 0.2f),         15),
            ("Ghost Rare",               10, true,  new Color(1f, 0.2f, 0.6f),         15),
        };

        foreach (var (name, rank, isHigh, color, xp) in definitions)
        {
            var rarity = ScriptableObject.CreateInstance<RarityDefinition>();
            rarity.displayName = name;
            rarity.sortingRank = rank;
            rarity.isHighRarity = isHigh;
            rarity.rarityColor = color;
            rarity.xpReward = xp;

            string fileName = $"Rarity_{name.Replace(" ", "")}.asset";
            string path = $"Assets/ScriptableObjects/Rarities/{fileName}";
            AssetDatabase.CreateAsset(rarity, path);

            rarities[name] = rarity;
        }

        Debug.Log($"[SampleDataGenerator] Created {rarities.Count} RarityDefinition assets.");
        return rarities;
    }

    private static List<CardData> CreateSampleCards(Dictionary<string, RarityDefinition> rarities)
    {
        var cards = new List<CardData>();

        var cardDefs = new[]
        {
            ("sv01-001", "Charizard ex",  "sv01", "sv", "001", "Ultra Rare",  45.99f, 180),
            ("sv01-002", "Pikachu",       "sv01", "sv", "002", "Common",        0.15f,  60),
            ("sv01-003", "Mewtwo",        "sv01", "sv", "003", "Rare",           3.50f, 120),
            ("sv01-004", "Bulbasaur",     "sv01", "sv", "004", "Common",         0.10f,  70),
            ("sv01-005", "Squirtle",      "sv01", "sv", "005", "Uncommon",       0.50f,  80),
        };

        foreach (var (id, name, setId, seriesId, number, rarityName, value, hp) in cardDefs)
        {
            var card = ScriptableObject.CreateInstance<CardData>();
            card.cardId = id;
            card.cardName = name;
            card.setId = setId;
            card.seriesId = seriesId;
            card.cardNumber = number;
            card.marketValue = value;
            card.baseHp = hp;

            if (rarities.TryGetValue(rarityName, out var rarity))
                card.rarity = rarity;

            string path = $"Assets/ScriptableObjects/Cards/Card_{name.Replace(" ", "_")}.asset";
            AssetDatabase.CreateAsset(card, path);
            cards.Add(card);
        }

        Debug.Log($"[SampleDataGenerator] Created {cards.Count} CardData assets.");
        return cards;
    }

    private static PackData CreateSamplePack(
        Dictionary<string, RarityDefinition> rarities,
        List<CardData> cards)
    {
        var pack = ScriptableObject.CreateInstance<PackData>();
        pack.packId = "pack_sv01";
        pack.packName = "Scarlet & Violet Base Set Booster Pack";
        pack.sourceSetId = "sv01";
        pack.generationName = "GENERATION IX";
        pack.buyCost = 4.99f;
        pack.defaultSellPrice = 7.99f;
        pack.requiredShopLevel = 1;
        pack.cardsPerPack = 6;
        pack.availableCards = new List<CardData>(cards);

        // =====================================================================
        // DROP TABLE: 6 slots với bảng weight khác nhau
        // Thiết kế theo chuẩn TCG thực tế:
        //   Slot 1-3: Chắc chắn Common
        //   Slot 4-5: Common/Uncommon mix
        //   Slot 6:   Rare guaranteed (có thể Ultra Rare)
        // =====================================================================
        pack.dropTable = new List<DropTableSlot>
        {
            // Slot 1: Common guaranteed
            new DropTableSlot
            {
                slotLabel = "Slot 1 (Common Guaranteed)",
                rarityWeights = new List<RarityWeight>
                {
                    new RarityWeight { rarity = rarities["Common"], weight = 100f }
                }
            },
            // Slot 2: Common guaranteed
            new DropTableSlot
            {
                slotLabel = "Slot 2 (Common Guaranteed)",
                rarityWeights = new List<RarityWeight>
                {
                    new RarityWeight { rarity = rarities["Common"], weight = 100f }
                }
            },
            // Slot 3: Common guaranteed
            new DropTableSlot
            {
                slotLabel = "Slot 3 (Common Guaranteed)",
                rarityWeights = new List<RarityWeight>
                {
                    new RarityWeight { rarity = rarities["Common"], weight = 100f }
                }
            },
            // Slot 4: Common/Uncommon mix
            new DropTableSlot
            {
                slotLabel = "Slot 4 (Common/Uncommon)",
                rarityWeights = new List<RarityWeight>
                {
                    new RarityWeight { rarity = rarities["Common"],   weight = 70f },
                    new RarityWeight { rarity = rarities["Uncommon"], weight = 30f }
                }
            },
            // Slot 5: Uncommon/Rare mix
            new DropTableSlot
            {
                slotLabel = "Slot 5 (Uncommon/Rare Mix)",
                rarityWeights = new List<RarityWeight>
                {
                    new RarityWeight { rarity = rarities["Common"],   weight = 30f },
                    new RarityWeight { rarity = rarities["Uncommon"], weight = 55f },
                    new RarityWeight { rarity = rarities["Rare"],     weight = 15f }
                }
            },
            // Slot 6: Rare Guaranteed Slot (có chance Ultra Rare)
            new DropTableSlot
            {
                slotLabel = "Slot 6 (Rare Guaranteed ★)",
                rarityWeights = new List<RarityWeight>
                {
                    new RarityWeight { rarity = rarities["Rare"],       weight = 70f },
                    new RarityWeight { rarity = rarities["Double Rare"],weight = 20f },
                    new RarityWeight { rarity = rarities["Ultra Rare"], weight = 10f }
                }
            }
        };

        string path = "Assets/ScriptableObjects/Packs/Pack_SV01_BaseSet.asset";
        AssetDatabase.CreateAsset(pack, path);

        Debug.Log($"[SampleDataGenerator] Created PackData '{pack.packName}' " +
                  $"with {pack.dropTable.Count} slots.");
        return pack;
    }

    private static void CreateCardDatabase(PackData pack)
    {
        var db = ScriptableObject.CreateInstance<CardDatabase>();
        db.allPacks = new List<PackData> { pack };

        string path = "Assets/ScriptableObjects/Database/CardDatabase_Main.asset";
        AssetDatabase.CreateAsset(db, path);

        Debug.Log($"[SampleDataGenerator] Created CardDatabase with {db.allPacks.Count} pack(s).");
    }
}
#endif
```

---

## 6. Thiết Lập Scene Cho Test

Cursor phải thêm các component sau vào `GameScene.unity` để test hoạt động:

```
GameScene Hierarchy:
├── _Bootstrapper          [SceneBootstrapper.cs]          (từ Bước 1)
├── Main Camera            [Camera] [CameraController]     (từ Bước 1)
│
├── _InventoryManager      [InventoryManager.cs]           ← MỚI
│   Inspector:
│     Card Database: CardDatabase_Main  (kéo asset vào đây)
│
└── _GachaDebugTester      [GachaDebugTester.cs]           ← MỚI (chỉ để test)
    Inspector:
      Pack To Test:          Pack_SV01_BaseSet  (kéo asset vào đây)
      Number Of Packs:       10
      Tolerance Percent:     0.15
      Print Individual Packs: true
      Disable After Test:    true
```

---

## 7. Giới Hạn & Quy Tắc Bắt Buộc

| Quy tắc | Lý do |
|---------|-------|
| **CẤM** hard-code tỷ lệ rơi trong GachaEngine.cs | Tỷ lệ phải đến từ PackData.dropTable trong ScriptableObject |
| **CẤM** dùng `List<T>.Find()` cho inventory lookup | Phải dùng `Dictionary<string, T>` để đảm bảo O(1) |
| **CẤM** `GameObject.Find()` trong `Update()` | Quy tắc từ Bước 1, áp dụng toàn dự án |
| **CẤM** `Random.Range()` trực tiếp để chọn rarity | Phải đi qua `GachaEngine.RollRarity()` |
| **BẮT BUỘC** `GachaDebugTester` phải tự disable sau test | Không để debug code trong production |
| **BẮT BUỘC** Mọi ScriptableObject phải có `IsValid()` | Phát hiện cấu hình sai sớm |
| **BẮT BUỘC** `CardDatabase.Initialize()` gọi trước mọi lookup | Dictionary chưa build sẽ trả về null |

---

## 8. Kịch Bản Kiểm Thử Đầy Đủ

### 8.1 Chuẩn Bị

```
Bước 1: Vào menu TCGShop > Setup > Generate Sample Data for Step 2
        → Kiểm tra thư mục Assets/ScriptableObjects/ có đủ assets

Bước 2: Mở GameScene.unity
        → Tạo GameObject "_InventoryManager", gắn InventoryManager.cs
        → Kéo CardDatabase_Main.asset vào field "Card Database"

Bước 3: Tạo GameObject "_GachaDebugTester", gắn GachaDebugTester.cs
        → Kéo Pack_SV01_BaseSet.asset vào field "Pack To Test"
        → Đặt Number Of Packs = 10
```

### 8.2 Chạy Test và Kiểm Tra Console

```
Nhấn Play. Kiểm tra Console theo thứ tự:

1. "[GameManager] Ready."                              ← Từ Bước 1
2. "[InventoryManager] Initialized."                   ← Mới
3. "[CardDatabase] Initialized: 1 packs, 5 unique cards, 1 sets."
4. "[GachaDebugTester] ════...════"
5. "[GachaDebugTester] PACK OPENING SIMULATION: 10 packs"
6. ... (10 dòng kết quả từng pack)
7. "[GachaDebugTester] RARITY DISTRIBUTION (60 total cards from 10 packs):"
8. ... (các dòng Common/Uncommon/Rare với % thực tế và lý thuyết)
9. "[GachaDebugTester] ✅ RNG BIAS CHECK: PASS ..."
```

### 8.3 Validate Kết Quả RNG

```
KẾT QUẢ MONG ĐỢI (xấp xỉ, biến động nhỏ là bình thường với 10 packs):

Slot 1-3 (Common 100%):
  Common: ~30 cards từ 3 slots × 10 packs = 30 (100% → luôn đúng)

Slot 4 (Common=70%, Uncommon=30%):
  Common:   ~7 cards / 10 packs  (70% của slot 4)
  Uncommon: ~3 cards / 10 packs  (30% của slot 4)

Slot 5 (Common=30%, Uncommon=55%, Rare=15%):
  Common:   ~3 cards / 10 packs
  Uncommon: ~5-6 cards / 10 packs
  Rare:     ~1-2 cards / 10 packs

Slot 6 (Rare=70%, DoubleRare=20%, UltraRare=10%):
  Rare:        ~7 cards / 10 packs
  Double Rare: ~2 cards / 10 packs
  Ultra Rare:  ~1 card / 10 packs (có thể 0 với chỉ 10 packs, bình thường)

✅ PASS: Tỷ lệ thực tế chênh lệch dưới 15% so với lý thuyết
⚠ WARN: Chênh lệch vượt 15% → Bình thường với 10 packs, tăng lên 100 packs để verify
❌ FAIL: NullReferenceException xuất hiện → Kiểm tra Inspector assignment
```

### 8.4 Kiểm Tra O(1) Lookup

```
Trong Console, không được thấy log về "searching" hay "iterating".
Mọi truy cập inventory phải instant.

Test thủ công (tạo script tạm thời):
  InventoryManager.Instance.AddPack("pack_sv01", 100);
  // Mở 100 pack liên tiếp — Console không lag
  for (int i = 0; i < 100; i++)
      InventoryManager.Instance.OpenPack("pack_sv01");

✅ PASS: 100 pack mở trong < 1 giây, không lag, không error
```

---

## 9. Định Nghĩa "Hoàn Thành" (Definition of Done)

Bước 2 được coi là **HOÀN THÀNH** khi và chỉ khi:

- [ ] Tất cả 9 file `.cs` đã được tạo đúng thư mục
- [ ] Menu `TCGShop > Setup > Generate Sample Data for Step 2` chạy thành công
- [ ] Tất cả sample assets tồn tại trong `Assets/ScriptableObjects/`
- [ ] Nhấn Play: Console in đúng thứ tự log không lỗi
- [ ] `[GachaDebugTester]` in đủ 10 pack results và bảng phân phối
- [ ] Log kết thúc bằng `✅ RNG BIAS CHECK: PASS`
- [ ] **Không có** `NullReferenceException` nào trong toàn bộ test
- [ ] `GachaDebugTester` tự disable sau khi test xong (`disableAfterTest = true`)
- [ ] `GachaEngine.RollRarity()` dùng đúng thuật toán Cumulative Weighted Probability
- [ ] `InventoryManager` dùng `Dictionary<string, int>` (không phải `List`)
- [ ] Không có hard-coded tỷ lệ rơi nào trong file `.cs` — mọi weight đến từ ScriptableObject

**Chỉ sau khi tất cả checkbox trên được check, mới chuyển sang Bước 3.**
```