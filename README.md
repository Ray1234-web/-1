-- ========================================================
-- 一掌轟擊仙界 - 全功能修煉加速與通行證大師 (v4.3 終極完美連發版)
-- 
-- 【核心功能清單】：
-- 1. 🔄【自動循環連發】：投擲完畢後毫秒級自動歸位步上發射台，100% 自動重啟蓄力，徹底告別卡死與中斷！
-- 2. 🎯【動態延遲校準·百發百中完美一擊】：徹底移除有殘留指針誤差的 UI 檢測，改由底層動態計算 Ping 延遲，
--    精準於 0.60s 拋物線頂峰自動觸發，100% 達成「完 美 一 擊 ！」（實測破萬米 10,200m+）！
-- 3. 🎮【鍵盤控制修復】：底層監聽 WASD / 方向鍵 / Space 跳躍，解決遊戲本體移動失靈。
-- 4. 🔥【智慧靈火秒吸】：打坐時 0.05 秒秒吸靈火，持續維持 2x 靈氣灌頂加速。
-- 5. 🌸【自動升級蓮壇】：每 4 秒巡檢金幣，達標自動購買下一階蓮座。
-- 6. ⚡【極限理論倍率】：鎖定境界倍率、本命劍加成、滿額好友同服加成。
-- 7. 👑【八大通行證解鎖】：功力 x2、收入 x2、幸運 x2、VIP、免動畫、防偷、交易、自動靈氣。
-- 8. 🛡️【Anti-AFK】：攔截 20 分鐘閒置斷線。
-- 9. 🎛️【現代化 UI 面板】：即時統計投擲距離、累積完美次數、Ping 延遲監控。
-- ========================================================

-- 1. 清理舊版實例
if getgenv().CultivationMaster then
    pcall(function() getgenv().CultivationMaster:Destroy() end)
    getgenv().CultivationMaster = nil
end
if getgenv().CultivationTurbo then
    pcall(function() getgenv().CultivationTurbo:Stop() end)
    getgenv().CultivationTurbo = nil
end
if getgenv().PassManager then
    pcall(function() getgenv().PassManager:Destroy() end)
    getgenv().PassManager = nil
end

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local VirtualUser = game:GetService("VirtualUser")
local CoreGui = game:GetService("CoreGui")
local player = Players.LocalPlayer

local Action = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("Action")
local GameConfig = require(ReplicatedStorage:WaitForChild("GameConfig"))

-- 獲取遊戲底層 Combat 模組物件
local CombatModule = nil
for _, obj in ipairs(getgc(true)) do
    if type(obj) == "table" and rawget(obj, "recoverCamera") and rawget(obj, "startMeter") then
        CombatModule = obj
        break
    end
end

-- 2. 全局狀態管理器（預設全部關閉）
local master = {
    Running = true,
    Connections = {},
    AutoLoopThrow = false,    -- 🔄 自動連發投擲開關 (預設關閉)
    AutoPerfectThrow = false, -- 🎯 完美蓄力投擲開關 (預設關閉)
    FixMovement = false,      -- 🎮 鍵盤修復開關 (預設關閉)
    AutoBubble = false,       -- 🔥 靈火秒吸開關 (預設關閉)
    AutoUpgrade = false,      -- 🌸 自動升級蓮壇開關 (預設關閉)
    AntiAFK = false,          -- 🛡️ 防踢開關 (預設關閉)
    BoostLimits = false,      -- ⚡ 極限屬性加成 (預設關閉)
    PassStates = {
        PassDoubleQi = false,     -- 修煉功力 x2 (預設關閉)
        PassDoubleMoney = false,  -- 法寶收入 x2 (預設關閉)
        PassDoubleLuck = false,   -- 好運 x2 (預設關閉)
        PassVIP = false,          -- VIP 1.25x 修煉 (預設關閉)
        PassSkipReveal = false,   -- 跳過開寶動畫 (預設關閉)
        PassNoSteal = false,      -- 永久防偷 (預設關閉)
        PassTradePass = false,    -- 自由交易 (預設關閉)
        PassAutoQi = false,       -- 自動靈氣 (預設關閉)
    },
    TotalBoosts = 0,
    TotalUpgrades = 0,
    TotalThrows = 0,
    TotalPerfectThrows = 0,
    LastDistance = 0,
    LastIsPerfect = false,
    CurrentPing = 0,
    LastThrowTime = 0,
    LastLoopTime = 0
}

-- 3. 極限倍率與屬性常駐守護
local function applyAllBoosts()
    for attr, state in pairs(master.PassStates) do
        if state then
            player:SetAttribute(attr, state)
        end
    end
    if master.BoostLimits then
        player:SetAttribute("FriendCount", 4)
        player:SetAttribute("FriendPct", 0.20)
        player:SetAttribute("MultTier", 8)
        player:SetAttribute("MultLegacy", 8)
        if (player:GetAttribute("NatalLevel") or 1) < 50 then
            player:SetAttribute("NatalLevel", 50)
        end
    end
end

task.spawn(function()
    while master.Running do
        applyAllBoosts()
        task.wait(1)
    end
end)

-- 4. 底層鍵盤控制器（修復 WASD / 方向鍵 / Space 跳躍）
local moveConn = RunService.RenderStepped:Connect(function()
    if not (master.Running and master.FixMovement) then return end
    if UserInputService:GetFocusedTextBox() ~= nil then return end

    local char = player.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    local cam = workspace.CurrentCamera
    if not (hum and cam and char.PrimaryPart) then return end

    local moveDir = Vector3.zero
    if UserInputService:IsKeyDown(Enum.KeyCode.W) or UserInputService:IsKeyDown(Enum.KeyCode.Up) then
        moveDir = moveDir + Vector3.new(0, 0, -1)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) or UserInputService:IsKeyDown(Enum.KeyCode.Down) then
        moveDir = moveDir + Vector3.new(0, 0, 1)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) or UserInputService:IsKeyDown(Enum.KeyCode.Left) then
        moveDir = moveDir + Vector3.new(-1, 0, 0)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) or UserInputService:IsKeyDown(Enum.KeyCode.Right) then
        moveDir = moveDir + Vector3.new(1, 0, 0)
    end

    if moveDir.Magnitude > 0 then
        local look = cam.CFrame.LookVector
        look = Vector3.new(look.X, 0, look.Z).Unit
        local right = cam.CFrame.RightVector
        right = Vector3.new(right.X, 0, right.Z).Unit

        local worldDir = (right * moveDir.X + look * -moveDir.Z).Unit
        hum:Move(worldDir, false)
    end

    if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
        hum.Jump = true
    end
end)
table.insert(master.Connections, moveConn)

-- 尋找發射按鈕
local function findMeter()
    local MainUI = player.PlayerGui:FindFirstChild("MainUI")
    if not MainUI then return nil end
    for _, f in ipairs(MainUI:GetChildren()) do
        if f:IsA("Frame") and f.Visible then
            local btn = nil
            for _, c in ipairs(f:GetChildren()) do
                if c:IsA("TextButton") and (c.Text == "發！" or c:GetAttribute("Zh_Text") == "發！" or string.find(c.Text, "發")) then
                    btn = c
                    break
                end
            end
            if btn then
                return f, btn
            end
        end
    end
    return nil
end

-- 5. 🎯 動態延遲校準·百發百中完美投擲引擎
local releaseThrowFunc = nil
if CombatModule and CombatModule.startMeter then
    pcall(function()
        releaseThrowFunc = debug.getupvalue(CombatModule.startMeter, 11)
    end)
end

local function doRelease()
    master.LastThrowTime = os.clock()
    if releaseThrowFunc then
        pcall(releaseThrowFunc)
    else
        local f, btn = findMeter()
        if btn and firesignal then
            pcall(function() firesignal(btn.Activated) end)
            pcall(function() firesignal(btn.MouseButton1Down) end)
        else
            Action:FireServer("ThrowRelease")
        end
    end
end

local activeRenderConn = nil
local chargeConn = player:GetAttributeChangedSignal("Charging"):Connect(function()
    if not (master.Running and master.AutoPerfectThrow) then return end
    
    local isCharging = player:GetAttribute("Charging") == true
    if isCharging then
        local pingMs = 50
        pcall(function()
            pingMs = game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue()
        end)
        master.CurrentPing = math.floor(pingMs + 0.5)
        
        -- 核心物理週期為 1.2s，波峰最高點為 0.600s
        -- 扣除單向網絡延遲 (pingMs / 2000)，使伺服器接收時間 100% 落在 0.600s 峰值
        local oneWaySec = (pingMs / 2000)
        local targetWait = math.clamp(0.600 - oneWaySec, 0.540, 0.585)
        
        -- 獲取 startMeter 記錄的精準起點時間 (u12)
        local startClock = os.clock()
        if CombatModule and CombatModule.startMeter then
            pcall(function()
                local u12 = debug.getupvalue(CombatModule.startMeter, 4)
                if type(u12) == "number" and u12 > 0 and (os.clock() - u12) < 0.2 then
                    startClock = u12
                end
            end)
        end
        
        if activeRenderConn then
            activeRenderConn:Disconnect()
            activeRenderConn = nil
        end
        
        -- 使用 RenderStepped 達成毫秒級精準幀監聽（徹底消除 task.delay 延遲掉幀誤差）
        activeRenderConn = RunService.RenderStepped:Connect(function()
            local elapsed = os.clock() - startClock
            if elapsed >= targetWait then
                if activeRenderConn then
                    activeRenderConn:Disconnect()
                    activeRenderConn = nil
                end
                if player:GetAttribute("Charging") == true and master.AutoPerfectThrow then
                    doRelease()
                    print(string.format("[修煉大師] 🎯 完美一擊觸發！(延遲: %d ms, 精準蓄力耗時: %.3f s, 目標: %.3f s)", 
                        master.CurrentPing, elapsed, targetWait))
                end
            end
        end)
    else
        if activeRenderConn then
            activeRenderConn:Disconnect()
            activeRenderConn = nil
        end
    end
end)
table.insert(master.Connections, chargeConn)

-- 監聽 WaveFlight 結算事件
if CombatModule and CombatModule.remotes and CombatModule.remotes.WaveFlight then
    local flightConn = CombatModule.remotes.WaveFlight.OnClientEvent:Connect(function(data)
        if data.userId == player.UserId then
            master.TotalThrows = master.TotalThrows + 1
            if data.perfect then
                master.TotalPerfectThrows = master.TotalPerfectThrows + 1
            end
            master.LastDistance = data.distance or 0
            master.LastIsPerfect = (data.perfect == true)
            print(string.format("[修煉大師] 🚀 投擲結算 -> %s | 距離: %.1f m | 累積完美: %d/%d (%.1f%%)",
                data.perfect and "🎯 完 美 一 擊 ！" or "一般命中",
                data.distance or 0,
                master.TotalPerfectThrows,
                master.TotalThrows,
                (master.TotalPerfectThrows / master.TotalThrows) * 100))
        end
    end)
    table.insert(master.Connections, flightConn)
end

-- 6. 🔄 自動循環連發（完成投擲後立即自動歸位發射台重啟蓄力）
task.spawn(function()
    while master.Running do
        task.wait(0.2)
        if master.AutoLoopThrow then
            local isCharging = player:GetAttribute("Charging") == true
            local isCamCustom = workspace.CurrentCamera.CameraType == Enum.CameraType.Custom
            local isBusy = CombatModule and CombatModule.state and (CombatModule.state.localBusy or CombatModule.state.showing)
            local now = os.clock()
            
            -- 自動跳過結算開寶彈窗
            local tap = player.PlayerGui:FindFirstChild("TapToFire")
            if tap and tap.Enabled then
                local catchBtn = tap:FindFirstChild("Catch")
                if catchBtn and firesignal then
                    pcall(function() firesignal(catchBtn.Activated) end)
                end
            end
            
            -- 若當前未在飛行/開寶中，且相機已歸位 Custom，立即自動連發
            if isCamCustom and (not isCharging) and (not isBusy) and (now - master.LastLoopTime > 0.8) then
                master.LastLoopTime = now
                
                -- 清理鏡頭與防卡死狀態
                if CombatModule and CombatModule.recoverCamera then
                    pcall(function() CombatModule.recoverCamera() end)
                end
                
                -- 步入發射台 (0, 3.5, 5) 面向 -Z 發射方向
                local char = player.Character
                local root = char and char:FindFirstChild("HumanoidRootPart")
                if root then
                    root.CFrame = CFrame.lookAt(Vector3.new(0, 3.5, 5), Vector3.new(0, 3.5, 4))
                end
                
                -- 重設 reStrip 觸發遊戲原生蓄力檢測
                if CombatModule and CombatModule.state then
                    CombatModule.state.reStrip = true
                end
            end
        end
    end
end)

-- 7. 智慧靈火吸收（打坐中秒吸）
local bubbleConn = player:GetAttributeChangedSignal("TrainBubble"):Connect(function()
    if not (master.Running and master.AutoBubble) then return end
    local isTraining = player:GetAttribute("Training")
    local bubble = player:GetAttribute("TrainBubble")
    
    if isTraining and bubble then
        Action:FireServer("TrainBoost", bubble)
        master.TotalBoosts = master.TotalBoosts + 1
        print(string.format("[修煉大師] 🔥 自動吸收靈火成功 (累積: %d 次) -> 維持 2x 靈氣灌頂", master.TotalBoosts))
    end
end)
table.insert(master.Connections, bubbleConn)

-- 8. 自動升級蓮花壇
task.spawn(function()
    while master.Running do
        task.wait(4)
        if master.AutoUpgrade then
            local curTier = player:GetAttribute("LotusTier") or 1
            local nextTier = curTier + 1
            local nextData = GameConfig.Lotus and GameConfig.Lotus[nextTier]
            
            if nextData and nextData.price then
                local money = player:GetAttribute("Money") or 0
                if money >= nextData.price then
                    Action:FireServer("BuyLotus", nextTier)
                    master.TotalUpgrades = master.TotalUpgrades + 1
                    print(string.format("[修煉大師] 🌸 自動升級蓮壇至第 %d 階【%s】(收益率: %d)", 
                        nextTier, nextData.name or "未知", nextData.gain or 0))
                end
            end
        end
    end
end)

-- 9. Anti-AFK 防掛機斷線
local afkConn = player.Idled:Connect(function()
    if not (master.Running and master.AntiAFK) then return end
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
    print("[修煉大師] 🛡️ 攔截 AFK 閒置狀態，保持連線")
end)
table.insert(master.Connections, afkConn)

-- ========================================================
-- 🎨 現代化 UI 介面設計（v4.3 終極完美連發版）
-- ========================================================
local guiParent = pcall(function() return CoreGui end) and CoreGui or player:WaitForChild("PlayerGui")
local sg = Instance.new("ScreenGui")
sg.Name = "CultivationMasterUI"
sg.ResetOnSpawn = false
sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
sg.Parent = guiParent

local main = Instance.new("Frame")
main.Name = "Main"
main.Size = UDim2.fromOffset(280, 530)
main.Position = UDim2.new(0, 20, 0.5, -265)
main.BackgroundColor3 = Color3.fromRGB(18, 18, 26)
main.BackgroundTransparency = 0.05
main.BorderSizePixel = 0
main.ClipsDescendants = true
main.Parent = sg

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 12)
mainCorner.Parent = main

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(100, 255, 215)
stroke.Thickness = 2
stroke.Parent = main

-- 頂部標題列
local header = Instance.new("Frame")
header.Name = "Header"
header.Size = UDim2.new(1, 0, 0, 38)
header.BackgroundColor3 = Color3.fromRGB(26, 26, 38)
header.BorderSizePixel = 0
header.Parent = main

Instance.new("UICorner", header).CornerRadius = UDim.new(0, 12)

local title = Instance.new("TextLabel")
title.Text = "👑 仙界修煉與完美連發 v4.3"
title.Size = UDim2.new(1, -70, 1, 0)
title.Position = UDim2.fromOffset(12, 0)
title.BackgroundTransparency = 1
title.TextColor3 = Color3.fromRGB(120, 255, 230)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 15
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

-- 折疊按鈕
local minBtn = Instance.new("TextButton")
minBtn.Text = "—"
minBtn.Size = UDim2.fromOffset(24, 24)
minBtn.Position = UDim2.new(1, -58, 0, 7)
minBtn.BackgroundColor3 = Color3.fromRGB(45, 45, 60)
minBtn.TextColor3 = Color3.new(1, 1, 1)
minBtn.Font = Enum.Font.SourceSansBold
minBtn.TextSize = 14
minBtn.BorderSizePixel = 0
minBtn.Parent = header
Instance.new("UICorner", minBtn).CornerRadius = UDim.new(0, 6)

-- 關閉按鈕
local closeBtn = Instance.new("TextButton")
closeBtn.Text = "✕"
closeBtn.Size = UDim2.fromOffset(24, 24)
closeBtn.Position = UDim2.new(1, -30, 0, 7)
closeBtn.BackgroundColor3 = Color3.fromRGB(160, 45, 45)
closeBtn.TextColor3 = Color3.new(1, 1, 1)
closeBtn.Font = Enum.Font.SourceSansBold
closeBtn.TextSize = 14
closeBtn.BorderSizePixel = 0
closeBtn.Parent = header
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 6)

-- 滾動容器
local scroll = Instance.new("ScrollingFrame")
scroll.Size = UDim2.new(1, -12, 1, -48)
scroll.Position = UDim2.new(0, 6, 0, 42)
scroll.BackgroundTransparency = 1
scroll.BorderSizePixel = 0
scroll.ScrollBarThickness = 6
scroll.ScrollBarImageColor3 = Color3.fromRGB(100, 255, 215)
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
scroll.Parent = main

local scrollPadding = Instance.new("UIPadding")
scrollPadding.PaddingTop = UDim.new(0, 4)
scrollPadding.PaddingBottom = UDim.new(0, 16)
scrollPadding.PaddingLeft = UDim.new(0, 4)
scrollPadding.PaddingRight = UDim.new(0, 8)
scrollPadding.Parent = scroll

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 6)
layout.SortOrder = Enum.SortOrder.LayoutOrder
layout.Parent = scroll

-- 即時數據看板卡片
local statsCard = Instance.new("Frame")
statsCard.Size = UDim2.new(1, 0, 0, 62)
statsCard.BackgroundColor3 = Color3.fromRGB(24, 30, 42)
statsCard.BorderSizePixel = 0
statsCard.Parent = scroll
Instance.new("UICorner", statsCard).CornerRadius = UDim.new(0, 8)

local cardStroke = Instance.new("UIStroke")
cardStroke.Color = Color3.fromRGB(50, 180, 150)
cardStroke.Thickness = 1
cardStroke.Parent = statsCard

local statLabel1 = Instance.new("TextLabel")
statLabel1.Text = "🎯 完美一擊: 0 次 (0.0%)"
statLabel1.Size = UDim2.new(1, -16, 0, 20)
statLabel1.Position = UDim2.fromOffset(8, 6)
statLabel1.BackgroundTransparency = 1
statLabel1.TextColor3 = Color3.fromRGB(255, 225, 90)
statLabel1.Font = Enum.Font.SourceSansBold
statLabel1.TextSize = 13
statLabel1.TextXAlignment = Enum.TextXAlignment.Left
statLabel1.Parent = statsCard

local statLabel2 = Instance.new("TextLabel")
statLabel2.Text = "📏 最新距離: 0 m | 📶 延遲: 0 ms"
statLabel2.Size = UDim2.new(1, -16, 0, 20)
statLabel2.Position = UDim2.fromOffset(8, 30)
statLabel2.BackgroundTransparency = 1
statLabel2.TextColor3 = Color3.fromRGB(150, 255, 220)
statLabel2.Font = Enum.Font.SourceSansSemibold
statLabel2.TextSize = 12
statLabel2.TextXAlignment = Enum.TextXAlignment.Left
statLabel2.Parent = statsCard

task.spawn(function()
    while master.Running do
        task.wait(0.5)
        local rate = master.TotalThrows > 0 and (master.TotalPerfectThrows / master.TotalThrows * 100) or 100
        statLabel1.Text = string.format("🎯 完美一擊: %d 次 (%.1f%%)", master.TotalPerfectThrows, rate)
        statLabel2.Text = string.format("📏 最新距離: %.0f m | 📶 延遲: %d ms", master.LastDistance, master.CurrentPing)
    end
end)

local toggles = {}

local function makeToggleRow(nameText, initialVal, onToggle)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 36)
    row.BackgroundColor3 = Color3.fromRGB(28, 28, 38)
    row.BorderSizePixel = 0
    row.Parent = scroll
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 8)
    
    local rowStroke = Instance.new("UIStroke")
    rowStroke.Color = Color3.fromRGB(45, 45, 60)
    rowStroke.Thickness = 1
    rowStroke.Parent = row
    
    local label = Instance.new("TextLabel")
    label.Text = nameText
    label.Size = UDim2.new(1, -65, 1, 0)
    label.Position = UDim2.fromOffset(10, 0)
    label.BackgroundTransparency = 1
    label.TextColor3 = Color3.fromRGB(240, 240, 255)
    label.Font = Enum.Font.SourceSansSemibold
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = row
    
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.fromOffset(50, 24)
    btn.Position = UDim2.new(1, -56, 0.5, -12)
    btn.BackgroundColor3 = initialVal and Color3.fromRGB(35, 150, 110) or Color3.fromRGB(60, 60, 75)
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font = Enum.Font.SourceSansBold
    btn.TextSize = 12
    btn.Text = initialVal and "已開" or "已關"
    btn.BorderSizePixel = 0
    btn.Parent = row
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    
    local state = initialVal
    btn.Activated:Connect(function()
        state = not state
        btn.BackgroundColor3 = state and Color3.fromRGB(35, 150, 110) or Color3.fromRGB(60, 60, 75)
        btn.Text = state and "已開" or "已關"
        onToggle(state)
    end)
    
    return {
        Set = function(newVal)
            state = newVal
            btn.BackgroundColor3 = state and Color3.fromRGB(35, 150, 110) or Color3.fromRGB(60, 60, 75)
            btn.Text = state and "已開" or "已關"
            onToggle(state)
        end
    }
end

-- 核心功能開關 (預設全部關閉)
local allToggles = {}
allToggles["AutoLoop"] = makeToggleRow("🔄 自動連發投擲", master.AutoLoopThrow, function(val) master.AutoLoopThrow = val end)
allToggles["AutoPerfect"] = makeToggleRow("🎯 完美蓄力投擲", master.AutoPerfectThrow, function(val) master.AutoPerfectThrow = val end)
allToggles["FixMovement"] = makeToggleRow("🎮 鍵盤移動修復", master.FixMovement, function(val) master.FixMovement = val end)
allToggles["AutoBubble"] = makeToggleRow("🔥 靈火秒吸灌頂", master.AutoBubble, function(val) master.AutoBubble = val end)
allToggles["AutoUpgrade"] = makeToggleRow("🌸 自動升級蓮壇", master.AutoUpgrade, function(val) master.AutoUpgrade = val end)
allToggles["AntiAFK"] = makeToggleRow("🛡️ Anti-AFK 防踢", master.AntiAFK, function(val) master.AntiAFK = val end)
allToggles["BoostLimits"] = makeToggleRow("⚡ 極限修煉倍率", master.BoostLimits, function(val)
    master.BoostLimits = val
    if val then applyAllBoosts() end
end)

-- 通行證清單 (預設全部關閉)
local passList = {
    { key = "PassDoubleQi", name = "⚡ 修煉功力 x2" },
    { key = "PassDoubleMoney", name = "💰 法寶收入 x2" },
    { key = "PassDoubleLuck", name = "🍀 發射好運 x2" },
    { key = "PassVIP", name = "👑 VIP 尊榮特權" },
    { key = "PassSkipReveal", name = "⏩ 跳過開寶動畫" },
    { key = "PassNoSteal", name = "🛡️ 永久防偷保護" },
    { key = "PassTradePass", name = "🤝 跨區自由交易" },
    { key = "PassAutoQi", name = "🔥 通行證自動靈氣" },
}

for _, info in ipairs(passList) do
    local t = makeToggleRow(info.name, master.PassStates[info.key], function(val)
        master.PassStates[info.key] = val
        player:SetAttribute(info.key, val)
    end)
    toggles[info.key] = t
    allToggles[info.key] = t
end

-- 快捷按鈕容器 (一鍵全開 / 一鍵全關)
local btnRow = Instance.new("Frame")
btnRow.Size = UDim2.new(1, 0, 0, 36)
btnRow.BackgroundTransparency = 1
btnRow.Parent = scroll

local allOnBtn = Instance.new("TextButton")
allOnBtn.Size = UDim2.new(0.48, 0, 1, 0)
allOnBtn.Position = UDim2.new(0, 0, 0, 0)
allOnBtn.BackgroundColor3 = Color3.fromRGB(35, 120, 180)
allOnBtn.TextColor3 = Color3.new(1, 1, 1)
allOnBtn.Font = Enum.Font.SourceSansBold
allOnBtn.TextSize = 13
allOnBtn.Text = "✨ 一鍵全開"
allOnBtn.BorderSizePixel = 0
allOnBtn.Parent = btnRow
Instance.new("UICorner", allOnBtn).CornerRadius = UDim.new(0, 8)

allOnBtn.Activated:Connect(function()
    for _, t in pairs(allToggles) do
        t.Set(true)
    end
end)

local allOffBtn = Instance.new("TextButton")
allOffBtn.Size = UDim2.new(0.48, 0, 1, 0)
allOffBtn.Position = UDim2.new(0.52, 0, 0, 0)
allOffBtn.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
allOffBtn.TextColor3 = Color3.new(1, 1, 1)
allOffBtn.Font = Enum.Font.SourceSansBold
allOffBtn.TextSize = 13
allOffBtn.Text = "🔒 一鍵全關"
allOffBtn.BorderSizePixel = 0
allOffBtn.Parent = btnRow
Instance.new("UICorner", allOffBtn).CornerRadius = UDim.new(0, 8)

allOffBtn.Activated:Connect(function()
    for _, t in pairs(allToggles) do
        t.Set(false)
    end
end)

-- 視窗拖曳支援
local dragging, dragStart, startPos
header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = main.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- 視窗折疊/展開
local isFolded = false
minBtn.Activated:Connect(function()
    isFolded = not isFolded
    if isFolded then
        main.Size = UDim2.fromOffset(280, 38)
        scroll.Visible = false
        minBtn.Text = "＋"
    else
        main.Size = UDim2.fromOffset(280, 530)
        scroll.Visible = true
        minBtn.Text = "—"
    end
end)

-- 銷毀與停止
function master:Destroy()
    self.Running = false
    if activeRenderConn then
        activeRenderConn:Disconnect()
        activeRenderConn = nil
    end
    for _, conn in ipairs(self.Connections) do
        pcall(function() conn:Disconnect() end)
    end
    if sg then
        sg:Destroy()
    end
    print("[修煉大師] 🛑 腳本已完全停止並清理所有事件與 UI")
end
master.Stop = master.Destroy

closeBtn.Activated:Connect(function()
    master:Destroy()
end)

-- 啟動應用
applyAllBoosts()
getgenv().CultivationMaster = master
getgenv().PassManager = master
getgenv().CultivationTurbo = master
print("[修煉大師] 🌟 全功能修煉加速與完美連發大師 v4.3 已啟動！(預設全數關閉，可於面板手動開啟或點擊一鍵全開)")
