--[[ 
    DIAMOND ADM - ULTIMATE EDITION (V11.5 - CUSTOM LAYOUT)
    Optimization Logs:
    - Added Remote Caching (Prevents scanning ReplicatedStorage on every click)
    - Added Save Debounce (Prevents disk lag when typing numbers quickly)
    - Optimized Player List (Updates existing buttons instead of destroying/recreating)
    - Cleaned up Thread Scheduling (Uses task.delay instead of spawn/wait)
    - Maintained full compatibility with DiamondShortcut_UI
    
    Update Log V11.5:
    - Select All agora persiste após reexecutar o script e seleciona players automaticamente.
    - Botões de Slot movidos para baixo da Resolução.
]]

-- ==============================================================================
-- 1. SISTEMA DE SALVAMENTO & CONFIG
-- ==============================================================================
local HttpService = game:GetService("HttpService")
local FileName = "DiamondADM_Config_V11.json" -- Mantendo mesmo arquivo

local COMMANDS = {
    {Name = "JAIL",      Cmd = "jail"},
    {Name = "ROCKET",    Cmd = "rocket"},
    {Name = "RAGDOLL",   Cmd = "ragdoll"},
    {Name = "TINY",      Cmd = "tiny"},
    {Name = "INVERSE",   Cmd = "inverse"},
    {Name = "BALLOON",   Cmd = "balloon"},
    {Name = "JUMPSCARE", Cmd = "jumpscare"},
    {Name = "MORPH",     Cmd = "morph"},
    {Name = "SPIN",      Cmd = "spin"},
    {Name = "UNJAIL",    Cmd = "unjail"},
}

local Config = {
    Slots = {
        [1] = {}, 
        [2] = {}, 
        [3] = {}
    },
    SelectAll = false, 
    Keybind = "None",
    ToggleKey = "P",
    SizeIndex = 2
}

-- Inicializa tabelas
for i = 1, 3 do
    for _, c in ipairs(COMMANDS) do
        Config.Slots[i][c.Cmd] = {Active = false, Delay = 0}
    end
end

local Resolutions = {
    {320, 180, "Small (320x180)"},
    {480, 270, "Medium (480x270)"},
    {640, 360, "Large (640x360)"},
    {800, 450, "X-Large (800x450)"}
}

-- OTIMIZAÇÃO: Debounce no salvamento para evitar lag de disco
local SaveTask = nil
local function RequestSaveConfig()
    if not writefile then return end
    if SaveTask then task.cancel(SaveTask) end
    
    SaveTask = task.delay(0.5, function()
        pcall(function()
            writefile(FileName, HttpService:JSONEncode(Config))
        end)
        SaveTask = nil
    end)
end

local function ForceSaveConfig()
    if writefile then
        pcall(function()
            writefile(FileName, HttpService:JSONEncode(Config))
        end)
    end
end

local function LoadConfig()
    if readfile and isfile and isfile(FileName) then
        pcall(function()
            local decoded = HttpService:JSONDecode(readfile(FileName))
            if decoded then
                if decoded.Slots then 
                    for i = 1, 3 do
                        if decoded.Slots[i] then
                            for k, v in pairs(decoded.Slots[i]) do
                                if Config.Slots[i][k] then Config.Slots[i][k] = v end
                            end
                        end
                    end
                end
                -- Carrega o estado do SelectAll
                if decoded.SelectAll ~= nil then Config.SelectAll = decoded.SelectAll end
                if decoded.Keybind then Config.Keybind = decoded.Keybind end
                if decoded.ToggleKey then Config.ToggleKey = decoded.ToggleKey end
                if decoded.SizeIndex and type(decoded.SizeIndex) == "number" then 
                    Config.SizeIndex = math.clamp(decoded.SizeIndex, 1, #Resolutions) 
                end
            end
        end)
    end
end
LoadConfig()

-- ==============================================================================
-- 2. SERVIÇOS & VARS
-- ==============================================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local LocalPlayer = Players.LocalPlayer

local State = {
    SelectedQueue = {}, 
    EditingSlot = 1,
    MasterKeybind = (Config.Keybind ~= "None" and Enum.KeyCode[Config.Keybind]) or nil, 
    ToggleKeybind = Enum.KeyCode[Config.ToggleKey] or Enum.KeyCode.P,
    IsBindingMaster = false,
    IsBindingToggle = false,
    PanelVisible = true,
    CurrentView = "Main"
}

local function SetCommandExclusive(slotIdx, cmdKey, active)
    if active then
        for i = 1, 3 do
            if i ~= slotIdx then
                Config.Slots[i][cmdKey].Active = false
            end
        end
    end
    Config.Slots[slotIdx][cmdKey].Active = active
end

-- Lógica para selecionar players automaticamente
local function AutoAssignTop3ByDistance()
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    
    -- Se o player ainda não carregou, tentamos pegar a posição da câmera ou esperar um pouco
    if not myRoot then return end

    local others = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local dist = (p.Character.HumanoidRootPart.Position - myRoot.Position).Magnitude
            table.insert(others, {Player = p, Dist = dist})
        end
    end

    table.sort(others, function(a, b) return a.Dist < b.Dist end)

    State.SelectedQueue = {}
    for i = 1, 3 do
        if others[i] then
            table.insert(State.SelectedQueue, others[i].Player)
        end
    end
end

-- Se já estiver ativado no load, seleciona imediatamente
if Config.SelectAll then
    AutoAssignTop3ByDistance()
end

-- Limpeza Limpa
for _, gui in ipairs(CoreGui:GetChildren()) do
    if gui.Name == "DiamondADM_UI" or gui.Name == "LazuritaADM_MultiSlot" then gui:Destroy() end
end

-- ==============================================================================
-- 3. TEMA
-- ==============================================================================
local Theme = {
    Background = Color3.fromHex("#0d1117"), 
    Header     = Color3.fromHex("#161b22"), 
    Element    = Color3.fromHex("#21262d"),     
    Text       = Color3.fromRGB(240, 240, 240),
    TextDim    = Color3.fromRGB(139, 148, 158), 
    Accent     = Color3.fromHex("#2d89ef"),
    Slot1      = Color3.fromHex("#2d89ef"), 
    Slot2      = Color3.fromHex("#a02def"), 
    Slot3      = Color3.fromHex("#ef892d"), 
    Positive   = Color3.fromRGB(50, 220, 100), 
    Negative   = Color3.fromRGB(220, 60, 60),
    Transparency = 0.05,
}

local SlotColors = {
    [1] = Theme.Slot1,
    [2] = Theme.Slot2,
    [3] = Theme.Slot3
}

-- ==============================================================================
-- 4. FUNÇÕES LÓGICAS & SHARE CODE
-- ==============================================================================
local CachedRemote = nil
local function GetCachedRemote()
    if CachedRemote and CachedRemote.Parent then
        return CachedRemote
    end
    for _, d in ipairs(ReplicatedStorage:GetDescendants()) do
        if d.Name:lower():find("executecommand") and (d:IsA("RemoteEvent") or d:IsA("RemoteFunction")) then
            CachedRemote = d
            return d
        end
    end
    return nil
end

local function ExecuteCommand(player, cmd)
    local remote = GetCachedRemote()
    if remote and player and player.Parent then
        pcall(function() remote:FireServer(player, cmd) end)
    end
end

local function TriggerMasterAction()
    for i, player in ipairs(State.SelectedQueue) do
        if i <= 3 and Config.Slots[i] then
            local currentConfig = Config.Slots[i]
            for cmdKey, data in pairs(currentConfig) do
                if data.Active then
                    local delayTime = tonumber(data.Delay) or 0
                    if delayTime > 0 then
                        task.delay(delayTime, function()
                            if player and player.Parent then
                                ExecuteCommand(player, cmdKey)
                            end
                        end)
                    else
                        ExecuteCommand(player, cmdKey)
                    end
                end
            end
        end
    end
end

local CodePrefix = "DiamondADM-"

local function GenerateShareCode()
    local parts = {}
    for i = 1, 3 do
        local slotParts = {}
        for _, cmd in ipairs(COMMANDS) do
            local d = Config.Slots[i][cmd.Cmd]
            local prefix = d.Active and "1" or "0"
            local delayVal = tostring(math.floor(d.Delay))
            table.insert(slotParts, prefix .. delayVal)
        end
        table.insert(parts, table.concat(slotParts, ","))
    end
    return CodePrefix .. table.concat(parts, "|")
end

local function LoadShareCode(codeStr)
    if string.sub(codeStr, 1, #CodePrefix) ~= CodePrefix then return false end
    codeStr = string.sub(codeStr, #CodePrefix + 1)
    local slotStrings = string.split(codeStr, "|")
    if #slotStrings ~= 3 then return false end
    for i = 1, 3 do
        local cmds = string.split(slotStrings[i], ",")
        if #cmds == #COMMANDS then
            for idx, val in ipairs(cmds) do
                local cmdName = COMMANDS[idx].Cmd
                local stateChar = string.sub(val, 1, 1)
                local delayStr = string.sub(val, 2)
                Config.Slots[i][cmdName].Active = (stateChar == "1")
                Config.Slots[i][cmdName].Delay = tonumber(delayStr) or 0
            end
        end
    end
    ForceSaveConfig()
    return true
end

-- ==============================================================================
-- 5. UI PRINCIPAL
-- ==============================================================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "DiamondADM_UI" 
ScreenGui.Parent = CoreGui
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
local initialSize = Resolutions[Config.SizeIndex] or Resolutions[2]
MainFrame.Size = UDim2.new(0, initialSize[1], 0, initialSize[2])
MainFrame.AnchorPoint = Vector2.new(1, 0) 
MainFrame.Position = UDim2.new(1, -30, 0, 30) 
MainFrame.BackgroundColor3 = Theme.Background
MainFrame.BackgroundTransparency = Theme.Transparency
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 6)
UICorner.Parent = MainFrame

local UIStroke = Instance.new("UIStroke")
UIStroke.Color = Theme.Accent 
UIStroke.Thickness = 1
UIStroke.Transparency = 0
UIStroke.Parent = MainFrame

-- HEADER
local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 35)
Header.BackgroundColor3 = Theme.Header
Header.BackgroundTransparency = 0
Header.BorderSizePixel = 0
Header.Parent = MainFrame
Instance.new("UICorner", Header).CornerRadius = UDim.new(0, 6)

local TitleContainer = Instance.new("Frame")
TitleContainer.Size = UDim2.new(0, 250, 1, 0)
TitleContainer.Position = UDim2.new(0, 15, 0, 0)
TitleContainer.BackgroundTransparency = 1
TitleContainer.Parent = Header

local TitleLayout = Instance.new("UIListLayout")
TitleLayout.FillDirection = Enum.FillDirection.Horizontal
TitleLayout.SortOrder = Enum.SortOrder.LayoutOrder
TitleLayout.Padding = UDim.new(0, 5) 
TitleLayout.VerticalAlignment = Enum.VerticalAlignment.Center
TitleLayout.Parent = TitleContainer

local DiamondLbl = Instance.new("TextLabel")
DiamondLbl.Text = "Diamond"
DiamondLbl.Font = Enum.Font.GothamBlack
DiamondLbl.TextSize = 20
DiamondLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
DiamondLbl.BackgroundTransparency = 1
DiamondLbl.AutomaticSize = Enum.AutomaticSize.X
DiamondLbl.Size = UDim2.new(0, 0, 1, 0)
DiamondLbl.Parent = TitleContainer

local UIGradient = Instance.new("UIGradient")
UIGradient.Rotation = 45 
UIGradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0.00, Theme.Accent),
    ColorSequenceKeypoint.new(0.20, Theme.Accent),                  
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(255, 255, 255)), 
    ColorSequenceKeypoint.new(0.80, Theme.Accent),                  
    ColorSequenceKeypoint.new(1.00, Theme.Accent)
}
UIGradient.Parent = DiamondLbl

task.spawn(function()
    while DiamondLbl.Parent do
        local tween = TweenService:Create(UIGradient, TweenInfo.new(2.0, Enum.EasingStyle.Linear), {Offset = Vector2.new(1, 0)})
        UIGradient.Offset = Vector2.new(-1, 0) 
        tween:Play()
        tween.Completed:Wait()
        task.wait(0.1)
    end
end)

local AdmLbl = Instance.new("TextLabel")
AdmLbl.Text = "ADM"
AdmLbl.Font = Enum.Font.GothamBlack
AdmLbl.TextSize = 20
AdmLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
AdmLbl.BackgroundTransparency = 1
AdmLbl.AutomaticSize = Enum.AutomaticSize.X
AdmLbl.Size = UDim2.new(0, 0, 1, 0)
AdmLbl.Parent = TitleContainer

local ConfigBtn = Instance.new("TextButton")
ConfigBtn.Size = UDim2.new(0, 35, 0, 35)
ConfigBtn.Position = UDim2.new(1, -35, 0, 0)
ConfigBtn.BackgroundTransparency = 1
ConfigBtn.Text = "⚙"
ConfigBtn.TextColor3 = Theme.TextDim
ConfigBtn.TextSize = 20
ConfigBtn.Font = Enum.Font.Gotham
ConfigBtn.Parent = Header

local ViewContainer = Instance.new("Frame")
ViewContainer.Size = UDim2.new(1, 0, 1, -35)
ViewContainer.Position = UDim2.new(0, 0, 0, 35)
ViewContainer.BackgroundTransparency = 1
ViewContainer.Parent = MainFrame

-- ==============================================================================
-- VISÃO 1: LISTA (Main)
-- ==============================================================================
local MainView = Instance.new("Frame")
MainView.Size = UDim2.new(1, 0, 1, 0)
MainView.BackgroundTransparency = 1
MainView.Visible = true
MainView.Parent = ViewContainer

local ScrollList = Instance.new("ScrollingFrame")
ScrollList.Size = UDim2.new(1, -20, 1, -60) 
ScrollList.Position = UDim2.new(0, 10, 0, 10)
ScrollList.BackgroundTransparency = 1
ScrollList.BorderSizePixel = 0
ScrollList.ScrollBarThickness = 4
ScrollList.ScrollBarImageColor3 = Theme.Element
ScrollList.Parent = MainView

local ListLayout = Instance.new("UIListLayout")
ListLayout.Padding = UDim.new(0, 4)
ListLayout.SortOrder = Enum.SortOrder.Name
ListLayout.Parent = ScrollList

local SelectAllBtn = Instance.new("TextButton")
SelectAllBtn.Name = "00_SelectAll"
SelectAllBtn.Size = UDim2.new(1, 0, 0, 26)
SelectAllBtn.BackgroundColor3 = Theme.Element 
SelectAllBtn.BackgroundTransparency = 0
SelectAllBtn.Text = "Select All"
SelectAllBtn.Font = Enum.Font.GothamMedium
SelectAllBtn.TextSize = 12
SelectAllBtn.Parent = ScrollList
Instance.new("UICorner", SelectAllBtn).CornerRadius = UDim.new(0, 4)

-- Aplica o estado inicial visual do botão Select All
if Config.SelectAll then
    SelectAllBtn.TextColor3 = Theme.Accent
    SelectAllBtn.BackgroundColor3 = Color3.fromRGB(Theme.Element.R*255+10, Theme.Element.G*255+10, Theme.Element.B*255+10)
else
    SelectAllBtn.TextColor3 = Theme.Text
    SelectAllBtn.BackgroundColor3 = Theme.Element
end

local function GetPlayerSlot(p)
    for i, pl in ipairs(State.SelectedQueue) do
        if pl == p then return i end
    end
    return nil
end

local function UpdatePlayerList()
    local seen = {}
    
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then
            seen[p.Name] = true
            local btn = ScrollList:FindFirstChild(p.Name)
            
            if not btn then
                btn = Instance.new("TextButton")
                btn.Name = p.Name
                btn.Size = UDim2.new(1, 0, 0, 26)
                btn.Text = "  " .. p.DisplayName
                btn.Font = Enum.Font.Gotham
                btn.TextSize = 13
                btn.TextXAlignment = Enum.TextXAlignment.Left
                btn.AutoButtonColor = false
                btn.Parent = ScrollList
                Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
                
                local slotInd = Instance.new("TextLabel")
                slotInd.Name = "SlotInd"
                slotInd.Size = UDim2.new(0, 40, 1, 0)
                slotInd.Position = UDim2.new(1, -5, 0, 0)
                slotInd.AnchorPoint = Vector2.new(1, 0)
                slotInd.BackgroundTransparency = 1
                slotInd.TextXAlignment = Enum.TextXAlignment.Right
                slotInd.Font = Enum.Font.Code
                slotInd.TextSize = 12
                slotInd.Parent = btn
                
                btn.MouseButton1Click:Connect(function()
                    local currentSlot = GetPlayerSlot(p)
                    if currentSlot then
                        table.remove(State.SelectedQueue, currentSlot)
                    else
                        table.insert(State.SelectedQueue, p)
                        if #State.SelectedQueue > 3 then
                            table.remove(State.SelectedQueue, 1)
                        end
                    end
                    
                    if Config.SelectAll then
                        Config.SelectAll = false
                        SelectAllBtn.TextColor3 = Theme.Text
                        SelectAllBtn.BackgroundColor3 = Theme.Element
                    end
                    UpdatePlayerList()
                end)
            end
            
            local slotInd = btn:FindFirstChild("SlotInd")
            local bar = btn:FindFirstChild("Bar") or Instance.new("Frame")
            
            if not btn:FindFirstChild("Bar") then
                bar.Name = "Bar"
                bar.Size = UDim2.new(0, 3, 1, 0)
                bar.BorderSizePixel = 0
                bar.Parent = btn
                Instance.new("UICorner", bar).CornerRadius = UDim.new(0, 2)
            end
            
            local s = GetPlayerSlot(p)
            if s then
                local color = SlotColors[s] or Theme.Accent
                btn.BackgroundColor3 = Theme.Element
                btn.TextColor3 = color
                btn.BackgroundTransparency = 0
                if slotInd then 
                    slotInd.Text = "["..s.."]"
                    slotInd.TextColor3 = color 
                end
                bar.Visible = true
                bar.BackgroundColor3 = color
            else
                btn.BackgroundColor3 = Theme.Background
                btn.TextColor3 = Color3.fromRGB(255, 255, 255) 
                btn.BackgroundTransparency = 1
                if slotInd then slotInd.Text = "" end
                bar.Visible = false
            end
        end
    end
    
    for _, child in pairs(ScrollList:GetChildren()) do
        if child:IsA("TextButton") and child.Name ~= "00_SelectAll" and not seen[child.Name] then
            child:Destroy()
        end
    end
end

SelectAllBtn.MouseButton1Click:Connect(function()
    Config.SelectAll = not Config.SelectAll
    RequestSaveConfig()
    if Config.SelectAll then
        SelectAllBtn.TextColor3 = Theme.Accent
        SelectAllBtn.BackgroundColor3 = Color3.fromRGB(Theme.Element.R*255+10, Theme.Element.G*255+10, Theme.Element.B*255+10)
        AutoAssignTop3ByDistance() 
    else
        SelectAllBtn.TextColor3 = Theme.Text
        SelectAllBtn.BackgroundColor3 = Theme.Element
        State.SelectedQueue = {}
    end
    UpdatePlayerList()
end)

Players.PlayerAdded:Connect(function(p)
    if Config.SelectAll then AutoAssignTop3ByDistance() end
    UpdatePlayerList()
end)
Players.PlayerRemoving:Connect(UpdatePlayerList)
UpdatePlayerList()

local ExecuteBtn = Instance.new("TextButton")
ExecuteBtn.Size = UDim2.new(1, -20, 0, 38)
ExecuteBtn.Position = UDim2.new(0, 10, 1, -45)
ExecuteBtn.BackgroundColor3 = Theme.Accent 
ExecuteBtn.BackgroundTransparency = 0
ExecuteBtn.Text = "Execute"
ExecuteBtn.Font = Enum.Font.GothamBold
ExecuteBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ExecuteBtn.TextSize = 14
ExecuteBtn.TextYAlignment = Enum.TextYAlignment.Center
ExecuteBtn.Parent = MainView
Instance.new("UICorner", ExecuteBtn).CornerRadius = UDim.new(0, 4)

local BindBtnMaster = Instance.new("TextButton")
BindBtnMaster.Size = UDim2.new(0, 100, 1, 0)
BindBtnMaster.Position = UDim2.new(1, -5, 0, 0)
BindBtnMaster.AnchorPoint = Vector2.new(1, 0)
BindBtnMaster.BackgroundTransparency = 1
BindBtnMaster.Font = Enum.Font.Code
BindBtnMaster.TextColor3 = Color3.fromRGB(255, 255, 255)
BindBtnMaster.TextTransparency = 0.5
BindBtnMaster.TextSize = 12
BindBtnMaster.TextXAlignment = Enum.TextXAlignment.Right
BindBtnMaster.Parent = ExecuteBtn

local function UpdateBindText()
    if Config.Keybind == "None" then
        BindBtnMaster.Text = "[ Bind ] "
    else
        BindBtnMaster.Text = "[" .. Config.Keybind .. "] "
    end
end
UpdateBindText()

BindBtnMaster.MouseButton1Click:Connect(function()
    State.IsBindingMaster = true
    BindBtnMaster.Text = "[...]"
    BindBtnMaster.TextColor3 = Theme.Positive
    BindBtnMaster.TextTransparency = 0
end)

ExecuteBtn.MouseButton1Click:Connect(function()
    TriggerMasterAction()
    ExecuteBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ExecuteBtn.TextColor3 = Theme.Accent
    task.wait(0.1)
    ExecuteBtn.BackgroundColor3 = Theme.Accent
    ExecuteBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
end)

-- ==============================================================================
-- VISÃO 2: CONFIG
-- ==============================================================================
local ConfigView = Instance.new("Frame")
ConfigView.Size = UDim2.new(1, 0, 1, 0)
ConfigView.BackgroundTransparency = 1
ConfigView.Visible = false
ConfigView.Parent = ViewContainer

local ConfigScroll = Instance.new("ScrollingFrame")
ConfigScroll.Size = UDim2.new(1, -20, 1, -20)
ConfigScroll.Position = UDim2.new(0, 10, 0, 10)
ConfigScroll.BackgroundTransparency = 1
ConfigScroll.BorderSizePixel = 0
ConfigScroll.ScrollBarThickness = 4
ConfigScroll.ScrollBarImageColor3 = Theme.Element
ConfigScroll.Parent = ConfigView

local ConfigLayout = Instance.new("UIListLayout")
ConfigLayout.SortOrder = Enum.SortOrder.LayoutOrder
ConfigLayout.Padding = UDim.new(0, 10)
ConfigLayout.Parent = ConfigScroll

-- CONFIG CONTAINER (Settings)
local SettingsContainer = Instance.new("Frame")
SettingsContainer.Size = UDim2.new(1, 0, 0, 0) 
SettingsContainer.AutomaticSize = Enum.AutomaticSize.Y
SettingsContainer.BackgroundColor3 = Theme.Element
SettingsContainer.BackgroundTransparency = 0.5
SettingsContainer.LayoutOrder = 1
SettingsContainer.Parent = ConfigScroll
Instance.new("UICorner", SettingsContainer).CornerRadius = UDim.new(0, 6)

local SCLayout = Instance.new("UIListLayout")
SCLayout.Padding = UDim.new(0, 5)
SCLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
SCLayout.VerticalAlignment = Enum.VerticalAlignment.Center
SCLayout.Parent = SettingsContainer

-- Padding Top
local PTop = Instance.new("Frame")
PTop.Size = UDim2.new(1, 0, 0, 2)
PTop.BackgroundTransparency = 1
PTop.Parent = SettingsContainer

-- Toggle Row
local ToggleRow = Instance.new("Frame")
ToggleRow.Size = UDim2.new(0.95, 0, 0, 25)
ToggleRow.BackgroundTransparency = 1
ToggleRow.Parent = SettingsContainer

local TRLbl = Instance.new("TextLabel")
TRLbl.Size = UDim2.new(0.6, 0, 1, 0)
TRLbl.BackgroundTransparency = 1
TRLbl.Text = "Menu Toggle Key"
TRLbl.Font = Enum.Font.GothamMedium
TRLbl.TextColor3 = Theme.Text
TRLbl.TextXAlignment = Enum.TextXAlignment.Left
TRLbl.TextSize = 12
TRLbl.Parent = ToggleRow

local TRBtn = Instance.new("TextButton")
TRBtn.Size = UDim2.new(0.3, 0, 1, 0)
TRBtn.Position = UDim2.new(0.7, 0, 0, 0)
TRBtn.BackgroundColor3 = Theme.Header
TRBtn.Text = "[" .. Config.ToggleKey .. "]"
TRBtn.TextColor3 = Theme.Accent
TRBtn.Font = Enum.Font.Code
TRBtn.TextSize = 12
TRBtn.Parent = ToggleRow
Instance.new("UICorner", TRBtn).CornerRadius = UDim.new(0, 4)

TRBtn.MouseButton1Click:Connect(function()
    State.IsBindingToggle = true
    TRBtn.Text = "..."
    TRBtn.TextColor3 = Theme.Positive
end)

-- Resolution Row
local ResRow = Instance.new("Frame")
ResRow.Size = UDim2.new(0.95, 0, 0, 25)
ResRow.BackgroundTransparency = 1
ResRow.Parent = SettingsContainer

local RRLbl = Instance.new("TextLabel")
RRLbl.Size = UDim2.new(0.4, 0, 1, 0)
RRLbl.BackgroundTransparency = 1
RRLbl.Text = "Menu Size"
RRLbl.Font = Enum.Font.GothamMedium
RRLbl.TextColor3 = Theme.Text
RRLbl.TextXAlignment = Enum.TextXAlignment.Left
RRLbl.TextSize = 12
RRLbl.Parent = ResRow

local RRBtn = Instance.new("TextButton")
RRBtn.Size = UDim2.new(0.5, 0, 1, 0)
RRBtn.Position = UDim2.new(0.5, 0, 0, 0)
RRBtn.BackgroundColor3 = Theme.Header
RRBtn.Text = Resolutions[Config.SizeIndex][3]
RRBtn.TextColor3 = Theme.TextDim
RRBtn.Font = Enum.Font.Gotham
RRBtn.TextSize = 10
RRBtn.Parent = ResRow
Instance.new("UICorner", RRBtn).CornerRadius = UDim.new(0, 4)

RRBtn.MouseButton1Click:Connect(function()
    Config.SizeIndex = Config.SizeIndex + 1
    if Config.SizeIndex > #Resolutions then Config.SizeIndex = 1 end
    local res = Resolutions[Config.SizeIndex]
    RRBtn.Text = res[3]
    MainFrame.Size = UDim2.new(0, res[1], 0, res[2])
    ForceSaveConfig()
end)

-- MOVED: Slot Selection Buttons (Now inside SettingsContainer, under Resolution)
local SlotSelContainer = Instance.new("Frame")
SlotSelContainer.Size = UDim2.new(0.95, 0, 0, 30)
SlotSelContainer.BackgroundTransparency = 1
SlotSelContainer.Parent = SettingsContainer -- Parented here instead of Scroll

local SlotGrid = Instance.new("UIGridLayout")
SlotGrid.CellSize = UDim2.new(0.3, 0, 1, 0)
SlotGrid.CellPadding = UDim2.new(0.05, 0, 0, 0)
SlotGrid.Parent = SlotSelContainer

local SlotBtns = {}
local function RefreshTrollGrid() end

for i = 1, 3 do
    local sBtn = Instance.new("TextButton")
    sBtn.Text = "Slot " .. i
    sBtn.Font = Enum.Font.GothamBold
    sBtn.TextSize = 12
    sBtn.Parent = SlotSelContainer
    Instance.new("UICorner", sBtn).CornerRadius = UDim.new(0, 4)
    
    sBtn.MouseButton1Click:Connect(function()
        State.EditingSlot = i
        RefreshTrollGrid()
    end)
    SlotBtns[i] = sBtn
end

-- Padding Bottom
local PBot = Instance.new("Frame")
PBot.Size = UDim2.new(1, 0, 0, 2)
PBot.BackgroundTransparency = 1
PBot.Parent = SettingsContainer


-- Trolls List
local TrollsContainer = Instance.new("Frame")
TrollsContainer.Size = UDim2.new(1, 0, 0, 0) 
TrollsContainer.AutomaticSize = Enum.AutomaticSize.Y
TrollsContainer.BackgroundTransparency = 1
TrollsContainer.LayoutOrder = 2
TrollsContainer.Parent = ConfigScroll

local TrollGrid = Instance.new("UIGridLayout")
TrollGrid.CellSize = UDim2.new(0, 140, 0, 30) 
TrollGrid.CellPadding = UDim2.new(0, 8, 0, 8)
TrollGrid.HorizontalAlignment = Enum.HorizontalAlignment.Center
TrollGrid.Parent = TrollsContainer

local TrollButtons = {} 

for _, cmd in ipairs(COMMANDS) do
    local Btn = Instance.new("TextButton")
    Btn.BackgroundColor3 = Theme.Element
    Btn.Text = ""
    Btn.AutoButtonColor = false
    Btn.Parent = TrollsContainer
    Instance.new("UICorner", Btn).CornerRadius = UDim.new(0, 4)
    
    local Bar = Instance.new("Frame")
    Bar.Size = UDim2.new(0, 4, 1, 0)
    Bar.BorderSizePixel = 0
    Bar.Parent = Btn
    Instance.new("UICorner", Bar).CornerRadius = UDim.new(0, 2)

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(0.6, 0, 1, 0)
    Label.Position = UDim2.new(0, 10, 0, 0)
    Label.BackgroundTransparency = 1
    Label.Text = cmd.Name
    Label.Font = Enum.Font.GothamMedium
    Label.TextSize = 11
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.TextColor3 = Theme.TextDim
    Label.Parent = Btn

    local TimeInput = Instance.new("TextBox")
    TimeInput.Size = UDim2.new(0.3, 0, 0.8, 0)
    TimeInput.Position = UDim2.new(0.68, 0, 0.1, 0)
    TimeInput.BackgroundColor3 = Theme.Background
    TimeInput.BackgroundTransparency = 0.5
    TimeInput.PlaceholderText = "0"
    TimeInput.TextColor3 = Theme.Accent
    TimeInput.Font = Enum.Font.Code
    TimeInput.TextSize = 11
    TimeInput.Parent = Btn
    Instance.new("UICorner", TimeInput).CornerRadius = UDim.new(0, 4)

    TrollButtons[cmd.Cmd] = {
        Btn = Btn,
        Bar = Bar,
        Label = Label,
        Input = TimeInput
    }

    Btn.MouseButton1Click:Connect(function()
        local s = State.EditingSlot
        local current = Config.Slots[s][cmd.Cmd]
        
        if not current.Active then
            SetCommandExclusive(s, cmd.Cmd, true)
        else
            SetCommandExclusive(s, cmd.Cmd, false)
        end
        RequestSaveConfig()
        RefreshTrollGrid() 
    end)

    TimeInput:GetPropertyChangedSignal("Text"):Connect(function()
        if TimeInput:IsFocused() then
            local s = State.EditingSlot
            local filtered = TimeInput.Text:gsub("[^0-9%.]", "")
            local val = tonumber(filtered) or 0
            Config.Slots[s][cmd.Cmd].Delay = val
        end
    end)

    TimeInput.FocusLost:Connect(function()
        local s = State.EditingSlot
        local val = Config.Slots[s][cmd.Cmd].Delay
        TimeInput.Text = tostring(val)
        RequestSaveConfig()
    end)
end

-- Share Code
local ShareContainer = Instance.new("Frame")
ShareContainer.Size = UDim2.new(1, 0, 0, 60)
ShareContainer.BackgroundColor3 = Theme.Element
ShareContainer.BackgroundTransparency = 0.5
ShareContainer.LayoutOrder = 3
ShareContainer.Parent = ConfigScroll
Instance.new("UICorner", ShareContainer).CornerRadius = UDim.new(0, 6)

local CodeInput = Instance.new("TextBox")
CodeInput.Size = UDim2.new(0.9, 0, 0, 25)
CodeInput.Position = UDim2.new(0.05, 0, 0.1, 0)
CodeInput.BackgroundColor3 = Theme.Background
CodeInput.Text = ""
CodeInput.PlaceholderText = "Paste Code Here (DiamondADM-...)"
CodeInput.TextColor3 = Theme.Text
CodeInput.Font = Enum.Font.Code
CodeInput.TextSize = 11
CodeInput.Parent = ShareContainer
Instance.new("UICorner", CodeInput).CornerRadius = UDim.new(0, 4)

local CopyBtn = Instance.new("TextButton")
CopyBtn.Size = UDim2.new(0.42, 0, 0, 20)
CopyBtn.Position = UDim2.new(0.05, 0, 0.6, 0)
CopyBtn.BackgroundColor3 = Theme.Header
CopyBtn.Text = "Copy Code"
CopyBtn.TextColor3 = Theme.Accent
CopyBtn.Font = Enum.Font.Gotham
CopyBtn.TextSize = 11
CopyBtn.Parent = ShareContainer
Instance.new("UICorner", CopyBtn).CornerRadius = UDim.new(0, 4)

local LoadBtn = Instance.new("TextButton")
LoadBtn.Size = UDim2.new(0.42, 0, 0, 20)
LoadBtn.Position = UDim2.new(0.53, 0, 0.6, 0)
LoadBtn.BackgroundColor3 = Theme.Header
LoadBtn.Text = "Load Code"
LoadBtn.TextColor3 = Theme.Positive
LoadBtn.Font = Enum.Font.Gotham
LoadBtn.TextSize = 11
LoadBtn.Parent = ShareContainer
Instance.new("UICorner", LoadBtn).CornerRadius = UDim.new(0, 4)

CopyBtn.MouseButton1Click:Connect(function()
    local c = GenerateShareCode()
    if setclipboard then 
        setclipboard(c)
        CopyBtn.Text = "Copied!"
        task.wait(1)
        CopyBtn.Text = "Copy Code"
    else
        CodeInput.Text = c
    end
end)

LoadBtn.MouseButton1Click:Connect(function()
    local s = LoadShareCode(CodeInput.Text)
    if s then
        LoadBtn.Text = "Loaded!"
        RefreshTrollGrid()
    else
        LoadBtn.Text = "Invalid!"
    end
    task.wait(1)
    LoadBtn.Text = "Load Code"
end)

function RefreshTrollGrid()
    local currentSlot = State.EditingSlot
    local currentConfig = Config.Slots[currentSlot]
    local color = SlotColors[currentSlot]

    for i, btn in ipairs(SlotBtns) do
        if i == currentSlot then
            btn.BackgroundColor3 = SlotColors[i]
            btn.TextColor3 = Color3.new(1,1,1)
            btn.BackgroundTransparency = 0
        else
            btn.BackgroundColor3 = Theme.Element
            btn.TextColor3 = Theme.TextDim
            btn.BackgroundTransparency = 0.5
        end
    end

    for cmd, objs in pairs(TrollButtons) do
        local data = currentConfig[cmd]
        if not objs.Input:IsFocused() then
            objs.Input.Text = tostring(data.Delay)
        end
        
        if data.Active then
            objs.Btn.BackgroundColor3 = Color3.fromRGB(Theme.Element.R*255+10, Theme.Element.G*255+10, Theme.Element.B*255+10)
            objs.Label.TextColor3 = Theme.Text
            objs.Bar.BackgroundColor3 = Theme.Positive 
        else
            objs.Btn.BackgroundColor3 = Theme.Element
            objs.Label.TextColor3 = Theme.TextDim
            objs.Bar.BackgroundColor3 = Theme.Background
        end
    end
end
RefreshTrollGrid()

ConfigBtn.MouseButton1Click:Connect(function()
    if State.CurrentView == "Main" then
        State.CurrentView = "Config"
        MainView.Visible = false
        ConfigView.Visible = true
        ConfigBtn.TextColor3 = Theme.Accent
        RefreshTrollGrid()
    else
        State.CurrentView = "Main"
        ConfigView.Visible = false
        MainView.Visible = true
        ConfigBtn.TextColor3 = Theme.TextDim
    end
end)

local dragging, dragInput, dragStart, startPos
Header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
Header.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then dragInput = input end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end

    if State.IsBindingMaster and input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.Backspace or input.KeyCode == Enum.KeyCode.Delete then
            Config.Keybind = "None"
            State.MasterKeybind = nil
            UpdateBindText()
        else
            State.MasterKeybind = input.KeyCode
            Config.Keybind = input.KeyCode.Name
            UpdateBindText()
        end
        BindBtnMaster.TextColor3 = Color3.fromRGB(255,255,255)
        BindBtnMaster.TextTransparency = 0.5
        State.IsBindingMaster = false
        ForceSaveConfig()
        return
    end

    if State.IsBindingToggle and input.UserInputType == Enum.UserInputType.Keyboard then
        State.ToggleKeybind = input.KeyCode
        Config.ToggleKey = input.KeyCode.Name
        TRBtn.Text = "[" .. Config.ToggleKey .. "]"
        TRBtn.TextColor3 = Theme.Accent
        State.IsBindingToggle = false
        ForceSaveConfig()
        return
    end

    if State.MasterKeybind and input.KeyCode == State.MasterKeybind then
        TriggerMasterAction()
        ExecuteBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        ExecuteBtn.TextColor3 = Theme.Accent
        task.wait(0.1)
        ExecuteBtn.BackgroundColor3 = Theme.Accent
        ExecuteBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end

    if input.KeyCode == State.ToggleKeybind and not State.IsBindingMaster and not State.IsBindingToggle then
        State.PanelVisible = not State.PanelVisible
        MainFrame.Visible = State.PanelVisible
    end
end)
