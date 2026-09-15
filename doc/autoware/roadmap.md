# 1. autoware 的代码如何学习

學習 Autoware 這樣一個基於 ROS 2 的企業級 L4 自動駕駛開源架構，因為其代碼量極其龐大（包含數百个 Packages）、涉及學科極廣（計算機視覺、控制工程、點雲處理、機器人學），很容易讓人迷失在代碼海中。
為了高效入門，建議採取「由外向內、由淺入深」的逆向學習法。以下為您梳理的核心學習路徑與方法：
------------------------------
在直接看 Autoware 代碼之前，必須先掌握以下三項工具。如果這三個基礎不穩，看代碼會像看天書：

   1. C++ 17/20 與 Python3： Autoware 核心算法（如點雲過濾、控制、規劃）100% 使用現代 C++ 編寫，以追求極致性能；部分上層工具與仿真腳本使用 Python。
   2. ROS 2 (Robot Operating System 2)： Autoware 是完全基於 ROS 2 架構開發的。你必須非常熟悉 Nodes（節點）、Topics（話題）、Services（服務）、Actions（動作） 以及 Component Nodes（組件節點，用於內存零拷貝傳輸） 的概念。
   3. Linux (Ubuntu) & Docker： Autoware 的開發與部署高度依賴 Ubuntu 系統。官方強烈推薦使用 Docker 環境來隔離依賴。

------------------------------
「不要一開始就讀代碼，先讓車子動起來。」 透過可視化界面理解系統的輸入與輸出，是建立全局觀最快的方式。

   1. 按照官方指南部署： 參考 [Autoware Documentation](https://autowarefoundation.github.io/autoware-documentation/main/)，使用 Docker 部署一套最新的 Autoware 環境。
   2. 跑通 AWSIM 仿真測試：
   * Autoware 目前官方推薦使用基於 Unity 開發的 AWSIM 仿真器。
      * 在仿真環境中加載高精地圖（HD Map），設定起點與終點，觀察虛擬車輛如何自動行駛。
   3. 使用 Rviz2 觀察數據流：
   * 打開 Rviz2（ROS 的可視化工具），觀察激光雷達（LiDAR）點雲數據。
      * 觀察 Perception（感知） 模組輸出的大型 3D 邊界框（Bounding Boxes）。
      * 觀察 Planning（規劃） 模組計算出的藍色/綠色局部路徑軌跡線（Trajectory）。
   
------------------------------
在深入某個特定的 C++ 檔案之前，先搞清楚各個模組之間是怎麼開會、怎麼通訊的。

   1. 研讀 Autoware Design 文檔： 官方的 Autoware Design 頁面 是極其寶貴的資源，裡面詳細繪製了節點關係圖和數據流向圖。
   2. 以 Tier IV 的 ADAPI 爲核心切入：
   * Autoware 定義了一套 Autonomous Driving Application Programming Interface (ADAPI)。
      * 從這套 API 的功能（如：如何發送目的地、如何切換自動駕駛模式、如何處理異常報警）出發，去追蹤背後的 ROS 2 話題，就能摸清整個系統的運作骨架。
   
------------------------------
不要試圖一次看懂所有代碼，選擇一個你最感興趣或最符合你專業背景的模組進入：
依據上圖的數據流向，你可以選擇其中一個切入點深入 autoware_universe/ 閱讀：

* 
* 如果你喜歡數學與控制理論 ➡️ 從 control 模組開始：
* 核心 Package： trajectory_follower_node
   * 學習重點： 閱讀其實現的 MPC（模型預測控制） 算法和 PID 算法。看它如何接收 Planning 輸出的軌跡點，並計算出車輛的轉向角（Steering）與加速度（Acceleration）。
* 如果你熟悉幾何與車道線規劃 ➡️ 從 planning 模組開始：
* 核心 Package： behavior_path_planner（行為路徑規劃）和 motion_velocity_planner（運動速度規劃）。
   * 學習重點： 看系統如何根據高精地圖（Lanelet2）進行換道（Lane Change）、避障（Avoidance）以及在紅綠燈前停下的狀態機邏輯。
* 如果你熟悉計算機視覺與點雲 ➡️ 從 perception 模組開始：
* 核心 Package： object_recognition_2d/3d
   * 學習重點： 研究它如何處理 LiDAR 點雲、如何做點雲地面過濾、如何透過卷積神經網絡（CNN）或 PointNet 進行障礙物檢測與多目標追蹤（Multi-Object Tracking）。
* 

------------------------------

* 
* 善用 colcon 編譯工具的軟連結： 在編譯時使用 colcon build --symlink-install，這樣當你修改 Python 腳本或某些配置文件時，不需要重新編譯就能直接生效，極大提升調試效率。
* 學會看 package.xml 和 CMakeLists.txt： 每當進入一個新的 Package，先看這兩個檔案。它會告訴你這個模組依賴了哪些其他模組，以及它的入口源代碼檔案（Executable）是哪一個。
* 利用日誌（RCLCPP_INFO）和 GDB： 在代碼中加入 ROS 2 的日誌打印，或者配置 VS Code 的 ROS 2 插件進行斷點調試，實時觀察變量的變化。
* 加入社區： 關注 Autoware Foundation GitHub Discussions，遇到報錯時，90% 的問題都能在裡面找到前人的解答。
* 

您目前具備 ROS / C++ 的基礎嗎？或者您的主要背景更偏向於計算機視覺（感知）還是車輛控制（規劃控制）？我可以針對您的背景，為您推薦第一個最適合深入閱讀的特定代碼包（Package）與算法核心檔案！

