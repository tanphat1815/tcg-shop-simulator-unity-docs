```markdown
# Step 4: Trí Tuệ Không Gian — Thuật Toán A* Cho Đám Đông Khách Hàng
## Cursor Instructions — TCG Shop Simulator (Unity Port)

**Phiên bản tài liệu:** 1.0  
**Giai đoạn:** AI & Navigation Layer  
**Yêu cầu tiên quyết:** Bước 1-3 hoàn thành. `GridSystem` đang hoạt động với `Dictionary<Vector2Int, GridNode>`. `PlacementManager` có thể đặt và giải phóng furniture. `GameManager` sẵn sàng.

---

## 1. Mục Tiêu Của Bước Này

Hệ thống cũ (Phaser/Vue) dùng `physics.moveTo()` — một lệnh "bay thẳng" đến đích, không né vật cản. Khi NPC bị kẹt, hệ thống dùng "Stuck Recovery" kiểm tra mỗi 500ms rồi kick lại velocity. Đây là giải pháp vá víu, không phải pathfinding thực sự.

**Ba vấn đề cốt lõi của hệ thống cũ:**
- NPC đi xuyên qua kệ hàng nếu physics collider không perfect
- Stuck Recovery tạo chuyển động giật cục, không tự nhiên
- Không có khái niệm "đường vòng" — NPC bị kẹt vĩnh viễn nếu bị bao vây

**Giải pháp:** A* Pathfinding đọc trực tiếp từ `GridSystem.Dictionary<Vector2Int, GridNode>` (đã xây ở Bước 3). Kệ hàng đã là `isOccupied = true` → tự động thành obstacle cho pathfinding. Không cần layer physics bổ sung.

Bước này xây dựng năm hệ thống:

1. **`MinHeap<T>`** — Cấu trúc dữ liệu Binary Min-Heap tối ưu Open List, O(log n) insert/extract
2. **`PathfindingCore`** — Thuật toán A* thuần, Manhattan heuristic, đọc GridSystem
3. **`PathfindingGrid`** — Adapter layer giữa GridSystem và PathfindingCore
4. **`CharacterMovement`** — Di chuyển tuần tự qua path, FlipX theo hướng, xử lý path stale
5. **`CustomerAIController`** — FSM khách hàng port từ NPCManager.ts cũ, dùng CharacterMovement

**Kết quả mong đợi:** NPC di chuyển vòng quanh kệ hàng mượt mà, khi kệ mới được đặt chặn đường NPC đang đi, NPC dừng 0.1s rồi tính lại đường vòng, Console in "Cập nhật lưới phát sinh".

---

## 2. Danh Sách Files Cần Tạo

```
Assets/
├── Scripts/
│   ├── Pathfinding/
│   │   ├── MinHeap.cs                  ← Binary Min-Heap cho Open List
│   │   ├── PathNode.cs                 ← Node data cho A* algorithm
│   │   ├── PathfindingGrid.cs          ← Adapter: GridSystem → Pathfinding nodes
│   │   └── PathfindingCore.cs          ← Thuật toán A* chính
│   ├── Character/
│   │   ├── CharacterMovement.cs        ← Di chuyển theo path, FlipX
│   │   └── CustomerAIController.cs     ← FSM khách hàng
│   └── Debug/
│       └── PathfindingDebugVisualizer.cs ← Vẽ path và nodes trong Scene view
```

---

## 3. Lý Thuyết Nền: A* Và Cấu Trúc Dữ Liệu

### 3.1 Tại Sao A* Thay Vì Physics.MoveTo

```
physics.moveTo (hệ thống cũ):
  Path: A → B (đường thẳng, bất kể vật cản)
  Khi có vật cản: stuck → kick velocity sau 500ms → stuck lại
  Độ phức tạp: O(1) nhưng sai về không gian

A* Pathfinding:
  Path: A → [C → D → E] → B (né vật cản, đường ngắn nhất)
  Khi có vật cản: tính path mới vòng quanh
  Độ phức tạp: O(n log n) với n = số node, nhưng đúng về không gian
```

### 3.2 Manhattan Distance Heuristic

Trong không gian lưới (grid), Manhattan Distance là heuristic chuẩn:

```
h(node) = |node.x - goal.x| + |node.y - goal.y|

Tại sao KHÔNG dùng Euclidean (√(dx² + dy²)):
  - Isometric grid chỉ di chuyển 4 hướng (không diagonal)
  - Manhattan không bao giờ overestimate → A* admissible
  - Euclidean overestimate trong 4-directional grid → path không tối ưu

Tại sao KHÔNG dùng Chebyshev (max(dx, dy)):
  - Chebyshev dành cho 8-directional movement
  - Cửa hàng isometric dùng 4-directional → Manhattan chính xác hơn
```

### 3.3 F = G + H

```
G(node): Chi phí thực tế từ START đến node này
         G(start) = 0
         G(neighbor) = G(current) + moveCost
         moveCost = 1 (di chuyển ngang/dọc)

H(node): Heuristic — ước tính chi phí từ node đến GOAL
         H(node) = |node.x - goal.x| + |node.y - goal.y|

F(node): Tổng chi phí ước tính
         F(node) = G(node) + H(node)
         Node có F thấp nhất được ưu tiên → Min-Heap

Ví dụ:
  Start=(0,0), Goal=(5,3)
  Node A=(2,1): G=3, H=|2-5|+|1-3|=5, F=8
  Node B=(3,2): G=5, H=|3-5|+|2-3|=3, F=8
  A* ưu tiên node nào dẫn đến goal nhanh nhất
```

### 3.4 Min-Heap vs List/SortedList

```
Open List cần hai thao tác chính:
  1. Thêm node mới: Insert
  2. Lấy node có F nhỏ nhất: ExtractMin

So sánh:
  List<PathNode> + sort mỗi frame:
    Insert: O(n)
    ExtractMin: O(n log n)
    → Với 1000 nodes: 1000 * sort(1000) = 10^6 operations/frame

  SortedList<float, PathNode>:
    Insert: O(log n)
    ExtractMin: O(log n)
    → Vẫn dùng balanced BST, overhead cao

  MinHeap<PathNode> (Binary Heap):
    Insert: O(log n)  ← Đẩy node mới lên đúng vị trí trong heap
    ExtractMin: O(log n)  ← Swap root với last, sift down
    → OPTIMAL: Với 1000 nodes: ~10 operations cho Insert và ExtractMin
```

### 3.5 Vấn Đề Path Stale và Cách Tránh Infinite Loop

```
Path Stale xảy ra khi:
  T=0: NPC đang đi path [A→B→C→D]
  T=1: Player đặt kệ tại node C → C.isWalkable = false
  T=2: NPC đến B, next node là C → C không thể đi
  → NPC bị kẹt

Giải pháp (tránh Infinite Loop):
  Khi NPC phát hiện next node không còn walkable:
    1. Dừng lại (velocity = 0) trong 0.1s (STALE_PAUSE_DURATION)
    2. Log: "Cập nhật lưới phát sinh"
    3. Yêu cầu recalculate path từ vị trí hiện tại
    4. Nếu path mới = null (không có đường): chuyển sang STUCK state
    5. Trong STUCK state: chờ MAX_STUCK_WAIT (5s) rồi abandon mục tiêu
    
  Anti-Infinite-Loop Guards:
    - pathRecalculationCount <= MAX_RECALCULATIONS (3)
    - Nếu vượt quá: abandon goal, chuyển về WANDER state
    - STALE_PAUSE_DURATION: đủ để Grid update propagate
    - Không bao giờ request path đến node isWalkable=false
```

---

## 4. Chi Tiết Kỹ Thuật Từng File

### 4.1 `PathNode.cs`

**Vị trí:** `Assets/Scripts/Pathfinding/PathNode.cs`

```csharp
// Assets/Scripts/Pathfinding/PathNode.cs

using UnityEngine;

/// <summary>
/// Dữ liệu của một node trong A* algorithm.
/// Dùng class (reference type) thay vì struct để:
///   - Cho phép lưu parent reference (linked list ngược về start)
///   - Tránh boxing/unboxing khi lưu trong MinHeap
///   - So sánh reference equality trong ClosedSet (HashSet)
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
///   Không có. Hệ thống cũ không có A* node representation.
///   NPCManager.ts dùng trực tiếp Phaser.Math.Distance.Between
///   và physics.moveTo — không có node-based pathfinding.
/// </summary>
public class PathNode : System.IComparable<PathNode>
{
    // =========================================================================
    // IDENTITY
    // =========================================================================

    /// <summary>
    /// Tọa độ cell trong GridSystem. Là key để tra cứu GridNode.
    /// Tương đương Vector2Int key trong Dictionary<Vector2Int, GridNode>.
    /// </summary>
    public Vector2Int CellCoord { get; }

    // =========================================================================
    // A* COSTS
    // =========================================================================

    /// <summary>
    /// G Cost: Chi phí thực tế từ START đến node này.
    /// G(start) = 0.
    /// G(neighbor) = G(current) + stepCost (thường = 10 để tránh float precision).
    /// </summary>
    public int GCost { get; set; }

    /// <summary>
    /// H Cost: Heuristic ước tính từ node này đến GOAL.
    /// Dùng Manhattan Distance: |dx| + |dy| nhân với BASE_COST.
    /// Không bao giờ được overestimate để A* admissible.
    /// </summary>
    public int HCost { get; set; }

    /// <summary>
    /// F Cost: Tổng G + H. Node có F nhỏ nhất được xử lý trước.
    /// Đây là key của Min-Heap.
    /// </summary>
    public int FCost => GCost + HCost;

    // =========================================================================
    // GRAPH STRUCTURE
    // =========================================================================

    /// <summary>
    /// Node cha trong path. Dùng để trace ngược từ GOAL về START.
    /// null nếu đây là start node.
    /// </summary>
    public PathNode Parent { get; set; }

    /// <summary>
    /// Index trong heap array. Dùng bởi MinHeap để update priority O(log n).
    /// -1 nếu node không trong heap.
    /// </summary>
    public int HeapIndex { get; set; } = -1;

    // =========================================================================
    // WALKABILITY
    // =========================================================================

    /// <summary>
    /// Node này có thể đi qua không.
    /// FALSE nếu GridNode.IsOccupied = true (có kệ hàng) hoặc ngoài bounds.
    /// Đọc từ GridSystem tại thời điểm PathfindingGrid.BuildGraph().
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   validatePlacement() check "Không được đè lên vật thể khác"
    ///   Nhưng ở đây dùng cho pathfinding thay vì placement.
    /// </summary>
    public bool IsWalkable { get; set; }

    // =========================================================================
    // CONSTRUCTOR
    // =========================================================================

    public PathNode(Vector2Int cellCoord, bool isWalkable)
    {
        CellCoord = cellCoord;
        IsWalkable = isWalkable;
        GCost = int.MaxValue; // Chưa visited → cost vô cùng
        HCost = 0;
        Parent = null;
        HeapIndex = -1;
    }

    // =========================================================================
    // COMPARISON — Dùng cho MinHeap
    // =========================================================================

    /// <summary>
    /// So sánh theo FCost để MinHeap sắp xếp đúng (nhỏ hơn = ưu tiên hơn).
    /// Tiebreak bằng HCost: nếu FCost bằng nhau, ưu tiên node gần goal hơn.
    /// </summary>
    public int CompareTo(PathNode other)
    {
        int compare = FCost.CompareTo(other.FCost);
        if (compare == 0)
            compare = HCost.CompareTo(other.HCost);
        return compare;
    }

    public override string ToString() =>
        $"PathNode[{CellCoord}|G={GCost}|H={HCost}|F={FCost}|Walk={IsWalkable}]";
}
```

---

### 4.2 `MinHeap.cs`

**Vị trí:** `Assets/Scripts/Pathfinding/MinHeap.cs`  
**Mục đích:** Binary Min-Heap tổng quát. Insert O(log n), ExtractMin O(log n). Thay thế `List<T>` + sort trong Open List của A*.

```csharp
// Assets/Scripts/Pathfinding/MinHeap.cs

using System;
using System.Collections.Generic;

/// <summary>
/// Binary Min-Heap tổng quát dùng cho A* Open List.
///
/// TẠI SAO MIN-HEAP:
///   A* cần liên tục lấy node có FCost nhỏ nhất từ Open List.
///   List + sort: O(n log n) mỗi lần extract → chậm với nhiều node.
///   MinHeap: Insert O(log n), ExtractMin O(log n) → tối ưu.
///
/// BINARY HEAP INVARIANT:
///   Với mọi node tại index i:
///     Parent(i) = (i-1)/2
///     LeftChild(i) = 2*i + 1
///     RightChild(i) = 2*i + 2
///   Luôn đảm bảo: heap[parent] <= heap[children]
///   → Phần tử nhỏ nhất luôn ở heap[0]
///
/// INTERFACE IHeapItem:
///   T phải implement IHeapItem để heap lưu HeapIndex.
///   HeapIndex cho phép UpdateItem() O(log n) thay vì O(n) search.
/// </summary>
public class MinHeap<T> where T : IComparable<T>, IHeapItem
{
    private readonly List<T> _items;

    // =========================================================================
    // CONSTRUCTORS
    // =========================================================================

    public MinHeap(int initialCapacity = 64)
    {
        _items = new List<T>(initialCapacity);
    }

    // =========================================================================
    // PROPERTIES
    // =========================================================================

    public int Count => _items.Count;
    public bool IsEmpty => _items.Count == 0;

    /// <summary>Phần tử nhỏ nhất (gốc heap) — O(1).</summary>
    public T Min => _items.Count > 0 ? _items[0] : default;

    // =========================================================================
    // CORE OPERATIONS
    // =========================================================================

    /// <summary>
    /// Thêm phần tử vào heap. O(log n).
    ///
    /// THUẬT TOÁN:
    ///   1. Thêm vào cuối array
    ///   2. SiftUp: so sánh với parent, swap nếu nhỏ hơn
    ///   3. Lặp đến khi heap invariant thỏa mãn
    /// </summary>
    public void Insert(T item)
    {
        _items.Add(item);
        int index = _items.Count - 1;
        item.HeapIndex = index;
        SiftUp(index);
    }

    /// <summary>
    /// Lấy và xóa phần tử nhỏ nhất (gốc). O(log n).
    ///
    /// THUẬT TOÁN:
    ///   1. Lưu root (phần tử min)
    ///   2. Đưa phần tử cuối lên root
    ///   3. Xóa phần tử cuối
    ///   4. SiftDown: so sánh root với children, swap với child nhỏ hơn
    ///   5. Lặp đến khi heap invariant thỏa mãn
    /// </summary>
    public T ExtractMin()
    {
        if (IsEmpty)
            throw new InvalidOperationException("MinHeap is empty.");

        T min = _items[0];
        int lastIndex = _items.Count - 1;

        // Đưa phần tử cuối lên root
        _items[0] = _items[lastIndex];
        _items[0].HeapIndex = 0;

        // Xóa phần tử cuối
        _items.RemoveAt(lastIndex);

        // Restore heap property
        if (!IsEmpty)
            SiftDown(0);

        min.HeapIndex = -1;
        return min;
    }

    /// <summary>
    /// Cập nhật priority của một item đã có trong heap. O(log n).
    /// Dùng khi tìm được G cost tốt hơn cho một node đã trong Open List.
    /// Nhờ HeapIndex, không cần tìm kiếm O(n) — chỉ cần SiftUp từ vị trí đã biết.
    /// </summary>
    public void UpdateItem(T item)
    {
        if (item.HeapIndex < 0 || item.HeapIndex >= _items.Count) return;
        SiftUp(item.HeapIndex);
    }

    /// <summary>Kiểm tra item có trong heap không — O(1) nhờ HeapIndex.</summary>
    public bool Contains(T item) =>
        item.HeapIndex >= 0 && item.HeapIndex < _items.Count && _items[item.HeapIndex].Equals(item);

    /// <summary>Xóa toàn bộ heap.</summary>
    public void Clear()
    {
        foreach (var item in _items)
            item.HeapIndex = -1;
        _items.Clear();
    }

    // =========================================================================
    // SIFT OPERATIONS — Duy trì heap invariant
    // =========================================================================

    /// <summary>
    /// SiftUp: Đẩy phần tử tại index lên đúng vị trí.
    /// Dùng sau Insert (phần tử mới ở cuối).
    ///
    /// THUẬT TOÁN:
    ///   Trong khi index > 0 VÀ item < parent:
    ///     Swap(item, parent)
    ///     index = parentIndex
    /// </summary>
    private void SiftUp(int index)
    {
        while (index > 0)
        {
            int parentIndex = (index - 1) / 2;

            // Nếu item >= parent → heap invariant thỏa mãn, dừng
            if (_items[index].CompareTo(_items[parentIndex]) >= 0)
                break;

            Swap(index, parentIndex);
            index = parentIndex;
        }
    }

    /// <summary>
    /// SiftDown: Đẩy phần tử tại index xuống đúng vị trí.
    /// Dùng sau ExtractMin (phần tử cuối được đưa lên root).
    ///
    /// THUẬT TOÁN:
    ///   Trong khi có ít nhất một child:
    ///     Tìm child nhỏ nhất
    ///     Nếu item <= child nhỏ nhất → dừng
    ///     Swap(item, child nhỏ nhất)
    ///     index = childIndex
    /// </summary>
    private void SiftDown(int index)
    {
        int count = _items.Count;

        while (true)
        {
            int leftChild = 2 * index + 1;
            int rightChild = 2 * index + 2;
            int smallest = index;

            // Tìm child nhỏ nhất
            if (leftChild < count && _items[leftChild].CompareTo(_items[smallest]) < 0)
                smallest = leftChild;

            if (rightChild < count && _items[rightChild].CompareTo(_items[smallest]) < 0)
                smallest = rightChild;

            // Nếu đã là nhỏ nhất trong nhóm → dừng
            if (smallest == index) break;

            Swap(index, smallest);
            index = smallest;
        }
    }

    private void Swap(int indexA, int indexB)
    {
        T temp = _items[indexA];
        _items[indexA] = _items[indexB];
        _items[indexB] = temp;

        // Cập nhật HeapIndex cho cả hai
        _items[indexA].HeapIndex = indexA;
        _items[indexB].HeapIndex = indexB;
    }
}

/// <summary>
/// Interface bắt buộc cho các item trong MinHeap.
/// HeapIndex cho phép UpdateItem O(log n) không cần tìm kiếm.
/// </summary>
public interface IHeapItem
{
    int HeapIndex { get; set; }
}
```

---

### 4.3 `PathfindingGrid.cs`

**Vị trí:** `Assets/Scripts/Pathfinding/PathfindingGrid.cs`  
**Mục đích:** Adapter layer. Đọc `GridSystem.Dictionary<Vector2Int, GridNode>` và chuyển đổi thành `Dictionary<Vector2Int, PathNode>` cho A* engine. Xử lý dynamic obstacle updates.

```csharp
// Assets/Scripts/Pathfinding/PathfindingGrid.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Adapter giữa GridSystem (game world) và PathfindingCore (A* engine).
///
/// TRÁCH NHIỆM:
///   1. Đọc GridNode.IsOccupied từ GridSystem → chuyển thành PathNode.IsWalkable
///   2. Cập nhật PathNode.IsWalkable khi furniture được đặt/dỡ (dynamic obstacles)
///   3. Cung cấp PathNode lookups O(1) cho PathfindingCore
///   4. Thông báo cho tất cả CharacterMovement khi grid thay đổi
///
/// THIẾT KẾ:
///   PathfindingGrid KHÔNG tự tạo PathNode mới mỗi lần A* chạy.
///   Thay vào đó, duy trì một bộ PathNode được reuse qua nhiều query.
///   Điều này tránh GC pressure từ việc tạo/hủy hàng nghìn objects/frame.
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
///   Không có. Hệ thống cũ dùng Phaser physics bodies làm obstacles.
///   PathfindingGrid thay thế hoàn toàn cơ chế collision detection cũ
///   bằng spatial index từ GridSystem (Bước 3).
/// </summary>
public class PathfindingGrid : MonoBehaviour
{
    // =========================================================================
    // SINGLETON
    // =========================================================================

    public static PathfindingGrid Instance { get; private set; }

    // =========================================================================
    // CONFIGURATION
    // =========================================================================

    [Header("Grid Settings")]
    [Tooltip("4 hướng di chuyển (không diagonal trong isometric).")]
    [SerializeField] private bool allowDiagonalMovement = false;

    [Header("Debug")]
    [SerializeField] private bool verboseLogging = true;

    // =========================================================================
    // PATHFINDING NODES
    // =========================================================================

    /// <summary>
    /// Tập hợp PathNode map với GridSystem cells.
    /// Key: Vector2Int cell coordinate (khớp với GridSystem._grid keys)
    /// Value: PathNode với IsWalkable đã được sync từ GridNode.IsOccupied
    ///
    /// Reuse nodes qua nhiều A* queries → tránh GC.
    /// Chỉ reset G/H/Parent khi bắt đầu query mới.
    /// </summary>
    private Dictionary<Vector2Int, PathNode> _pathNodes;

    /// <summary>
    /// Danh sách 4 hướng di chuyển hợp lệ.
    /// Không diagonal vì isometric shop layout dùng 4-directional.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   Phaser physics di chuyển 8 hướng nhưng NPC thực tế đi 4 hướng
    ///   (moveStates check vx và vy riêng).
    /// </summary>
    private static readonly Vector2Int[] FourDirections =
    {
        new Vector2Int(1, 0),   // East
        new Vector2Int(-1, 0),  // West
        new Vector2Int(0, 1),   // North
        new Vector2Int(0, -1),  // South
    };

    private static readonly Vector2Int[] EightDirections =
    {
        new Vector2Int(1, 0),   new Vector2Int(-1, 0),
        new Vector2Int(0, 1),   new Vector2Int(0, -1),
        new Vector2Int(1, 1),   new Vector2Int(-1, 1),
        new Vector2Int(1, -1),  new Vector2Int(-1, -1),
    };

    // =========================================================================
    // EVENTS — Thông báo CharacterMovement khi grid thay đổi
    // =========================================================================

    /// <summary>
    /// Kích hoạt khi một hoặc nhiều node thay đổi walkability.
    /// CharacterMovement subscribe để biết khi nào cần recalculate path.
    /// Tham số: danh sách cells bị thay đổi.
    /// </summary>
    public event System.Action<List<Vector2Int>> OnGridChanged;

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

        // Delay build đến Start để GridSystem có thời gian Initialize
        _pathNodes = new Dictionary<Vector2Int, PathNode>();
    }

    private void Start()
    {
        if (GridSystem.Instance == null)
        {
            Debug.LogError("[PathfindingGrid] GridSystem.Instance là null! " +
                           "Đảm bảo GridSystem tồn tại và đã Initialize.");
            enabled = false;
            return;
        }

        BuildFromGridSystem();

        // Subscribe vào PlacementManager events để cập nhật khi furniture thay đổi
        if (PlacementManager.Instance != null)
        {
            PlacementManager.Instance.OnFurniturePlaced += HandleFurniturePlaced;
            PlacementManager.Instance.OnPlacementCancelled += HandlePlacementCancelled;
        }

        Debug.Log($"[PathfindingGrid] Initialized: {_pathNodes.Count} path nodes.");
    }

    private void OnDestroy()
    {
        if (PlacementManager.Instance != null)
        {
            PlacementManager.Instance.OnFurniturePlaced -= HandleFurniturePlaced;
            PlacementManager.Instance.OnPlacementCancelled -= HandlePlacementCancelled;
        }
    }

    // =========================================================================
    // GRID BUILDING — Đọc từ GridSystem
    // =========================================================================

    /// <summary>
    /// Build PathNode graph từ GridSystem hiện tại.
    /// Gọi một lần tại Start, sau đó chỉ update incrementally.
    ///
    /// LUỒNG:
    ///   FOR EACH (coord, gridNode) IN GridSystem._grid:
    ///     pathNode.IsWalkable = gridNode.IsWithinShopBounds AND NOT gridNode.IsOccupied
    ///     pathNode.IsWalkable = false nếu gridNode.IsOccupied (có kệ hàng)
    ///
    /// Đây là điểm kết nối chính với Bước 3:
    ///   GridNode.IsOccupied = true (kệ hàng đặt xuống)
    ///   → PathNode.IsWalkable = false (A* né kệ hàng)
    /// </summary>
    public void BuildFromGridSystem()
    {
        _pathNodes.Clear();

        var gridSystem = GridSystem.Instance;

        // Duyệt toàn bộ cells trong GridSystem bounds
        for (int x = gridSystem.ShopMinCell.x; x <= gridSystem.ShopMaxCell.x; x++)
        {
            for (int y = gridSystem.ShopMinCell.y; y <= gridSystem.ShopMaxCell.y; y++)
            {
                var coord = new Vector2Int(x, y);
                GridNode gridNode = gridSystem.GetNode(coord);

                // QUAN TRỌNG: isOccupied = true → isWalkable = false
                // Kệ hàng trở thành obstacle tự động
                bool isWalkable = gridNode.IsWithinShopBounds && !gridNode.IsOccupied;

                _pathNodes[coord] = new PathNode(coord, isWalkable);
            }
        }
    }

    // =========================================================================
    // DYNAMIC OBSTACLE UPDATES — Khi furniture được đặt/dỡ
    // =========================================================================

    /// <summary>
    /// Cập nhật PathNodes khi furniture được đặt xuống (các cells trở thành obstacle).
    /// Gọi bởi event OnFurniturePlaced từ PlacementManager.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   Khi player đặt kệ → Phaser tạo physics static body → NPC tự né.
    ///   Ở đây: đặt kệ → các PathNode.IsWalkable = false → A* tự né.
    /// </summary>
    private void HandleFurniturePlaced(PlacedFurnitureInstance instance)
    {
        var changedCells = new List<Vector2Int>();

        // Lấy footprint từ definition
        var footprint = instance.Definition.GetFootprintCells(instance.PlacedRotation);

        foreach (var offset in footprint)
        {
            var absoluteCell = instance.OriginCell + offset;

            if (_pathNodes.TryGetValue(absoluteCell, out PathNode node))
            {
                node.IsWalkable = false;
                changedCells.Add(absoluteCell);
            }
        }

        if (changedCells.Count > 0)
        {
            // Thông báo tất cả CharacterMovement về grid change
            OnGridChanged?.Invoke(changedCells);

            if (verboseLogging)
            {
                Debug.Log($"[PathfindingGrid] Cập nhật lưới phát sinh: " +
                          $"{changedCells.Count} cells trở thành obstacles " +
                          $"do [{instance.Definition.furnitureType}].");
            }
        }
    }

    private void HandlePlacementCancelled()
    {
        // Không cần làm gì khi cancel — không có furniture được đặt
    }

    /// <summary>
    /// Cập nhật PathNodes khi furniture được dỡ (cells trở lại walkable).
    /// Gọi bởi GridSystem sau khi ReleaseCells() hoàn tất.
    /// </summary>
    public void NotifyFurnitureRemoved(List<Vector2Int> releasedCells)
    {
        var changedCells = new List<Vector2Int>();

        foreach (var cell in releasedCells)
        {
            if (_pathNodes.TryGetValue(cell, out PathNode node))
            {
                // Double-check với GridSystem (source of truth)
                GridNode gridNode = GridSystem.Instance.GetNode(cell);
                node.IsWalkable = gridNode.IsWithinShopBounds && !gridNode.IsOccupied;

                if (node.IsWalkable)
                    changedCells.Add(cell);
            }
        }

        if (changedCells.Count > 0)
        {
            OnGridChanged?.Invoke(changedCells);

            if (verboseLogging)
            {
                Debug.Log($"[PathfindingGrid] {changedCells.Count} cells " +
                          "đã được giải phóng và trở lại walkable.");
            }
        }
    }

    // =========================================================================
    // NODE ACCESS API
    // =========================================================================

    /// <summary>Lấy PathNode tại cell — O(1).</summary>
    public PathNode GetNode(Vector2Int cellCoord)
    {
        return _pathNodes.TryGetValue(cellCoord, out PathNode node) ? node : null;
    }

    /// <summary>Kiểm tra cell có walkable không — O(1).</summary>
    public bool IsWalkable(Vector2Int cellCoord)
    {
        var node = GetNode(cellCoord);
        return node != null && node.IsWalkable;
    }

    /// <summary>
    /// Lấy danh sách neighbors hợp lệ của một node.
    /// Chỉ trả về neighbors walkable và trong bounds.
    /// </summary>
    public List<PathNode> GetNeighbors(PathNode node)
    {
        var directions = allowDiagonalMovement ? EightDirections : FourDirections;
        var neighbors = new List<PathNode>(directions.Length);

        foreach (var dir in directions)
        {
            var neighborCoord = node.CellCoord + dir;
            var neighbor = GetNode(neighborCoord);

            if (neighbor != null && neighbor.IsWalkable)
                neighbors.Add(neighbor);
        }

        return neighbors;
    }

    /// <summary>Reset G/H/Parent của tất cả nodes trước mỗi A* query.</summary>
    public void ResetForNewQuery()
    {
        foreach (var node in _pathNodes.Values)
        {
            node.GCost = int.MaxValue;
            node.HCost = 0;
            node.Parent = null;
            node.HeapIndex = -1;
        }
    }
}
```

---

### 4.4 `PathfindingCore.cs`

**Vị trí:** `Assets/Scripts/Pathfinding/PathfindingCore.cs`  
**Mục đích:** Thuật toán A* hoàn chỉnh với Manhattan heuristic và MinHeap Open List.

```csharp
// Assets/Scripts/Pathfinding/PathfindingCore.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Thuật toán A* Pathfinding hoàn chỉnh.
///
/// ĐẶC ĐIỂM KỸ THUẬT:
///   - Heuristic: Manhattan Distance (|dx| + |dy|) — admissible cho 4-directional grid
///   - Open List: MinHeap<PathNode> — O(log n) insert/extract
///   - Closed Set: HashSet<PathNode> — O(1) membership check
///   - Node reuse: PathfindingGrid resets nodes trước mỗi query, tránh GC
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
///   Toàn bộ class này thay thế:
///     - physics.moveTo() trong NPCManager.ts
///     - handleStuckRecovery() trong NPCManager.ts
///     - scene.physics.moveTo(customer.sprite, ...) mỗi frame
///   Thay vì "fly straight + stuck recovery", ta tính đường đúng từ đầu.
///
/// THREAD SAFETY:
///   Class này là static utility — không có mutable state.
///   An toàn khi gọi từ nhiều CharacterMovement trong cùng frame.
///   (Nhưng PathfindingGrid.ResetForNewQuery() KHÔNG thread-safe — gọi sequential.)
/// </summary>
public static class PathfindingCore
{
    // =========================================================================
    // CONSTANTS
    // =========================================================================

    /// <summary>
    /// Chi phí di chuyển một bước ngang/dọc.
    /// Dùng integer 10 thay vì float 1.0 để:
    ///   - Tránh floating point precision errors khi so sánh FCost
    ///   - Diagonal có thể dùng 14 (≈ 10√2) nếu enable diagonal movement
    /// </summary>
    private const int STRAIGHT_COST = 10;
    private const int DIAGONAL_COST = 14; // 10 * √2 ≈ 14.14 → làm tròn xuống 14

    // =========================================================================
    // A* MAIN ALGORITHM
    // =========================================================================

    /// <summary>
    /// Tìm đường ngắn nhất từ startCell đến goalCell.
    ///
    /// THUẬT TOÁN A* CHI TIẾT:
    ///
    ///   BƯỚC 1: KHỞI TẠO
    ///     openList = MinHeap với startNode (G=0, H=Manhattan(start→goal))
    ///     closedSet = HashSet rỗng
    ///
    ///   BƯỚC 2: VÒNG LẶP CHÍNH
    ///     WHILE openList không rỗng:
    ///       current = openList.ExtractMin()  ← O(log n)
    ///       closedSet.Add(current)
    ///
    ///       IF current == goalNode:
    ///         RETURN TracePath(current)  ← Tìm thấy đường!
    ///
    ///       FOR EACH neighbor OF current:
    ///         IF neighbor IN closedSet: SKIP
    ///         IF NOT neighbor.IsWalkable: SKIP
    ///
    ///         newG = current.G + MoveCost(current, neighbor)
    ///
    ///         IF newG < neighbor.G:  ← Tìm được đường tốt hơn
    ///           neighbor.G = newG
    ///           neighbor.H = Manhattan(neighbor → goal)
    ///           neighbor.Parent = current
    ///
    ///           IF neighbor IN openList:
    ///             openList.UpdateItem(neighbor)  ← O(log n) nhờ HeapIndex
    ///           ELSE:
    ///             openList.Insert(neighbor)      ← O(log n)
    ///
    ///   BƯỚC 3: KHÔNG TÌM THẤY
    ///     RETURN null
    ///
    /// ĐỘ PHỨC TẠP:
    ///   Time:  O(n log n) với n = số nodes được thăm
    ///   Space: O(n) cho openList và closedSet
    ///
    /// </summary>
    /// <param name="startCell">Cell xuất phát (vị trí NPC hiện tại).</param>
    /// <param name="goalCell">Cell đích (vị trí mục tiêu).</param>
    /// <param name="grid">PathfindingGrid đã build.</param>
    /// <returns>
    ///   List<Vector2Int>: Danh sách cells từ start đến goal (không bao gồm start).
    ///   null: Không có đường đi.
    /// </returns>
    public static List<Vector2Int> FindPath(
        Vector2Int startCell,
        Vector2Int goalCell,
        PathfindingGrid grid)
    {
        // --- Validate inputs ---
        PathNode startNode = grid.GetNode(startCell);
        PathNode goalNode = grid.GetNode(goalCell);

        if (startNode == null)
        {
            Debug.LogWarning($"[PathfindingCore] Start cell {startCell} không tồn tại trong grid.");
            return null;
        }

        if (goalNode == null)
        {
            Debug.LogWarning($"[PathfindingCore] Goal cell {goalCell} không tồn tại trong grid.");
            return null;
        }

        if (!goalNode.IsWalkable)
        {
            Debug.LogWarning($"[PathfindingCore] Goal cell {goalCell} không walkable " +
                             "(bị kệ hàng chiếm). Không thể tìm đường đến đây.");
            return null;
        }

        // --- Reset all nodes for new query ---
        grid.ResetForNewQuery();

        // --- BƯỚC 1: Khởi tạo ---
        var openList = new MinHeap<PathNode>(256);
        var closedSet = new HashSet<PathNode>();

        startNode.GCost = 0;
        startNode.HCost = CalculateManhattanDistance(startCell, goalCell);
        openList.Insert(startNode);

        // --- BƯỚC 2: Vòng lặp chính ---
        while (!openList.IsEmpty)
        {
            // ExtractMin: O(log n) — Lấy node có FCost nhỏ nhất
            PathNode current = openList.ExtractMin();
            closedSet.Add(current);

            // Kiểm tra đã đến đích chưa
            if (current.CellCoord == goalCell)
            {
                return TracePath(current, startCell);
            }

            // Duyệt các neighbors
            List<PathNode> neighbors = grid.GetNeighbors(current);

            foreach (PathNode neighbor in neighbors)
            {
                // Skip nếu đã trong closed set
                if (closedSet.Contains(neighbor)) continue;

                // Tính G cost mới
                int newGCost = current.GCost + GetMoveCost(current, neighbor);

                // Nếu tìm được đường tốt hơn đến neighbor
                if (newGCost < neighbor.GCost)
                {
                    neighbor.GCost = newGCost;
                    neighbor.HCost = CalculateManhattanDistance(neighbor.CellCoord, goalCell);
                    neighbor.Parent = current;

                    if (openList.Contains(neighbor))
                    {
                        // Update priority trong heap — O(log n) nhờ HeapIndex
                        openList.UpdateItem(neighbor);
                    }
                    else
                    {
                        // Thêm vào open list lần đầu — O(log n)
                        openList.Insert(neighbor);
                    }
                }
            }
        }

        // --- BƯỚC 3: Không tìm thấy đường ---
        return null;
    }

    // =========================================================================
    // HEURISTIC: MANHATTAN DISTANCE
    // =========================================================================

    /// <summary>
    /// Tính Manhattan Distance giữa hai cells.
    ///
    /// CÔNG THỨC: |ax - bx| + |ay - by|
    ///
    /// TẠI SAO MANHATTAN:
    ///   - Grid 4-directional: chỉ đi ngang/dọc, không chéo
    ///   - Manhattan là lower bound chính xác → admissible heuristic
    ///   - Admissible → A* đảm bảo tìm đường ngắn nhất
    ///   - Euclidean: tính √, tốn kém hơn, overestimate trong 4-directional
    ///   - Chebyshev: dành cho 8-directional (cho phép diagonal)
    ///
    /// Nhân với STRAIGHT_COST (10) để khớp với GCost scale.
    /// </summary>
    private static int CalculateManhattanDistance(Vector2Int a, Vector2Int b)
    {
        return (Mathf.Abs(a.x - b.x) + Mathf.Abs(a.y - b.y)) * STRAIGHT_COST;
    }

    // =========================================================================
    // MOVE COST
    // =========================================================================

    /// <summary>
    /// Chi phí di chuyển từ current sang neighbor.
    /// Straight (ngang/dọc) = 10, Diagonal = 14.
    /// Trong 4-directional, luôn là STRAIGHT_COST.
    /// </summary>
    private static int GetMoveCost(PathNode current, PathNode neighbor)
    {
        int dx = Mathf.Abs(current.CellCoord.x - neighbor.CellCoord.x);
        int dy = Mathf.Abs(current.CellCoord.y - neighbor.CellCoord.y);

        // Diagonal nếu cả dx và dy đều != 0
        bool isDiagonal = dx != 0 && dy != 0;
        return isDiagonal ? DIAGONAL_COST : STRAIGHT_COST;
    }

    // =========================================================================
    // PATH RECONSTRUCTION
    // =========================================================================

    /// <summary>
    /// Trace ngược từ goal về start để lấy path.
    ///
    /// THUẬT TOÁN:
    ///   path = []
    ///   current = goalNode
    ///   WHILE current.Parent != null:
    ///     path.Add(current.CellCoord)
    ///     current = current.Parent
    ///   path.Reverse()  ← Đảo ngược để path đi từ start → goal
    ///   RETURN path (không bao gồm start cell)
    ///
    /// Kết quả: List<Vector2Int> từ cell ngay sau start đến goal.
    /// CharacterMovement sẽ di chuyển tuần tự qua từng cell.
    /// </summary>
    private static List<Vector2Int> TracePath(PathNode goalNode, Vector2Int startCell)
    {
        var path = new List<Vector2Int>();
        PathNode current = goalNode;

        // Trace ngược từ goal
        while (current.Parent != null)
        {
            path.Add(current.CellCoord);
            current = current.Parent;
        }

        // Đảo ngược: path hiện tại là goal→start, cần start→goal
        path.Reverse();

        return path;
    }

    // =========================================================================
    // UTILITY
    // =========================================================================

    /// <summary>
    /// Ước tính có đường đi không mà không cần tìm full path.
    /// Dùng flood fill đơn giản — nhanh hơn A* cho connectivity check.
    /// </summary>
    public static bool HasPath(Vector2Int startCell, Vector2Int goalCell, PathfindingGrid grid)
    {
        return FindPath(startCell, goalCell, grid) != null;
    }
}
```

---

### 4.5 `CharacterMovement.cs`

**Vị trí:** `Assets/Scripts/Character/CharacterMovement.cs`  
**Mục đích:** Di chuyển Character theo path từ A*, xử lý FlipX, phát hiện path stale và recalculate.

```csharp
// Assets/Scripts/Character/CharacterMovement.cs

using System.Collections;
using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Quản lý di chuyển của một Character (NPC hoặc Player) theo A* path.
///
/// LUỒNG DI CHUYỂN:
///   1. RequestPath(goalCell) → PathfindingCore.FindPath() → lưu _currentPath
///   2. Mỗi frame: MoveAlongPath()
///      - Lấy waypoint hiện tại (cell đầu tiên trong path)
///      - Vector3.MoveTowards đến world position của waypoint
///      - Khi đến waypoint: pop khỏi path, lấy waypoint tiếp theo
///      - FlipX sprite theo hướng di chuyển ngang
///   3. Khi grid thay đổi: StartCoroutine(HandlePathStale())
///      - Dừng 0.1s (STALE_PAUSE_DURATION)
///      - Log "Cập nhật lưới phát sinh"
///      - Recalculate path
///
/// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ (NPCManager.ts):
///   physics.moveTo(sprite, targetX, targetY, speed)  → RequestPath() + MoveAlongPath()
///   handleStuckRecovery()                            → HandlePathStale()
///   sprite.anims.play('npc-left')                   → UpdateSpriteDirection() + FlipX
///   customer.state = 'SEEK_ITEM'                    → CustomerAIController states
/// </summary>
[RequireComponent(typeof(SpriteRenderer))]
public class CharacterMovement : MonoBehaviour
{
    // =========================================================================
    // CONSTANTS
    // =========================================================================

    /// <summary>
    /// Thời gian dừng khi phát hiện path stale (giây).
    /// 0.1s đủ để grid update propagate và tránh rapid recalculation.
    /// YÊU CẦU: Phải là 0.1s theo spec của Bước 4.
    /// </summary>
    private const float STALE_PAUSE_DURATION = 0.1f;

    /// <summary>
    /// Số lần recalculate tối đa trước khi abandon goal.
    /// Tránh Infinite Loop khi không có đường nào khả thi.
    /// </summary>
    private const int MAX_RECALCULATIONS = 3;

    /// <summary>
    /// Thời gian chờ tối đa khi stuck trước khi abandon goal.
    /// </summary>
    private const float MAX_STUCK_WAIT = 5f;

    /// <summary>
    /// Khoảng cách ngưỡng để coi là "đã đến" waypoint.
    /// Nhỏ hơn → chính xác hơn nhưng có thể overshoot.
    /// </summary>
    private const float WAYPOINT_REACH_THRESHOLD = 0.05f;

    // =========================================================================
    // CONFIGURATION
    // =========================================================================

    [Header("Movement")]
    [Tooltip("Tốc độ di chuyển (world units/giây). " +
             "Tương đương npcSpeed = 100 trong NPCManager.ts cũ.")]
    [SerializeField] private float moveSpeed = 3f;

    [Tooltip("Tốc độ xoay nhân vật về hướng di chuyển. " +
             "Cao = xoay ngay lập tức, thấp = xoay mượt.")]
    [SerializeField][Range(1f, 20f)] private float rotationSpeed = 10f;

    [Header("Sprite Direction")]
    [Tooltip("True nếu sprite mặc định nhìn sang phải (hướng +X). " +
             "False nếu sprite mặc định nhìn sang trái.")]
    [SerializeField] private bool defaultFacingRight = true;

    [Header("Debug")]
    [SerializeField] private bool showPathGizmos = true;
    [SerializeField] private bool verboseLogging = false;

    // =========================================================================
    // COMPONENT REFERENCES — Cache trong Awake, không GetComponent trong Update
    // =========================================================================

    private SpriteRenderer _spriteRenderer;

    // =========================================================================
    // MOVEMENT STATE
    // =========================================================================

    /// <summary>
    /// Path hiện tại: danh sách cells còn lại cần đi qua.
    /// Index 0 = waypoint kế tiếp cần đến.
    /// Pop từ đầu khi đến waypoint (dùng Queue cho O(1) dequeue).
    /// </summary>
    private Queue<Vector2Int> _currentPath;

    /// <summary>World position của waypoint đang hướng đến.</summary>
    private Vector3 _currentWaypointWorldPos;

    /// <summary>Cell của waypoint đang hướng đến.</summary>
    private Vector2Int _currentWaypointCell;

    /// <summary>Cell đích cuối cùng.</summary>
    private Vector2Int _goalCell;

    /// <summary>Đang di chuyển không.</summary>
    public bool IsMoving { get; private set; }

    /// <summary>Đã đến đích chưa.</summary>
    public bool HasReachedGoal { get; private set; }

    // =========================================================================
    // PATH STALE STATE — Xử lý dynamic obstacles
    // =========================================================================

    /// <summary>Coroutine xử lý path stale (chỉ một instance chạy cùng lúc).</summary>
    private Coroutine _staleHandlerCoroutine;

    /// <summary>Đang trong quá trình xử lý stale path không.</summary>
    private bool _isHandlingStale;

    /// <summary>Số lần đã recalculate path cho mục tiêu hiện tại.</summary>
    private int _recalculationCount;

    /// <summary>Thời điểm bắt đầu stuck (để timeout).</summary>
    private float _stuckStartTime;

    // =========================================================================
    // DIRECTION — Cho FlipX và animation
    // =========================================================================

    private Vector2 _movementDirection = Vector2.right;
    private bool _isFacingRight;

    // =========================================================================
    // EVENTS
    // =========================================================================

    /// <summary>Kích hoạt khi đến đích.</summary>
    public event System.Action OnReachedGoal;

    /// <summary>Kích hoạt khi không tìm được đường (goal unreachable).</summary>
    public event System.Action OnPathNotFound;

    /// <summary>Kích hoạt khi abandon goal do quá nhiều recalculations.</summary>
    public event System.Action OnGoalAbandoned;

    // =========================================================================
    // VÒNG ĐỜI
    // =========================================================================

    private void Awake()
    {
        // Cache — KHÔNG GetComponent trong Update
        _spriteRenderer = GetComponent<SpriteRenderer>();
        _isFacingRight = defaultFacingRight;
        _currentPath = new Queue<Vector2Int>();
    }

    private void OnEnable()
    {
        // Subscribe vào PathfindingGrid events khi character active
        if (PathfindingGrid.Instance != null)
            PathfindingGrid.Instance.OnGridChanged += HandleGridChanged;
    }

    private void OnDisable()
    {
        // Unsubscribe khi disable (tránh memory leak)
        if (PathfindingGrid.Instance != null)
            PathfindingGrid.Instance.OnGridChanged -= HandleGridChanged;
    }

    private void Update()
    {
        // KHÔNG gọi GetComponent, GameObject.Find, hay FindObjectOfType ở đây
        if (!IsMoving || _isHandlingStale) return;

        MoveAlongPath();
    }

    // =========================================================================
    // PUBLIC API — Gọi từ CustomerAIController
    // =========================================================================

    /// <summary>
    /// Yêu cầu di chuyển đến goalCell.
    /// Tự động tính A* path và bắt đầu di chuyển.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   customer.targetX = shelf.x
    ///   customer.targetY = shelf.y + 45
    ///   scene.physics.moveTo(customer.sprite, targetX, targetY, npcSpeed)
    /// </summary>
    public void RequestPath(Vector2Int goalCell)
    {
        if (GridSystem.Instance == null || PathfindingGrid.Instance == null) return;

        _goalCell = goalCell;
        _recalculationCount = 0;
        HasReachedGoal = false;

        CalculateAndStartPath();
    }

    /// <summary>Dừng di chuyển ngay lập tức và xóa path.</summary>
    public void StopMovement()
    {
        IsMoving = false;
        _currentPath.Clear();

        if (_staleHandlerCoroutine != null)
        {
            StopCoroutine(_staleHandlerCoroutine);
            _staleHandlerCoroutine = null;
            _isHandlingStale = false;
        }
    }

    /// <summary>Cell hiện tại của character trên grid.</summary>
    public Vector2Int CurrentCell =>
        GridSystem.Instance != null
            ? GridSystem.Instance.WorldToCell(transform.position)
            : Vector2Int.zero;

    // =========================================================================
    // PATH CALCULATION
    // =========================================================================

    /// <summary>
    /// Tính A* path từ vị trí hiện tại đến _goalCell và bắt đầu di chuyển.
    /// </summary>
    private void CalculateAndStartPath()
    {
        if (PathfindingGrid.Instance == null || GridSystem.Instance == null) return;

        Vector2Int startCell = CurrentCell;

        // Nếu đã đứng tại goal
        if (startCell == _goalCell)
        {
            HandleGoalReached();
            return;
        }

        // Tìm đường A*
        List<Vector2Int> path = PathfindingCore.FindPath(
            startCell,
            _goalCell,
            PathfindingGrid.Instance
        );

        if (path == null || path.Count == 0)
        {
            Debug.LogWarning($"[CharacterMovement] {gameObject.name}: " +
                             $"Không tìm được đường từ {startCell} đến {_goalCell}.");
            OnPathNotFound?.Invoke();
            IsMoving = false;
            return;
        }

        // Chuyển path thành Queue
        _currentPath.Clear();
        foreach (var cell in path)
            _currentPath.Enqueue(cell);

        // Lấy waypoint đầu tiên
        AdvanceToNextWaypoint();
        IsMoving = true;

        if (verboseLogging)
        {
            Debug.Log($"[CharacterMovement] {gameObject.name}: " +
                      $"Path tìm được {path.Count} bước: {startCell} → {_goalCell}.");
        }
    }

    // =========================================================================
    // DI CHUYỂN THEO PATH — Gọi mỗi frame khi IsMoving
    // =========================================================================

    /// <summary>
    /// Di chuyển từng bước theo path sử dụng Vector3.MoveTowards.
    ///
    /// VECTOR3.MOVETOWARDS vs LERP:
    ///   MoveTowards: Di chuyển tối đa maxDistanceDelta mỗi frame, không vượt quá target.
    ///   Lerp: Di chuyển tỷ lệ phần trăm → chậm lại khi gần đến, không đảm bảo đến đích.
    ///   → MoveTowards phù hợp hơn cho grid movement (đảm bảo snap đúng vào waypoint).
    ///
    /// KIỂM TRA PATH STALE REAL-TIME:
    ///   Trước khi di chuyển đến waypoint kế tiếp, kiểm tra waypoint đó có còn walkable không.
    ///   Nếu không → trigger HandlePathStale ngay lập tức.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   handleSeekItem(), handleGoCashier(), handleSeekTable() trong NPCManager.ts
    ///   Kiểm tra distance < threshold rồi chuyển state.
    /// </summary>
    private void MoveAlongPath()
    {
        if (_currentPath.Count == 0 && !IsMoving) return;

        // === KIỂM TRA WAYPOINT HIỆN TẠI CÒN HỢP LỆ KHÔNG ===
        // Đây là core logic phòng chống path stale
        if (!PathfindingGrid.Instance.IsWalkable(_currentWaypointCell))
        {
            // Waypoint bị chặn (kệ mới đặt vào) → cần recalculate
            TriggerPathStale();
            return;
        }

        // === DI CHUYỂN VỀ WAYPOINT ===
        float step = moveSpeed * Time.deltaTime;
        Vector3 newPosition = Vector3.MoveTowards(
            transform.position,
            _currentWaypointWorldPos,
            step
        );

        // Tính hướng di chuyển để FlipX
        Vector3 moveDir = newPosition - transform.position;
        if (moveDir.sqrMagnitude > 0.0001f)
        {
            _movementDirection = new Vector2(moveDir.x, moveDir.y).normalized;
            UpdateSpriteDirection(_movementDirection.x);
        }

        transform.position = newPosition;

        // === KIỂM TRA ĐÃ ĐẾN WAYPOINT CHƯA ===
        float distToWaypoint = Vector3.Distance(transform.position, _currentWaypointWorldPos);

        if (distToWaypoint <= WAYPOINT_REACH_THRESHOLD)
        {
            // Snap chính xác vào waypoint để tránh float drift
            transform.position = _currentWaypointWorldPos;

            if (_currentPath.Count > 0)
            {
                // Còn waypoint tiếp theo → tiếp tục đi
                AdvanceToNextWaypoint();
            }
            else
            {
                // Đã đi qua tất cả waypoints → đến đích
                HandleGoalReached();
            }
        }
    }

    /// <summary>Lấy waypoint tiếp theo từ queue.</summary>
    private void AdvanceToNextWaypoint()
    {
        if (_currentPath.Count == 0) return;

        _currentWaypointCell = _currentPath.Dequeue();
        _currentWaypointWorldPos = GridSystem.Instance.CellToWorld(_currentWaypointCell);
    }

    private void HandleGoalReached()
    {
        IsMoving = false;
        HasReachedGoal = true;
        _currentPath.Clear();
        _recalculationCount = 0;

        if (verboseLogging)
            Debug.Log($"[CharacterMovement] {gameObject.name}: Đã đến đích {_goalCell}.");

        OnReachedGoal?.Invoke();
    }

    // =========================================================================
    // SPRITE DIRECTION — FlipX theo hướng di chuyển
    // =========================================================================

    /// <summary>
    /// FlipX sprite dựa trên hướng di chuyển ngang.
    ///
    /// LOGIC:
    ///   Nếu đang đi phải (horizontalDirection > 0) VÀ sprite mặc định nhìn phải:
    ///     → Không flip (isFacingRight = true)
    ///   Nếu đang đi trái (horizontalDirection < 0) VÀ sprite mặc định nhìn phải:
    ///     → Flip X (isFacingRight = false)
    ///   Nếu horizontalDirection ≈ 0 (đang đi lên/xuống):
    ///     → Giữ nguyên flip hiện tại
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   sprite.anims.play(vx < 0 ? 'npc-left' : 'npc-right', true)
    ///   Phaser dùng animation direction, Unity dùng SpriteRenderer.flipX.
    /// </summary>
    private void UpdateSpriteDirection(float horizontalDirection)
    {
        if (Mathf.Abs(horizontalDirection) < 0.1f) return; // Không đổi hướng khi đi dọc

        bool shouldFaceRight = horizontalDirection > 0;

        if (shouldFaceRight != _isFacingRight)
        {
            _isFacingRight = shouldFaceRight;
            // FlipX: true = nhìn trái (flip), false = nhìn phải (gốc)
            _spriteRenderer.flipX = defaultFacingRight ? !_isFacingRight : _isFacingRight;
        }
    }

    // =========================================================================
    // PATH STALE HANDLER — Xử lý khi grid thay đổi giữa chừng
    // =========================================================================

    /// <summary>
    /// Nhận thông báo từ PathfindingGrid khi grid thay đổi.
    /// Kiểm tra xem có ảnh hưởng đến path hiện tại không.
    ///
    /// TƯƠNG ĐƯƠNG HỆ THỐNG CŨ:
    ///   handleStuckRecovery() chạy mỗi 500ms.
    ///   Ở đây: event-driven, chỉ chạy khi thực sự có grid change.
    /// </summary>
    private void HandleGridChanged(List<Vector2Int> changedCells)
    {
        if (!IsMoving || _isHandlingStale) return;

        // Kiểm tra xem changed cells có overlap với path hiện tại không
        bool pathAffected = false;

        // Kiểm tra waypoint hiện tại
        foreach (var cell in changedCells)
        {
            if (cell == _currentWaypointCell)
            {
                pathAffected = true;
                break;
            }
        }

        // Nếu chưa bị ảnh hưởng tại waypoint hiện tại,
        // kiểm tra xem trong _currentPath queue có cell bị ảnh hưởng không
        if (!pathAffected)
        {
            // Convert Queue to check (không thể iterate Queue trực tiếp hiệu quả)
            // Chấp nhận O(n) ở đây vì path ngắn (< 50 steps trong shop)
            foreach (var pathCell in _currentPath)
            {
                foreach (var changedCell in changedCells)
                {
                    if (pathCell == changedCell)
                    {
                        pathAffected = true;
                        break;
                    }
                }
                if (pathAffected) break;
            }
        }

        if (pathAffected)
        {
            TriggerPathStale();
        }
    }

    /// <summary>
    /// Kích hoạt xử lý path stale.
    /// Chỉ một instance coroutine chạy cùng lúc.
    /// </summary>
    private void TriggerPathStale()
    {
        if (_isHandlingStale) return;

        if (_staleHandlerCoroutine != null)
            StopCoroutine(_staleHandlerCoroutine);

        _staleHandlerCoroutine = StartCoroutine(HandlePathStaleCoroutine());
    }

    /// <summary>
    /// Coroutine xử lý path stale theo đúng spec:
    ///   1. Dừng di chuyển 0.1s
    ///   2. Log "Cập nhật lưới phát sinh"
    ///   3. Recalculate path
    ///   4. Anti-Infinite-Loop guard
    ///
    /// TRÁNH INFINITE LOOP:
    ///   - Đếm số lần recalculate (_recalculationCount)
    ///   - Nếu vượt MAX_RECALCULATIONS → abandon goal
    ///   - Nếu recalculate thành công → reset counter
    ///   - Timeout MAX_STUCK_WAIT → abandon goal
    /// </summary>
    private IEnumerator HandlePathStaleCoroutine()
    {
        _isHandlingStale = true;
        IsMoving = false;
        _stuckStartTime = Time.time;

        // BƯỚC 1: Dừng 0.1s — YÊU CẦU THEO SPEC
        yield return new WaitForSeconds(STALE_PAUSE_DURATION);

        // BƯỚC 2: Log — YÊU CẦU THEO SPEC
        // Thông điệp chính xác: "Cập nhật lưới phát sinh"
        Debug.Log($"[CharacterMovement] Cập nhật lưới phát sinh — " +
                  $"{gameObject.name} đang recalculate path đến {_goalCell}. " +
                  $"Lần #{_recalculationCount + 1}/{MAX_RECALCULATIONS}.");

        _recalculationCount++;

        // BƯỚC 3: Anti-Infinite-Loop Guard
        if (_recalculationCount > MAX_RECALCULATIONS)
        {
            Debug.LogWarning($"[CharacterMovement] {gameObject.name}: " +
                             $"Đã recalculate {MAX_RECALCULATIONS} lần, abandon goal {_goalCell}. " +
                             "Có thể không có đường đi khả thi.");
            _isHandlingStale = false;
            IsMoving = false;
            OnGoalAbandoned?.Invoke();
            yield break;
        }

        // Timeout check
        if (Time.time - _stuckStartTime > MAX_STUCK_WAIT)
        {
            Debug.LogWarning($"[CharacterMovement] {gameObject.name}: " +
                             $"Stuck timeout ({MAX_STUCK_WAIT}s). Abandon goal {_goalCell}.");
            _isHandlingStale = false;
            IsMoving = false;
            OnGoalAbandoned?.Invoke();
            yield break;
        }

        // BƯỚC 4: Recalculate path
        _isHandlingStale = false;
        _currentPath.Clear();
        CalculateAndStartPath();

        if (!IsMoving)
        {
            // Recalculate thất bại (không có đường)
            Debug.LogWarning($"[CharacterMovement] {gameObject.name}: " +
                             "Recalculate thất bại — không tìm được đường sau grid change.");
            OnPathNotFound?.Invoke();
        }
        else if (verboseLogging)
        {
            Debug.Log($"[CharacterMovement] {gameObject.name}: " +
                      "Recalculate thành công — tiếp tục di chuyển theo đường mới.");
        }

        _staleHandlerCoroutine = null;
    }

    // =========================================================================
    // DEBUG GIZMOS
    // =========================================================================

#if UNITY_EDITOR
    private void OnDrawGizmos()
    {
        if (!showPathGizmos || !IsMoving) return;

        // Vẽ waypoint hiện tại
        Gizmos.color = Color.yellow;
        Gizmos.DrawSphere(_currentWaypointWorldPos, 0.15f);

        // Vẽ path còn lại
        if (GridSystem.Instance == null) return;

        Vector3 prevPos = transform.position;
        Gizmos.color = Color.cyan;

        foreach (var cell in _currentPath)
        {
            Vector3 cellWorld = GridSystem.Instance.CellToWorld(cell);
            Gizmos.DrawLine(prevPos, cellWorld);
            Gizmos.DrawSphere(cellWorld, 0.1f);
            prevPos = cellWorld;
        }
    }
#endif
}
```

---

### 4.6 `CustomerAIController.cs`

**Vị trí:** `Assets/Scripts/Character/CustomerAIController.cs`  
**Mục đích:** FSM khách hàng port từ `NPCManager.ts` cũ, dùng `CharacterMovement` thay vì `physics.moveTo`.

```csharp
// Assets/Scripts/Character/CustomerAIController.cs

using UnityEngine;

/// <summary>
/// Finite State Machine điều khiển hành vi khách hàng.
///
/// PORT TỪ HỆ THỐNG CŨ (NPCManager.ts):
///   NPCState enum:
///     'SPAWN'      → CustomerState.Spawning
///     'WANDER'     → CustomerState.Wandering
///     'SEEK_ITEM'  → CustomerState.SeekingItem
///     'INTERACT'   → CustomerState.Interacting
///     'GO_CASHIER' → CustomerState.GoingToCashier
///     'WAITING'    → CustomerState.WaitingInLine
///     'LEAVE'      → CustomerState.Leaving
///
///   physics.moveTo(sprite, x, y, speed) → _movement.RequestPath(cellCoord)
///   handleStuckRecovery()               → CharacterMovement.HandlePathStale() (tự động)
///   boredomThreshold (45000ms)          → BOREDOM_THRESHOLD (45s)
///
/// THAY ĐỔI QUAN TRỌNG:
///   - Không dùng world coordinates trực tiếp cho movement
///   - Tất cả movement đi qua CharacterMovement → A* path
///   - Targets được convert từ world → cell trước khi RequestPath
/// </summary>
[RequireComponent(typeof(CharacterMovement))]
public class CustomerAIController : MonoBehaviour
{
    // =========================================================================
    // CONSTANTS
    // =========================================================================

    private const float BOREDOM_THRESHOLD = 45f;    // 45s như hệ thống cũ
    private const float DECISION_INTERVAL = 1.5f;   // 1500ms như hệ thống cũ
    private const float INTERACT_DURATION = 1f;     // 1000ms như hệ thống cũ

    // =========================================================================
    // CUSTOMER STATE MACHINE
    // =========================================================================

    public enum CustomerState
    {
        Spawning,
        Wandering,
        SeekingItem,
        Interacting,
        GoingToCashier,
        WaitingInLine,
        Leaving,
        Stuck  // Không có trong hệ thống cũ — state mới khi A* không tìm được đường
    }

    public CustomerState CurrentState { get; private set; } = CustomerState.Spawning;

    // =========================================================================
    // COMPONENT REFERENCES — Cache trong Awake
    // =========================================================================

    private CharacterMovement _movement;

    // =========================================================================
    // AI STATE
    // =========================================================================

    private float _spawnTime;
    private float _lastDecisionTime;
    private float _interactStartTime;

    // Intent: BUY hoặc PLAY (tương đương intent trong hệ thống cũ)
    public enum CustomerIntent { Buy, Play }
    public CustomerIntent Intent { get; private set; }

    // Instance ID duy nhất
    public string InstanceId { get; private set; }

    // =========================================================================
    // VÒNG ĐỜI
    // =========================================================================

    private void Awake()
    {
        // Cache — KHÔNG GetComponent trong Update
        _movement = GetComponent<CharacterMovement>();

        // Subscribe vào movement events
        _movement.OnReachedGoal += HandleReachedGoal;
        _movement.OnPathNotFound += HandlePathNotFound;
        _movement.OnGoalAbandoned += HandleGoalAbandoned;
    }

    private void OnDestroy()
    {
        if (_movement != null)
        {
            _movement.OnReachedGoal -= HandleReachedGoal;
            _movement.OnPathNotFound -= HandlePathNotFound;
            _movement.OnGoalAbandoned -= HandleGoalAbandoned;
        }
    }

    /// <summary>
    /// Khởi tạo NPC với intent và ID.
    /// Tương đương constructor của Customer object trong NPCManager.ts cũ.
    /// </summary>
    public void Initialize(string instanceId, CustomerIntent intent)
    {
        InstanceId = instanceId;
        Intent = intent;
        _spawnTime = Time.time;
        _lastDecisionTime = Time.time;

        TransitionToState(CustomerState.Spawning);
    }

    private void Update()
    {
        // KHÔNG gọi GetComponent hay Find ở đây
        UpdateStateMachine();
        CheckBoredom();
    }

    // =========================================================================
    // STATE MACHINE
    // =========================================================================

    private void UpdateStateMachine()
    {
        switch (CurrentState)
        {
            case CustomerState.Spawning:
                HandleSpawning();
                break;
            case CustomerState.Wandering:
                HandleWandering();
                break;
            case CustomerState.Interacting:
                HandleInteracting();
                break;
            case CustomerState.WaitingInLine:
                HandleWaitingInLine();
                break;
            case CustomerState.Stuck:
                HandleStuck();
                break;
            // SeekingItem, GoingToCashier, Leaving: 
            // CharacterMovement đang di chuyển → đợi OnReachedGoal event
        }
    }

    private void HandleSpawning()
    {
        // Chờ 0.5s rồi bắt đầu wander (tương đương timer=now+500 trong cũ)
        if (Time.time - _spawnTime > 0.5f)
        {
            TransitionToState(CustomerState.Wandering);
            MoveToRandomWanderPoint();
        }
    }

    private void HandleWandering()
    {
        if (Time.time - _lastDecisionTime < DECISION_INTERVAL) return;
        _lastDecisionTime = Time.time;

        if (Intent == CustomerIntent.Buy)
        {
            TryFindShelf();
        }
        // PLAY intent: TryFindTable() — implement ở bước sau
    }

    private void HandleInteracting()
    {
        // Đứng tại kệ trong INTERACT_DURATION rồi "lấy hàng"
        if (Time.time - _interactStartTime >= INTERACT_DURATION)
        {
            // Placeholder: Logic lấy hàng sẽ implement với InventoryManager
            Debug.Log($"[CustomerAI] {InstanceId}: Đã tương tác với kệ. " +
                      "Đi về cashier...");
            TransitionToState(CustomerState.GoingToCashier);
            MoveToNearestCashier();
        }
    }

    private void HandleWaitingInLine()
    {
        // Placeholder: Chờ được phục vụ
        // Logic thực tế sẽ implement với InventoryManager ở bước sau
    }

    private void HandleStuck()
    {
        // Timeout: Nếu stuck quá 5s → Leave
        if (Time.time - _spawnTime > BOREDOM_THRESHOLD)
        {
            TransitionToState(CustomerState.Leaving);
            MoveToExit();
        }
    }

    // =========================================================================
    // STATE TRANSITIONS
    // =========================================================================

    private void TransitionToState(CustomerState newState)
    {
        if (verboseLogging)
            Debug.Log($"[CustomerAI] {InstanceId}: {CurrentState} → {newState}");

        CurrentState = newState;
    }

    // =========================================================================
    // MOVEMENT REQUESTS — Chuyển đổi target sang cell rồi RequestPath
    // =========================================================================

    private void MoveToRandomWanderPoint()
    {
        if (GridSystem.Instance == null) return;

        // Tìm random cell trong shop bounds (walkable)
        var bounds = GridSystem.Instance;
        for (int attempt = 0; attempt < 10; attempt++)
        {
            int x = Random.Range(bounds.ShopMinCell.x, bounds.ShopMaxCell.x + 1);
            int y = Random.Range(bounds.ShopMinCell.y, bounds.ShopMaxCell.y + 1);
            var cell = new Vector2Int(x, y);

            if (PathfindingGrid.Instance != null && PathfindingGrid.Instance.IsWalkable(cell))
            {
                _movement.RequestPath(cell);
                return;
            }
        }
    }

    private void TryFindShelf()
    {
        // Placeholder: Tìm kệ gần nhất có hàng
        // Sẽ implement đầy đủ khi tích hợp với InventoryManager ở bước sau
        Debug.Log($"[CustomerAI] {InstanceId}: Đang tìm kệ...");
    }

    private void MoveToNearestCashier()
    {
        // Placeholder: Di chuyển đến cashier
        Debug.Log($"[CustomerAI] {InstanceId}: Di chuyển đến cashier...");
    }

    private void MoveToExit()
    {
        if (GridSystem.Instance == null) return;
        // Di chuyển đến rìa dưới của shop (cửa ra)
        var exitCell = new Vector2Int(0, GridSystem.Instance.ShopMinCell.y);
        _movement.RequestPath(exitCell);
    }

    // =========================================================================
    // EVENT HANDLERS — Từ CharacterMovement
    // =========================================================================

    private void HandleReachedGoal()
    {
        switch (CurrentState)
        {
            case CustomerState.Wandering:
                // Đến điểm wander → làm quyết định mới
                _lastDecisionTime = 0; // Force decision ngay
                break;

            case CustomerState.SeekingItem:
                // Đến kệ → bắt đầu interact
                _interactStartTime = Time.time;
                TransitionToState(CustomerState.Interacting);
                break;

            case CustomerState.GoingToCashier:
                TransitionToState(CustomerState.WaitingInLine);
                break;

            case CustomerState.Leaving:
                // Đến cửa → destroy
                Destroy(gameObject);
                break;
        }
    }

    private void HandlePathNotFound()
    {
        Debug.LogWarning($"[CustomerAI] {InstanceId}: Path not found, switching to Wander.");
        TransitionToState(CustomerState.Wandering);
        MoveToRandomWanderPoint();
    }

    private void HandleGoalAbandoned()
    {
        Debug.LogWarning($"[CustomerAI] {InstanceId}: Goal abandoned, leaving shop.");
        TransitionToState(CustomerState.Leaving);
        MoveToExit();
    }

    // =========================================================================
    // UTILITIES
    // =========================================================================

    private void CheckBoredom()
    {
        if (CurrentState == CustomerState.Leaving) return;
        if (Time.time - _spawnTime > BOREDOM_THRESHOLD)
        {
            TransitionToState(CustomerState.Leaving);
            MoveToExit();
        }
    }

    [SerializeField] private bool verboseLogging = false;
}
```

---

### 4.7 `PathfindingDebugVisualizer.cs`

**Vị trí:** `Assets/Scripts/Debug/PathfindingDebugVisualizer.cs`

```csharp
// Assets/Scripts/Debug/PathfindingDebugVisualizer.cs

using System.Collections.Generic;
using UnityEngine;

/// <summary>
/// Visualizer cho pathfinding debug.
/// Vẽ walkable/non-walkable nodes và active paths trong Scene view.
/// Disable trong production build.
/// </summary>
public class PathfindingDebugVisualizer : MonoBehaviour
{
    [Header("Visualization")]
    [SerializeField] private bool showWalkableNodes = true;
    [SerializeField] private bool showNonWalkableNodes = true;
    [SerializeField] private Color walkableColor = new Color(0f, 1f, 0f, 0.1f);
    [SerializeField] private Color nonWalkableColor = new Color(1f, 0f, 0f, 0.3f);
    [SerializeField] private Color pathColor = Color.cyan;

    [Header("Test Controls")]
    [Tooltip("Cell xuất phát cho test path (nhấn T).")]
    [SerializeField] private Vector2Int testStartCell = new Vector2Int(-5, -5);
    [Tooltip("Cell đích cho test path (nhấn T).")]
    [SerializeField] private Vector2Int testGoalCell = new Vector2Int(5, 5);

    private List<Vector2Int> _debugPath;

    private void Update()
    {
        // Nhấn T để test pathfinding từ testStartCell đến testGoalCell
        if (UnityEngine.InputSystem.Keyboard.current?.tKey.wasPressedThisFrame == true)
        {
            TestPathfinding();
        }
    }

    private void TestPathfinding()
    {
        if (PathfindingGrid.Instance == null)
        {
            Debug.LogError("[PathfindingDebugVisualizer] PathfindingGrid.Instance is null!");
            return;
        }

        float startTime = Time.realtimeSinceStartup;

        _debugPath = PathfindingCore.FindPath(
            testStartCell,
            testGoalCell,
            PathfindingGrid.Instance
        );

        float elapsed = (Time.realtimeSinceStartup - startTime) * 1000f;

        if (_debugPath != null)
        {
            Debug.Log($"[PathfindingDebugVisualizer] Path tìm thấy: " +
                      $"{_debugPath.Count} bước từ {testStartCell} đến {testGoalCell}. " +
                      $"Thời gian: {elapsed:F3}ms.");
        }
        else
        {
            Debug.LogWarning($"[PathfindingDebugVisualizer] Không tìm được path " +
                             $"từ {testStartCell} đến {testGoalCell}.");
        }
    }

    private void OnDrawGizmos()
    {
        if (PathfindingGrid.Instance == null || GridSystem.Instance == null) return;

        // Vẽ nodes
        if (showWalkableNodes || showNonWalkableNodes)
        {
            for (int x = GridSystem.Instance.ShopMinCell.x;
                 x <= GridSystem.Instance.ShopMaxCell.x; x++)
            {
                for (int y = GridSystem.Instance.ShopMinCell.y;
                     y <= GridSystem.Instance.ShopMaxCell.y; y++)
                {
                    var cell = new Vector2Int(x, y);
                    var node = PathfindingGrid.Instance.GetNode(cell);
                    if (node == null) continue;

                    if (node.IsWalkable && showWalkableNodes)
                        Gizmos.color = walkableColor;
                    else if (!node.IsWalkable && showNonWalkableNodes)
                        Gizmos.color = nonWalkableColor;
                    else
                        continue;

                    Vector3 worldPos = GridSystem.Instance.CellToWorld(cell);
                    Gizmos.DrawCube(worldPos, GridSystem.Instance.IsometricGrid.cellSize * 0.85f);
                }
            }
        }

        // Vẽ debug path
        if (_debugPath != null && _debugPath.Count > 0)
        {
            Gizmos.color = pathColor;
            Vector3 prev = GridSystem.Instance.CellToWorld(testStartCell);

            foreach (var cell in _debugPath)
            {
                Vector3 worldPos = GridSystem.Instance.CellToWorld(cell);
                Gizmos.DrawLine(prev, worldPos);
                Gizmos.DrawSphere(worldPos, 0.12f);
                prev = worldPos;
            }
        }
    }
}
```

---

## 5. Cập Nhật GridSystem — Thêm Public Accessors

Cursor phải thêm các dòng sau vào `GridSystem.cs` (file đã tạo ở Bước 3) để `PathfindingGrid` có thể đọc bounds:

```csharp
// Thêm vào GridSystem.cs — public accessors cho PathfindingGrid

/// <summary>Public accessor cho PathfindingGrid.</summary>
public Vector2Int ShopMinCell => shopMinCell;

/// <summary>Public accessor cho PathfindingGrid.</summary>
public Vector2Int ShopMaxCell => shopMaxCell;
```

---

## 6. Thiết Lập Scene

```
GameScene Hierarchy (cập nhật từ Bước 3):
│
├── _PathfindingGrid           [PathfindingGrid.cs]          ← MỚI
│   Inspector:
│     Allow Diagonal: false (4-directional)
│     Verbose Logging: true
│
├── _PathfindingDebugVisualizer [PathfindingDebugVisualizer]  ← MỚI (debug only)
│   Inspector:
│     Show Walkable Nodes:     false (tắt khi shop lớn)
│     Show Non-Walkable Nodes: true
│     Test Start Cell: (-3, -3)
│     Test Goal Cell:  (3, 3)
│
└── NPCs/                      (Empty parent)
    └── NPC_Test_001           [SpriteRenderer]
                               [CharacterMovement]
                               [CustomerAIController]
        Inspector (CharacterMovement):
          Move Speed: 3
          Default Facing Right: true
          Show Path Gizmos: true
        Inspector (CustomerAIController):
          Verbose Logging: true
```

---

## 7. Kịch Bản Kiểm Thử Đầy Đủ

### 7.1 Chuẩn Bị

```
Bước 1: Mở GameScene, đảm bảo GridSystem, PlacementManager, PathfindingGrid đều có trong scene.

Bước 2: Tạo NPC test:
  - Tạo GameObject "NPC_Test_001"
  - Thêm SpriteRenderer (sprite đơn giản, hình vuông màu xanh)
  - Thêm CharacterMovement
  - Thêm CustomerAIController
  - Đặt tại world position = GridSystem.CellToWorld(0, 0)

Bước 3: Tạo script test tạm thời "Step4TestRunner.cs":
```

```csharp
// Assets/Scripts/Debug/Step4TestRunner.cs — XÓA SAU KHI TEST

using UnityEngine;

public class Step4TestRunner : MonoBehaviour
{
    [SerializeField] private CustomerAIController testNPC;

    private void Start()
    {
        if (testNPC != null)
            testNPC.Initialize("npc_test_001", CustomerAIController.CustomerIntent.Buy);
    }
}
```

### 7.2 Test A* Cơ Bản

```
Test 1 — Nhấn T để test pathfinding thủ công:
  PathfindingDebugVisualizer.TestStartCell = (-3, -3)
  PathfindingDebugVisualizer.TestGoalCell  = (3, 3)
  Nhấn T trong Play mode

  KỲ VỌNG Console:
    "[PathfindingDebugVisualizer] Path tìm thấy: N bước từ (-3,-3) đến (3,3). Thời gian: X.XXXms."

  KỲ VỌNG Scene view:
    Đường màu cyan từ (-3,-3) đến (3,3) hiển thị trong Scene view

Test 2 — Không có đường:
  Đặt kệ hàng bao vây hoàn toàn cell (3, 3)
  Nhấn T

  KỲ VỌNG Console:
    "[PathfindingDebugVisualizer] Không tìm được path từ (-3,-3) đến (3,3)."
    "[PathfindingCore] Goal cell (3,3) không walkable..."
```

### 7.3 Test Core — Đặt Kệ Chặn Đường NPC Đang Đi

```
ĐÂY LÀ TEST QUAN TRỌNG NHẤT THEO SPEC

Setup:
  NPC đang di chuyển từ (0,0) đến (10,10) — path xuyên qua (5,5)
  Quan sát NPC đang di chuyển trong Game view

Hành động:
  Khi NPC đang trên đoạn đường gần (5,5):
  Dùng PlacementManager để đặt kệ tại cell (5,5)

KỲ VỌNG Console (theo đúng thứ tự):
  1. "[PathfindingGrid] Cập nhật lưới phát sinh: 1 cells trở thành obstacles do [ShelfSingle]."
  2. "[CharacterMovement] Cập nhật lưới phát sinh — NPC_Test_001 đang recalculate path đến (10,10). Lần #1/3."

KỲ VỌNG Scene:
  1. NPC dừng lại ngay lập tức (velocity = 0)
  2. Sau đúng 0.1 giây: NPC bắt đầu di chuyển lại theo đường vòng quanh kệ
  3. Path gizmo (cyan) hiển thị đường mới tránh cell (5,5)

KỲ VỌNG TUYỆT ĐỐI KHÔNG CÓ:
  - NPC đi xuyên qua kệ
  - Infinite loop (Console spam không dừng)
  - NullReferenceException
```

### 7.4 Test Anti-Infinite-Loop

```
Test: Đặt kệ bao vây NPC từ nhiều phía

Setup:
  NPC tại (0,0), target (5,0)
  Đặt kệ chặn mọi đường ra khỏi (0,0):
    Kệ tại (1,0), (0,1), (-1,0), (0,-1)

KỲ VỌNG Console:
  "[CharacterMovement] Cập nhật lưới phát sinh — recalculate Lần #1/3."
  "[CharacterMovement] Cập nhật lưới phát sinh — recalculate Lần #2/3."
  "[CharacterMovement] Cập nhật lưới phát sinh — recalculate Lần #3/3."
  "[CharacterMovement] NPC_Test_001: Đã recalculate 3 lần, abandon goal (5,0)."
  → CustomerAIController.HandleGoalAbandoned() được gọi
  → NPC chuyển sang Leaving state

TUYỆT ĐỐI KHÔNG CÓ:
  - Console spam vô tận
  - Game freeze
  - NPC stuck vĩnh viễn
```

### 7.5 Test FlipX Sprite

```
NPC di chuyển sang phải (x tăng):
  KỲ VỌNG: SpriteRenderer.flipX = false (mặt nhìn phải)

NPC di chuyển sang trái (x giảm):
  KỲ VỌNG: SpriteRenderer.flipX = true (mặt nhìn trái)

NPC di chuyển lên/xuống:
  KỲ VỌNG: flipX giữ nguyên từ lần di chuyển ngang cuối
```

### 7.6 Test Performance

```
Tạo 10 NPC đồng thời, đặt nhiều kệ hàng.

KỲ VỌNG:
  FPS không drop dưới 30 FPS
  Console không spam pathfinding logs
  Mỗi A* query < 1ms (theo PathfindingDebugVisualizer timing log)
```

---

## 8. Định Nghĩa "Hoàn Thành" (Definition of Done)

- [ ] `MinHeap<T>` hoạt động đúng: Insert O(log n), ExtractMin O(log n), heap invariant duy trì
- [ ] `PathfindingCore.FindPath()` dùng Manhattan Distance làm heuristic (KHÔNG Euclidean)
- [ ] `PathfindingGrid` đọc `GridNode.IsOccupied` từ `GridSystem` → `PathNode.IsWalkable = false`
- [ ] A* tìm đúng đường vòng quanh kệ hàng trong test cơ bản
- [ ] NPC dừng **đúng 0.1s** khi path stale, không nhiều hơn không ít hơn
- [ ] Console in **chính xác** chuỗi `"Cập nhật lưới phát sinh"` khi path bị chặn
- [ ] Anti-infinite-loop: Không recalculate quá `MAX_RECALCULATIONS = 3` lần
- [ ] `Vector3.MoveTowards` được dùng trong `CharacterMovement.MoveAlongPath()`
- [ ] FlipX sprite đúng theo hướng di chuyển ngang
- [ ] `OnGridChanged` event fire khi furniture đặt xuống, CharacterMovement nhận được
- [ ] Không có `GameObject.Find()` hay `GetComponent()` trong bất kỳ `Update()` nào
- [ ] Không có `NullReferenceException` trong toàn bộ test session
- [ ] Test 10 NPC đồng thời không freeze game

**Chỉ sau khi tất cả checkbox được check, mới chuyển sang Bước 5.**
```