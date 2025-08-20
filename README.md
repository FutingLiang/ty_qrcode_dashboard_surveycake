Taoyuan 公車站點資料集說明

本檔案 taoyuan_citybus_stops_OpenData_with_operator_lookup.xlsx 是依據「桃園市政府交通局」公開資料所生成的公車站牌資料集，包含桃園市所有市區公車路線的站序、站牌名稱、地址、座標及對應公車業者資訊。本 README 將詳述資料來源、處理流程、欄位意義與使用方式，並附上參考來源。

資料來源

本資料集主要使用桃園市政府資料開放平台提供的公車路線靜態資訊與公車動態資訊兩個來源進行彙整：

公車路線資訊 (路線靜態資料集)：該資料集提供每條市區公車路線的「路線編號(Route)」、「中文名稱(nameZh)」、「起點起訖站(departureZh, destinationZh)」等基本資訊，並包含路線圖與站牌PDF連結（stopAddrUrl）。資料由桃園市交通局提供，並以即時方式更新
twn.databasesets.com
twn.databasesets.com
。例如，從資料描述可知此資料集中欄位包含路線代號(ID)、業者代號(ProviderID)、路線中文名稱(nameZh)等
twn.databasesets.com
。

公車動態資訊 (行駛業者資料)：該資料集主要記錄公車即時動態，包括車輛位置、營運狀態等，同時含有「業者代號(OperatorID, ProviderID)」與「業者中文名稱(OperatorName, ProviderName)」等欄位。此資料集用於建立「業者對照表」，將各路線之業者代號對應到對應的業者名稱。此對照表由程式函式 build_operator_lookup 以多筆資料匯總方式建立（詳見下文）。同樣由交通局提供。

參考政府資料平台說明，此類資料由桃園市交通局發布，屬政府資料開放授權條款-第1版
twn.databasesets.com
。資料透過網路服務 API 存取，格式多為 JSON 或 XML。
twn.databasesets.com
說明「ProviderID(業者代號) 對應動態資料API中的ID」，可知路線靜態資料與動態資料之間可透過此ID欄位關聯。

處理流程

本資料集由 Python 腳本整合上述來源並解析站點PDF生成，主要流程如下：

建立業者對照表：使用 build_operator_lookup 函式，從「桃園市市區公車動態」資料集（OP_BASE_URL）以分頁方式拉取所有記錄，取得每筆資料中的業者代號與業者名稱欄位。此函式對多種欄位名稱提供容錯 (如 OperatorID、ProviderID、ProviderName 等)，將同一代號的不同版本名稱保留較長者（視為較完整名稱）。如果一個業者代號在多筆資料中有不同名稱，會保留字串較長的名稱。建立好的字典將用於後續替換業者名稱。

多值業者處理：若資料中的業者代號欄位 (如 "1001,1002") 包含多個編號，使用正則 [,\uFF0C/;|\uFF5C、]+ 進行切分，對應到各業者名稱後再以中文頓號「、」連接。例如，編號串 "1001,1002" 可能被映射為 "桃園客運、統聯客運"。函式會保留原始代號串中無法對應的部分，確保不丟失資訊。

輸出結果：函式打印對照表條數與示例，如「業者對照表筆數：X；前幾筆： [(代號, 名稱), ...]」。成功建立的對照表將用於替換路線資料中的業者名稱欄位。

抓取路線列表：使用 fetch_pages 函式對「公車路線資訊」API（ROUTE_BASE_URL）分頁請求，拉取所有路線的靜態資料。每筆資料行主要欄位包括路線編號(route 或 masterrouteno)、路線名稱(nameZh 或 masterroutename)、起點(departureZh)及終點(destinationZh)、業者代號欄位(OperatorID、ProviderID等多種可能的鍵名) 以及停站PDF連結(stopAddrUrl 或 stop_address_url)。如某筆資料缺少停站PDF連結，該路線會在後續步驟中被略過。

解析每條路線的停站PDF：對每條取得的路線記錄，依序執行以下步驟：

取出路線編號、中文名稱、起訖站資訊、業者ID與停站PDF網址。若路線缺少停站PDF網址，則跳過並記錄未處理路線數量。

使用 requests 下載 PDF 內容，若遇 HTTP/HTTPS 問題嘗試切換協定，並以 pdfplumber 開啟PDF檔案內容。pdfplumber 是一個常用的 Python 套件，可 「從 PDF 中提取文本和表格」
github.com
。腳本以 pdfplumber.open(io.BytesIO(pdf_bytes)) 載入PDF，並透過 page.extract_text() 擷取每頁文字；再將所有頁的文字合併為一個長字串。

站牌資料抽取：使用正則表達式 ENTRY_RE = re.compile(r'(\d+)\s+(.+?)\s+([0-9]{2}\.[0-9]+)\s+([0-9]{3}\.[0-9]+)') 解析文本，每筆匹配結果捕捉站序號、站牌名稱＋地址、緯度 (lat)、經度 (lon)。由於原PDF站牌表通常左右兩列排版，程式將同一行中的每兩個匹配結果拆成左右兩列：左邊列對應「去程方向」站點清單，右邊列對應「返程方向」站點。程式依照站序 (StopSequence) 排序後，分別得到 left 與 right 兩份站牌列表。

站牌名稱與地址分離：正則式取得的第二組群組 (.+?) 常包含站牌名稱及路名地址，如 "中壢北站 中壢區中山東路2段"。腳本以空白分隔第1個詞當作站牌名稱 (StopName_Zh)，剩餘字串當作地址 (Address)。若匹配結果只包含一個詞，地址則留空。

結果組合：將左列站點標記為 Direction=0（去程），右列標記為 Direction=1（返程）。生成每個站點的完整記錄，包括路線編號、中文名稱、方向、方向標籤(DirectionLabel)、起訖站(From/To)、站序、站名、地址、緯經度、業者ID/名稱及來源。方向標籤以中文格式表示，例如 往 {destination} 或 去程（若終點資料缺失）；返程標籤則用 往 {departure} 或 返程。

業者名稱填補：若路線有業者代號，且於對照表中找到對應名稱，則在每筆站點資料中填入 OperatorName，並將 OperatorSource 設為 "lookup"；若找不到，保留空或原 ID 作為名稱，OperatorSource 標記為 "missing"。程式會統計業者代號填補情況，如補到名稱的路線數與有業者ID路線總數。

合併與輸出：將每條路線所有站點資料（含去程與返程）累計在單一列表中，並以 pandas.DataFrame 形式整理，欄位包含：Route, RouteName_Zh, Direction, DirectionLabel, StopSequence, StopName_Zh, Address, Lat, Lon, From, To, OperatorId, OperatorName, OperatorSource。最後依照路線編號、方向及站序排序，並輸出成 Excel (.xlsx) 檔案。Excel 使用 openpyxl 格式，可直接以 Excel 或 LibreOffice 等軟體開啟；也可用程式載入（如 Python/pandas）。輸出的檔名為 taoyuan_citybus_stops_OpenData_with_operator_lookup.xlsx。

整個執行過程有多重備援：資料請求使用 JSON 解析，失敗時改用 XML 解析；下載PDF若使用 http 失敗，改用 https。透過 tqdm 顯示進度條以利長時間任務監控。程式執行結束後，會印出有停站表的路線數、缺少PDF的路線數、站點總數及業者名稱補齊率等資訊，以供檢查。

使用方式與相依套件

執行此腳本需要 Python 3.x 環境，並安裝以下第三方套件：

requests：發送 HTTP 請求以取得開放資料 JSON 或下載PDF。

pdfplumber：解析PDF檔案，抽取文字內容
github.com
。

pandas、openpyxl：資料框(DataFrame)操作及寫出Excel檔案。

xmltodict：遇到某些XML格式回應時，用於解析備援。

tqdm：顯示進度條(非必需，但方便觀察處理進度)。

可以使用 pip 安裝上述套件，例如：

pip install requests pdfplumber pandas openpyxl xmltodict tqdm


然後執行腳本（可命名為 generate_taoyuan_stops.py），例如在終端機輸入：

python generate_taoyuan_stops.py


程式會自動從網路抓取資料並產生 Excel 檔案。記得在執行時機器需能連網，因為需要訪問桃園市開放資料 API 及下載站牌PDF。若要重現結果，可確保使用與本程式相容的開放資料版本與欄位結構。

輸出欄位說明

Excel 檔案中的欄位意義如下：

Route：路線編號。例如「602」或「A1」等。如原資料無編號則以名稱取代。

RouteName_Zh：路線中文名稱，如「區公所-保生路口」。

Direction：方向代碼，0 代表去程（從起點出發至終點），1 代表返程（從終點回起點）。

DirectionLabel：方向標籤，以中文顯示。如「往 {終點名稱}」或「去程」、「往 {起點名稱}」或「返程」。程式邏輯是若有提供終點字串，去程標記「往 XX」，否則以「去程」；返程類似。

StopSequence：站牌在該路線方向上的序號 (數字)。由PDF表格解析得到，代表公車路線中站牌出發順序。

StopName_Zh：站牌中文名稱。程式解析時取匹配字串的首個詞作為站名。

Address：站牌附加地址資訊（若有）。通常是站牌名稱後面其餘的文字。如格式「站名 地址」。程式將第一個詞切出作為站名，剩餘部分存為地址。本欄有些站牌可能為空。

Lat, Lon：站牌座標，分別為緯度與經度（十進位格式）。來自PDF上的GPS欄位，用於地圖定位。

From：該路線去程的起點站名稱（中文）。對應 departureZh 欄位。

To：該路線去程的終點站名稱（中文）。對應 destinationZh 欄位。

OperatorId：業者代號。來源於路線資料集中的 OperatorID 或等效欄位。若該路線由多家業者聯營，這欄可能包含多個編號（以逗號等分隔）。

OperatorName：業者中文名稱。程式使用第二步驟取得的業者對照表將上述代號轉換為名稱。如無對應名稱則留空或保留原ID。若一條路線有多個業者，名稱以「、」連接。

OperatorSource：名稱來源標記。若 OperatorName 成功從對照表查到，則為 "lookup"；否則為 "missing"，表示該業者代號在動態資料集中沒有對應名稱。

注意事項

站點重複：若路線同時有去程與返程停站表，會分別列出並分別標示方向。去、返程可能站牌名稱重複，但屬於不同方向。

缺少PDF連結：部分舊版或特例路線資料可能沒有提供 stopAddrUrl。此類路線不會被納入輸出，程式會在開始時印出略過訊息。

資料一致性：此檔案內容反映資料抓取當時的情況，包含桃園市公車路線與站牌資料。如要更新資料，需要重新執行程式以抓取最新開放資料。

字元編碼：所有中文欄位以 UTF-8 編碼儲存。Excel 檔以 .xlsx 格式輸出，可正常顯示繁體中文。

參考來源：本檔案依據桃園市交通局公開資料集製作，受桃園市政府開放資料授權條款規範
twn.databasesets.com
。此外，使用 pdfplumber 等開源工具進行PDF解析
github.com
。

參考資料

桃園市政府交通局提供之「公車路線資訊」資料集欄位說明
twn.databasesets.com
。

桃園市政府資料開放平台官網說明
twn.databasesets.com
。

pdfplumber 套件說明文件，支援從PDF抽取文字與表格
