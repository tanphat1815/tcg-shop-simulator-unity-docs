# Feature Extraction Specification
## TCG Cards Shop Simulator → Unity C# Migration

---

## 1. Hệ Thống Dữ Liệu Thẻ Bài & Các Gói (Cards & Packs Data System)

### 1.1 Mục Đích

Quản lý toàn bộ catalogue thẻ bài từ cơ sở dữ liệu SQLite, tổ chức chúng thành các "Set" và "Series", định giá động dựa trên độ hiếm, và cung cấp pool thẻ cho cơ chế mở gói Gacha.

---

### 1.2 Cấu Trúc Dữ Liệu

#### Schema SQLite (Nguồn Gốc)

```
TABLE series
  id: TEXT (PK)          -- vd: "sv", "swsh", "base"
  name: TEXT             -- vd: "Scarlet & Violet"

TABLE sets
  id: TEXT (PK)          -- vd: "sv01", "base1"
  name: TEXT
  serieId: TEXT (FK)
  cardCount: INTEGER

TABLE cards
  id: TEXT (PK)          -- vd: "sv01-001"
  name: TEXT
  supertype: TEXT        -- "Pokémon" | "Trainer" | "Energy"
  hp: TEXT               -- lưu dạng string "120", cần parse
  types: TEXT            -- JSON array: ["Fire"]
  attacks: TEXT          -- JSON array of attack objects
  weaknesses: TEXT       -- JSON array: [{type, value}]
  resistances: TEXT      -- JSON array: [{type, value}]
  retreatCost: INTEGER
  rarity: TEXT           -- "Common" | "Uncommon" | "Rare" | "Double Rare" | ...
  set_id: TEXT (FK)
  series_id: TEXT (FK)
  image: TEXT            -- URL base path
  pricing: TEXT          -- JSON: {tcgplayer: {...}, cardmarket: {...}}
```

#### Rarity Tier Enum (Suy Ra Từ Code)

```
Tier 0 - Common
Tier 1 - Uncommon
Tier 2 - Rare
Tier 3 - Double Rare / Rare Holo
Tier 4 - Ultra Rare / Illustration Rare
Tier 5 - Special Illustration Rare / Secret Rare
Tier 6 - Hyper Secret Rare / Ghost Rare / Mega Secret Rare
```

Bảng ánh xạ từ `rarityRegistry.ts`:
```
HIGH_RARITY_LIST = [
  'Rare', 'Double Rare', 'Ultra Rare', 'Secret Rare',
  'Illustration Rare', 'Special Illustration Rare',
  'Hyper Secret Rare', 'Mega Secret Rare', 'Ghost Rare'
]
```

#### ShopItem (Runtime Object)

```
StockItemInfo {
  id: string               -- "pack_sv01" | "box_sv01"
  name: string
  buyPrice: float          -- Giá shop mua vào từ hệ thống
  sellPrice: float         -- Giá bán ra mặc định cho NPC
  basePrice: float         -- EV gốc (Expected Value từ cards)
  rarityBonusPercent: float -- % markup dựa trên độ hot của set
  requiredLevel: int        -- Level tối thiểu để unlock
  type: "pack" | "box"
  volume: int              -- Số slot chiếm dụng trong kho
  contains?: {itemId, amount}  -- Box chứa 64 packs
  sourceSetId: string      -- Liên kết về set gốc trong DB
  generation: string       -- "GENERATION I" ... "GENERATION IX"
}
```

---

### 1.3 Thuật Toán Định Giá (Pricing Algorithm)

**Input:** `evPrice` (giá trung bình của cards trong set từ DB query), `seriesId`

**Bước 1: Tính Required Level theo Series**
```
Mapping seriesId → requiredLevel:
  base/gym/neo/lc/ecard → 1
  ex                    → 11
  dp/pl/hgss/col        → 21
  bw                    → 31
  xy                    → 41
  sm                    → 51
  swsh                  → 61
  sv                    → 71
  special/misc          → 80
```

**Bước 2: Tính Base EV Price**
```
baseEVPrice = evPrice × 10
-- Lý do: Giả định mỗi pack có 10 cards đại diện
basePackPrice = MAX(baseEVPrice, 2.5)  -- Floor $2.50
```

**Bước 3: Tính Rarity Bonus**
```
rarityBonusPercent = CLAMP(requiredLevel / 80 × 60, 10, 60)
-- Range: 10% (Level 1 sets) đến 60% (Level 80 sets)
```

**Bước 4: Tính Final Prices**
```
packBuyPrice  = ROUND(basePackPrice × (1 + rarityBonusPercent/100), 2)
packSellPrice = ROUND(packBuyPrice × 1.6, 2)       -- Markup 60%
boxBuyPrice   = ROUND(packBuyPrice × 64 × 0.85, 2) -- Bulk discount 15%
boxSellPrice  = ROUND(boxBuyPrice × 1.4, 2)        -- Markup 40%
```

---

### 1.4 Thuật Toán Mở Gói Gacha (Pack Opening RNG)

**File tham chiếu:** `inventoryStore.ts` → `tearPack()`, `apiStore.ts` → `getWeightedRandomCardsFromSet()`

**Lưu ý quan trọng:** Hệ thống hiện tại **không có weighted probability table** được hard-code. Thay vào đó nó dùng `ORDER BY RANDOM()` của SQLite rồi **sort sau theo rarity**. Đây là thiết kế cần tái tư duy khi port sang Unity.

#### Luồng Thực Thi

```
Bước 1: VALIDATION
  - Kiểm tra shopInventory[packId] > 0
  - Lấy sourceSetId từ shopItems[packId]

Bước 2: UI PHASE TRANSITION (tách biệt với logic)
  - currentPack = []
  - packPhase = 'pack_visible'  ← Hiển thị ảnh vỏ pack ngay
  - isOpeningPack = true

Bước 3: DATA FETCH (async)
  SQL: SELECT * FROM cards WHERE set_id = ? ORDER BY RANDOM() LIMIT 6
  → Trả về 6 cards ngẫu nhiên đều nhau (không có trọng số)

Bước 4: RARITY SORT
  Mỗi card được tính rarityRank:
    'Ghost Rare'                  → 10
    'Hyper/Mega Secret Rare'      → 9
    'Special Illustration Rare'   → 8
    'Illustration Rare'           → 7
    'Secret Rare'                 → 6
    'Ultra Rare'                  → 5
    'Double Rare'                 → 4
    'Rare Holo'                   → 3
    'Rare'                        → 3
    string contains 'Rare'        → 2 (fallback)
    'Uncommon'                    → 1
    'Common' / 'None'             → 0

  sortedCards = sort(cards, ASC by rarityRank)
  → Thẻ hiếm nhất luôn ở vị trí cuối cùng (index 5)
  → Đây là thiết kế UX: "reveal thẻ hiếm cuối cùng"

Bước 5: INVENTORY UPDATE
  shopInventory[packId]--
  FOR each card in sortedCards:
    personalBinder[card.id]++
    IF rarityRank >= 3: gainExp(XP_RARE=15)
    ELSE:               gainExp(XP_COMMON=2)

Bước 6: STATE UPDATE
  currentPack = sortedCards
  packPhase = 'cards_visible'
```

#### Điểm Khác Biệt Quan Trọng Cho Unity

Hệ thống hiện tại **không có tỷ lệ rơi (drop rate) được định nghĩa rõ ràng**. Xác suất ra thẻ hiếm hoàn toàn phụ thuộc vào tỷ lệ thực tế của thẻ hiếm trong set đó trong database. Khi port sang Unity, cần **thiết kế lại** bảng drop rate theo tiêu chuẩn TCG thực tế hoặc custom game design.

---

## 2. Hệ Thống Kho Đồ & Trưng Bày (Inventory & Display System)

### 2.1 Mục Đích

Quản lý hai loại inventory riêng biệt: (1) Kho hàng thương mại (packs/boxes chờ bán), (2) Bộ sưu tập cá nhân (thẻ bài đã mở). Điều phối luồng hàng từ Giao Hàng → Kho → Kệ Trưng Bày.

---

### 2.2 Cấu Trúc Dữ Liệu

#### Inventory State

```
shopInventory: Record<itemId, quantity>
  -- vd: { "pack_sv01": 24, "box_sv02": 3 }
  -- Chỉ lưu quantity, metadata lấy từ shopItems

personalBinder: Record<cardId, quantity>
  -- vd: { "sv01-001": 2, "sv01-045": 1 }
  -- Thẻ đã mở, dùng cho Battle và Binder UI

shopItems: Record<itemId, StockItemInfo>
  -- Master catalogue, populated từ API/DB
```

#### Shelf Structure (Kệ Hàng)

```
ShelfData {
  id: string           -- "shelf_1234567890"
  furnitureId: string  -- "shelf_single" | "shelf_double" | "storage_shelf"
  x, y: float         -- World coordinates
  role: "selling" | "storage"
  tiers: ShelfTier[]   -- Array of tiers (3-4 tiers per shelf)
}

ShelfTier {
  itemId: string | null  -- NULL nếu tầng trống; tất cả slots cùng loại
  slots: string[]        -- Array of itemId (length = số lượng thực)
  maxSlots: int         -- Sức chứa tối đa
}
```

#### Shelf Config (từ FURNITURE_ITEMS)

```
shelf_single:  numTiers=3, slotsPerTier=16, role=selling
shelf_double:  numTiers=4, slotsPerTier=32, role=selling
storage_shelf: numTiers=3, slotsPerTier=4,  role=storage
```

#### Slot Capacity Rules

```
IF itemType == 'box':
  maxSlots = FLOOR(slotsPerTier / 4)  -- Box chiếm 4 slot
  -- shelf_single: 16/4 = 4 boxes/tier
ELSE (pack):
  maxSlots = slotsPerTier
  -- shelf_single: 16 packs/tier
```

---

### 2.3 Luồng Hàng Hóa (Goods Flow Pipeline)

```
[Checkout Cart]
    ↓ cartStore.checkout()
    ↓ statsStore.spendMoney(total)
    ↓ deliveryStore.scheduleDelivery(items)
    
[Delivery Queue] → pendingDeliveries[]
    ↓ Phaser DeliveryManager spawns physical box sprites
    ↓ Player walks to box, presses [F]
    
[Player Carries Box]
    ↓ deliveryStore.carriedBox = item
    ↓ Player presses [E] near shelf
    
[Shelf Interaction - Two Paths]
  Path A: shelf.role == 'selling'
    → Open ShelfManagementMenu UI
    → Player manually selects tier
    → furnitureStore.fillTierFromItem(shelfId, itemId, tierIndex, qty)
    → Box removed from world (consumed)
    → Optional: SetPriceModal opens
    
  Path B: shelf.role == 'storage'
    → inventoryStore.shopInventory[itemId] += quantity
    → Box removed from world
    → No price setting needed
```

---

### 2.4 Thuật Toán Xếp Hàng Lên Kệ

#### moveToTierSlot (1 item)

```
FUNCTION moveToTierSlot(shelfId, itemId, tierIndex):
  shelf = placedShelves[shelfId]
  itemData = shopItems[itemId]
  currentStock = shopInventory[itemId] ?? 0
  
  IF currentStock <= 0: RETURN
  
  tier = shelf.tiers[tierIndex]
  maxSlots = (itemData.type == 'box') 
             ? FLOOR(slotsPerTier / 4) 
             : slotsPerTier
  
  IF tier.itemId == NULL:
    tier.itemId = itemId
    tier.maxSlots = maxSlots
    tier.slots = []
  
  IF tier.itemId != itemId: RETURN  -- Không trộn loại
  IF tier.slots.length >= tier.maxSlots: RETURN  -- Đầy
  
  tier.slots.push(itemId)
  shopInventory[itemId]--
  IF shopInventory[itemId] == 0: DELETE shopInventory[itemId]
```

#### fillTier (Fill tối đa có thể)

```
FUNCTION fillTier(shelfId, itemId, tierIndex):
  -- Tương tự moveToTierSlot nhưng loop:
  spaceLeft = tier.maxSlots - tier.slots.length
  available = shopInventory[itemId] ?? 0
  toAdd = MIN(spaceLeft, available)
  
  REPEAT toAdd TIMES:
    tier.slots.push(itemId)
  
  shopInventory[itemId] -= toAdd
  IF shopInventory[itemId] <= 0: DELETE
```

#### clearTier (Thu hồi về kho)

```
FUNCTION clearTier(shelfId, tierIndex):
  tier = shelf.tiers[tierIndex]
  IF tier.itemId == NULL: RETURN
  
  shopInventory[tier.itemId] += tier.slots.length
  tier.itemId = NULL
  tier.slots = []
  tier.maxSlots = 0
```

---

### 2.5 Logic NPC Mua Hàng Từ Kệ

```
FUNCTION npcTakeItemFromSlot(shelfId):
  shelf = placedShelves[shelfId]
  
  -- CHỈ mua từ kệ selling, không phải storage
  IF shelf.role == 'storage': RETURN NULL
  
  filledTiers = shelf.tiers.filter(t => t.itemId != null AND t.slots.length > 0)
  IF filledTiers.length == 0: RETURN NULL
  
  -- Chọn ngẫu nhiên một tầng có hàng
  pickedTier = filledTiers[RANDOM_INT(0, filledTiers.length)]
  itemId = pickedTier.itemId
  
  pickedTier.slots.pop()
  IF pickedTier.slots.length == 0:
    pickedTier.itemId = NULL
    pickedTier.maxSlots = 0
  
  dailyStats.itemsSold++
  RETURN itemId
```

---

## 3. Hệ Thống Lưới & Xây Dựng Cửa Hàng (Grid Placement System)

### 3.1 Mục Đích

Cho phép người chơi mua và đặt nội thất vào không gian cửa hàng với validation vị trí, hỗ trợ rotation, và chế độ chỉnh sửa để di chuyển/cất nội thất đã đặt.

---

### 3.2 Cấu Trúc Dữ Liệu Không Gian

#### Coordinate System

```
World Origin: (0, 0)
Shop Origin:  (START_X=1000, START_Y=1000)
  -- Shop được offset vào giữa world rộng 5500×3000
  -- Lý do: Tránh đụng với Town area (x > 3000)

Shop Base Size: 400×400 px
Expansion Step: +200px rộng, +80px cao mỗi level

Town Area: x >= 3000, y từ 0 đến 1500
```

#### Shop Bounds (Dynamic)

```
shopBounds = {
  x: START_X,
  y: START_Y,
  w: BASE_WIDTH + expansionLevel × 200,
  h: BASE_HEIGHT + expansionLevel × 80
}
```

#### Placed Furniture State

```
placedShelves:  Record<id, ShelfData>    -- x, y là world coords
placedTables:   Record<id, PlayTableData>
placedCashiers: Record<id, CashierData>

PlayTableData {
  id, furnitureId, x, y
  occupants: [string|null, string|null]  -- 2 ghế
  matchStartedAt: timestamp | null
  rotation: 0 | 90  -- Ngang hoặc dọc
}
```

---

### 3.3 Thuật Toán Validation Đặt Vật (Placement Validation)

```
FUNCTION validatePlacement(worldX, worldY, ghostSize):
  bounds = shopBounds
  PAD = 10  -- Buffer từ tường
  
  -- RULE 1: Phải nằm trong shop
  IF worldX < bounds.x + PAD: RETURN INVALID
  IF worldX > bounds.x + bounds.w - PAD: RETURN INVALID
  IF worldY < bounds.y + PAD: RETURN INVALID
  IF worldY > bounds.y + bounds.h - PAD: RETURN INVALID
  
  -- RULE 2: Không đè lên vật khác
  placementRect = Rectangle(
    worldX - ghostW/2, worldY - ghostH/2, ghostW, ghostH
  )
  
  FOR EACH group IN [walls, shelves, tables, cashiers]:
    FOR EACH item IN group:
      IF item.id == editFurnitureData?.id: CONTINUE  -- Bỏ qua chính nó khi edit
      IF Intersects(placementRect, item.bounds): RETURN INVALID
  
  -- RULE 3: Không đè lên player
  IF Distance(worldX, worldY, player.x, player.y) < 50: RETURN INVALID
  
  RETURN VALID
```

**Ghost Sizes (Hitbox nhỏ hơn visual để dễ đặt):**
```
Table ghost:  width=50, height=30
Other ghost:  width=30, height=30
```

---

### 3.4 Thuật Toán Đặt Nội Thất

```
STATE MACHINE:
  IDLE → [Player opens Build Menu] → SELECTING
  SELECTING → [Player clicks furniture] → PLACING
  PLACING → [Valid click] → PLACED
  PLACING → [ESC / Right-click] → CANCELLED → IDLE
  
  EDIT_MODE: [Click on placed furniture] → PICKING_UP
  PICKING_UP → [Mouse move] → PLACING (re-use same flow)
```

#### Build Mode Update Loop (mỗi frame)

```
FUNCTION handleBuildMode(pointer):
  -- Spawn ghost sprite nếu chưa có
  IF ghostSprite == NULL:
    CREATE ghost at (pointer.worldX, pointer.worldY)
    SET ghost.alpha = 0.6
  
  -- Update ghost position
  ghost.position = (pointer.worldX, pointer.worldY)
  
  -- Validate và cập nhật màu ghost
  isValid = validatePlacement(pointer.worldX, pointer.worldY)
  ghost.tint = isValid ? WHITE : RED
  
  -- Cooldown tránh double-place (300ms)
  IF pointer.isDown AND isValid AND (now - lastPlacementTime > 300):
    placeFurniture(pointer.worldX, pointer.worldY, currentRotation)
    lastPlacementTime = now
    
  IF ESC_pressed OR rightMouseDown:
    cancelPlacement()
```

#### Rotation

```
currentRotation: 0 | 90  -- Phím R để toggle
-- Chỉ có Play Table hỗ trợ rotation
-- rotation=90 → Table vertical (chairs ở trên/dưới)
-- rotation=0  → Table horizontal (chairs ở trái/phải)
```

#### Place Furniture

```
FUNCTION placeFurniture(x, y, rotation):
  furnitureId = buildItemId OR editFurnitureData.furnitureId
  
  IF furnitureId == 'play_table':
    id = 'table_' + timestamp
    placedTables[id] = { id, furnitureId, x, y, occupants:[null,null], rotation }
    
  ELSE IF furnitureId == 'cashier_desk':
    id = 'cashier_' + timestamp
    placedCashiers[id] = { id, furnitureId, x, y }
    
  ELSE (shelves):
    id = 'shelf_' + timestamp
    numTiers = FURNITURE_ITEMS[furnitureId].numTiers
    placedShelves[id] = {
      id, furnitureId, x, y,
      tiers: Array(numTiers).fill({ itemId:null, slots:[], maxSlots:0 }),
      role: FURNITURE_ITEMS[furnitureId].role
    }
  
  -- Trừ kho nếu không phải edit mode
  IF NOT editMode:
    purchasedFurniture[furnitureId]--
    IF purchasedFurniture[furnitureId] == 0: DELETE
  
  currentRotation = 0  -- Reset
  clearGhost()
```

---

### 3.5 Shop Expansion System

```
EXPANSION_LEVELS[] = [
  { id:1, requiredLevel:2,  cost:300,   rentIncrease:20 },
  { id:2, requiredLevel:3,  cost:400,   rentIncrease:20 },
  ... (20 levels total)
  { id:20, requiredLevel:47, cost:6000, rentIncrease:60 }
]

FUNCTION buyExpansion():
  nextId = expansionLevel + 1
  config = EXPANSION_LEVELS.find(e => e.id == nextId)
  
  IF config == NULL: RETURN false  -- Max level
  IF money < config.cost: RETURN false
  IF level < config.requiredLevel: RETURN false
  
  money -= config.cost
  expansionLevel = nextId
  
  -- Trigger environment refresh
  newWidth  = 400 + expansionLevel × 200
  newHeight = 400 + expansionLevel × 80
  -- Recalculate wall colliders, redraw floor, update camera bounds
```

---

## 4. Trí Tuệ Nhân Tạo Khách Hàng (Customer AI Logic)

### 4.1 Mục Đích

Mô phỏng hành vi khách hàng trong cửa hàng qua Finite State Machine (FSM) với hai mục đích: mua hàng và chơi bài. Bao gồm pathfinding đơn giản, stuck recovery, và quyết định kinh tế.

---

### 4.2 FSM States

```
SPAWN → WANDER → SEEK_ITEM → INTERACT → GO_CASHIER → WAITING → [serve] → LEAVE

Alternative path:
SPAWN → WANT_TO_PLAY → SEEK_TABLE → PLAYING → LEAVE

Cross-transitions:
WANDER: Nếu chán (>45s không làm gì) → LEAVE
WANDER: Nếu PLAY intent không tìm được bàn (>10s hoặc 20% chance) → Chuyển intent sang BUY
WAITING/GO_CASHIER: Nếu không còn trong queue (đã được serve) → LEAVE
Any state khi closing time (>=20:00): → LEAVE (với điều kiện riêng)
```

---

### 4.3 Chi Tiết Từng State

#### SPAWN
```
Trigger: NPC được tạo tại cửa ra vào
Duration: 500ms
Action: Di chuyển vào điểm ngẫu nhiên bên trong shop
Next state:
  IF intent == 'PLAY': → WANT_TO_PLAY
  ELSE:                → WANDER
```

#### WANDER
```
Decision interval: mỗi 1500ms

IF intent == 'PLAY':
  Scan all placedTables for (occupants.includes(null))
  IF found AND joinTable() succeeded:
    state = SEEK_TABLE
  ELSE IF searchTime > 10000 OR random() < 0.2:
    intent = 'BUY'
    state = WANDER

IF intent == 'BUY':
  Scan placedShelves WHERE:
    shelf.id NOT IN checkedShelfIds  -- Chưa kiểm tra
    AND shelf.tiers.some(t => t.slots.length > 0)  -- Có hàng
  
  IF foundShelf:
    state = SEEK_ITEM
    target = (shelf.x, shelf.y + 45)  -- Đứng phía trước kệ
  ELSE IF random() < 0.4:
    IF random() < 0.5: intent = 'PLAY', state = WANT_TO_PLAY
    ELSE: → LEAVE

Boredom check:
  IF (now - spawnTime) > 45000ms: → LEAVE
```

#### SEEK_ITEM
```
NPC di chuyển về phía kệ
Distance threshold: 12px

IF distance < 12:
  velocity = 0
  state = INTERACT
  timer = now + 1000ms  -- "Shopping time"
```

#### INTERACT
```
Sau 1000ms "suy nghĩ":
  Tìm kệ gần nhất (distance < 15px)
  
  IF kệ tìm thấy:
    itemId = npcTakeItemFromSlot(shelfId)  -- Lấy 1 món
    
    IF itemId != null:
      itemData = shopItems[itemId]
      targetPrice = itemData.sellPrice
      
      -- Thêm vào queue thanh toán
      customerStore.addWaitingCustomer(targetPrice, instanceId)
      
      -- Tính vị trí xếp hàng
      cashier = placedCashiers[0]
      myIndex = waitingQueue.indexOf(instanceId)
      targetY = cashier.y + 60 + (myIndex × 40)  -- Social distancing
      
      state = GO_CASHIER
      
    ELSE:  -- Kệ hết hàng
      checkedShelfIds.push(shelfId)
      state = WANDER
```

**Lưu ý quan trọng:** Không có logic kiểm tra giá trong INTERACT. NPC **luôn mua** nếu kệ có hàng. Quyết định kinh tế được xử lý ở cấp độ `sellPrice` trên kệ (player set giá, NPC không compare với budget).

#### GO_CASHIER
```
Di chuyển đến vị trí trong hàng chờ
Distance threshold: 5px

IF distance <= 5:
  velocity = 0
  state = WAITING
```

#### WAITING
```
Mỗi frame: Kiểm tra vị trí trong queue
  myIndex = waitingQueue.indexOf(instanceId)
  
  IF myIndex == -1:  -- Đã được serve hoặc bị remove
    → LEAVE
  
  expectedY = cashier.y + 60 + (myIndex × 40)
  IF |currentY - expectedY| > 5:
    state = GO_CASHIER  -- Di chuyển lên vị trí mới (người trước đã rời)
```

#### PLAYING
```
Cần 2 occupants để bắt đầu match
IF table.occupants.all(!=null) AND table.matchStartedAt == null:
  table.matchStartedAt = now
  
IF table.matchStartedAt != null:
  elapsed = now - matchStartedAt
  matchDuration = 12000ms  -- 12 giây

  -- Visual effect: 🃏 emoji float mỗi 1s
  
  IF elapsed >= 12000:
    IF seatIndex == 0:  -- Chỉ tính 1 lần per bàn
      finishMatch(tableId)
      gainExp(50)
    → LEAVE
```

---

### 4.4 Stuck Recovery System

```
stuckCheckInterval = 500ms
minimumVelocityThreshold = 100 (velocity.lengthSq)

Mỗi 500ms, với NPC đang trong di chuyển states [WANDER, SEEK_ITEM, SEEK_TABLE, GO_CASHIER, LEAVE]:
  distance = Distance(npc.pos, target.pos)
  isStuck = velocity.lengthSq < 100  -- Gần như đứng yên
  
  IF distance > 15 AND isStuck:
    physics.moveTo(npc, target, speed=100)  -- Kick lại physics
```

---

### 4.5 Spawn System

```
Spawn interval: 3000ms (timer lặp)
Max NPC count: 15

Spawn conditions:
  shopState == 'OPEN'
  timeInMinutes < 1200  -- Trước 20:00
  currentNPCCount < 15

Spawn location: doorLocation (x, y+50)

Intent assignment:
  30% chance → intent = 'PLAY' (tìm bàn chơi bài)
  70% chance → intent = 'BUY'  (mua hàng)
```

---

### 4.6 Closing Time Behavior

```
isClosingTime = (timeInMinutes >= 1200) OR (shopState == 'CLOSED')

IF isClosingTime:
  State PLAYING / WANT_TO_PLAY / SEEK_TABLE → LEAVE (ngay lập tức)
  State WANDER / SEEK_ITEM (targetPrice == 0) → LEAVE
  -- NPC đang GO_CASHIER / WAITING được phép hoàn tất giao dịch
```

---

### 4.7 Animation State

```
Velocity-based animation selection:
  IF |vx| > |vy|:
    vx < 0 → play 'npc-left'
    vx > 0 → play 'npc-right'
  ELSE:
    vy < 0 → play 'npc-up'
    vy > 0 → play 'npc-down'
  
  IF velocity ≈ 0: stop animation
```

---

## 5. Nền Kinh Tế & Thời Gian (Economy & Time Systems)

### 5.1 Mục Đích

Quản lý vòng đời tài chính của cửa hàng qua chu kỳ ngày/đêm, bao gồm doanh thu, chi phí vận hành, hệ thống level/XP, và các cơ chế mở khóa nội dung.

---

### 5.2 Cấu Trúc Dữ Liệu Tài Chính

```
StatsStore {
  money: float           -- Số dư hiện tại
  level: int             -- Shop level (1-80+)
  currentExp: float      -- XP tích lũy trong level hiện tại
  currentDay: int        -- Ngày thứ N
  timeInMinutes: int     -- 480 = 8:00 AM, 1200 = 20:00 (closing)
  expansionLevel: int    -- 0-20 (số lần mở rộng shop)
  
  dailyStats: {
    revenue: float          -- Tổng doanh thu ngày
    customersServed: int    -- Khách đã phục vụ
    itemsSold: int          -- Số món đã bán
  }
}
```

---

### 5.3 Hệ Thống Thời Gian

#### Time Tick

```
REAL TIME → GAME TIME conversion:
  1 second real time = 1 minute game time
  Timer: delay=1000ms, loop=true

Conditions for time to advance:
  shopState == 'OPEN'
  NOT isBuildMode
  NOT isEditMode

Time range:
  480  = 08:00 AM (shop opens)
  1200 = 20:00   (closing time, hard cap)
  
IF timeInMinutes >= 1200: STOP incrementing
```

#### Display Format

```
hours = FLOOR(minutes / 60)
remainingMins = minutes MOD 60
ampm = hours >= 12 ? 'PM' : 'AM'
displayHours = hours > 12 ? hours-12 : (hours == 0 ? 12 : hours)
```

#### Time → NPC Spawn Relationship

```
Spawn condition: timeInMinutes < 1200
-- Không có time-of-day based spawn rate variation hiện tại
-- Mọi giờ trong ngày đều spawn đều nhau (3s interval)
-- Đây là điểm có thể tối ưu khi port sang Unity
```

---

### 5.4 Hệ Thống Doanh Thu (Revenue Flow)

#### Customer Purchase

```
FUNCTION serveCustomer():
  entry = waitingQueue.shift()  -- FIFO
  
  money += entry.price
  dailyStats.revenue += entry.price
  dailyStats.customersServed++
  dailyStats.itemsSold++  -- Trong npcTakeItemFromSlot()
  gainExp(5)  -- Base XP per transaction
  
  RETURN entry.instanceId  -- Để Phaser giải phóng NPC sprite
```

#### Auto Checkout (Staff System)

```
FUNCTION handleAutoCheckout(now):
  IF waitingCustomers == 0: RETURN
  
  cashierWorker = hiredWorkers.find(w => w.duty == 'CHECKOUT')
  IF NOT cashierWorker: RETURN
  
  workerData = WORKERS.find(w.dataId)
  cooldown = SPEED_TO_MS[workerData.checkoutSpeed]
  -- Slow: 5000ms | Normal: 3000ms | Fast: 1500ms | Very Fast: 800ms
  
  IF now > lastAutoCheckoutTime + cooldown:
    serveCustomer()
    lastAutoCheckoutTime = now
```

---

### 5.5 Hệ Thống XP & Level

#### XP Rewards Table

```
SERVE_CUSTOMER: 5 XP    (per checkout)
OPEN_PACK_RARE: 15 XP   (Rare tier và cao hơn, rank >= 3)
OPEN_PACK_COMMON: 2 XP  (Common/Uncommon)
MATCH_FINISHED: 50 XP   (Battle match completed)
-- Note: Một số constants trong config (SERVE_CUSTOMER=50, ITEM_SOLD=10)
-- chưa được dùng trong implementation thực tế
```

#### Level Up Formula

```
XP_PER_LEVEL = 1000
requiredXP(level) = level × 1000

FUNCTION gainExp(amount):
  currentExp += amount
  req = level × 1000
  
  WHILE currentExp >= req:
    level++
    currentExp -= req
    req = level × 1000  -- Recalculate for new level
    showLevelUpNext = true  -- Trigger UI toast
```

---

### 5.6 Hệ Thống Chi Phí Vận Hành

#### End of Day Calculations

```
FUNCTION startNewDay(totalSalary):
  -- 1. CHI PHÍ NHÂN SỰ
  money -= totalSalary
  
  totalSalary = SUM(WORKERS[w.dataId].salary for each hiredWorker)
  -- Trả trước khi bắt đầu ngày mới
  
  -- 2. TIỀN THUÊ MẶT BẰNG (Lũy tiến theo expansion)
  baseRent = 50  -- Tiền thuê cơ bản
  FOR i IN 0..expansionLevel-1:
    baseRent += EXPANSION_LEVELS[i].rentIncrease
  money -= baseRent
  
  -- Ví dụ expansionLevel=3:
  -- 50 + 20 + 20 + 20 = 110/ngày
  
  -- 3. RESET
  currentDay++
  timeInMinutes = 480
  showEndDayModal = false
  dailyStats = { revenue:0, customersServed:0, itemsSold:0 }
```

#### Rent Progression

```
Level 1: base=50, rentIncrease=20 → daily rent = 70
Level 2: base=50, +20+20         → daily rent = 90
Level 5: base=50, +20+20+20+20+60 → daily rent = 170
Level 10: ...tích lũy đến ~330/ngày
Level 20: ~810/ngày
```

---

### 5.7 Hệ Thống Staff (Workers)

#### Worker Data

```
WORKERS[] = [
  { id, name, levelUnlocked, restockSpeed, checkoutSpeed, hiringFee, salary }
]

checkoutSpeed → auto-checkout cooldown:
  'Slow':      5000ms
  'Normal':    3000ms
  'Fast':      1500ms
  'Very Fast':  800ms
```

#### Hiring Validation

```
FUNCTION hireWorker(workerId):
  -- Không thuê duplicate
  IF hiredWorkers.any(w.dataId == workerId): RETURN false
  
  workerData = WORKERS.find(workerId)
  IF NOT spendMoney(workerData.hiringFee): RETURN false
  IF level < workerData.levelUnlocked: RETURN false
  
  hiredWorkers.push({
    instanceId: unique_id,
    dataId: workerId,
    duty: 'NONE',
    targetDeskId: null
  })
```

#### Worker Duties

```
'NONE':     Đứng yên, không làm gì
'CHECKOUT': Tự động thanh toán theo cooldown tương ứng checkoutSpeed
'RESTOCK':  AI di chuyển đến kệ ngẫu nhiên (visual only, không có logic xếp hàng thực)
```

---

### 5.8 Hệ Thống Giá Bán (Sell Price System)

```
SetPriceTarget {
  shelfId, tierIndex, itemId
  currentPrice: float   -- Giá đang set
  marketPrice: float    -- Reference từ shopItems.sellPrice
  buyPrice: float       -- Giá nhập về (floor)
}

profit = sellPrice - buyPrice
profitPercent = (sellPrice / buyPrice - 1) × 100

Quick actions:
  Set to Market: customPrice = marketPrice
  +10%: customPrice = customPrice × 1.1
  -10%: customPrice = customPrice × 0.9
  Round: customPrice = ROUND(customPrice × 10) / 10

Apply:
  shopItems[itemId].sellPrice = customPrice
  -- NPC khi mua sẽ dùng giá này: customer.targetPrice = itemData.sellPrice
```

**Quan trọng:** Hiện tại NPC không so sánh giá với "budget" hay "willingness to pay". Mọi NPC đều mua bất kể giá bao nhiêu miễn là kệ có hàng. Đây là điểm thiếu hoàn chỉnh về mặt kinh tế học game mà bản Unity nên cân nhắc thêm vào.

---

### 5.9 Save/Load System

#### Persisted Data

```
localStorage key: 'tcg-shop-save'

{
  money, level, currentExp, expansionLevel, currentDay,
  shopInventory: Record<itemId, qty>,
  personalBinder: Record<cardId, qty>,
  placedShelves: Record<id, ShelfData>,
  placedTables: Record<id, PlayTableData>,
  placedCashiers: Record<id, CashierData>,
  purchasedFurniture: Record<furnitureId, qty>,
  shopState: 'OPEN' | 'CLOSED',
  gymLeaders: GymLeader[]
}
```

#### Auto-save Trigger

```
Subscribe to ALL store changes (deep watch):
  statsStore.$subscribe → saveGame()
  inventoryStore.$subscribe → saveGame()
  furnitureStore.$subscribe → saveGame()
  customerStore.$subscribe → saveGame()
-- Bất kỳ thay đổi state nào đều trigger save ngay lập tức
-- Không có debounce → cần tối ưu khi port sang Unity (dùng dirty flag)
```

---

## Tổng Kết: Các Điểm Cần Thiết Kế Lại Cho Unity

| Vấn Đề | Mô Tả | Khuyến Nghị |
|--------|--------|-------------|
| **Gacha Drop Rate** | Hiện dùng `ORDER BY RANDOM()` SQL, không có weighted table | Thiết kế bảng tỷ lệ theo TCG standard (Common 60%, Uncommon 30%, Rare 10%...) |
| **NPC Price Check** | NPC mua bất kể giá | Thêm `maxBudget` và `willingnessMultiplier` cho từng NPC |
| **Auto-save** | Save mỗi state change, không có debounce | Dùng dirty flag, save định kỳ hoặc khi end of day |
| **Time-NPC Correlation** | Spawn rate cố định 3s bất kể giờ | Thêm spawn rate curve theo giờ trong ngày |
| **Pathfinding** | Dùng physics.moveTo đơn giản + stuck recovery thủ công | Unity NavMesh hoặc A* pathfinding chính thức |
| **Worker Restock Logic** | RESTOCK duty chỉ là visual, không thực sự xếp hàng | Cần implement logic tự động xếp hàng lên kệ |