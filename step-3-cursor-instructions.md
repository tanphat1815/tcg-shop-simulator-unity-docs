```markdown
# Step 3: Toán Học Xây Dựng Và Hệ Thống Xếp Hàng Trên Lưới
## Cursor Instructions — TCG Shop Simulator (Unity Port)

**Phiên bản tài liệu:** 1.0  
**Giai đoạn:** World Interaction Layer  
**Yêu cầu tiên quyết:** Bước 1 hoàn thành (`GameManager`, `CameraController` hoạt động), Bước 2 hoàn thành (`CardDatabase`, `InventoryManager`, `PackData` sẵn sàng).

---

## 1. Mục Tiêu Của Bước Này

Hệ thống cũ (Phaser/Vue) xử lý placement bằng cách kiểm tra va chạm vật lý Phaser (`physics.add.collider`) và validate bằng hàm `validatePlacement()` dùng `Phaser.Geom.Intersects.RectangleToRectangle`. Cách này có hai vấn đề khi port sang Unity:

- **Vấn đề 1 — Floating point drift:** Kiểm tra bằng hitbox pixel không đảm bảo snap chính xác vào lưới, gây "kẹt khe" giữa các vật thể.
- **Vấn đề 2 — Không có spatial index:** Mỗi lần validate phải duyệt toàn bộ danh sách vật thể O(n), chậm khi shop lớn.

**Giải pháp Unity:** `Grid` component + `Dictionary<Vector2Int, GridNode>` — mỗi ô lưới là một node trong dictionary, lookup O(1), snap tuyệt đối không drift.

Bước này xây dựng bốn hệ thống:

1. **`GridSystem`** — Quản lý ma trận lưới, spatial index O(1), kiểm tra va chạm chính xác theo cell
2. **`FurnitureDefinition`** — ScriptableObject khai báo kích thước và footprint của từng loại nội thất
3. **`PlacementManager`** — Điều phối toàn bộ luồng placement: di chuột → ghost preview → click → instantiate
4. **`GhostObject`** — Visual feedback đổi màu xanh/đỏ theo tính hợp lệ của vị trí

**Kết quả mong đợi:** Di chuột thấy ghost preview đổi màu, nhấn R để xoay, click để đặt, đặt đè lên ô đã có log lỗi chính xác.

---

## 2. Danh Sách Files Cần Tạo

Cursor phải tạo đúng các file sau. Không được tự ý thay đổi đường dẫn.

```
Assets/
├── Scripts/
│   ├── Grid/
│   │   ├── GridSystem.cs               ← Ma trận lưới, Dictionary<Vector2Int, GridNode>
│   │   ├── GridNode.cs                 ← Data struct cho một ô lưới
│   │   └── GridVisualizer.cs           ← Vẽ debug lines cho lưới (Editor only)
│   ├── Placement/
│   │   ├── PlacementManager.cs         ← Điều phối luồng đặt nội thất
│   │   ├── GhostObject.cs              ← Visual ghost preview xanh/đỏ
│   │   └── PlacedFurnitureInstance.cs  ← Component gắn lên prefab sau khi đặt
│   └── Data/
│       └── FurnitureDefinition.cs      ← ScriptableObject khai báo nội thất
│
├── ScriptableObjects/
│   └── Furniture/                      ← Chứa FurnitureDefinition assets
│       ├── Furniture_ShelfSingle.asset
│       ├── Furniture_ShelfDouble.asset
│       ├── Furniture_StorageShelf.asset
│       ├── Furniture_PlayTable.asset
│       └── Furniture_CashierDesk.asset
│
├── Prefabs/
│   └── Furniture/                      ← Prefab cho từng loại nội thất
│       ├── Prefab_ShelfSingle.prefab
│       ├── Prefab_ShelfDouble.prefab
│       ├── Prefab_StorageShelf.prefab
│       ├── Prefab_PlayTable.prefab
│       └── Prefab_CashierDesk.prefab
│
└── Editor/
    └── FurnitureDefinitionEditor.cs    ← Custom Inspector hiển thị footprint preview
```

---

## 3. Lý Thuyết Nền: Không Gian Lưới Isometric

### 3.1 Coordinate System

Unity `Grid` component cung cấp API chuyển đổi giữa ba hệ tọa độ:

```
World Space    → Grid.WorldToCell(worldPos)    → Cell Space (Vector3Int)
Cell Space     → Grid.CellToWorld(cellPos)     → World Space
Cell Space     → Vector2Int(cell.x, cell.y)    → Dictionary Key
```

**Quan trọng:** Trong Isometric Z-as-Y (đã cấu hình ở Bước 1), trục Y của cell space tương ứng với chiều sâu. `WorldToCell` tự xử lý đúng phép chiếu isometric — không cần tính thủ công.

### 3.2 Footprint — Kích Thước Vật Thể Trên Lưới

Mỗi vật thể chiếm một **footprint** — tập hợp các cell tương đối so với cell gốc (origin cell):

```
Ví dụ ShelfSingle (1×1):
  Footprint: [(0,0)]
  → Chiếm đúng 1 cell tại vị trí đặt

Ví dụ ShelfDouble (2×1):
  Footprint rotation=0°:  [(0,0), (1,0)]
  Footprint rotation=90°: [(0,0), (0,1)]
  → Chiếm 2 cells liên tiếp

Ví dụ PlayTable (2×2):
  Footprint rotation=0°:  [(0,0), (1,0), (0,1), (1,1)]
  Footprint rotation=90°: [(0,0), (-1,0), (0,-1), (-1,-1)]
  → Chiếm 4 cells
```

**Khi validate placement:**
```
FOR EACH cell IN footprint:
  absoluteCell = originCell + cell
  IF GridNode[absoluteCell].isOccupied → INVALID
  IF absoluteCell ngoài shop bounds     → INVALID
```

**Khi confirm placement:**
```
FOR EACH cell IN footprint:
  absoluteCell = originCell + cell
  GridNode[absoluteCell].isOccupied = true
  GridNode[absoluteCell].occupantId = furnitureInstanceId
```

### 3.3 Rotation Logic

Phím R xoay footprint 90° theo chiều kim đồng hồ:

```
Công thức xoay một điểm (x, y) quanh gốc 90° CW:
  x' = y
  y' = -x

Ví dụ ShelfDouble rotation=0° → 90°:
  (1, 0) → (0, -1)
  Footprint mới: [(0,0), (0,-1)]
```

---

## 4. Chi Tiết Kỹ Thuật Từng File

### 4.1 `GridNode.cs`

**Vị trí:** `Assets/Scripts/Grid/GridNode.cs`  
**Mục đích:** Data struct thuần cho một ô lưới. Đây là value type (`struct`) để tránh overhead của heap allocation khi dictionary có hàng nghìn nodes.

```csharp
// Assets/Scripts/Grid/GridNode.cs

using UnityEngine;

/// <summary>
/// Đại diện cho một ô đơn trong ma trận lưới cửa hàng.
/// Dùng struct thay vì class để tránh heap allocation overhead
/// khi Dictionary chứa hàng nghìn nodes.
///
/// MAPPING TỪ HỆ THỐNG CŨ (Phaser/Vue):
///   Không có cấu trúc tương đương trực tiếp.
///   Hệ thống cũ dùng Phaser physics bodies và validatePlacement()
///   để kiểm tra va chạm. GridNode thay thế toàn bộ cơ chế đó
///   bằng spatial index O(1).
/// </summary>
public struct GridNode
{
    // =========================================================================
    // TRẠNG THÁI Ô LƯỚI
    // =========================================================================

    /// <summary>
    /// Ô này có đang bị chiếm dụng bởi một vật thể không.
    /// true  = không thể đặt thêm vật thể vào đây.
    /// false = ô trống, có thể đặt.
    /// Tương đương logic: "Không được đè lên vật thể khác" trong
    /// validatePlacement() của MainScene.ts cũ.
    /// </summary>
    public bool IsOccupied;

    /// <summary>
    /// ID của vật thể đang chiếm dụng ô này.
    /// Dùng để tìm vật thể khi người chơi click vào ô (Edit Mode).
    /// String.Empty nếu ô trống.
    /// Tương đương furniture.getData('id') trong Phaser sprites cũ.
    /// </summary>
    public string OccupantId;

    /// <summary>
    /// Loại vật thể chiếm dụng ô (dùng để hiển thị tooltip và logic đặc biệt).
    /// Tương đương furniture type trong FURNITURE_ITEMS config cũ.
    /// </summary>
    public FurnitureType OccupantType;

    /// <summary>
    /// Ô này có nằm trong shop bounds hợp lệ không.
    /// false = ô nằm ngoài biên giới shop (không cho phép đặt bất kỳ gì).
    /// Tương đương kiểm tra bounds trong validatePlacement() cũ:
    ///   worldX < bounds.x + PAD → INVALID
    /// </summary>
    public bool IsWithinShopBounds;

    // =========================================================================
    // CONSTRUCTORS
    // =========================================================================

    /// <summary>
    /// Tạo node trống hợp lệ (trong shop bounds, chưa bị chiếm).
    /// </summary>
    public static GridNode Empty => new GridNode
    {
        IsOccupied = false,
        OccupantId = string.Empty,
        OccupantType = FurnitureType.None,
        IsWithinShopBounds = true
    };

    /// <summary>
    /// Tạo node ngoài bounds (không thể đặt).
    /// </summary>
    public static GridNode OutOfBounds => new GridNode
    {
        IsOccupied = false,
        OccupantId = string.Empty,
        OccupantType = FurnitureType.None,
        IsWithinShopBounds = false
    };

    // =========================================================================
    // TIỆN ÍCH
    // =========================================================================

    /// <summary>
    /// Ô này có thể đặt vật thể không.
    /// Tổng hợp cả hai điều kiện: trong bounds VÀ chưa bị chiếm.
    /// </summary>
    public bool IsPlaceable => IsWithinShopBounds && !IsOccupied;

    public override string ToString() =>
        $"GridNode[Occupied={IsOccupied}, InBounds={IsWithinShopBounds}, " +
        $"OccupantId={OccupantId}]";
}

/// <summary>
/// Enum phân loại vật thể nội thất.
/// Tương đương furnitureId strings trong FURNITURE_ITEMS config cũ.
/// </summary>
public enum FurnitureType
{
    None,
    ShelfSingle,   // "shelf_single"
    ShelfDouble,   // "shelf_double"
    StorageShelf,  // "storage_shelf"
    PlayTable,     // "play_table"
    CashierDesk    // "cashier_desk"
}
```

---

### 4.2 `FurnitureDefinition.cs`

**Vị trí:** `Assets/Scripts/Data/FurnitureDefinition.cs`  
**Mục đích:** ScriptableObject khai báo toàn bộ metadata của một loại nội thất, bao gồm footprint kích thước, giá mua, level yêu cầu, và prefab reference.

**Mapping từ `FURNITURE_ITEMS` config cũ:**
```
furniture/config/index.ts (cũ)    →   FurnitureDefinition ScriptableObject
─────────────────────────────────────────────────────────────────────────────
id: 'shelf_single'                →   furnitureId (enum FurnitureType)
name: 'Single Sided Shelf'        →   displayName
buyPrice: 300                     →   buyCost
requiredLevel: 3                  →   requiredShopLevel
numTiers: 3                       →   numberOfTiers
slotsPerTier: 16                  →   slotsPerTier
role: 'selling'                   →   role (enum ShelfRole)
-- MỚI THÊM --
footprintWidth: int               ←   Không có trong hệ thống cũ
footprintHeight: int              ←   Không có trong hệ thống cũ
prefab: GameObject                ←   Không có trong hệ thống cũ
```

```csharp
// Assets/Scripts/Data/FurnitureDefinition.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// ScriptableObject khai báo metadata của một loại nội thất.
/// Là nguồn dữ liệu duy nhất (Single Source of Truth) cho mọi thông tin
/// liên quan đến một loại furniture.
///
/// CÁCH TẠO ASSET:
///   Right-click > Create > TCGShop > Data > Furniture Definition
/// </summary>
[CreateAssetMenu(
    fileName = "Furniture_New",
    menuName = "TCGShop/Data/Furniture Definition",
    order = 4
)]
public class FurnitureDefinition : ScriptableObject
{
    // =========================================================================
    // IDENTITY
    // =========================================================================

    [Header("Identity")]
    [Tooltip("Loại nội thất. Dùng làm key để lookup trong code.")]
    public FurnitureType furnitureType = FurnitureType.ShelfSingle;

    [Tooltip("Tên hiển thị trong UI Shop. Vd: 'Single Sided Shelf'.")]
    public string displayName = "New Furniture";

    [Tooltip("Mô tả ngắn hiển thị trong Build Menu.")]
    [TextArea(2, 3)]
    public string description;

    // =========================================================================
    // VISUAL
    // =========================================================================

    [Header("Visual")]
    [Tooltip("Prefab được Instantiate khi người chơi đặt nội thất xuống lưới. " +
             "Prefab này PHẢI có PlacedFurnitureInstance component.")]
    public GameObject furniturePrefab;

    [Tooltip("Prefab Ghost (bóng ma preview). Nếu null, tự tạo từ furniturePrefab " +
             "với material bán trong suốt.")]
    public GameObject ghostPrefab;

    [Tooltip("Sprite icon hiển thị trong Build Menu UI.")]
    public Sprite menuIcon;

    // =========================================================================
    // GRID FOOTPRINT — Kích thước trên lưới
    // =========================================================================

    [Header("Grid Footprint")]
    [Tooltip("Số cell chiếm theo trục X (ngang). " +
             "ShelfSingle=1, ShelfDouble=2, PlayTable=2.")]
    [Range(1, 4)]
    public int footprintWidth = 1;

    [Tooltip("Số cell chiếm theo trục Y (dọc/sâu trong isometric). " +
             "ShelfSingle=1, PlayTable=2.")]
    [Range(1, 4)]
    public int footprintHeight = 1;

    [Tooltip("Nếu true, vật thể có thể xoay bằng phím R. " +
             "false cho những vật thể đối xứng (1x1).")]
    public bool canRotate = false;

    // =========================================================================
    // ECONOMY
    // =========================================================================

    [Header("Economy")]
    [Tooltip("Giá mua nội thất từ Online Shop. " +
             "Tương đương buyPrice trong FURNITURE_ITEMS cũ.")]
    [Min(0f)]
    public float buyCost = 300f;

    [Tooltip("Level shop tối thiểu để mở khóa. " +
             "Tương đương requiredLevel trong FURNITURE_ITEMS cũ.")]
    [Range(1, 80)]
    public int requiredShopLevel = 1;

    // =========================================================================
    // SHELF CONFIG — Chỉ áp dụng cho loại shelf
    // =========================================================================

    [Header("Shelf Config (Shelf types only)")]
    [Tooltip("Số tầng kệ. 0 nếu không phải shelf. " +
             "Tương đương numTiers trong FURNITURE_ITEMS cũ.")]
    [Range(0, 6)]
    public int numberOfTiers = 0;

    [Tooltip("Số slot tối đa mỗi tầng (cho pack). Box = slotsPerTier/4. " +
             "Tương đương slotsPerTier trong FURNITURE_ITEMS cũ.")]
    [Range(0, 64)]
    public int slotsPerTier = 0;

    [Tooltip("Vai trò của kệ: Selling (NPC mua được) hay Storage (chỉ lưu trữ). " +
             "Tương đương role trong FURNITURE_ITEMS cũ.")]
    public ShelfRole shelfRole = ShelfRole.Selling;

    // =========================================================================
    // COMPUTED PROPERTIES
    // =========================================================================

    /// <summary>
    /// Tổng số cell chiếm khi rotation = 0°.
    /// </summary>
    public int TotalCells => footprintWidth * footprintHeight;

    /// <summary>
    /// Tạo danh sách relative cell positions cho footprint ở rotation 0°.
    /// Gốc (0,0) là cell trái-dưới của footprint.
    ///
    /// Ví dụ ShelfDouble (2×1):
    ///   Width=2, Height=1 → [(0,0), (1,0)]
    ///
    /// Ví dụ PlayTable (2×2):
    ///   Width=2, Height=2 → [(0,0), (1,0), (0,1), (1,1)]
    /// </summary>
    public List<Vector2Int> GetFootprintCells(int rotationDegrees = 0)
    {
        // Tạo footprint cơ bản (rotation = 0°)
        var cells = new List<Vector2Int>();
        for (int x = 0; x < footprintWidth; x++)
        {
            for (int y = 0; y < footprintHeight; y++)
            {
                cells.Add(new Vector2Int(x, y));
            }
        }

        // Áp dụng rotation nếu cần
        int normalizedRotation = ((rotationDegrees % 360) + 360) % 360;
        int steps = normalizedRotation / 90;

        for (int step = 0; step < steps; step++)
        {
            cells = RotateFootprint90CW(cells);
        }

        return cells;
    }

    /// <summary>
    /// Xoay danh sách cells 90° theo chiều kim đồng hồ.
    /// Công thức xoay 90° CW: (x, y) → (y, -x)
    ///
    /// Ví dụ ShelfDouble [(0,0), (1,0)] sau khi xoay:
    ///   (0,0) → (0, 0)
    ///   (1,0) → (0,-1)
    ///   Kết quả: [(0,0), (0,-1)]
    /// </summary>
    private List<Vector2Int> RotateFootprint90CW(List<Vector2Int> cells)
    {
        var rotated = new List<Vector2Int>(cells.Count);
        foreach (var cell in cells)
        {
            // Công thức xoay 90° CW: x' = y, y' = -x
            rotated.Add(new Vector2Int(cell.y, -cell.x));
        }
        return rotated;
    }

    /// <summary>
    /// Tính bounding box của footprint sau khi xoay.
    /// Dùng để căn chỉnh ghost preview đúng tâm.
    /// </summary>
    public (Vector2Int min, Vector2Int max) GetFootprintBounds(int rotationDegrees = 0)
    {
        var cells = GetFootprintCells(rotationDegrees);
        if (cells.Count == 0) return (Vector2Int.zero, Vector2Int.zero);

        int minX = int.MaxValue, minY = int.MaxValue;
        int maxX = int.MinValue, maxY = int.MinValue;

        foreach (var cell in cells)
        {
            minX = Mathf.Min(minX, cell.x);
            minY = Mathf.Min(minY, cell.y);
            maxX = Mathf.Max(maxX, cell.x);
            maxY = Mathf.Max(maxY, cell.y);
        }

        return (new Vector2Int(minX, minY), new Vector2Int(maxX, maxY));
    }

    /// <summary>
    /// Validation: Đảm bảo definition có đủ dữ liệu để sử dụng.
    /// </summary>
    public bool IsValid()
    {
        if (furniturePrefab == null) return false;
        if (string.IsNullOrEmpty(displayName)) return false;
        if (footprintWidth <= 0 || footprintHeight <= 0) return false;
        return true;
    }

    public override string ToString() =>
        $"FurnitureDef[{furnitureType}|{footprintWidth}×{footprintHeight}|${buyCost}]";
}

/// <summary>
/// Vai trò của kệ hàng.
/// Tương đương ShelfRole type trong furniture/types/index.ts cũ.
/// </summary>
public enum ShelfRole
{
    Selling,  // 'selling' — NPC có thể mua
    Storage   // 'storage' — Chỉ lưu trữ, NPC không mua
}
```

---

### 4.3 `GridSystem.cs`

**Vị trí:** `Assets/Scripts/Grid/GridSystem.cs`  
**Mục đích:** Singleton quản lý toàn bộ ma trận lưới. Dictionary O(1) lookup, xử lý shop bounds, validate và confirm placement.

```csharp
// Assets/Scripts/Grid/GridSystem.cs

using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Tilemaps;

/// <summary>
/// Singleton quản lý ma trận lưới của cửa hàng.
///
/// KIẾN TRÚC:
///   _grid: Dictionary<Vector2Int, GridNode>
///     → Key:   Tọa độ cell (x, y) trong không gian lưới
///     → Value: GridNode struct lưu trạng thái ô (occupied, occupantId...)
///
/// TẠI SAO DICTIONARY THAY VÌ MẢNG 2D:
///   - Shop mở rộng động (expansionLevel tăng) → kích thước thay đổi runtime
///   - Dictionary cho phép thêm cell mới mà không cần resize toàn bộ array
///   - Lookup O(1) thay vì O(n) của List
///   - Sparse representation: chỉ tạo node cho cells thực sự tồn tại
///
/// MAPPING TỪ HỆ THỐNG CŨ (MainScene.ts):
///   validatePlacement()     → ValidatePlacement()
///   placeFurniture()        → ConfirmPlacement()
///   shopBounds check        → IsWithinShopBounds() + GridNode.IsWithinShopBounds
///   furnitureManager.remove → ReleaseCells()
/// </summary>
public class GridSystem : MonoBehaviour
{
    // =========================================================================
    // SINGLETON
    // =========================================================================

    public static GridSystem Instance { get; private set; }

    // =========================================================================
    // CONFIGURATION
    // =========================================================================

    [Header("Grid Reference")]
    [Tooltip("Unity Grid component trên scene. PHẢI là Isometric Z as Y layout. " +
             "Dùng API Grid.WorldToCell() và Grid.CellToWorld().")]
    [SerializeField] private Grid isometricGrid;

    [Header("Shop Bounds")]
    [Tooltip("Tọa độ cell góc dưới-trái của shop (inclusive).")]
    [SerializeField] private Vector2Int shopMinCell = new Vector2Int(-10, -10);

    [Tooltip("Tọa độ cell góc trên-phải của shop (inclusive).")]
    [SerializeField] private Vector2Int shopMaxCell = new Vector2Int(10, 10);

    [Header("Debug")]
    [Tooltip("Bật log chi tiết cho mọi placement operation. Tắt trong production.")]
    [SerializeField] private bool verboseLogging = true;

    // =========================================================================
    // GRID STATE
    // =========================================================================

    /// <summary>
    /// Ma trận lưới chính.
    /// Key:   Vector2Int — tọa độ cell (x, y)
    /// Value: GridNode  — trạng thái ô (occupied, occupantId, inBounds...)
    ///
    /// Tương đương spatially-indexed replacement cho toàn bộ
    /// physics collision system của Phaser/MainScene.ts cũ.
    /// </summary>
    private Dictionary<Vector2Int, GridNode> _grid;

    /// <summary>
    /// Map từ furnitureInstanceId → danh sách cells đang chiếm.
    /// Dùng để release cells khi xóa/di chuyển furniture.
    /// Key:   furnitureInstanceId (string)
    /// Value: List<Vector2Int> cells đang bị chiếm
    /// </summary>
    private Dictionary<string, List<Vector2Int>> _furnitureFootprints;

    // =========================================================================
    // KHỞI TẠO
    // =========================================================================

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        if (isometricGrid == null)
        {
            Debug.LogError("[GridSystem] isometricGrid chưa được assign! " +
                           "Kéo thả Grid GameObject vào field 'Isometric Grid'.");
            enabled = false;
            return;
        }

        InitializeGrid();
    }

    /// <summary>
    /// Khởi tạo dictionary và đánh dấu tất cả cells trong shop bounds là Empty.
    /// Cells ngoài bounds sẽ có IsWithinShopBounds = false khi query.
    /// </summary>
    private void InitializeGrid()
    {
        _grid = new Dictionary<Vector2Int, GridNode>();
        _furnitureFootprints = new Dictionary<string, List<Vector2Int>>();

        int cellCount = 0;
        for (int x = shopMinCell.x; x <= shopMaxCell.x; x++)
        {
            for (int y = shopMinCell.y; y <= shopMaxCell.y; y++)
            {
                var cellCoord = new Vector2Int(x, y);
                _grid[cellCoord] = GridNode.Empty;
                cellCount++;
            }
        }

        Debug.Log($"[GridSystem] Initialized: {cellCount} cells " +
                  $"({shopMinCell} to {shopMaxCell}).");
    }

    // =========================================================================
    // COORDINATE CONVERSION API
    // =========================================================================

    /// <summary>
    /// Chuyển đổi world position sang cell coordinate.
    /// Dùng Grid.WorldToCell() — API chuẩn Unity, xử lý đúng Isometric Z-as-Y.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   Không có — Phaser dùng world coordinates trực tiếp.
    ///   Đây là cải tiến mới: snap-to-grid thay vì pixel-perfect positioning.
    /// </summary>
    public Vector2Int WorldToCell(Vector3 worldPosition)
    {
        Vector3Int cell3D = isometricGrid.WorldToCell(worldPosition);
        return new Vector2Int(cell3D.x, cell3D.y);
    }

    /// <summary>
    /// Chuyển đổi cell coordinate sang world position (tâm cell).
    /// Dùng Grid.CellToWorld() rồi offset về tâm cell.
    /// </summary>
    public Vector3 CellToWorld(Vector2Int cellCoord)
    {
        Vector3Int cell3D = new Vector3Int(cellCoord.x, cellCoord.y, 0);
        // CellToWorld trả về góc dưới-trái của cell, cộng thêm nửa cellSize để ra tâm
        Vector3 bottomLeft = isometricGrid.CellToWorld(cell3D);
        return bottomLeft + isometricGrid.cellSize * 0.5f;
    }

    /// <summary>
    /// Snap một world position về tâm của cell gần nhất.
    /// Dùng để hiển thị ghost preview đúng vị trí.
    /// </summary>
    public Vector3 SnapToGrid(Vector3 worldPosition)
    {
        Vector2Int cell = WorldToCell(worldPosition);
        return CellToWorld(cell);
    }

    // =========================================================================
    // GRID QUERY API — O(1) lookups
    // =========================================================================

    /// <summary>
    /// Lấy GridNode tại một cell. Trả về OutOfBounds nếu cell không tồn tại.
    /// O(1) complexity.
    /// </summary>
    public GridNode GetNode(Vector2Int cellCoord)
    {
        return _grid.TryGetValue(cellCoord, out GridNode node)
            ? node
            : GridNode.OutOfBounds;
    }

    /// <summary>
    /// Kiểm tra cell có nằm trong shop bounds không.
    /// </summary>
    public bool IsWithinShopBounds(Vector2Int cellCoord)
    {
        return cellCoord.x >= shopMinCell.x && cellCoord.x <= shopMaxCell.x &&
               cellCoord.y >= shopMinCell.y && cellCoord.y <= shopMaxCell.y;
    }

    /// <summary>
    /// Kiểm tra cell có thể đặt vật thể không.
    /// </summary>
    public bool IsCellPlaceable(Vector2Int cellCoord)
    {
        return GetNode(cellCoord).IsPlaceable;
    }

    // =========================================================================
    // PLACEMENT VALIDATION
    // =========================================================================

    /// <summary>
    /// Kiểm tra một furniture definition có thể đặt tại origin cell không.
    /// Kiểm tra TẤT CẢ cells trong footprint — không chỉ origin cell.
    ///
    /// ĐÂY LÀ THUẬT TOÁN LÕI thay thế validatePlacement() của MainScene.ts cũ.
    ///
    /// LUỒNG KIỂM TRA:
    ///   FOR EACH relativeCellOffset IN footprint:
    ///     absoluteCell = originCell + relativeCellOffset
    ///     IF absoluteCell out of bounds → INVALID
    ///     IF _grid[absoluteCell].isOccupied → INVALID
    ///   RETURN VALID
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   validatePlacement() trong MainScene.ts, nhưng:
    ///   - Cũ dùng Phaser.Geom.Intersects.RectangleToRectangle (pixel-level, O(n))
    ///   - Mới dùng Dictionary lookup (cell-level, O(footprintSize))
    ///   footprintSize <= 4 → thực tế là O(1)
    /// </summary>
    /// <param name="originCell">Cell gốc (thường là cell trái-dưới của footprint).</param>
    /// <param name="definition">Loại furniture cần đặt.</param>
    /// <param name="rotationDegrees">Góc xoay hiện tại (0, 90, 180, 270).</param>
    /// <param name="failReason">Lý do thất bại nếu trả về false.</param>
    public bool ValidatePlacement(
        Vector2Int originCell,
        FurnitureDefinition definition,
        int rotationDegrees,
        out string failReason)
    {
        failReason = string.Empty;

        if (definition == null)
        {
            failReason = "FurnitureDefinition là null.";
            return false;
        }

        List<Vector2Int> footprint = definition.GetFootprintCells(rotationDegrees);

        foreach (Vector2Int relativeOffset in footprint)
        {
            Vector2Int absoluteCell = originCell + relativeOffset;

            // Kiểm tra 1: Nằm trong shop bounds
            if (!IsWithinShopBounds(absoluteCell))
            {
                failReason = $"Vị trí không hợp lệ: cell {absoluteCell} nằm ngoài biên giới cửa hàng.";
                return false;
            }

            // Kiểm tra 2: Ô chưa bị chiếm
            GridNode node = GetNode(absoluteCell);
            if (node.IsOccupied)
            {
                // LOG LỖI CHÍNH XÁC THEO YÊU CẦU
                // Thông điệp này sẽ xuất hiện khi người chơi cố click đè lên kệ đã đặt
                failReason = $"Vị trí không hợp lệ, lưới bị chiếm dụng tại {absoluteCell} " +
                             $"bởi [{node.OccupantType}] ID='{node.OccupantId}'.";
                return false;
            }
        }

        return true;
    }

    // =========================================================================
    // PLACEMENT CONFIRMATION — Cập nhật ma trận sau khi đặt
    // =========================================================================

    /// <summary>
    /// Đánh dấu tất cả cells trong footprint là OCCUPIED sau khi đặt furniture.
    /// Gọi SAU KHI ValidatePlacement() trả về true VÀ prefab đã được Instantiate.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   placeFurniture() → this.furnitureManager.addFurnitureToScene(placedData)
    ///   Thay vì Phaser tạo physics body, ta đánh dấu cells trong dictionary.
    /// </summary>
    /// <param name="originCell">Cell gốc của furniture.</param>
    /// <param name="definition">Loại furniture.</param>
    /// <param name="rotationDegrees">Góc xoay.</param>
    /// <param name="furnitureInstanceId">ID duy nhất của instance (để release sau).</param>
    public void ConfirmPlacement(
        Vector2Int originCell,
        FurnitureDefinition definition,
        int rotationDegrees,
        string furnitureInstanceId)
    {
        List<Vector2Int> footprint = definition.GetFootprintCells(rotationDegrees);
        var occupiedCells = new List<Vector2Int>();

        foreach (Vector2Int relativeOffset in footprint)
        {
            Vector2Int absoluteCell = originCell + relativeOffset;

            if (_grid.ContainsKey(absoluteCell))
            {
                _grid[absoluteCell] = new GridNode
                {
                    IsOccupied = true,
                    OccupantId = furnitureInstanceId,
                    OccupantType = definition.furnitureType,
                    IsWithinShopBounds = true
                };
                occupiedCells.Add(absoluteCell);
            }
        }

        // Lưu footprint của instance để release sau
        _furnitureFootprints[furnitureInstanceId] = occupiedCells;

        if (verboseLogging)
        {
            Debug.Log($"[GridSystem] Placed [{definition.furnitureType}] " +
                      $"ID='{furnitureInstanceId}' at origin={originCell}, " +
                      $"rotation={rotationDegrees}°, " +
                      $"cells occupied: {occupiedCells.Count}.");
        }
    }

    // =========================================================================
    // RELEASE CELLS — Khi xóa hoặc di chuyển furniture
    // =========================================================================

    /// <summary>
    /// Giải phóng tất cả cells của một furniture instance.
    /// Gọi khi người chơi nhặt furniture lên (Edit Mode) hoặc xóa.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   furnitureManager.removeFurniture(id, type) trong MainScene.ts
    ///   handleFurniturePickup() → delete placedShelves[id]
    /// </summary>
    /// <param name="furnitureInstanceId">ID của instance cần giải phóng.</param>
    /// <returns>true nếu release thành công, false nếu không tìm thấy.</returns>
    public bool ReleaseCells(string furnitureInstanceId)
    {
        if (!_furnitureFootprints.TryGetValue(furnitureInstanceId, out List<Vector2Int> cells))
        {
            Debug.LogWarning($"[GridSystem] ReleaseCells: Không tìm thấy footprint " +
                             $"cho ID='{furnitureInstanceId}'.");
            return false;
        }

        foreach (Vector2Int cell in cells)
        {
            if (_grid.ContainsKey(cell))
            {
                _grid[cell] = GridNode.Empty;
            }
        }

        _furnitureFootprints.Remove(furnitureInstanceId);

        if (verboseLogging)
        {
            Debug.Log($"[GridSystem] Released {cells.Count} cells " +
                      $"from furniture ID='{furnitureInstanceId}'.");
        }

        return true;
    }

    // =========================================================================
    // SHOP EXPANSION — Mở rộng bounds khi shop level tăng
    // =========================================================================

    /// <summary>
    /// Mở rộng grid bounds khi người chơi mua expansion.
    /// Thêm cells mới vào dictionary mà không xóa cells cũ.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   buyExpansion() → statsStore.expansionLevel++
    ///   → environmentManager.refreshEnvironment()
    ///   → shopBounds thay đổi → wallsGroup.updateFromGameObject()
    /// </summary>
    public void ExpandShopBounds(Vector2Int newMinCell, Vector2Int newMaxCell)
    {
        int newCellCount = 0;

        for (int x = newMinCell.x; x <= newMaxCell.x; x++)
        {
            for (int y = newMinCell.y; y <= newMaxCell.y; y++)
            {
                var cellCoord = new Vector2Int(x, y);
                if (!_grid.ContainsKey(cellCoord))
                {
                    _grid[cellCoord] = GridNode.Empty;
                    newCellCount++;
                }
            }
        }

        shopMinCell = newMinCell;
        shopMaxCell = newMaxCell;

        Debug.Log($"[GridSystem] Shop expanded to ({newMinCell} → {newMaxCell}). " +
                  $"Added {newCellCount} new cells. " +
                  $"Total cells: {_grid.Count}.");
    }

    // =========================================================================
    // DEBUG UTILITIES
    // =========================================================================

    /// <summary>
    /// In trạng thái toàn bộ grid ra Console. Dùng khi debug.
    /// </summary>
    [ContextMenu("Debug: Print Grid State")]
    public void PrintGridState()
    {
        int occupied = 0, empty = 0;
        foreach (var node in _grid.Values)
        {
            if (node.IsOccupied) occupied++;
            else empty++;
        }

        Debug.Log($"[GridSystem] Grid State: " +
                  $"Total={_grid.Count}, Occupied={occupied}, Empty={empty}, " +
                  $"Furniture instances={_furnitureFootprints.Count}");
    }

    /// <summary>
    /// Expose Grid component cho GhostObject sử dụng WorldToCell.
    /// </summary>
    public Grid IsometricGrid => isometricGrid;
}
```

---

### 4.4 `GhostObject.cs`

**Vị trí:** `Assets/Scripts/Placement/GhostObject.cs`  
**Mục đích:** Visual feedback ghost preview. Di chuyển theo chuột, đổi màu xanh (valid) / đỏ (invalid) theo kết quả ValidatePlacement().

```csharp
// Assets/Scripts/Placement/GhostObject.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Component điều khiển ghost preview khi đang trong placement mode.
/// Tự cập nhật vị trí theo con trỏ chuột và đổi màu xanh/đỏ.
///
/// VÒNG ĐỜI:
///   1. PlacementManager.StartPlacement() → Instantiate ghost
///   2. Mỗi frame: ghost.UpdatePreview(mouseWorldPos) → snap + validate + recolor
///   3. PlacementManager.ConfirmPlacement() → Destroy ghost
///   4. PlacementManager.CancelPlacement()  → Destroy ghost
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ (MainScene.ts):
///   ghostSprite / ghostRectangle / ghostText → GhostObject component
///   updateGhostPosition() → UpdatePreview()
///   updateGhostVisual()   → UpdateGhostColor()
///   isPlacementValid      → _isCurrentPositionValid
/// </summary>
public class GhostObject : MonoBehaviour
{
    // =========================================================================
    // CONFIGURATION
    // =========================================================================

    [Header("Color Settings")]
    [Tooltip("Màu ghost khi vị trí HỢP LỆ (có thể đặt). Xanh lá bán trong suốt.")]
    [SerializeField] private Color validColor = new Color(0f, 1f, 0f, 0.6f);

    [Tooltip("Màu ghost khi vị trí KHÔNG HỢP LỆ (không thể đặt). Đỏ bán trong suốt.")]
    [SerializeField] private Color invalidColor = new Color(1f, 0f, 0f, 0.6f);

    [Tooltip("Tốc độ di chuyển của ghost về vị trí mới. " +
             "Cao hơn = mượt hơn nhưng có độ trễ nhỏ. Dùng Mathf.Lerp.")]
    [SerializeField][Range(1f, 30f)] private float followSpeed = 15f;

    // =========================================================================
    // RUNTIME STATE — Cache, không GetComponent trong Update
    // =========================================================================

    private FurnitureDefinition _definition;
    private int _currentRotation = 0;
    private bool _isCurrentPositionValid = false;
    private Vector2Int _currentCellPosition;
    private Vector3 _targetWorldPosition;

    // Cache renderer references — set khi Awake, không tìm lại trong Update
    private List<SpriteRenderer> _spriteRenderers = new List<SpriteRenderer>();
    private List<MeshRenderer> _meshRenderers = new List<MeshRenderer>();

    // Material instances (tạo bản copy để không ảnh hưởng shared material)
    private List<Material> _materialInstances = new List<Material>();

    // =========================================================================
    // VÒNG ĐỜI
    // =========================================================================

    private void Awake()
    {
        // Cache tất cả renderers — KHÔNG GetComponent trong Update
        GetComponentsInChildren(true, _spriteRenderers);
        GetComponentsInChildren(true, _meshRenderers);

        // Tạo material instances để tô màu độc lập (không ảnh hưởng shared material)
        foreach (var sr in _spriteRenderers)
        {
            if (sr.material != null)
            {
                var matInstance = new Material(sr.material);
                sr.material = matInstance;
                _materialInstances.Add(matInstance);
            }
        }

        foreach (var mr in _meshRenderers)
        {
            if (mr.material != null)
            {
                var matInstance = new Material(mr.material);
                mr.material = matInstance;
                _materialInstances.Add(matInstance);
            }
        }

        // Đặt màu invalid mặc định khi mới tạo
        ApplyColor(invalidColor);
    }

    private void OnDestroy()
    {
        // Dọn dẹp material instances tránh memory leak
        foreach (var mat in _materialInstances)
        {
            if (mat != null)
                Destroy(mat);
        }
        _materialInstances.Clear();
    }

    // =========================================================================
    // PUBLIC API — Gọi từ PlacementManager
    // =========================================================================

    /// <summary>
    /// Khởi tạo ghost với definition của furniture sẽ đặt.
    /// Gọi một lần ngay sau khi ghost được Instantiate.
    /// </summary>
    public void Initialize(FurnitureDefinition definition)
    {
        _definition = definition;
        _currentRotation = 0;
    }

    /// <summary>
    /// Cập nhật vị trí và màu ghost mỗi frame.
    /// Gọi từ PlacementManager.Update() với vị trí world của con trỏ chuột.
    ///
    /// LUỒNG:
    ///   1. Snap mouseWorldPos về tâm cell gần nhất (Grid.WorldToCell + CellToWorld)
    ///   2. Smooth move ghost về vị trí đó (Mathf.Lerp)
    ///   3. Validate placement tại cell đó
    ///   4. Đổi màu ghost xanh/đỏ tương ứng
    /// </summary>
    public void UpdatePreview(Vector3 mouseWorldPosition)
    {
        if (_definition == null || GridSystem.Instance == null) return;

        // BƯỚC 1: Snap về cell gần nhất
        _currentCellPosition = GridSystem.Instance.WorldToCell(mouseWorldPosition);
        _targetWorldPosition = GridSystem.Instance.CellToWorld(_currentCellPosition);

        // Điều chỉnh offset cho footprint lớn hơn 1x1
        // Tâm footprint phải nằm dưới con trỏ, không phải góc trái-dưới
        _targetWorldPosition = AdjustForFootprintCenter(_targetWorldPosition);

        // BƯỚC 2: Di chuyển mượt ghost về vị trí target
        // Dùng Mathf.Lerp thay vì teleport để ghost "trượt" mượt mà
        transform.position = Vector3.Lerp(
            transform.position,
            _targetWorldPosition,
            followSpeed * Time.deltaTime
        );

        // BƯỚC 3: Validate placement tại cell hiện tại
        bool isValid = GridSystem.Instance.ValidatePlacement(
            _currentCellPosition,
            _definition,
            _currentRotation,
            out string _  // Bỏ qua fail reason ở đây — PlacementManager sẽ log khi cần
        );

        // BƯỚC 4: Đổi màu nếu trạng thái thay đổi (tránh set material mỗi frame)
        if (isValid != _isCurrentPositionValid)
        {
            _isCurrentPositionValid = isValid;
            ApplyColor(isValid ? validColor : invalidColor);
        }
    }

    /// <summary>
    /// Xoay ghost 90° theo chiều kim đồng hồ và cập nhật visual.
    /// Gọi khi người chơi nhấn phím R.
    /// </summary>
    public void Rotate()
    {
        if (_definition == null || !_definition.canRotate) return;

        _currentRotation = (_currentRotation + 90) % 360;

        // Xoay Transform của ghost object
        transform.rotation = Quaternion.Euler(0f, 0f, 0f);

        // Với Isometric, xoay visual thường là thay đổi sprite
        // thay vì xoay Transform. Đây là placeholder — 
        // thay thế bằng logic sprite switching khi có sprite assets.
        Debug.Log($"[GhostObject] Rotated to {_currentRotation}°. " +
                  $"New footprint: {_definition.GetFootprintCells(_currentRotation).Count} cells.");
    }

    // =========================================================================
    // GETTERS — Cho PlacementManager
    // =========================================================================

    public bool IsCurrentPositionValid => _isCurrentPositionValid;
    public Vector2Int CurrentCellPosition => _currentCellPosition;
    public int CurrentRotation => _currentRotation;

    // =========================================================================
    // PRIVATE HELPERS
    // =========================================================================

    /// <summary>
    /// Điều chỉnh world position để tâm của footprint nằm dưới con trỏ
    /// thay vì góc trái-dưới của footprint.
    ///
    /// Ví dụ footprint 2x1: origin cell = cursor cell - (0.5, 0)
    /// Giúp ghost "bao quanh" con trỏ thay vì lệch sang một bên.
    /// </summary>
    private Vector3 AdjustForFootprintCenter(Vector3 originWorldPos)
    {
        if (_definition == null) return originWorldPos;

        var (minBounds, maxBounds) = _definition.GetFootprintBounds(_currentRotation);

        // Tính offset để căn tâm footprint về con trỏ
        float centerOffsetX = (minBounds.x + maxBounds.x) * 0.5f;
        float centerOffsetY = (minBounds.y + maxBounds.y) * 0.5f;

        // Chuyển offset cell sang world units
        Vector3 cellSize = GridSystem.Instance.IsometricGrid.cellSize;
        return originWorldPos - new Vector3(
            centerOffsetX * cellSize.x,
            centerOffsetY * cellSize.y,
            0f
        );
    }

    /// <summary>
    /// Áp dụng màu cho tất cả renderers của ghost.
    /// Dùng material instances đã tạo trong Awake — không set shared material.
    /// </summary>
    private void ApplyColor(Color color)
    {
        foreach (var sr in _spriteRenderers)
        {
            if (sr != null)
                sr.color = color;
        }

        foreach (var mr in _meshRenderers)
        {
            if (mr != null && mr.material != null)
                mr.material.color = color;
        }
    }
}
```

---

### 4.5 `PlacedFurnitureInstance.cs`

**Vị trí:** `Assets/Scripts/Placement/PlacedFurnitureInstance.cs`  
**Mục đích:** Component gắn lên prefab sau khi đặt xuống lưới. Lưu trữ metadata về instance và cung cấp API để EditMode nhặt lên.

```csharp
// Assets/Scripts/Placement/PlacedFurnitureInstance.cs

using UnityEngine;

/// <summary>
/// Component gắn lên mỗi furniture GameObject đã được đặt vào scene.
/// Lưu trữ "identity" của instance: ai là nó, đặt ở đâu, xoay bao nhiêu.
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
///   sprite.setData('id', shelf.id)   → InstanceId
///   sprite.setData('type', 'shelf')  → Definition.furnitureType
///   shelf.x, shelf.y                 → OriginCell
///   table.rotation                   → PlacedRotation
/// </summary>
[DisallowMultipleComponent]
public class PlacedFurnitureInstance : MonoBehaviour
{
    // =========================================================================
    // IDENTITY — Được set bởi PlacementManager.ConfirmPlacement()
    // =========================================================================

    /// <summary>
    /// ID duy nhất của instance này. Format: "furniture_{type}_{timestamp}_{random}".
    /// Tương đương 'shelf_' + Date.now() trong furnitureStore.ts cũ.
    /// </summary>
    public string InstanceId { get; private set; }

    /// <summary>
    /// Definition của loại furniture này.
    /// </summary>
    public FurnitureDefinition Definition { get; private set; }

    /// <summary>
    /// Cell gốc trên lưới (trái-dưới của footprint).
    /// </summary>
    public Vector2Int OriginCell { get; private set; }

    /// <summary>
    /// Góc xoay khi đặt (0, 90, 180, 270).
    /// </summary>
    public int PlacedRotation { get; private set; }

    /// <summary>
    /// Thời điểm được đặt xuống (dùng cho save/load ordering).
    /// </summary>
    public float PlacedAt { get; private set; }

    // =========================================================================
    // KHỞI TẠO
    // =========================================================================

    /// <summary>
    /// Khởi tạo instance sau khi Instantiate.
    /// Gọi bởi PlacementManager.ConfirmPlacement().
    /// </summary>
    public void Initialize(
        string instanceId,
        FurnitureDefinition definition,
        Vector2Int originCell,
        int rotation)
    {
        InstanceId = instanceId;
        Definition = definition;
        OriginCell = originCell;
        PlacedRotation = rotation;
        PlacedAt = Time.time;

        // Đặt tên GameObject cho dễ debug trong Hierarchy
        gameObject.name = $"[Furniture] {definition.furnitureType} ({instanceId[^8..]})";
    }

    // =========================================================================
    // TIỆN ÍCH
    // =========================================================================

    public override string ToString() =>
        $"FurnitureInstance[{InstanceId}|{Definition?.furnitureType}|" +
        $"cell={OriginCell}|rot={PlacedRotation}°]";
}
```

---

### 4.6 `PlacementManager.cs`

**Vị trí:** `Assets/Scripts/Placement/PlacementManager.cs`  
**Mục đích:** Điều phối toàn bộ luồng placement từ đầu đến cuối. Đây là state machine chính cho Build Mode.

```csharp
// Assets/Scripts/Placement/PlacementManager.cs

using System;
using UnityEngine;
using UnityEngine.InputSystem;

/// <summary>
/// Quản lý toàn bộ luồng đặt nội thất (Build Mode).
///
/// STATE MACHINE:
///   IDLE
///     ↓ StartPlacement(definition)
///   PLACING
///     - Mỗi frame: UpdateGhostPreview(mousePos)
///     - Phím R:    ghost.Rotate()
///     - Click:     TryConfirmPlacement()
///       ↓ Valid    → ConfirmPlacement() → IDLE
///       ↓ Invalid  → Log "Vị trí không hợp lệ" → tiếp tục PLACING
///     - ESC/RClick: CancelPlacement() → IDLE
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ (MainScene.ts):
///   handleBuildMode()    → Update() trong PlacingState
///   updateGhostPosition()→ ghost.UpdatePreview()
///   placeFurniture()     → TryConfirmPlacement() + ConfirmPlacement()
///   cancelPlacement()    → CancelPlacement()
///   validatePlacement()  → GridSystem.ValidatePlacement()
///   clearGhost()         → DestroyGhost()
///
/// INPUT SYSTEM: Unity New Input System (không dùng Input.* cũ)
/// </summary>
public class PlacementManager : MonoBehaviour
{
    // =========================================================================
    // SINGLETON
    // =========================================================================

    public static PlacementManager Instance { get; private set; }

    // =========================================================================
    // CONFIGURATION
    // =========================================================================

    [Header("Placement Settings")]
    [Tooltip("Layer mask cho Raycast tìm vị trí chuột trên world plane.")]
    [SerializeField] private LayerMask groundLayerMask;

    [Tooltip("Camera chính. Dùng để Raycast từ screen space sang world space. " +
             "Được cache trong Awake — không gọi Camera.main trong Update.")]
    [SerializeField] private Camera mainCamera;

    [Tooltip("Cooldown tối thiểu giữa hai lần placement (giây). " +
             "Tránh double-place do click lag. " +
             "Tương đương: this.time.now > this.lastPlacementTime + 300 trong MainScene.ts cũ.")]
    [SerializeField][Range(0.1f, 1f)] private float placementCooldown = 0.3f;

    [Header("Ghost Settings")]
    [Tooltip("Parent Transform để chứa ghost object. Tổ chức Hierarchy gọn gàng.")]
    [SerializeField] private Transform ghostParent;

    [Header("Furniture Parent")]
    [Tooltip("Parent Transform để chứa tất cả placed furniture instances.")]
    [SerializeField] private Transform furnitureParent;

    [Header("Debug")]
    [SerializeField] private bool verboseLogging = true;

    // =========================================================================
    // PLACEMENT STATE
    // =========================================================================

    /// <summary>Trạng thái hiện tại của PlacementManager.</summary>
    public enum PlacementState { Idle, Placing }
    public PlacementState CurrentState { get; private set; } = PlacementState.Idle;

    /// <summary>Definition của furniture đang được đặt.</summary>
    private FurnitureDefinition _activeFurnitureDefinition;

    /// <summary>Ghost object hiện tại (null khi Idle).</summary>
    private GhostObject _activeGhost;

    /// <summary>Thời điểm lần đặt gần nhất (cooldown).</summary>
    private float _lastPlacementTime = -999f;

    // Input devices — Cache trong Awake
    private Mouse _mouse;
    private Keyboard _keyboard;

    // =========================================================================
    // EVENTS — Các hệ thống khác (UI, Inventory) subscribe để phản ứng
    // =========================================================================

    /// <summary>Kích hoạt khi đặt furniture thành công. Tham số: instance đã đặt.</summary>
    public event Action<PlacedFurnitureInstance> OnFurniturePlaced;

    /// <summary>Kích hoạt khi placement bị hủy.</summary>
    public event Action OnPlacementCancelled;

    /// <summary>Kích hoạt khi thử đặt vào vị trí không hợp lệ. Tham số: lý do.</summary>
    public event Action<string> OnPlacementFailed;

    // =========================================================================
    // VÒNG ĐỜI
    // =========================================================================

    private void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        // Cache camera — KHÔNG gọi Camera.main trong Update
        if (mainCamera == null)
            mainCamera = Camera.main;

        if (mainCamera == null)
        {
            Debug.LogError("[PlacementManager] mainCamera không tìm thấy! " +
                           "Assign Camera vào field hoặc đảm bảo có Camera tag 'MainCamera'.");
            enabled = false;
            return;
        }

        // Cache input devices
        _mouse = Mouse.current;
        _keyboard = Keyboard.current;

        Debug.Log("[PlacementManager] Initialized.");
    }

    private void Update()
    {
        // Cập nhật input device references (hot-plug support)
        if (_mouse == null) _mouse = Mouse.current;
        if (_keyboard == null) _keyboard = Keyboard.current;

        switch (CurrentState)
        {
            case PlacementState.Idle:
                HandleIdleState();
                break;

            case PlacementState.Placing:
                HandlePlacingState();
                break;
        }
    }

    // =========================================================================
    // STATE HANDLERS
    // =========================================================================

    private void HandleIdleState()
    {
        // Trong Idle, PlacementManager không làm gì.
        // Chờ StartPlacement() được gọi từ UI.
    }

    /// <summary>
    /// Xử lý toàn bộ logic khi đang trong Placing state.
    /// Gọi mỗi frame — KHÔNG gọi GetComponent hay Find ở đây.
    /// </summary>
    private void HandlePlacingState()
    {
        if (_activeGhost == null || _mouse == null) return;

        // BƯỚC 1: Lấy vị trí world của con trỏ chuột
        Vector3 mouseWorldPosition = GetMouseWorldPosition();
        if (mouseWorldPosition == Vector3.negativeInfinity) return;

        // BƯỚC 2: Cập nhật ghost preview
        _activeGhost.UpdatePreview(mouseWorldPosition);

        // BƯỚC 3: Xử lý phím R — Xoay 90°
        if (_keyboard != null && _keyboard.rKey.wasPressedThisFrame)
        {
            HandleRotation();
        }

        // BƯỚC 4: Click chuột trái — Thử đặt
        if (_mouse.leftButton.wasPressedThisFrame)
        {
            // Cooldown check tương đương: this.time.now > this.lastPlacementTime + 300
            if (Time.time - _lastPlacementTime >= placementCooldown)
            {
                TryConfirmPlacement();
            }
        }

        // BƯỚC 5: ESC hoặc chuột phải — Hủy
        if (_mouse.rightButton.wasPressedThisFrame ||
            (_keyboard != null && _keyboard.escapeKey.wasPressedThisFrame))
        {
            CancelPlacement();
        }
    }

    // =========================================================================
    // PUBLIC API — Gọi từ UI (Build Menu)
    // =========================================================================

    /// <summary>
    /// Bắt đầu placement mode cho một loại furniture.
    /// Gọi từ Build Menu khi người chơi chọn furniture muốn đặt.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   gameStore.startBuildMode(furnitureId) trong OnlineShopMenu.vue / BuildMenu.vue
    ///   → useUIStore().showBuildMenu = false
    ///   → useFurnitureStore().startBuildMode(furnitureId)
    ///   Sau đó MainScene.ts handleBuildMode() bắt đầu chạy.
    /// </summary>
    public void StartPlacement(FurnitureDefinition definition)
    {
        if (definition == null)
        {
            Debug.LogError("[PlacementManager] StartPlacement: definition là null!");
            return;
        }

        if (!definition.IsValid())
        {
            Debug.LogError($"[PlacementManager] StartPlacement: Definition '{definition.name}' " +
                           "không hợp lệ. Kiểm tra furniturePrefab và kích thước footprint.");
            return;
        }

        // Hủy placement cũ nếu đang có
        if (CurrentState == PlacementState.Placing)
        {
            CancelPlacement();
        }

        _activeFurnitureDefinition = definition;

        // Tạo ghost object
        SpawnGhost(definition);

        CurrentState = PlacementState.Placing;

        if (verboseLogging)
        {
            Debug.Log($"[PlacementManager] Started placement: {definition.furnitureType} " +
                      $"({definition.footprintWidth}×{definition.footprintHeight}).");
        }
    }

    /// <summary>
    /// Hủy placement mode hiện tại.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   cancelPlacement(store) → clearGhost()
    ///   store.cancelBuildMode() → isBuildMode = false, buildItemId = null
    /// </summary>
    public void CancelPlacement()
    {
        DestroyGhost();
        _activeFurnitureDefinition = null;
        CurrentState = PlacementState.Idle;

        OnPlacementCancelled?.Invoke();

        if (verboseLogging)
        {
            Debug.Log("[PlacementManager] Placement cancelled.");
        }
    }

    // =========================================================================
    // PLACEMENT EXECUTION
    // =========================================================================

    /// <summary>
    /// Thử xác nhận placement tại vị trí ghost hiện tại.
    /// Nếu không hợp lệ, log lỗi và tiếp tục PLACING state (không hủy).
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   handleBuildMode() → if (pointer.isDown && isPlacementValid):
    ///     placeFurniture(pointer, store)
    ///   handleBuildMode() → else nếu invalid: không làm gì (tiếp tục hiện ghost đỏ)
    /// </summary>
    private void TryConfirmPlacement()
    {
        if (_activeGhost == null || _activeFurnitureDefinition == null) return;

        Vector2Int targetCell = _activeGhost.CurrentCellPosition;
        int rotation = _activeGhost.CurrentRotation;

        // Validate một lần nữa với failReason chi tiết
        bool isValid = GridSystem.Instance.ValidatePlacement(
            targetCell,
            _activeFurnitureDefinition,
            rotation,
            out string failReason
        );

        if (!isValid)
        {
            // LOG LỖI CHO NGƯỜI CHƠI THẤY
            // Đây là thông điệp bắt buộc theo yêu cầu:
            // "Vị trí không hợp lệ, lưới bị chiếm dụng"
            Debug.LogWarning($"[PlacementManager] {failReason}");
            OnPlacementFailed?.Invoke(failReason);
            return;
        }

        // Placement hợp lệ → Thực thi
        ConfirmPlacement(targetCell, rotation);
    }

    /// <summary>
    /// Thực thi đặt furniture: Instantiate prefab, cập nhật grid, ghi nhận cooldown.
    ///
    /// LUỒNG CHI TIẾT:
    ///   1. Tạo ID duy nhất cho instance
    ///   2. Tính world position từ cell (Grid.CellToWorld)
    ///   3. Instantiate furniturePrefab
    ///   4. Initialize PlacedFurnitureInstance component
    ///   5. GridSystem.ConfirmPlacement() → đánh dấu cells occupied
    ///   6. Ghi nhận lastPlacementTime (cooldown)
    ///   7. Fire OnFurniturePlaced event
    ///   8. Reset về Idle (placement hoàn tất)
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   store.placeFurniture(x, y, rotation) → placedShelves[id] = ShelfData
    ///   furnitureManager.addFurnitureToScene(placedData) → Phaser sprite
    /// </summary>
    private void ConfirmPlacement(Vector2Int targetCell, int rotation)
    {
        // BƯỚC 1: Tạo instance ID duy nhất
        string instanceId = GenerateInstanceId(_activeFurnitureDefinition.furnitureType);

        // BƯỚC 2: Tính world position (tâm của cell gốc)
        Vector3 worldPosition = GridSystem.Instance.CellToWorld(targetCell);

        // BƯỚC 3: Instantiate prefab
        Transform parent = furnitureParent != null ? furnitureParent : transform;
        GameObject furnitureGO = Instantiate(
            _activeFurnitureDefinition.furniturePrefab,
            worldPosition,
            Quaternion.identity,
            parent
        );

        // BƯỚC 4: Initialize PlacedFurnitureInstance
        PlacedFurnitureInstance instance = furnitureGO.GetComponent<PlacedFurnitureInstance>();
        if (instance == null)
        {
            // Auto-add component nếu prefab thiếu (safety net)
            instance = furnitureGO.AddComponent<PlacedFurnitureInstance>();
            Debug.LogWarning($"[PlacementManager] Prefab '{_activeFurnitureDefinition.name}' " +
                             "thiếu PlacedFurnitureInstance component. Đã auto-add.");
        }

        instance.Initialize(instanceId, _activeFurnitureDefinition, targetCell, rotation);

        // BƯỚC 5: Cập nhật GridSystem
        GridSystem.Instance.ConfirmPlacement(
            targetCell,
            _activeFurnitureDefinition,
            rotation,
            instanceId
        );

        // BƯỚC 6: Ghi nhận cooldown
        _lastPlacementTime = Time.time;

        // BƯỚC 7: Fire event
        OnFurniturePlaced?.Invoke(instance);

        if (verboseLogging)
        {
            Debug.Log($"[PlacementManager] ✅ Placed {instance}");
        }

        // BƯỚC 8: Reset về Idle
        DestroyGhost();
        _activeFurnitureDefinition = null;
        CurrentState = PlacementState.Idle;
    }

    // =========================================================================
    // ROTATION
    // =========================================================================

    /// <summary>
    /// Xoay ghost 90° và log kết quả.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   input.keyboard.on('keydown-R', () => {
    ///     this.currentRotation = (this.currentRotation === 0) ? 90 : 0
    ///     this.clearGhost()
    ///   }) trong MainScene.ts
    ///
    ///   Hệ thống cũ chỉ hỗ trợ 0° và 90°, hệ thống mới hỗ trợ 0/90/180/270°.
    /// </summary>
    private void HandleRotation()
    {
        if (_activeGhost == null) return;

        if (!_activeFurnitureDefinition.canRotate)
        {
            Debug.Log($"[PlacementManager] {_activeFurnitureDefinition.furnitureType} " +
                      "không thể xoay (canRotate = false).");
            return;
        }

        _activeGhost.Rotate();

        Debug.Log($"[PlacementManager] Rotated to {_activeGhost.CurrentRotation}°. " +
                  $"New footprint: " +
                  $"{_activeFurnitureDefinition.GetFootprintCells(_activeGhost.CurrentRotation).Count} cells.");
    }

    // =========================================================================
    // GHOST MANAGEMENT
    // =========================================================================

    private void SpawnGhost(FurnitureDefinition definition)
    {
        DestroyGhost(); // Xóa ghost cũ nếu có

        GameObject ghostSource = definition.ghostPrefab != null
            ? definition.ghostPrefab
            : definition.furniturePrefab;

        if (ghostSource == null)
        {
            Debug.LogError($"[PlacementManager] Không có prefab để tạo ghost " +
                           $"cho '{definition.furnitureType}'!");
            return;
        }

        Transform parent = ghostParent != null ? ghostParent : transform;
        GameObject ghostGO = Instantiate(ghostSource, Vector3.zero, Quaternion.identity, parent);
        ghostGO.name = "[Ghost] " + definition.furnitureType;

        // Thêm GhostObject component nếu chưa có
        _activeGhost = ghostGO.GetComponent<GhostObject>();
        if (_activeGhost == null)
            _activeGhost = ghostGO.AddComponent<GhostObject>();

        _activeGhost.Initialize(definition);

        // Disable colliders trên ghost để không ảnh hưởng physics
        foreach (var col in ghostGO.GetComponentsInChildren<Collider2D>())
            col.enabled = false;
        foreach (var col in ghostGO.GetComponentsInChildren<Collider>())
            col.enabled = false;
    }

    private void DestroyGhost()
    {
        if (_activeGhost != null)
        {
            Destroy(_activeGhost.gameObject);
            _activeGhost = null;
        }
    }

    // =========================================================================
    // RAYCAST — Tìm vị trí world của con trỏ chuột
    // =========================================================================

    /// <summary>
    /// Raycast từ camera xuống ground plane để tìm world position của chuột.
    /// Trả về Vector3.negativeInfinity nếu raycast thất bại.
    ///
    /// Dùng New Input System: Mouse.current.position.ReadValue()
    /// KHÔNG dùng Input.mousePosition (legacy).
    /// </summary>
    private Vector3 GetMouseWorldPosition()
    {
        if (_mouse == null || mainCamera == null) return Vector3.negativeInfinity;

        Vector2 screenPos = _mouse.position.ReadValue();
        Ray ray = mainCamera.ScreenPointToRay(screenPos);

        // Raycast xuống ground layer (plane tại Z=0)
        if (Physics.Raycast(ray, out RaycastHit hit, 100f, groundLayerMask))
        {
            return hit.point;
        }

        // Fallback cho 2D: tính giao điểm với Z=0 plane
        if (Mathf.Abs(ray.direction.z) > 0.001f)
        {
            float t = -ray.origin.z / ray.direction.z;
            return ray.origin + ray.direction * t;
        }

        return Vector3.negativeInfinity;
    }

    // =========================================================================
    // UTILITIES
    // =========================================================================

    /// <summary>
    /// Tạo ID duy nhất cho một furniture instance.
    /// Format: "furniture_{type}_{timestamp}_{random4}"
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   'shelf_' + Date.now() trong furnitureStore.ts
    ///   'table_' + Date.now()
    ///   'cashier_' + Date.now()
    /// </summary>
    private string GenerateInstanceId(FurnitureType type)
    {
        string timestamp = DateTime.UtcNow.Ticks.ToString();
        string random = UnityEngine.Random.Range(1000, 9999).ToString();
        return $"furniture_{type}_{timestamp[^8..]}_{random}";
    }

    // =========================================================================
    // PUBLIC QUERY API
    // =========================================================================

    public bool IsInPlacementMode => CurrentState == PlacementState.Placing;
    public FurnitureDefinition ActiveDefinition => _activeFurnitureDefinition;
}
```

---

### 4.7 `GridVisualizer.cs`

**Vị trí:** `Assets/Scripts/Grid/GridVisualizer.cs`  
**Mục đích:** Vẽ debug lines cho lưới trong Editor và Play mode. Chỉ chạy khi debug flag bật.

```csharp
// Assets/Scripts/Grid/GridVisualizer.cs

using UnityEngine;

/// <summary>
/// Vẽ debug visualization của lưới grid.
/// Chỉ active trong Editor hoặc khi debug mode bật.
/// KHÔNG ảnh hưởng gameplay — component này có thể disable bất kỳ lúc nào.
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
///   drawPlacementVisualizer() trong MainScene.ts
///   placementGraphics.fillStyle(0xff0000, 0.3) → occupied cells màu đỏ
///   settings.showDebugPhysics → showGridLines toggle
/// </summary>
public class GridVisualizer : MonoBehaviour
{
    [Header("Visualization")]
    [Tooltip("Bật/tắt vẽ debug grid. Tương đương showDebugPhysics trong settings cũ.")]
    [SerializeField] private bool showGridLines = true;

    [Tooltip("Bật/tắt highlight ô đang bị occupied.")]
    [SerializeField] private bool highlightOccupied = true;

    [Tooltip("Màu đường kẻ lưới.")]
    [SerializeField] private Color gridLineColor = new Color(1f, 1f, 1f, 0.1f);

    [Tooltip("Màu highlight ô occupied.")]
    [SerializeField] private Color occupiedColor = new Color(1f, 0f, 0f, 0.25f);

    [Tooltip("Màu highlight ô empty.")]
    [SerializeField] private Color emptyColor = new Color(0f, 1f, 0f, 0.05f);

    private void OnDrawGizmos()
    {
        if (!showGridLines && !highlightOccupied) return;
        if (GridSystem.Instance == null) return;

        // Tham chiếu qua Instance — cache sẵn, không Find()
        var gridSystem = GridSystem.Instance;

        for (int x = gridSystem.transform.GetComponent<GridSystem>() != null
                 ? -15 : -15; x <= 15; x++)
        {
            for (int y = -15; y <= 15; y++)
            {
                var cellCoord = new Vector2Int(x, y);
                var node = gridSystem.GetNode(cellCoord);

                if (!node.IsWithinShopBounds) continue;

                Vector3 worldPos = gridSystem.CellToWorld(cellCoord);
                Vector3 cellSize = gridSystem.IsometricGrid != null
                    ? gridSystem.IsometricGrid.cellSize
                    : Vector3.one;

                // Vẽ ô
                if (highlightOccupied)
                {
                    Gizmos.color = node.IsOccupied ? occupiedColor : emptyColor;
                    Gizmos.DrawCube(worldPos, cellSize * 0.9f);
                }

                // Vẽ viền
                if (showGridLines)
                {
                    Gizmos.color = gridLineColor;
                    Gizmos.DrawWireCube(worldPos, cellSize);
                }
            }
        }
    }
}
```

---

## 5. Editor Script: Auto-Generate Sample Assets

```csharp
// Assets/Editor/FurnitureDataGenerator.cs

#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;

/// <summary>
/// Tạo FurnitureDefinition ScriptableObject assets cho tất cả 5 loại furniture.
/// Chạy qua menu: TCGShop > Setup > Generate Furniture Definitions
/// </summary>
public static class FurnitureDataGenerator
{
    [MenuItem("TCGShop/Setup/Generate Furniture Definitions")]
    public static void GenerateFurnitureDefinitions()
    {
        // Đảm bảo thư mục tồn tại
        if (!AssetDatabase.IsValidFolder("Assets/ScriptableObjects/Furniture"))
        {
            AssetDatabase.CreateFolder("Assets/ScriptableObjects", "Furniture");
        }

        CreateFurnitureDef("Furniture_ShelfSingle",  FurnitureType.ShelfSingle,
            "Single Sided Shelf", 1, 1, false, 300f, 3, 3, 16, ShelfRole.Selling,
            "Kệ gỗ 1 mặt tiêu chuẩn. NPCs có thể mua hàng từ kệ này.");

        CreateFurnitureDef("Furniture_ShelfDouble",  FurnitureType.ShelfDouble,
            "Double Sided Shelf", 2, 1, true, 750f, 11, 4, 32, ShelfRole.Selling,
            "Kệ trung tâm 2 mặt cao cấp. Sinh lời cực mạnh.");

        CreateFurnitureDef("Furniture_StorageShelf", FurnitureType.StorageShelf,
            "Storage Shelf", 1, 1, false, 150f, 1, 3, 4, ShelfRole.Storage,
            "Kệ kho đơn giản. Dùng để cất thùng hàng. NPCs KHÔNG mua từ đây.");

        CreateFurnitureDef("Furniture_PlayTable",    FurnitureType.PlayTable,
            "Play Table", 2, 2, true, 400f, 5, 0, 0, ShelfRole.Selling,
            "Bàn chơi bài cho khách hàng. Tạo XP thụ động khi có người thi đấu.");

        CreateFurnitureDef("Furniture_CashierDesk",  FurnitureType.CashierDesk,
            "Cashier Desk", 1, 1, false, 500f, 1, 0, 0, ShelfRole.Selling,
            "Quầy thu ngân tiêu chuẩn. Nơi khách mang hàng tới thanh toán.");

        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();

        Debug.Log("[FurnitureDataGenerator] ✅ Đã tạo 5 FurnitureDefinition assets.");
        EditorUtility.DisplayDialog("Done",
            "Đã tạo 5 FurnitureDefinition assets tại Assets/ScriptableObjects/Furniture/",
            "OK");
    }

    private static void CreateFurnitureDef(
        string fileName, FurnitureType type, string displayName,
        int width, int height, bool canRotate,
        float buyCost, int reqLevel,
        int numTiers, int slotsPerTier, ShelfRole role,
        string description)
    {
        string path = $"Assets/ScriptableObjects/Furniture/{fileName}.asset";

        // Không ghi đè nếu đã tồn tại
        var existing = AssetDatabase.LoadAssetAtPath<FurnitureDefinition>(path);
        if (existing != null)
        {
            Debug.Log($"[FurnitureDataGenerator] Skipped (already exists): {path}");
            return;
        }

        var def = ScriptableObject.CreateInstance<FurnitureDefinition>();
        def.furnitureType = type;
        def.displayName = displayName;
        def.footprintWidth = width;
        def.footprintHeight = height;
        def.canRotate = canRotate;
        def.buyCost = buyCost;
        def.requiredShopLevel = reqLevel;
        def.numberOfTiers = numTiers;
        def.slotsPerTier = slotsPerTier;
        def.shelfRole = role;
        def.description = description;

        AssetDatabase.CreateAsset(def, path);
        Debug.Log($"[FurnitureDataGenerator] Created: {path}");
    }
}
#endif
```

---

## 6. Thiết Lập Scene

Cursor phải cấu hình `GameScene.unity` với hierarchy sau:

```
GameScene (Scene Root)
│
├── _Bootstrapper              [SceneBootstrapper]       (Bước 1)
├── Main Camera                [Camera][CameraController] (Bước 1)
├── _InventoryManager          [InventoryManager]         (Bước 2)
│
├── Grid                       [Grid component]           ← MỚI
│   Inspector — Grid component:
│     Cell Layout: Isometric Z as Y
│     Cell Size:   (1, 0.5, 1)    ← Standard 2:1 isometric
│     Cell Gap:    (0, 0, 0)
│   Children:
│   └── Tilemap                [Tilemap][TilemapRenderer] ← Ground layer (trống, dùng sau)
│
├── _GridSystem                [GridSystem]               ← MỚI
│   Inspector:
│     Isometric Grid:  (kéo Grid GameObject vào đây)
│     Shop Min Cell:   (-10, -10)
│     Shop Max Cell:   (10, 10)
│     Verbose Logging: true
│
├── _PlacementManager          [PlacementManager]         ← MỚI
│   Inspector:
│     Ground Layer Mask: Default (hoặc tạo layer "Ground")
│     Main Camera:       (kéo Main Camera vào đây)
│     Placement Cooldown: 0.3
│     Ghost Parent:      (kéo _GhostContainer vào đây)
│     Furniture Parent:  (kéo _PlacedFurniture vào đây)
│
├── _GhostContainer            (Empty GameObject, parent cho ghost)
├── _PlacedFurniture           (Empty GameObject, parent cho placed furniture)
│
└── _GridVisualizer            [GridVisualizer]           ← Debug only
    Inspector:
      Show Grid Lines:     true
      Highlight Occupied:  true
```

---

## 7. Giới Hạn & Quy Tắc Bắt Buộc

| Quy tắc | Lý do |
|---------|-------|
| **CẤM** `GameObject.Find()` trong `Update()` | Quy tắc toàn dự án từ Bước 1 |
| **CẤM** `GetComponent<T>()` trong `Update()` | Phải cache trong `Awake()` |
| **CẤM** hard-code shop bounds | Phải lấy từ `GridSystem.shopMinCell/shopMaxCell` |
| **BẮT BUỘC** `ValidatePlacement()` trả về `failReason` string | Thông điệp lỗi phải chứa "Vị trí không hợp lệ, lưới bị chiếm dụng" |
| **BẮT BUỘC** `ConfirmPlacement()` gọi sau khi `Instantiate` | Grid state phải khớp với scene objects |
| **BẮT BUỘC** `ReleaseCells()` khi xóa furniture | Tránh grid "phantom occupation" |
| **BẮT BUỘC** Dùng `Mouse.current` từ New Input System | Không dùng `Input.GetMouseButton()` |
| **BẮT BUỘC** Material instances trong `GhostObject.Awake()` | Không set shared material — gây bug với mọi prefab |

---

## 8. Kịch Bản Kiểm Thử Đầy Đủ

### 8.1 Chuẩn Bị

```
Bước 1: Menu TCGShop > Setup > Generate Furniture Definitions
        → 5 assets xuất hiện trong Assets/ScriptableObjects/Furniture/

Bước 2: Tạo prefab đơn giản Prefab_ShelfSingle:
        - Sprite có SpriteRenderer
        - Thêm PlacedFurnitureInstance component
        - Kéo vào FurnitureDefinition.Furniture_ShelfSingle.furniturePrefab

Bước 3: Cấu hình Scene theo Mục 6

Bước 4: Tạo một script test tạm thời TestBuildMode.cs gắn vào scene:
        void Start() {
            var def = Resources.Load<FurnitureDefinition>("Furniture/Furniture_ShelfSingle");
            PlacementManager.Instance.StartPlacement(def);
        }
```

### 8.2 Kiểm Tra Placement Hợp Lệ

```
Test 1 — Ghost Preview:
  Nhấn Play
  Di chuột trong shop bounds
  KỲ VỌNG: Ghost sprite xuất hiện, màu XANH, di chuyển mượt theo chuột

Test 2 — Ghost Snap to Grid:
  Di chuột chậm theo đường thẳng
  KỲ VỌNG: Ghost "nhảy" từ cell này sang cell khác, không drift pixel

Test 3 — Đặt Furniture:
  Di chuột đến ô trống, ghost màu xanh
  Click chuột trái
  KỲ VỌNG:
    - Ghost biến mất
    - Prefab Instantiate tại đúng vị trí cell
    - Console: "[PlacementManager] ✅ Placed FurnitureInstance[...]"
    - Console: "[GridSystem] Placed [ShelfSingle] ID='...' at origin=(...)"
```

### 8.3 Kiểm Tra Vị Trí Không Hợp Lệ — LOG LỖI BẮT BUỘC

```
Test 4 — Đặt Đè Lên Furniture Cũ:
  Đặt ShelfSingle tại cell (0, 0) thành công
  Bắt đầu placement mode mới
  Di chuột về đúng cell (0, 0) — ghost phải chuyển màu ĐỎ
  Click chuột trái

  KỲ VỌNG Console:
    [PlacementManager] Vị trí không hợp lệ, lưới bị chiếm dụng tại (0, 0)
    bởi [ShelfSingle] ID='furniture_ShelfSingle_...'

  KỲ VỌNG Scene:
    KHÔNG có prefab mới được tạo
    PlacementManager vẫn ở PLACING state (không thoát)
    Ghost vẫn hiển thị màu đỏ

Test 5 — Đặt Ngoài Bounds:
  Di chuột ra ngoài shop area
  KỲ VỌNG: Ghost màu ĐỎ
  Click chuột trái
  KỲ VỌNG Console:
    [PlacementManager] Vị trí không hợp lệ: cell (...) nằm ngoài biên giới cửa hàng.
```

### 8.4 Kiểm Tra Rotation

```
Test 6 — Xoay ShelfDouble (2×1):
  StartPlacement(Furniture_ShelfDouble) — footprint 2×1
  Ghost hiển thị chiều ngang
  Nhấn R
  KỲ VỌNG:
    Ghost xoay 90° → chiều dọc (footprint 1×2)
    Console: "[PlacementManager] Rotated to 90°. New footprint: 2 cells."
  Nhấn R lần nữa
  KỲ VỌNG: Quay về 180° (tương đương 0° với footprint đối xứng)

Test 7 — Vật Thể Không Thể Xoay (ShelfSingle, canRotate=false):
  StartPlacement(Furniture_ShelfSingle)
  Nhấn R
  KỲ VỌNG Console:
    [PlacementManager] ShelfSingle không thể xoay (canRotate = false).
```

### 8.5 Kiểm Tra FootPrint Nhiều Cells

```
Test 8 — ShelfDouble chiếm 2 cells:
  Đặt ShelfDouble tại cell (0, 0), rotation 0°
  Footprint: [(0,0), (1,0)] → cả hai cell phải bị occupied

  Verify trong GridSystem.PrintGridState():
    Console: "[GridSystem] Grid State: ..., Occupied=2, ..."

  Thử đặt vật thể mới tại cell (1, 0):
  KỲ VỌNG Console:
    "[PlacementManager] Vị trí không hợp lệ, lưới bị chiếm dụng tại (1, 0)
    bởi [ShelfDouble] ID='...'"

Test 9 — ReleaseCells khi xóa:
  Lấy reference PlacedFurnitureInstance của ShelfDouble
  Gọi GridSystem.Instance.ReleaseCells(instance.InstanceId)
  KỲ VỌNG Console: "[GridSystem] Released 2 cells from furniture ID='...'"
  Thử đặt tại (0,0) lại → Ghost màu XANH, có thể đặt
```

### 8.6 Kiểm Tra Không Có Lỗi

```
Thực hiện tất cả test trên liên tục
Mở Console, filter chỉ Errors

KỲ VỌNG: 0 NullReferenceException, 0 MissingReferenceException

Test đặc biệt — CancelPlacement:
  StartPlacement, di chuột, nhấn ESC (hoặc chuột phải)
  KỲ VỌNG:
    Ghost biến mất ngay lập tức
    Console: "[PlacementManager] Placement cancelled."
    PlacementManager.CurrentState == Idle
```

---

## 9. Định Nghĩa "Hoàn Thành" (Definition of Done)

Bước 3 được coi là **HOÀN THÀNH** khi và chỉ khi:

- [ ] Tất cả 8 file `.cs` đã được tạo đúng thư mục
- [ ] Menu `TCGShop > Setup > Generate Furniture Definitions` tạo đủ 5 assets
- [ ] `GridSystem` khởi tạo thành công với log cell count
- [ ] Ghost preview xuất hiện màu **xanh** trên ô trống
- [ ] Ghost preview chuyển màu **đỏ** trên ô occupied hoặc ngoài bounds
- [ ] Ghost snap đúng vào cell — không drift pixel
- [ ] Click đặt hợp lệ → Prefab Instantiate đúng vị trí
- [ ] Click đặt vào ô đã có → Log chứa **"Vị trí không hợp lệ, lưới bị chiếm dụng"**
- [ ] Phím R xoay ghost 90° với furniture có `canRotate = true`
- [ ] Phím R log warning với furniture có `canRotate = false`
- [ ] Footprint 2×1 (ShelfDouble) đánh dấu đúng 2 cells trong GridSystem
- [ ] `ReleaseCells()` giải phóng đúng số cells
- [ ] ESC / chuột phải hủy placement, ghost biến mất, state về Idle
- [ ] **Không có** `NullReferenceException` nào trong toàn bộ test session
- [ ] `PlacementManager.Update()` không chứa `GameObject.Find()` hay `GetComponent()`

**Chỉ sau khi tất cả checkbox trên được check, mới chuyển sang Bước 4.**
```