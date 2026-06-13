# 2026 OOPL Final Report

## 組別資訊

組別：T03
組員：陳柏全
復刻遊戲：異形工廠

## 專案簡介

### 遊戲簡介
異形工廠是一款專注於自動化生產線的沙盒益智遊戲。玩家需要在無邊界的地圖上，開採、切割、旋轉、染色並組合不同形狀與顏色的幾何圖形，以滿足特定的交貨需求。

### 組別分工
陳柏全 - 100%

## 遊戲介紹

### 遊戲規則
玩家需要使用礦機開採地圖上的資源，透過數種方法將資源加工，並送到基地完成目標要求的特定形狀與數量。
遊戲中有許多運送物品的裝置，主要為：
- 輸送帶：以固定速度將一個物品從一端運送到另一端
- 平衡機：將兩個輸入平均分配到兩個輸出
- 分流器：將一個輸入平均分配到兩個輸出
- 匯流器：將兩個輸入匯流到一個輸送帶上
- 隧道：運送物品的另一個方式，可以穿過其他機器，但長度有限
<!-- -->
遊戲中有許多加工機器和其變體，主要如下：
- 切割機：將一個形狀分為左右兩半
- 旋轉機：將一個形狀旋轉
- 堆疊機：將一個形狀堆疊在另一個形狀上
- 混色機：依照 RGB 光混色方式對兩個顏料進行混色
- 上色機：輸入一個形狀和一個顏料，將整個圖形上色
- 垃圾桶：將所有輸入的物品銷毀

### 遊戲畫面
<img width="2562" height="1467" alt="image" src="https://github.com/user-attachments/assets/c34986eb-d97c-43d2-98ac-d872fc308251" />
<img width="2562" height="1467" alt="image" src="https://github.com/user-attachments/assets/335b3bf9-8999-4dd6-a967-6c7a00c16a23" />

## 程式設計

### 程式架構
`Machine`為所有機器的 base class。
`ItemAcceptor`和`ItemEjector`以組合的方式表示一個機器的輸入和輸出。
這些機器放在`World.hpp`中的一個陣列和一個以座標為key的雜湊表中。
機器的`ItemAcceptor`和`ItemEjector`放在一個以 {x座標, y座標, 旋轉方向} 為key的雜湊表中。

`Item`為所有物品的 base class，以組合的方式存在於`ItemAcceptor`，`ItemEjector`，機器，世界網格，或UI元素中。
主要有`Shape`形狀和`Color`顏料。
`ItemAcceptor`可以選擇只輸入形狀或顏料。

場景主要處理玩家輸入和使用者介面，`Scene` 為場景的 base class。
最主要有`TitleScene`（標題畫面），和`GameScene`（遊戲場景）。
`TitleScene`負責建立、進入、刪除與重新命名存檔。
`GameScene`則處理玩家放置和拆除機器，選擇機器和其變體，以及顯示其他功能。

整個程式每一幀先更新場景，再更新世界和所有機器，最後更新 Renderer。

### 程式技術
大量使用繼承和組合。
有些地方使用到 function overloading，例如顏料的建立：
```cpp
Color::Color(int color)
    : Item(ItemType::COLOR) {
    this->color = color;
    SetDrawable(colorTextures[color]);
    SetZIndex(20);
}

Color::Color(std::string code)
    : Item(ItemType::COLOR) {
    if (code == "Color-u") {color = 0;} // 000
    else if (code == "Color-b") {color = 1;} // 001
    else if (code == "Color-g") {color = 2;} // 010
    else if (code == "Color-c") {color = 3;} // 011
    else if (code == "Color-r") {color = 4;} // 100
    else if (code == "Color-p") {color = 5;} // 101
    else if (code == "Color-y") {color = 6;} // 110
    else if (code == "Color-w") {color = 7;} // 111
    else {throw std::invalid_argument("Invalid color " + code);}
    SetDrawable(colorTextures[color]);
    SetZIndex(20);
}
```

### 使用到 AI/AI Agent 的部分 (沒有用到者，不需要寫這篇)
主要使用 ChatGPT、Gemini 與 Claude 等 AI 聊天機器人。
未使用到 AI Agent。

## 結語

### 問題與解決方法
第一個問題是效能：
PTSD給的Renderer太慢了，因此我以其為原型，修改出了適合這個專案的Renderer。
單線程不足以支撐這麼多台機器，因此我把多線程套用在大部分機器更新步驟和Renderer的部分步驟。
第二個問題是更新順序：
同樣一條輸送帶，由後往前更新和由前往後更新，物品會差一幀。在高速輸送帶上或是輸送帶堵住的時候，這個問題會特別明顯。
解決方法是透過`Update()`、`Restore()`和`Promote()`三個函式解決這些問題。
- `Update()` 假設前方機器是空的並直接將物品輸送過去。
- 如果機器後方輸送物品，但機器不是空的，就會觸發遞迴函式`Restore()`。
- 所有機器處理完後，所有機器`Promote()`，確認物品轉移。
<!-- -->
其中`Update()`和`Promote()`沒有順序性，因此可以使用多線程完成這兩個步驟。
```cpp
hub->Update();
UpdateMachines(pool, LstMachines);

std::vector<std::shared_ptr<ItemAcceptor>> acceptorConflicts;
std::vector<std::shared_ptr<ItemEjector>> ejectorConflicts;

for (auto& [pos, ac]: MapAcceptors) {
    if (ac->CheckConflict()) {acceptorConflicts.push_back(ac);}
}
for (auto& [pos, ej]: MapEjectors) {
    if (ej->CheckConflict()) {ejectorConflicts.push_back(ej);}
}

for (auto& ac: acceptorConflicts) {ac->StartRestore();}
for (auto& ej: ejectorConflicts) {ej->StartRestore();
}

hub->Promote();
PromoteMachines(pool, LstMachines);
```

### 自評

| 項次 | 項目                   | 完成 |
|------|------------------------|-------|
| 1    | 完成Proposal的所有目標 |  V  |
| 2    | 完成專案權限改為 public |  V  |
| 3    | 具有 debug mode 的功能  |  V  |
| 4    | 解決專案上所有 Memory Leak 的問題  |  V  |
| 5    | 報告中沒有任何錯字，以及沒有任何一項遺漏  |  V  |
| 6    | 報告至少保持基本的美感，人類可讀  |  V  |

### 心得
因為異形工廠原本是個網頁.io遊戲，而且是開源專案，因此可以直接從 GitHub 和遊戲本身取得大部分素材。
但同時因為它是網頁遊戲，原本很容易透過html和css實作的UI介面，在PTSD中就要花很多時間。
PTSD 不支援調整圖片顏色或透明度，這讓使用者介面的實作雪上加霜，就連Scratch都支援調顏色和透明度。
Renderer 雖然讓使用者不需要直接操作 OpenGL，但效能真的很差。
如果之後還有這種課程的話，換個遊戲引擎吧，至少遇到問題時，可以在網路上找到相關解答。

### 貢獻比例
陳柏全 - 100%
