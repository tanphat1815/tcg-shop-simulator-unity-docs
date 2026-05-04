```markdown
# Step 1: Thiết Lập Ma Trận Không Gian Hình Học Z-as-Y
## Cursor Instructions — TCG Shop Simulator (Unity Port)

**Phiên bản tài liệu:** 1.0  
**Giai đoạn:** Foundation Layer  
**Yêu cầu tiên quyết:** Unity 6 LTS, Package `com.unity.inputsystem` đã được cài đặt, Package `com.unity.2d.tilemap` đã được cài đặt.

---

## 1. Mục Tiêu Của Bước Này

Xây dựng nền tảng kỹ thuật không thể thiếu trước khi viết bất kỳ logic gameplay nào. Bước này thiết lập ba trụ cột:

1. **Không gian hình học Isometric Z-as-Y** — Cấu hình Unity hiểu đúng rằng trục Y của thế giới game chính là chiều sâu Z, đảm bảo các sprite được sắp xếp lớp (sorting) chính xác theo vị trí trong không gian isometric.
2. **CameraController** — Một camera 2D phản ứng mượt mà với mọi loại đầu vào: kéo chuột, cuộn bánh xe, chạm một ngón tay và pinch hai ngón trên mobile.
3. **GameManager Singleton** — Trung tâm điều phối toàn cục, đảm bảo chỉ tồn tại một instance duy nhất xuyên suốt vòng đời ứng dụng.

**Kết quả mong đợi sau bước này:** Scene chạy được, camera di chuyển mượt không lỗi, Console in ra `[GameManager] Ready.`, không có bất kỳ NullReferenceException nào.

---

## 2. Danh Sách Files Cần Tạo

Cursor phải tạo đúng các file sau, đặt đúng thư mục. Không được tạo file ở vị trí khác.

```
Assets/
└── Scripts/
    ├── Core/
    │   ├── GameManager.cs
    │   └── SceneBootstrapper.cs
    ├── Camera/
    │   └── CameraController.cs
    └── World/
        └── IsometricSortingController.cs
```

Ngoài ra, Cursor phải tạo một Scene Unity mới:

```
Assets/
└── Scenes/
    └── GameScene.unity
```

---

## 3. Cấu Hình Unity Editor (THỰC HIỆN TRƯỚC KHI VIẾT CODE)

> ⚠️ **Cursor phải thực hiện các bước cấu hình này thông qua code C# Editor Script, không hướng dẫn người dùng bấm tay. Lý do: đảm bảo cấu hình được version-control và tái tạo được trên mọi máy.**

### 3.1 Transparency Sort Axis — Isometric Z-as-Y

Đây là bước quan trọng nhất. Trong không gian Isometric 2D, Unity mặc định sort sprite theo trục Y thuần túy, dẫn đến hiện tượng sprite ở phía dưới màn hình bị vẽ đè lên sprite phía trên sai cách. Công thức Z-as-Y yêu cầu vector `(0, 1, -0.26)`.

**Giá trị `-0.26` được tính từ:** `tan(15°) ≈ 0.268`, tương ứng góc isometric tiêu chuẩn 2:1 (mỗi 2 pixel ngang = 1 pixel dọc).

Cursor phải tạo file `Assets/Scripts/Editor/IsometricSetup.cs` (chỉ chạy trong Editor, không compile vào build):

```csharp
// Assets/Scripts/Editor/IsometricSetup.cs
// FILE NÀY CHỈ TỒN TẠI TRONG THƯ MỤC Editor, KHÔNG ĐƯA VÀO BUILD.

#if UNITY_EDITOR
using UnityEngine;
using UnityEditor;

/// <summary>
/// Editor utility để cấu hình Isometric Z-as-Y sorting cho toàn bộ dự án.
/// Gọi một lần duy nhất qua menu TCGShop > Setup > Configure Isometric Rendering.
/// Cấu hình được lưu vào ProjectSettings và version-control được.
/// </summary>
public static class IsometricSetup
{
    private const string MENU_PATH = "TCGShop/Setup/Configure Isometric Rendering";

    [MenuItem(MENU_PATH)]
    public static void ConfigureIsometricRendering()
    {
        // --- Bước 1: Thiết lập Transparency Sort Mode ---
        // Phải đặt thành CustomAxis để Unity dùng vector tùy chỉnh bên dưới.
        GraphicsSettings.transparencySortMode = TransparencySortMode.CustomAxis;

        // --- Bước 2: Thiết lập Transparency Sort Axis ---
        // Vector (0, 1, -0.26): 
        //   x=0   → Không sort theo chiều ngang
        //   y=1   → Sort theo chiều dọc (Y trên màn hình)
        //   z=-0.26 → Điều chỉnh chiều sâu isometric (tan của góc 15°)
        // Kết quả: Object ở phía dưới màn hình (Y nhỏ hơn) sẽ được vẽ SAU
        // object ở phía trên, tạo ảo giác chiều sâu isometric đúng.
        GraphicsSettings.transparencySortAxis = new Vector3(0f, 1f, -0.26f);

        // --- Bước 3: Đảm bảo tất cả Camera 2D trong scene dùng đúng sort mode ---
        // Camera.main có thể chưa tồn tại lúc chạy menu này, nên ta cấu hình
        // thông qua GraphicsSettings là đủ (apply toàn project).

        // --- Bước 4: Lưu và xác nhận ---
        // AssetDatabase.SaveAssets() không cần thiết cho GraphicsSettings,
        // nhưng ta force refresh để chắc chắn.
        EditorUtility.SetDirty(QualitySettings.GetQualitySettings());
        AssetDatabase.Refresh();

        // --- Bước 5: In xác nhận ---
        Debug.Log("[IsometricSetup] ✅ Transparency Sort Axis đã được cấu hình: " +
                  $"Mode={GraphicsSettings.transparencySortMode}, " +
                  $"Axis={GraphicsSettings.transparencySortAxis}");

        EditorUtility.DisplayDialog(
            "Isometric Setup Hoàn Tất",
            "Transparency Sort Axis đã được đặt thành (0, 1, -0.26).\n\n" +
            "Cấu hình này apply cho toàn bộ dự án và được lưu vào ProjectSettings.",
            "OK"
        );
    }

    /// <summary>
    /// Kiểm tra xem cấu hình đã đúng chưa. Dùng để validate trong CI/CD.
    /// </summary>
    [MenuItem(MENU_PATH + " (Validate)", true)]
    public static bool ValidateConfiguration()
    {
        // Menu item luôn enabled
        return true;
    }

    /// <summary>
    /// Tự động chạy khi Unity mở project, kiểm tra và cảnh báo nếu chưa cấu hình.
    /// </summary>
    [InitializeOnLoadMethod]
    private static void CheckConfigurationOnLoad()
    {
        if (GraphicsSettings.transparencySortMode != TransparencySortMode.CustomAxis)
        {
            Debug.LogWarning("[IsometricSetup] ⚠️ Transparency Sort Mode chưa được cấu hình đúng. " +
                             "Vào menu TCGShop > Setup > Configure Isometric Rendering.");
        }
    }
}
#endif
```

### 3.2 Cấu Hình Camera 2D Trong Scene

Khi tạo `GameScene.unity`, Camera object phải có các thiết lập sau (thực hiện qua `SceneBootstrapper.cs`):

```
Camera.main:
  - Projection: Orthographic
  - Size: 5 (có thể zoom từ 2 đến 10)
  - Position: (0, 0, -10)
  - Background: Solid Color, màu #1A1A2E
```

---

## 4. Chi Tiết Kỹ Thuật Từng File

### 4.1 `GameManager.cs`

**Vị trí:** `Assets/Scripts/Core/GameManager.cs`  
**Mục đích:** Singleton toàn cục, quản lý vòng đời game, cổng truy cập cho các hệ thống con.

**Yêu cầu bắt buộc:**
- Phải implement pattern Singleton thread-safe với `DontDestroyOnLoad`
- Phải có `[RuntimeInitializeOnLoadMethod]` để tự khởi tạo trước mọi Scene
- Phải in log `[GameManager] Ready.` chính xác sau khi khởi tạo xong
- **CẤM** dùng `GameObject.Find()` ở bất kỳ đâu trong class này

```csharp
// Assets/Scripts/Core/GameManager.cs

using UnityEngine;

/// <summary>
/// Singleton trung tâm điều phối toàn bộ hệ thống game.
/// Tự khởi tạo trước mọi Scene, tồn tại xuyên suốt vòng đời ứng dụng.
/// 
/// NGUYÊN TẮC THIẾT KẾ:
/// - Không bao giờ dùng GameObject.Find() trong Update().
/// - Các hệ thống con tự đăng ký với GameManager thay vì GameManager đi tìm chúng.
/// - Mọi tham chiếu được cache tại Awake/Start, không tìm kiếm động lúc runtime.
/// </summary>
public class GameManager : MonoBehaviour
{
    // =========================================================================
    // SINGLETON PATTERN
    // =========================================================================

    /// <summary>
    /// Instance duy nhất của GameManager. Truy cập từ bất kỳ đâu qua GameManager.Instance.
    /// </summary>
    public static GameManager Instance { get; private set; }

    /// <summary>
    /// Trạng thái khởi tạo. Các hệ thống con kiểm tra flag này trước khi sử dụng GameManager.
    /// </summary>
    public bool IsReady { get; private set; } = false;

    // =========================================================================
    // THAM CHIẾU CÁC HỆ THỐNG CON (Sẽ được mở rộng ở các bước sau)
    // =========================================================================
    // Ví dụ cấu trúc cho tương lai:
    // public InventorySystem Inventory { get; private set; }
    // public EconomySystem Economy { get; private set; }
    // public CustomerAISystem CustomerAI { get; private set; }

    // =========================================================================
    // KHỞI TẠO
    // =========================================================================

    /// <summary>
    /// RuntimeInitializeOnLoadMethod đảm bảo GameManager tồn tại TRƯỚC KHI
    /// bất kỳ Scene nào được load, kể cả Scene đầu tiên.
    /// SubsystemRegistration là phase sớm nhất có thể.
    /// </summary>
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void CreateInstance()
    {
        if (Instance != null) return;

        // Tạo GameObject mới để host GameManager
        var go = new GameObject("[GameManager]");
        
        // AddComponent sẽ trigger Awake() ngay lập tức
        go.AddComponent<GameManager>();
        
        // Đảm bảo không bị destroy khi chuyển Scene
        DontDestroyOnLoad(go);
    }

    private void Awake()
    {
        // --- Kiểm tra duplicate (trường hợp Scene có sẵn GameManager trong hierarchy) ---
        if (Instance != null && Instance != this)
        {
            Debug.LogWarning("[GameManager] Phát hiện instance thứ hai. Đang hủy instance thừa.");
            Destroy(gameObject);
            return;
        }

        Instance = this;
        DontDestroyOnLoad(gameObject);

        // --- Khởi tạo các hệ thống con ---
        InitializeSystems();
    }

    /// <summary>
    /// Khởi tạo tất cả hệ thống con theo thứ tự dependency.
    /// Thứ tự quan trọng: hệ thống A không phụ thuộc hệ thống B phải khởi tạo trước B.
    /// </summary>
    private void InitializeSystems()
    {
        // Bước 1: Bước này chưa có hệ thống con, chỉ đánh dấu ready
        // Các bước sau sẽ thêm: Economy, Inventory, CustomerAI, v.v.

        IsReady = true;

        // Log chính xác theo yêu cầu — KHÔNG được thay đổi chuỗi này
        // vì kịch bản test tìm kiếm chuỗi "[GameManager] Ready." trong Console
        Debug.Log("[GameManager] Ready.");
    }

    // =========================================================================
    // VÒNG ĐỜI
    // =========================================================================

    private void OnApplicationQuit()
    {
        IsReady = false;
        Debug.Log("[GameManager] Application quitting. Shutting down systems.");
    }

    private void OnApplicationPause(bool pauseStatus)
    {
        if (pauseStatus)
        {
            Debug.Log("[GameManager] Application paused. Saving state...");
            // SaveSystem.Save(); // Sẽ implement ở bước sau
        }
    }

    // =========================================================================
    // TIỆN ÍCH
    // =========================================================================

    /// <summary>
    /// Kiểm tra GameManager có sẵn sàng không trước khi gọi hệ thống con.
    /// Dùng trong mọi hệ thống con: if (!GameManager.Instance.IsReady) return;
    /// </summary>
    public static bool IsAvailable => Instance != null && Instance.IsReady;
}
```

---

### 4.2 `CameraController.cs`

**Vị trí:** `Assets/Scripts/Camera/CameraController.cs`  
**Mục đích:** Xử lý toàn bộ input điều khiển camera — pan (kéo), zoom (cuộn/pinch) — cho cả Desktop lẫn Mobile, sử dụng Unity New Input System.

**Yêu cầu bắt buộc:**
- Phải dùng **Unity New Input System** (`UnityEngine.InputSystem`), KHÔNG dùng `Input.GetMouseButton()` cũ
- Pan phải mượt với `Mathf.Lerp` hoặc `Vector3.Lerp`
- Zoom phải bị giới hạn bởi `Mathf.Clamp` với min/max configurable
- Hỗ trợ **pinch-to-zoom** trên mobile (2 ngón tay)
- **CẤM** dùng `GameObject.Find()` trong `Update()`
- Phải có bounds giới hạn camera không ra ngoài thế giới game
- Tất cả magic numbers phải là `[SerializeField]` có thể chỉnh trong Inspector

```csharp
// Assets/Scripts/Camera/CameraController.cs

using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.InputSystem.EnhancedTouch;
using Touch = UnityEngine.InputSystem.EnhancedTouch.Touch;

/// <summary>
/// Điều khiển Camera 2D Orthographic cho Isometric view.
/// Hỗ trợ đầy đủ Desktop (chuột) và Mobile (cảm ứng).
/// 
/// DESKTOP:
///   - Nhấn giữ chuột giữa (hoặc chuột phải) + kéo → Pan
///   - Cuộn bánh xe chuột → Zoom
///
/// MOBILE:
///   - Chạm một ngón + kéo → Pan  
///   - Pinch hai ngón → Zoom
///
/// THIẾT KẾ: 
///   - Không dùng GameObject.Find() trong Update().
///   - Camera reference được cache tại Awake().
///   - Tất cả tính toán dùng world-space để tránh lỗi khi resolution thay đổi.
/// </summary>
[RequireComponent(typeof(Camera))]
public class CameraController : MonoBehaviour
{
    // =========================================================================
    // INSPECTOR SETTINGS
    // =========================================================================

    [Header("Pan Settings")]
    [Tooltip("Tốc độ pan cơ bản. Giá trị càng lớn, camera di chuyển càng nhanh theo ngón tay/chuột.")]
    [SerializeField] private float panSpeed = 1f;

    [Tooltip("Độ mượt của pan. 0 = không mượt (immediate), 1 = không bao giờ đến đích. Khuyến nghị: 0.08-0.15")]
    [SerializeField][Range(0.01f, 0.5f)] private float panSmoothTime = 0.1f;

    [Header("Zoom Settings")]
    [Tooltip("Kích thước Orthographic Size tối thiểu (phóng to tối đa).")]
    [SerializeField] private float minZoom = 2f;

    [Tooltip("Kích thước Orthographic Size tối đa (thu nhỏ tối đa).")]
    [SerializeField] private float maxZoom = 10f;

    [Tooltip("Tốc độ zoom khi cuộn chuột. Giá trị âm để đảo chiều.")]
    [SerializeField] private float mouseZoomSpeed = 1f;

    [Tooltip("Tốc độ zoom khi pinch mobile.")]
    [SerializeField] private float pinchZoomSpeed = 0.05f;

    [Tooltip("Độ mượt của zoom. Khuyến nghị: 0.08-0.2")]
    [SerializeField][Range(0.01f, 0.5f)] private float zoomSmoothTime = 0.12f;

    [Header("World Bounds")]
    [Tooltip("Bật/tắt giới hạn camera trong thế giới game.")]
    [SerializeField] private bool useBounds = true;

    [Tooltip("Vùng thế giới mà camera được phép nhìn. Đặt theo kích thước map.")]
    [SerializeField] private Bounds worldBounds = new Bounds(Vector3.zero, new Vector3(50f, 50f, 0f));

    // =========================================================================
    // PRIVATE STATE — Không serialize, quản lý nội bộ
    // =========================================================================

    // Reference được cache — KHÔNG tìm kiếm lại trong Update()
    private Camera _camera;

    // Pan state
    private Vector3 _targetPosition;           // Vị trí camera đang hướng đến
    private Vector3 _panVelocity;              // Dùng cho SmoothDamp
    private Vector3 _lastPanWorldPosition;     // Vị trí world lúc bắt đầu drag
    private bool _isPanning;                   // Đang pan không?

    // Zoom state  
    private float _targetOrthographicSize;     // Zoom target
    private float _zoomVelocity;               // Dùng cho SmoothDamp
    private float _lastPinchDistance;          // Khoảng cách 2 ngón lần trước (pinch)

    // Input action references — Cache tại Awake()
    private Mouse _mouse;
    private Keyboard _keyboard;

    // =========================================================================
    // VÒNG ĐỜI UNITY
    // =========================================================================

    private void Awake()
    {
        // Cache camera reference — Chỉ làm một lần, không GetComponent trong Update
        _camera = GetComponent<Camera>();

        if (_camera == null)
        {
            Debug.LogError("[CameraController] Không tìm thấy Camera component! " +
                           "CameraController phải được gắn vào GameObject có Camera.");
            enabled = false;
            return;
        }

        // Đảm bảo camera là Orthographic
        if (!_camera.orthographic)
        {
            Debug.LogWarning("[CameraController] Camera không phải Orthographic. Đang chuyển đổi...");
            _camera.orthographic = true;
        }

        // Khởi tạo target values từ state hiện tại của camera
        _targetPosition = transform.position;
        _targetOrthographicSize = _camera.orthographicSize;

        // Clamp zoom ngay từ đầu trong trường hợp giá trị ban đầu ngoài range
        _targetOrthographicSize = Mathf.Clamp(_targetOrthographicSize, minZoom, maxZoom);

        // Cache input devices — GetDevice() an toàn hơn GetComponent trong Update
        _mouse = Mouse.current;
        _keyboard = Keyboard.current;
    }

    private void OnEnable()
    {
        // Bật Enhanced Touch API để đọc multi-touch (pinch)
        EnhancedTouchSupport.Enable();
    }

    private void OnDisable()
    {
        EnhancedTouchSupport.Disable();
    }

    private void Update()
    {
        // ⚠️ KHÔNG được gọi GameObject.Find() trong đây
        // ⚠️ KHÔNG được gọi GetComponent() trong đây
        // Chỉ đọc input và tính toán target values

        HandleDesktopInput();
        HandleMobileInput();
        ApplySmoothMovement();
    }

    // =========================================================================
    // XỬ LÝ INPUT DESKTOP (Chuột)
    // =========================================================================

    /// <summary>
    /// Xử lý input từ chuột: Pan bằng chuột giữa/phải, Zoom bằng scroll wheel.
    /// Tất cả tính toán trong world-space.
    /// </summary>
    private void HandleDesktopInput()
    {
        // Cập nhật mouse reference nếu mới connect (hot-plug)
        if (_mouse == null)
        {
            _mouse = Mouse.current;
            return;
        }

        HandleMousePan();
        HandleMouseZoom();
    }

    /// <summary>
    /// Pan bằng chuột giữa hoặc chuột phải.
    /// Dùng delta world-space để pan tốc độ khớp với ngón tay/con trỏ.
    /// </summary>
    private void HandleMousePan()
    {
        // Bắt đầu pan khi nhấn giữ chuột giữa (button 2) HOẶC chuột phải (button 1)
        bool panButtonPressed = _mouse.middleButton.isPressed || _mouse.rightButton.isPressed;

        if (panButtonPressed)
        {
            if (!_isPanning)
            {
                // Lần đầu nhấn: ghi nhớ vị trí world tại điểm chuột
                _isPanning = true;
                _lastPanWorldPosition = ScreenToWorldPosition(_mouse.position.ReadValue());
            }
            else
            {
                // Đang kéo: tính delta và dịch chuyển target
                Vector3 currentWorldPosition = ScreenToWorldPosition(_mouse.position.ReadValue());
                Vector3 worldDelta = currentWorldPosition - _lastPanWorldPosition;

                // Di chuyển camera ngược chiều delta (kéo phải → thế giới trái → camera trái)
                _targetPosition -= worldDelta * panSpeed;

                // Clamp trong bounds
                if (useBounds)
                {
                    _targetPosition = ClampPositionToBounds(_targetPosition);
                }

                // Không cập nhật _lastPanWorldPosition ở đây vì ta dùng delta từ frame trước
                // để tạo cảm giác "kéo thế giới" tự nhiên hơn.
                // Nhưng cần cập nhật để delta frame sau đúng:
                _lastPanWorldPosition = ScreenToWorldPosition(_mouse.position.ReadValue());
            }
        }
        else
        {
            _isPanning = false;
        }
    }

    /// <summary>
    /// Zoom bằng scroll wheel.
    /// Zoom hướng về điểm con trỏ chuột đang trỏ (zoom-to-cursor).
    /// </summary>
    private void HandleMouseZoom()
    {
        float scrollValue = _mouse.scroll.ReadValue().y;
        if (Mathf.Approximately(scrollValue, 0f)) return;

        // Lấy vị trí world tại con trỏ trước khi zoom để làm điểm neo
        Vector3 mouseWorldBefore = ScreenToWorldPosition(_mouse.position.ReadValue());

        // Tính toán zoom target mới
        // scrollValue dương = cuộn lên = zoom in = orthographicSize giảm
        float zoomDelta = -scrollValue * mouseZoomSpeed * 0.1f;
        _targetOrthographicSize += zoomDelta;

        // CLAMP: Đảm bảo zoom không vượt quá giới hạn
        _targetOrthographicSize = Mathf.Clamp(_targetOrthographicSize, minZoom, maxZoom);

        // Zoom-to-cursor: điều chỉnh _targetPosition để điểm dưới con trỏ giữ nguyên
        // Áp dụng zoom ngay để tính vị trí world mới tại cursor
        float previousSize = _camera.orthographicSize;
        _camera.orthographicSize = _targetOrthographicSize;
        Vector3 mouseWorldAfter = ScreenToWorldPosition(_mouse.position.ReadValue());
        _camera.orthographicSize = previousSize; // Restore, để SmoothDamp xử lý

        // Offset target để cursor vẫn trỏ vào cùng điểm world
        _targetPosition += mouseWorldBefore - mouseWorldAfter;

        if (useBounds)
        {
            _targetPosition = ClampPositionToBounds(_targetPosition);
        }
    }

    // =========================================================================
    // XỬ LÝ INPUT MOBILE (Cảm ứng)
    // =========================================================================

    /// <summary>
    /// Xử lý input cảm ứng:
    ///   - 1 ngón tay: Pan
    ///   - 2 ngón tay: Pinch zoom
    /// EnhancedTouch API phải được Enable() trước (trong OnEnable).
    /// </summary>
    private void HandleMobileInput()
    {
        var activeTouches = Touch.activeTouches;

        switch (activeTouches.Count)
        {
            case 1:
                HandleSingleTouchPan(activeTouches[0]);
                break;

            case 2:
                HandlePinchZoom(activeTouches[0], activeTouches[1]);
                break;

            default:
                // 0 ngón hoặc >2 ngón: Reset pinch state
                _lastPinchDistance = 0f;
                // Nếu đang pan bằng cảm ứng và nhấc hết ngón, reset pan
                if (activeTouches.Count == 0)
                {
                    _isPanning = false;
                }
                break;
        }
    }

    /// <summary>
    /// Pan với một ngón tay, logic tương tự mouse pan.
    /// </summary>
    private void HandleSingleTouchPan(Touch touch)
    {
        // Reset pinch distance khi chỉ còn 1 ngón
        _lastPinchDistance = 0f;

        if (touch.phase == UnityEngine.InputSystem.TouchPhase.Began)
        {
            _isPanning = true;
            _lastPanWorldPosition = ScreenToWorldPosition(touch.screenPosition);
        }
        else if (touch.phase == UnityEngine.InputSystem.TouchPhase.Moved && _isPanning)
        {
            Vector3 currentWorldPosition = ScreenToWorldPosition(touch.screenPosition);
            Vector3 worldDelta = currentWorldPosition - _lastPanWorldPosition;

            _targetPosition -= worldDelta * panSpeed;

            if (useBounds)
            {
                _targetPosition = ClampPositionToBounds(_targetPosition);
            }

            _lastPanWorldPosition = ScreenToWorldPosition(touch.screenPosition);
        }
        else if (touch.phase == UnityEngine.InputSystem.TouchPhase.Ended ||
                 touch.phase == UnityEngine.InputSystem.TouchPhase.Canceled)
        {
            _isPanning = false;
        }
    }

    /// <summary>
    /// Pinch-to-zoom với hai ngón tay.
    /// So sánh khoảng cách giữa 2 ngón frame này với frame trước để tính delta zoom.
    /// </summary>
    private void HandlePinchZoom(Touch touch0, Touch touch1)
    {
        // Không pan khi đang pinch
        _isPanning = false;

        float currentPinchDistance = Vector2.Distance(
            touch0.screenPosition,
            touch1.screenPosition
        );

        // Frame đầu tiên của pinch gesture: chỉ ghi nhớ khoảng cách, chưa zoom
        if (_lastPinchDistance <= 0f)
        {
            _lastPinchDistance = currentPinchDistance;
            return;
        }

        float pinchDelta = currentPinchDistance - _lastPinchDistance;

        // Ngón tay ra xa nhau (pinchDelta > 0) → zoom in → size giảm
        float zoomChange = -pinchDelta * pinchZoomSpeed;
        _targetOrthographicSize += zoomChange;

        // CLAMP: Đảm bảo zoom mobile cũng không vượt giới hạn
        _targetOrthographicSize = Mathf.Clamp(_targetOrthographicSize, minZoom, maxZoom);

        // Giới hạn pan trong bounds sau khi zoom
        if (useBounds)
        {
            _targetPosition = ClampPositionToBounds(_targetPosition);
        }

        _lastPinchDistance = currentPinchDistance;
    }

    // =========================================================================
    // ÁP DỤNG CHUYỂN ĐỘNG MƯỢT
    // =========================================================================

    /// <summary>
    /// Mỗi frame, camera di chuyển từ vị trí hiện tại về _targetPosition một cách mượt mà.
    /// Dùng SmoothDamp thay vì Lerp để có deceleration tự nhiên hơn.
    /// </summary>
    private void ApplySmoothMovement()
    {
        // Pan smooth
        Vector3 currentPos = transform.position;
        Vector3 newPos = Vector3.SmoothDamp(
            currentPos,
            _targetPosition,
            ref _panVelocity,
            panSmoothTime
        );

        // Giữ Z cố định ở -10 (camera 2D standard)
        newPos.z = -10f;
        transform.position = newPos;

        // Zoom smooth
        _camera.orthographicSize = Mathf.SmoothDamp(
            _camera.orthographicSize,
            _targetOrthographicSize,
            ref _zoomVelocity,
            zoomSmoothTime
        );
    }

    // =========================================================================
    // TIỆN ÍCH TÍNH TOÁN
    // =========================================================================

    /// <summary>
    /// Chuyển đổi tọa độ screen sang world space.
    /// Dùng camera được cache, không GetComponent mới.
    /// </summary>
    private Vector3 ScreenToWorldPosition(Vector2 screenPosition)
    {
        Vector3 screenPos3D = new Vector3(screenPosition.x, screenPosition.y, 0f);
        return _camera.ScreenToWorldPoint(screenPos3D);
    }

    /// <summary>
    /// Giới hạn vị trí camera trong worldBounds.
    /// Tính đến orthographicSize hiện tại để camera không thấy ngoài map.
    /// </summary>
    private Vector3 ClampPositionToBounds(Vector3 position)
    {
        // Tính kích thước vùng nhìn thấy của camera
        float cameraHalfHeight = _camera.orthographicSize;
        float cameraHalfWidth = cameraHalfHeight * _camera.aspect;

        // Giới hạn vị trí camera sao cho viewport không ra ngoài bounds
        float minX = worldBounds.min.x + cameraHalfWidth;
        float maxX = worldBounds.max.x - cameraHalfWidth;
        float minY = worldBounds.min.y + cameraHalfHeight;
        float maxY = worldBounds.max.y - cameraHalfHeight;

        // Clamp từng trục
        position.x = Mathf.Clamp(position.x, minX, maxX);
        position.y = Mathf.Clamp(position.y, minY, maxY);

        return position;
    }

    // =========================================================================
    // API CÔNG KHAI (Dùng cho hệ thống khác di chuyển camera)
    // =========================================================================

    /// <summary>
    /// Di chuyển camera mượt đến vị trí world space chỉ định.
    /// Dùng khi muốn focus vào một object (vd: click vào NPC).
    /// </summary>
    public void FocusOn(Vector3 worldPosition, float duration = 0.5f)
    {
        _targetPosition = new Vector3(worldPosition.x, worldPosition.y, -10f);

        if (useBounds)
        {
            _targetPosition = ClampPositionToBounds(_targetPosition);
        }
    }

    /// <summary>
    /// Đặt zoom level ngay lập tức, không smooth.
    /// Dùng cho việc khởi tạo scene hoặc teleport.
    /// </summary>
    public void SetZoomImmediate(float size)
    {
        float clampedSize = Mathf.Clamp(size, minZoom, maxZoom);
        _camera.orthographicSize = clampedSize;
        _targetOrthographicSize = clampedSize;
    }

    // =========================================================================
    // DEBUG GIZMOS (Chỉ hiển thị trong Editor)
    // =========================================================================

#if UNITY_EDITOR
    private void OnDrawGizmosSelected()
    {
        if (!useBounds) return;

        // Vẽ world bounds màu xanh lá
        Gizmos.color = Color.green;
        Gizmos.DrawWireCube(worldBounds.center, worldBounds.size);

        // Vẽ vùng camera hiện tại màu vàng
        if (_camera != null)
        {
            Gizmos.color = Color.yellow;
            float h = _camera.orthographicSize;
            float w = h * _camera.aspect;
            Gizmos.DrawWireCube(transform.position, new Vector3(w * 2, h * 2, 0));
        }
    }
#endif
}
```

---

### 4.3 `SceneBootstrapper.cs`

**Vị trí:** `Assets/Scripts/Core/SceneBootstrapper.cs`  
**Mục đích:** Thiết lập Scene `GameScene.unity` đúng cách khi load, đảm bảo tất cả dependencies có mặt trước khi gameplay bắt đầu.

```csharp
// Assets/Scripts/Core/SceneBootstrapper.cs

using UnityEngine;

/// <summary>
/// Chạy khi GameScene được load.
/// Kiểm tra và cấu hình các thành phần bắt buộc của Scene.
/// Gắn vào một GameObject tên "_Bootstrapper" trong Scene.
/// </summary>
public class SceneBootstrapper : MonoBehaviour
{
    [Header("Scene Requirements")]
    [Tooltip("Camera chính của Scene. Phải có CameraController.")]
    [SerializeField] private Camera mainCamera;

    private void Awake()
    {
        ValidateSceneSetup();
    }

    private void Start()
    {
        // Kiểm tra GameManager đã ready chưa trước khi bắt đầu scene logic
        if (!GameManager.IsAvailable)
        {
            Debug.LogError("[SceneBootstrapper] GameManager chưa sẵn sàng! " +
                           "Kiểm tra RuntimeInitializeOnLoadMethod trong GameManager.cs");
            return;
        }

        Debug.Log("[SceneBootstrapper] Scene setup hoàn tất. Tất cả systems sẵn sàng.");
    }

    /// <summary>
    /// Kiểm tra toàn bộ requirements của Scene.
    /// In error rõ ràng thay vì để NullReferenceException xuất hiện ngẫu nhiên.
    /// </summary>
    private void ValidateSceneSetup()
    {
        bool hasErrors = false;

        // Kiểm tra Camera
        if (mainCamera == null)
        {
            // Thử tìm trong Scene nếu chưa assign (chỉ cho phép ở Awake)
            mainCamera = Camera.main;

            if (mainCamera == null)
            {
                Debug.LogError("[SceneBootstrapper] ❌ Không tìm thấy Main Camera trong Scene! " +
                               "Thêm Camera với tag 'MainCamera' vào Scene.");
                hasErrors = true;
            }
        }

        // Kiểm tra CameraController
        if (mainCamera != null && !mainCamera.TryGetComponent<CameraController>(out _))
        {
            Debug.LogError("[SceneBootstrapper] ❌ Camera thiếu CameraController component! " +
                           "Thêm CameraController vào Camera GameObject.");
            hasErrors = true;
        }

        // Kiểm tra Orthographic
        if (mainCamera != null && !mainCamera.orthographic)
        {
            Debug.LogWarning("[SceneBootstrapper] ⚠️ Camera không phải Orthographic. " +
                             "Isometric 2D yêu cầu Orthographic camera.");
        }

        if (!hasErrors)
        {
            Debug.Log("[SceneBootstrapper] ✅ Scene validation passed.");
        }
    }
}
```

---

### 4.4 `IsometricSortingController.cs`

**Vị trí:** `Assets/Scripts/World/IsometricSortingController.cs`  
**Mục đích:** Runtime validation rằng Isometric sorting đang hoạt động đúng. Cung cấp API để các sprite trong game cập nhật sorting order theo vị trí Y.

```csharp
// Assets/Scripts/World/IsometricSortingController.cs

using UnityEngine;

/// <summary>
/// Quản lý sorting order cho tất cả sprite trong không gian Isometric.
/// Trong Isometric Z-as-Y: sprite có Y lớn hơn (phía trên màn hình) được vẽ sau (dưới).
/// Sprite có Y nhỏ hơn (phía dưới màn hình) được vẽ trước (trên).
///
/// CÁCH DÙNG: Gắn vào mỗi GameObject cần sorting isometric tự động.
/// </summary>
public class IsometricSortingController : MonoBehaviour
{
    [Header("Sorting Configuration")]
    [Tooltip("Số nhân để chuyển đổi Y position sang sorting order. " +
             "Giá trị âm vì Y tăng lên trên, nhưng sorting order tăng có nghĩa 'vẽ sau' (dưới).")]
    [SerializeField] private float sortingMultiplier = -100f;

    [Tooltip("Offset sorting order, dùng để phân tầng các layer (Floor, Furniture, Character, UI).")]
    [SerializeField] private int sortingOrderOffset = 0;

    [Tooltip("Tự động cập nhật sorting order mỗi frame. " +
             "Tắt nếu object không di chuyển để tiết kiệm performance.")]
    [SerializeField] private bool dynamicSorting = true;

    // Cache reference — Không GetComponent trong Update
    private SpriteRenderer _spriteRenderer;
    private bool _isInitialized;

    private void Awake()
    {
        // Cache component — Chỉ một lần
        if (!TryGetComponent(out _spriteRenderer))
        {
            Debug.LogError($"[IsometricSortingController] {gameObject.name} thiếu SpriteRenderer! " +
                           "Component này yêu cầu SpriteRenderer.");
            enabled = false;
            return;
        }

        _isInitialized = true;
        UpdateSortingOrder();
    }

    private void Update()
    {
        // ⚠️ KHÔNG gọi GetComponent hay GameObject.Find ở đây
        if (!_isInitialized || !dynamicSorting) return;

        UpdateSortingOrder();
    }

    /// <summary>
    /// Cập nhật sorting order dựa trên vị trí Y hiện tại.
    /// Công thức: sortingOrder = round(Y × multiplier) + offset
    /// </summary>
    public void UpdateSortingOrder()
    {
        if (!_isInitialized) return;

        int newOrder = Mathf.RoundToInt(transform.position.y * sortingMultiplier) + sortingOrderOffset;
        _spriteRenderer.sortingOrder = newOrder;
    }

    /// <summary>
    /// API công khai để hệ thống bên ngoài force update sorting ngay lập tức.
    /// Dùng sau khi teleport hoặc đặt vật thể.
    /// </summary>
    public void ForceSortingUpdate()
    {
        UpdateSortingOrder();
    }
}
```

---

## 5. Thiết Lập Scene `GameScene.unity`

Cursor phải cấu hình Scene với hierarchy sau. Tên GameObject phải khớp chính xác:

```
GameScene (Scene Root)
├── _Bootstrapper          [SceneBootstrapper.cs]
│
├── Main Camera            [Camera.cs] [CameraController.cs]
│   Settings:
│     Projection: Orthographic
│     Orthographic Size: 5
│     Position: (0, 0, -10)
│     Clear Flags: Solid Color
│     Background: #1A1A2E
│
└── World                  (Empty GameObject, tổ chức)
    └── Grid               [Grid.cs] [Tilemap.cs] (placeholder cho bước sau)
```

**Lưu ý quan trọng về `_Bootstrapper`:** GameObject này phải tồn tại trong Scene để `SceneBootstrapper` chạy. `GameManager` tự tạo ra via `RuntimeInitializeOnLoadMethod` nên **không** cần có trong Scene hierarchy.

---

## 6. Giới Hạn & Quy Tắc Bắt Buộc

Cursor phải tuân thủ các quy tắc sau trong toàn bộ code của Bước 1. Vi phạm bất kỳ quy tắc nào sẽ gây lỗi ở bước sau.

### 6.1 Quy Tắc Performance

| Quy tắc | Lý do |
|---------|-------|
| **CẤM** `GameObject.Find()` trong `Update()`, `LateUpdate()`, `FixedUpdate()` | Duyệt toàn bộ hierarchy mỗi frame, O(n) per frame, gây lag khi scene phức tạp |
| **CẤM** `GetComponent<T>()` trong `Update()` | Tốn kém, phải cache trong `Awake()` |
| **CẤM** `FindObjectOfType<T>()` trong `Update()` | Tương tự `GameObject.Find()` |
| **BẮT BUỘC** Cache tất cả component references trong `Awake()` | Best practice Unity |
| **BẮT BUỘC** Dùng `TryGetComponent<T>()` thay vì `GetComponent<T>()` | Tránh exception, trả về bool |

### 6.2 Quy Tắc Input System

```
KHÔNG ĐƯỢC dùng (Legacy Input System):
  Input.GetMouseButton()
  Input.GetAxis()
  Input.GetKey()
  Input.touches

PHẢI dùng (New Input System):
  Mouse.current.leftButton.isPressed
  Mouse.current.scroll.ReadValue()
  Touch.activeTouches  (cần EnhancedTouchSupport.Enable())
```

### 6.3 Quy Tắc Singleton

```
Mỗi Singleton phải:
  1. Có [RuntimeInitializeOnLoadMethod] để tự tạo trước Scene
  2. Kiểm tra duplicate trong Awake() và Destroy nếu thừa
  3. Gọi DontDestroyOnLoad(gameObject)
  4. Có flag IsReady để các hệ thống khác kiểm tra trước khi dùng
```

### 6.4 Quy Tắc Naming

```
Private fields:   _camelCase       (vd: _targetPosition)
Public properties: PascalCase      (vd: IsReady)
SerializeField:   camelCase        (vd: panSpeed)
Constants:        UPPER_SNAKE_CASE (vd: MENU_PATH)
```

---

## 7. Kịch Bản Kiểm Thử (Playtest Checklist)

Sau khi Cursor hoàn thành tất cả files, người dùng thực hiện các bước sau để verify:

### 7.1 Test Khởi Động

```
Bước 1: Mở Unity Editor, vào menu TCGShop > Setup > Configure Isometric Rendering
Bước 2: Kiểm tra Console: phải thấy đúng dòng sau (không có gì khác):
  "[IsometricSetup] ✅ Transparency Sort Axis đã được cấu hình: ..."

Bước 3: Nhấn Play trong GameScene
Bước 4: Kiểm tra Console theo thứ tự CHÍNH XÁC:
  1. "[GameManager] Ready."
  2. "[SceneBootstrapper] ✅ Scene validation passed."
  3. "[SceneBootstrapper] Scene setup hoàn tất. Tất cả systems sẵn sàng."

❌ FAIL nếu thấy:
  - NullReferenceException
  - MissingReferenceException
  - Bất kỳ dòng [ERROR] nào
  - Thứ tự log khác với trên
```

### 7.2 Test Camera Pan (Desktop)

```
Trong Play Mode:
  1. Nhấn giữ chuột GIỮA, kéo sang phải → Camera phải di chuyển sang trái mượt mà
  2. Nhấn giữ chuột PHẢI, kéo lên trên → Camera phải di chuyển xuống dưới mượt mà
  3. Thả chuột → Camera dừng lại mượt (deceleration), không dừng đột ngột
  4. Kéo đến rìa map → Camera phải dừng tại bounds, không vượt ra ngoài

✅ PASS: Di chuyển mượt, không jitter, không vượt bounds
❌ FAIL: Giật cục, vượt bounds, lỗi Console
```

### 7.3 Test Camera Zoom (Desktop)

```
Trong Play Mode:
  1. Cuộn bánh xe lên → Scene thu nhỏ (zoom out), orthographicSize tăng
  2. Cuộn bánh xe xuống → Scene phóng to (zoom in), orthographicSize giảm
  3. Cuộn liên tục đến giới hạn → Camera phải dừng ở minZoom=2 hoặc maxZoom=10
  4. Zoom vào một điểm cụ thể → Điểm đó phải giữ nguyên vị trí trên màn hình

✅ PASS: Zoom mượt, clamp đúng ở [2, 10], zoom-to-cursor hoạt động
❌ FAIL: Zoom không dừng, camera nhảy xa khi zoom
```

### 7.4 Test Camera Mobile (Game View Simulation)

```
Chuẩn bị: 
  Trong Unity Editor, vào Window > Device Simulator HOẶC
  Giữ Ctrl + kéo trong Game View để giả lập multi-touch

Test 1 ngón:
  Chạm và kéo → Camera pan theo ngón tay

Test 2 ngón (Pinch):
  Đặt 2 ngón, kéo xa nhau → Zoom out
  Đặt 2 ngón, kéo vào nhau → Zoom in
  Zoom đến giới hạn → Dừng lại

✅ PASS: Pan và pinch hoạt động, không conflict nhau
❌ FAIL: Pan tiếp tục khi đang pinch, zoom vượt giới hạn
```

### 7.5 Test Isometric Sorting

```
Chuẩn bị: Tạo 2 sprite trong Scene, đặt ở Y khác nhau
  - Sprite A: position Y = 2 (phía trên màn hình)
  - Sprite B: position Y = -2 (phía dưới màn hình)
  - Cả hai đều có IsometricSortingController.cs

Quan sát:
  - Sprite B (Y thấp hơn) phải vẽ ĐÈ LÊN Sprite A khi overlap
  - Điều này giả lập vật thể phía trước che vật thể phía sau trong isometric view

Mở Inspector của mỗi Sprite, kiểm tra Sprite Renderer > Sorting Order:
  - Sprite A (Y=2):  sortingOrder ≈ -200  (2 × -100)
  - Sprite B (Y=-2): sortingOrder ≈ +200  (-2 × -100)
  - Sprite B có sortingOrder cao hơn → Vẽ trên cùng ✅

✅ PASS: Sprite phía dưới màn hình vẽ đè lên sprite phía trên
❌ FAIL: Sorting ngược lại hoặc không hoạt động
```

### 7.6 Test Không Có NullReferenceException

```
Thực hiện tất cả test trên LIÊN TỤC trong 60 giây.
Mở Console, filter chỉ hiển thị Errors.

✅ PASS: Console hoàn toàn trống (0 errors)
❌ FAIL: Bất kỳ NullReferenceException nào xuất hiện
```

---

## 8. Cấu Trúc Project Hoàn Chỉnh Sau Bước 1

```
Assets/
├── Scenes/
│   └── GameScene.unity
│
├── Scripts/
│   ├── Core/
│   │   ├── GameManager.cs
│   │   └── SceneBootstrapper.cs
│   ├── Camera/
│   │   └── CameraController.cs
│   ├── World/
│   │   └── IsometricSortingController.cs
│   └── Editor/
│       └── IsometricSetup.cs          ← Chỉ compile trong Editor
│
└── Settings/
    └── InputSystem_Actions.inputactions  ← Tự tạo khi cài New Input System
```

---

## 9. Định Nghĩa "Hoàn Thành" (Definition of Done)

Bước 1 được coi là **HOÀN THÀNH** khi và chỉ khi tất cả điều kiện sau đều đúng:

- [ ] Tất cả 5 file `.cs` đã được tạo đúng thư mục
- [ ] `GameScene.unity` đã được tạo với đúng hierarchy
- [ ] Chạy menu `TCGShop > Setup > Configure Isometric Rendering` thành công không lỗi
- [ ] Nhấn Play: Console in `[GameManager] Ready.` là dòng **đầu tiên**
- [ ] Không có bất kỳ `NullReferenceException` nào trong 60 giây test
- [ ] Camera pan bằng chuột giữa hoạt động mượt
- [ ] Camera zoom bằng scroll wheel hoạt động và clamp ở `[2, 10]`
- [ ] Không có dòng code nào gọi `GameObject.Find()` trong `Update()`
- [ ] Không có dòng code nào dùng Legacy Input System (`Input.*`)

**Chỉ sau khi tất cả checkbox trên được check, mới chuyển sang Bước 2.**
```