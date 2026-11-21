# -1
-- ========================================================
-- 一掌轟擊仙界 - 全功能修煉加速與通行證大師 (v4.1 完美投擲旗艦版)
-- 
-- 【核心功能清單】：
-- 1. 🎯【百發百中·完美投擲】：即時監控光波蓄力條，指針達 92%~98% 完美區間自動釋放，100% 觸發「完美一擊」極限滿距！
-- 2. 🎮【鍵盤控制修復】：底層監聽 WASD / 方向鍵 / Space 跳躍，解決遊戲本體移動失靈。
-- 3. 🔥【智慧靈火秒吸】：打坐時 0.05 秒秒吸靈火，持續維持 2x 灌頂加速。
-- 4. 🌸【自動升級蓮壇】：每 4 秒巡檢金幣，達標自動購買下一階蓮座。
-- 5. ⚡【極限理論倍率】：鎖定境界倍率、本命劍加成、滿額好友同服加成。
-- 6. 👑【八大通行證解鎖】：功力 x2、收入 x2、幸運 x2、VIP、免動畫、防偷、交易、自動靈氣。
-- 7. 🛡️【Anti-AFK】：攔截 20 分鐘閒置斷線。
-- 8. 🎛️【現代化 UI 面板】：自適應滾動、拖曳、折疊、開關全掌控。
-- ========================================================

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

local master = {
    Running = true,
    Connections = {},
    AutoPerfectThrow = true,  -- 🎯 完美投擲輔助開關
    FixMovement = true,       -- 🎮 鍵盤修復開關
    AutoBubble = true,        -- 🔥 靈火秒吸開關
    AutoUpgrade = true,       -- 🌸 自動升級蓮壇開關
    AntiAFK = true,           -- 🛡️ 防踢開關
    BoostLimits = true,       -- ⚡ 極限屬性加成 (境界/本命/好友)
    PassStates = {
        PassDoubleQi = true,      -- 修煉功力 x2
        PassDoubleMoney = true,   -- 法寶收入 x2
        PassDoubleLuck = true,    -- 好運 x2
        PassVIP = true,           -- VIP 1.25x 修煉
        PassSkipReveal = true,    -- 跳過開寶動畫
        PassNoSteal = true,       -- 永久防偷
        PassTradePass = true,     -- 自由交易
        PassAutoQi = true,        -- 自動靈氣
    },
    TotalBoosts = 0,
    TotalUpgrades = 0,
    TotalPerfectThrows = 0,
    LastThrowTime = 0
}

local function applyAllBoosts()
    for attr, state in pairs(master.PassStates) do
        player:SetAttribute(attr, state)
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

local function findMeter()
    local MainUI = player.PlayerGui:FindFirstChild("MainUI")
    if not MainUI then return nil end
    for _, f in ipairs(MainUI:GetChildren()) do
        if f:IsA("Frame") then
            local btn = nil
            for _, c in ipairs(f:GetChildren()) do
                if c:IsA("TextButton") and (c.Text == "發！" or c:GetAttribute("Zh_Text") == "發！" or string.find(c.Text, "發")) then
                    btn = c
                    break
                end
            end
            if btn then
                local barContainer = f:FindFirstChild("Frame")
                local needle = nil
                if barContainer then
                    for _, sub in ipairs(barContainer:GetChildren()) do
                        if sub:IsA("Frame") and sub.Size.X.Offset == 6 then
                            needle = sub
                            break
                        end
                    end
                end
                return f, btn, needle
            end
        end
    end
    return nil
end

local perfectConn = RunService.RenderStepped:Connect(function()
    if not (master.Running and master.AutoPerfectThrow) then return end
    
    local f, btn, needle = findMeter()
    if f and f.Visible and btn and needle then
        local ratio = needle.Position.X.Scale
        local now = os.clock()
        if ratio >= 0.92 and ratio <= 0.99 and (now - master.LastThrowTime) > 0.8 then
            master.LastThrowTime = now
            master.TotalPerfectThrows = master.TotalPerfectThrows + 1
            
            if firesignal then
                pcall(function() firesignal(btn.Activated) end)
                pcall(function() firesignal(btn.MouseButton1Down) end)
                pcall(function() firesignal(btn.MouseButton1Click) end)
            else
                pcall(function()
                    VirtualUser:CaptureController()
                    VirtualUser:ClickButton1(Vector2.new())
                end)
            end
            print(string.format("[修煉大師] 🎯 完美蓄力觸發！蓄力比例: %.1f%% (累積完美一擊: %d 次)", ratio * 100, master.TotalPerfectThrows))
        end
    end
end)
table.insert(master.Connections, perfectConn)

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

local afkConn = player.Idled:Connect(function()
    if not (master.Running and master.AntiAFK) then return end
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
    print("[修煉大師] 🛡️ 攔截 AFK 閒置狀態，保持連線")
end)
table.insert(master.Connections, afkConn)

local guiParent = pcall(function() return CoreGui end) and CoreGui or player:WaitForChild("PlayerGui")
local sg = Instance.new("ScreenGui")
sg.Name = "CultivationMasterUI"
sg.ResetOnSpawn = false
sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
sg.Parent = guiParent

local main = Instance.new("Frame")
main.Name = "Main"
main.Size = UDim2.fromOffset(280, 500)
main.Position = UDim2.new(0, 20, 0.5, -250)
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

local header = Instance.new("Frame")
header.Name = "Header"
header.Size = UDim2.new(1, 0, 0, 38)
header.BackgroundColor3 = Color3.fromRGB(26, 26, 38)
header.BorderSizePixel = 0
header.Parent = main

Instance.new("UICorner", header).CornerRadius = UDim.new(0, 12)

local title = Instance.new("TextLabel")
title.Text = "👑 仙界修煉與完美投擲 v4.1"
title.Size = UDim2.new(1, -70, 1, 0)
title.Position = UDim2.fromOffset(12, 0)
title.BackgroundTransparency = 1
title.TextColor3 = Color3.fromRGB(120, 255, 230)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 15
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

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

makeToggleRow("🎯 完美蓄力投擲", master.AutoPerfectThrow, function(val) master.AutoPerfectThrow = val end)
makeToggleRow("🎮 鍵盤移動修復", master.FixMovement, function(val) master.FixMovement = val end)
makeToggleRow("🔥 靈火秒吸灌頂", master.AutoBubble, function(val) master.AutoBubble = val end)
makeToggleRow("🌸 自動升級蓮壇", master.AutoUpgrade, function(val) master.AutoUpgrade = val end)
makeToggleRow("🛡️ Anti-AFK 防踢", master.AntiAFK, function(val) master.AntiAFK = val end)
makeToggleRow("⚡ 極限修煉倍率", master.BoostLimits, function(val)
    master.BoostLimits = val
    if val then applyAllBoosts() end
end)

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
end

local allBtn = Instance.new("TextButton")
allBtn.Size = UDim2.new(1, 0, 0, 36)
allBtn.BackgroundColor3 = Color3.fromRGB(35, 120, 180)
allBtn.TextColor3 = Color3.new(1, 1, 1)
allBtn.Font = Enum.Font.SourceSansBold
allBtn.TextSize = 14
allBtn.Text = "✨ 一鍵全部解鎖通行證"
allBtn.BorderSizePixel = 0
allBtn.Parent = scroll
Instance.new("UICorner", allBtn).CornerRadius = UDim.new(0, 8)

allBtn.Activated:Connect(function()
    for _, t in pairs(toggles) do
        t.Set(true)
    end
end)

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

local isFolded = false
minBtn.Activated:Connect(function()
    isFolded = not isFolded
    if isFolded then
        main.Size = UDim2.fromOffset(280, 38)
        scroll.Visible = false
        minBtn.Text = "＋"
    else
        main.Size = UDim2.fromOffset(280, 500)
        scroll.Visible = true
        minBtn.Text = "—"
    end
end)

function master:Destroy()
    self.Running = false
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

applyAllBoosts()
getgenv().CultivationMaster = master
getgenv().PassManager = master
getgenv().CultivationTurbo = master
print("[修煉大師] 🌟 全功能修煉加速與完美投擲大師 v4.1 已啟動！")

