--[[ 
  Optimized GPT UI Script with Backup Teleport Reactivation, Adjusted Layout, 
  Persistent Forced Mouse Unlock/Zoom, and Persistence.
  
  Features:
    • Tracker GUIs (spawned by entering a part name)
      - Teleport tracked parts to player (customizable delay & limit)
      - Unanchor tracked parts
      - Toggle collision for tracked parts
    • Fly Script
    • Show Items (with remote/module script check)
    • Radius teleportation (customizable hitbox, teleport sidebar)
    • Infinite Health
    • Show Inventory (CoreGui Backpack)
    • Classic Sword (built from parts with guard, custom sizing, slash animation)
    • Restore Jump
    • Value Override (scan for NumberValue objects, toggle selection, set new value)
    • Page 4: Highlight Mode & Click Activator
         - Highlight Mode: Toggles a SelectionBox around the part under the mouse and displays its name.
         - Click Activator: Opens a GUI listing all parts with ClickDetectors and fires them when toggled.
    • Persistence: On teleport, a minimal bootstrap GUI appears. When its button is clicked 
         (or when the "=" key is pressed), the camera is set for free zoom, the mouse is unlocked,
         and the full GPT UI is loaded from your public GitHub README.
    • Backup: If queue_on_teleport is unavailable, a CharacterAdded event is used.
    • Forced Mouse Unlock: Once the "=" key is pressed at any time, a persistent heartbeat forces
         the mouse to unlock and the camera to allow unlimited zoom.
--]]

-- Persistent flag: if the GUI was manually closed, do not reload.
if getgenv().GuiClosed then
    return
end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer

-- Create and name the main ScreenGui so we can check for it later.
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "GPTUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

-- Global variables
local hitboxIndicator, baseIndicatorSize
local radiusToggled = true
local activeTargets, outlinedAdorns, sidebarEntries = {}, {}, {}
local teleportDelay, teleportDuration = 0, 0.1
local infiniteHealth = false

-- Helper: Create a standard delete button.
local function createDeleteButton(parent)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0,30,0,30)
    btn.Position = UDim2.new(0,0,0,0)
    btn.Text = "X"
    btn.BackgroundColor3 = Color3.fromRGB(255,0,0)
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Font = Enum.Font.SourceSansBold
    btn.TextSize = 20
    btn.Parent = parent
    btn.MouseButton1Click:Connect(function()
        parent:Destroy()
        if parent == mainFrame then
            getgenv().GuiClosed = true
        end
    end)
end

-- Main GUI frame
local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0,250,0,250)
mainFrame.Position = UDim2.new(0.5,-125,0.05,0)
mainFrame.BackgroundColor3 = Color3.fromRGB(50,50,50)
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

createDeleteButton(mainFrame)

-- Title label
local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1,-35,0,30)
titleLabel.Position = UDim2.new(0,35,0,0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "GPT UI"
titleLabel.TextColor3 = Color3.new(1,1,1)
titleLabel.Font = Enum.Font.SourceSansBold
titleLabel.TextSize = 20
titleLabel.Parent = mainFrame

-- Create 4 pages
local pages = {}
for i = 1,4 do
    local page = Instance.new("Frame")
    page.Name = "Page" .. i
    page.Size = UDim2.new(1,0,1,-70)
    page.Position = UDim2.new(0,0,0,30)
    page.BackgroundTransparency = 1
    page.Visible = (i == 1)
    page.Parent = mainFrame
    pages[i] = page
end

-- Page navigation bar (4 buttons)
local pageNav = Instance.new("Frame")
pageNav.Size = UDim2.new(1,0,0,40)
pageNav.Position = UDim2.new(0,0,1,-40)
pageNav.BackgroundTransparency = 1
pageNav.Parent = mainFrame

local navButtons = {}
for i = 1,4 do
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0,30,1,0)
    btn.Position = UDim2.new(0,(i-1)*40 + 10,0,0)
    btn.Text = tostring(i)
    btn.BackgroundColor3 = (i==1) and Color3.fromRGB(0,255,0) or Color3.fromRGB(100,100,100)
    btn.Parent = pageNav
    btn.MouseButton1Click:Connect(function()
        for j = 1,4 do
            pages[j].Visible = (j == i)
            navButtons[j].BackgroundColor3 = (j == i) and Color3.fromRGB(0,255,0) or Color3.fromRGB(100,100,100)
        end
    end)
    navButtons[i] = btn
end

-----------------------------------------------------------
-- PAGE 1: Tracker GUI, Fly Script, Show Items
-----------------------------------------------------------
local page1 = pages[1]

local trackerTextBox = Instance.new("TextBox")
trackerTextBox.Size = UDim2.new(1,-20,0,50)
trackerTextBox.Position = UDim2.new(0,10,0,10)
trackerTextBox.PlaceholderText = "Enter part name"
trackerTextBox.Parent = page1

local flyScriptButton = Instance.new("TextButton")
flyScriptButton.Size = UDim2.new(0,100,0,30)
flyScriptButton.Position = UDim2.new(0.5,-50,0,70)
flyScriptButton.Text = "Fly Script"
flyScriptButton.BackgroundColor3 = Color3.fromRGB(0,255,255)
flyScriptButton.Parent = page1
flyScriptButton.MouseButton1Click:Connect(function()
    loadstring(game:HttpGet("https://raw.githubusercontent.com/XNEOFF/FlyGuiV3/main/FlyGuiV3.txt"))()
end)

local showItemsButton = Instance.new("TextButton")
showItemsButton.Size = UDim2.new(0,150,0,30)
showItemsButton.Position = UDim2.new(0.5,-75,0,120)
showItemsButton.Text = "Show Items"
showItemsButton.BackgroundColor3 = Color3.fromRGB(0,255,255)
showItemsButton.Parent = page1
showItemsButton.MouseButton1Click:Connect(function()
    local itemGui = Instance.new("ScreenGui")
    itemGui.ResetOnSpawn = false
    itemGui.Parent = player:WaitForChild("PlayerGui")
    
    local listFrame = Instance.new("Frame")
    listFrame.Size = UDim2.new(0,300,0,400)
    listFrame.Position = UDim2.new(0.5,-150,0.5,-200)
    listFrame.BackgroundColor3 = Color3.fromRGB(50,50,50)
    listFrame.Parent = itemGui
    createDeleteButton(listFrame)
    
    local listTitle = Instance.new("TextLabel")
    listTitle.Size = UDim2.new(1,0,0,30)
    listTitle.Position = UDim2.new(0,0,0,0)
    listTitle.BackgroundTransparency = 1
    listTitle.Text = "Items in Game"
    listTitle.TextColor3 = Color3.new(1,1,1)
    listTitle.Font = Enum.Font.SourceSansBold
    listTitle.TextSize = 18
    listTitle.Parent = listFrame
    
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1,-20,1,-50)
    scrollFrame.Position = UDim2.new(0,10,0,40)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.ScrollBarThickness = 12
    scrollFrame.Parent = listFrame
    
    local contentFrame = Instance.new("Frame")
    contentFrame.Size = UDim2.new(1,0,0,0)
    contentFrame.BackgroundTransparency = 1
    contentFrame.Parent = scrollFrame
    
    local yOffset = 0
    local function canHoldOrUse(item)
        return item:IsA("Tool") or (item:IsA("BasePart") and item:FindFirstChild("CanBeHeld"))
    end
    for _, obj in ipairs(game:GetDescendants()) do
        if canHoldOrUse(obj) then
            local row = Instance.new("Frame")
            row.Size = UDim2.new(1,0,0,30)
            row.Position = UDim2.new(0,0,0,yOffset)
            row.BackgroundTransparency = 1
            row.Parent = contentFrame

            local label = Instance.new("TextLabel")
            label.Size = UDim2.new(0,200,1,0)
            label.Position = UDim2.new(0,10,0,0)
            label.Text = obj.Name
            label.TextColor3 = Color3.new(1,1,1)
            label.Font = Enum.Font.SourceSans
            label.TextSize = 16
            label.BackgroundTransparency = 1
            label.Parent = row

            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(0,60,1,0)
            btn.Position = UDim2.new(0,220,0,0)
            btn.Text = "Give"
            btn.BackgroundColor3 = Color3.fromRGB(0,255,0)
            btn.Parent = row
            btn.MouseButton1Click:Connect(function()
                local backpack = player:WaitForChild("Backpack")
                if backpack then
                    local found = false
                    for _, instance in ipairs(ReplicatedStorage:GetChildren()) do
                        if (instance:IsA("RemoteEvent") or instance:IsA("ModuleScript")) and instance:GetAttribute("GiveItem") == obj.Name then
                            found = true
                            if instance:IsA("RemoteEvent") then
                                instance:FireServer()
                            elseif instance:IsA("ModuleScript") then
                                local f = require(instance)
                                if type(f)=="function" then f(player) end
                            end
                            break
                        end
                    end
                    if not found then
                        if obj:IsA("Tool") then
                            local clone = obj:Clone()
                            clone.Parent = backpack
                        elseif obj:IsA("BasePart") then
                            local clone = obj:Clone()
                            clone:BreakJoints()
                            clone.Anchored = false
                            clone.CanCollide = false
                            local tool = Instance.new("Tool")
                            tool.Name = clone.Name
                            tool.RequiresHandle = true
                            tool.Grip = CFrame.new()
                            for _, weld in ipairs(clone:GetChildren()) do
                                if weld:IsA("Weld") or weld:IsA("WeldConstraint") then weld:Destroy() end
                            end
                            clone.Name = "Handle"
                            clone.Parent = tool
                            tool.Handle = clone
                            tool.Parent = backpack
                        end
                    end
                end
            end)

            yOffset = yOffset + 35
        end
    end
    contentFrame.Size = UDim2.new(1,0,0,yOffset)
end)

trackerTextBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then
        local name = trackerTextBox.Text
        for _, part in ipairs(workspace:GetDescendants()) do
            if part:IsA("BasePart") and part.Name == name then
                createTrackerGui(name)
                break
            end
        end
    end
end)

-- Tracker GUI function
local function createTrackerGui(partName)
    local trackerGui = Instance.new("ScreenGui")
    trackerGui.ResetOnSpawn = false
    trackerGui.Parent = player:WaitForChild("PlayerGui")
    
    local trackerFrame = Instance.new("Frame")
    trackerFrame.Size = UDim2.new(0,250,0,200)
    trackerFrame.Position = UDim2.new(0.5,-125,0.2,0)
    trackerFrame.BackgroundColor3 = Color3.fromRGB(50,50,50)
    trackerFrame.Active = true
    trackerFrame.Draggable = true
    trackerFrame.Parent = trackerGui
    
    createDeleteButton(trackerFrame)
    
    local trackerTitle = Instance.new("TextLabel")
    trackerTitle.Size = UDim2.new(1,0,0,30)
    trackerTitle.Position = UDim2.new(0,0,0,0)
    trackerTitle.BackgroundTransparency = 1
    trackerTitle.Text = "Tracking: " .. partName
    trackerTitle.TextColor3 = Color3.new(1,1,1)
    trackerTitle.Font = Enum.Font.SourceSansBold
    trackerTitle.TextSize = 18
    trackerTitle.Parent = trackerFrame
    
    local toggleButton = Instance.new("TextButton")
    toggleButton.Size = UDim2.new(0,100,0,30)
    toggleButton.Position = UDim2.new(0.5,-50,0.75,0)
    toggleButton.Text = "OFF"
    toggleButton.BackgroundColor3 = Color3.fromRGB(255,0,0)
    toggleButton.Parent = trackerFrame
    
    local activated = false
    local delayTime = 1
    local teleportLimit = math.huge
    local lastTeleportTime = 0

    local unanchorButton = Instance.new("TextButton")
    unanchorButton.Size = UDim2.new(0,100,0,30)
    unanchorButton.Position = UDim2.new(0.5,-50,0.85,0)
    unanchorButton.Text = "Unanchor Parts"
    unanchorButton.BackgroundColor3 = Color3.fromRGB(255,255,0)
    unanchorButton.Parent = trackerFrame
    unanchorButton.MouseButton1Click:Connect(function()
        for _, p in ipairs(workspace:GetDescendants()) do
            if p:IsA("BasePart") and p.Name == partName then
                p.Anchored = false
            end
        end
    end)
    
    local collisionToggled = false
    local disableCollisionButton = Instance.new("TextButton")
    disableCollisionButton.Size = UDim2.new(0,100,0,30)
    disableCollisionButton.Position = UDim2.new(0.5,-50,0.95,0)
    disableCollisionButton.Text = "Collision: ON"
    disableCollisionButton.BackgroundColor3 = Color3.fromRGB(255,0,0)
    disableCollisionButton.Parent = trackerFrame
    disableCollisionButton.MouseButton1Click:Connect(function()
        collisionToggled = not collisionToggled
        if collisionToggled then
            disableCollisionButton.Text = "Collision: OFF"
            disableCollisionButton.BackgroundColor3 = Color3.fromRGB(0,255,0)
            for _, p in ipairs(workspace:GetDescendants()) do
                if p:IsA("BasePart") and p.Name == partName then
                    p.CanCollide = false
                end
            end
        else
            disableCollisionButton.Text = "Collision: ON"
            disableCollisionButton.BackgroundColor3 = Color3.fromRGB(255,0,0)
            for _, p in ipairs(workspace:GetDescendants()) do
                if p:IsA("BasePart") and p.Name == partName then
                    p.CanCollide = true
                end
            end
        end
    end)
    
    local delayTextBox = Instance.new("TextBox")
    delayTextBox.Size = UDim2.new(0,80,0,30)
    delayTextBox.Position = UDim2.new(0,10,0,80)
    delayTextBox.Text = tostring(delayTime)
    delayTextBox.PlaceholderText = "Delay"
    delayTextBox.Parent = trackerFrame
    
    local setDelayButton = Instance.new("TextButton")
    setDelayButton.Size = UDim2.new(0,50,0,30)
    setDelayButton.Position = UDim2.new(0,100,0,80)
    setDelayButton.Text = "Set"
    setDelayButton.BackgroundColor3 = Color3.fromRGB(0,255,0)
    setDelayButton.Parent = trackerFrame
    setDelayButton.MouseButton1Click:Connect(function()
        local newDelay = tonumber(delayTextBox.Text)
        if newDelay then
            delayTime = newDelay
        end
    end)
    
    local limitTextBox = Instance.new("TextBox")
    limitTextBox.Size = UDim2.new(0,80,0,30)
    limitTextBox.Position = UDim2.new(0,10,0,120)
    limitTextBox.Text = "∞"
    limitTextBox.PlaceholderText = "Limit"
    limitTextBox.Parent = trackerFrame
    
    local setLimitButton = Instance.new("TextButton")
    setLimitButton.Size = UDim2.new(0,50,0,30)
    setLimitButton.Position = UDim2.new(0,100,0,120)
    setLimitButton.Text = "Set"
    setLimitButton.BackgroundColor3 = Color3.fromRGB(0,255,0)
    setLimitButton.Parent = trackerFrame
    setLimitButton.MouseButton1Click:Connect(function()
        local newLimit = tonumber(limitTextBox.Text)
        if newLimit then
            teleportLimit = newLimit
        else
            teleportLimit = math.huge
            limitTextBox.Text = "∞"
        end
    end)
    
    toggleButton.MouseButton1Click:Connect(function()
        activated = not activated
        if activated then
            toggleButton.Text = "ON"
            toggleButton.BackgroundColor3 = Color3.fromRGB(0,255,0)
        else
            toggleButton.Text = "OFF"
            toggleButton.BackgroundColor3 = Color3.fromRGB(255,0,0)
        end
    end)
    
    RunService.Heartbeat:Connect(function()
        if activated and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local currentTime = tick()
            if currentTime - lastTeleportTime >= delayTime then
                local teleported = 0
                for _, p in ipairs(workspace:GetDescendants()) do
                    if p:IsA("BasePart") and p.Name == partName then
                        if teleported >= teleportLimit then break end
                        p.CFrame = player.Character.HumanoidRootPart.CFrame
                        teleported = teleported + 1
                    end
                end
                lastTeleportTime = currentTime
            end
        end
    end)
end

trackerTextBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then
        local name = trackerTextBox.Text
        for _, part in ipairs(workspace:GetDescendants()) do
            if part:IsA("BasePart") and part.Name == name then
                createTrackerGui(name)
                break
            end
        end
    end
end)

-----------------------------------------------------------
-- PAGE 2: Hitbox & Teleport Controls, Sidebar, Infinite Health
-----------------------------------------------------------
local page2 = pages[2]

-- Adjusted hitbox controls: TextBox and Button each roughly half the width.
local function createHitboxControls(parent)
    local textboxWidth = 115  -- roughly half the width of main UI (approx.)
    
    local box = Instance.new("TextBox")
    box.Size = UDim2.new(0, textboxWidth, 0, 30)
    box.Position = UDim2.new(0,10,0,60)
    box.PlaceholderText = "Size (X,Y,Z)"
    box.Text = "4,4,4"
    box.Parent = parent

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, textboxWidth, 0, 30)
    btn.Position = UDim2.new(0,10,0,95) -- 35 pixels below the textbox
    btn.Text = "Set Size"
    btn.BackgroundColor3 = Color3.fromRGB(0,0,255)
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Font = Enum.Font.SourceSansBold
    btn.TextSize = 20
    btn.Parent = parent

    btn.MouseButton1Click:Connect(function()
        local values = {}
        for v in string.gmatch(box.Text, "[^,]+") do
            table.insert(values, tonumber(v))
        end
        if #values==3 and values[1] and values[2] and values[3] then
            baseIndicatorSize = Vector3.new(values[1], values[2], values[3])
            if hitboxIndicator then hitboxIndicator:Destroy() end
            hitboxIndicator = Instance.new("Part")
            hitboxIndicator.Size = baseIndicatorSize
            hitboxIndicator.Anchored = true
            hitboxIndicator.CanCollide = false
            hitboxIndicator.Transparency = 0.5
            hitboxIndicator.BrickColor = BrickColor.new("Really red")
            if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                hitboxIndicator.CFrame = player.Character.HumanoidRootPart.CFrame
            end
            hitboxIndicator.Parent = workspace
            radiusToggled = true
        end
    end)
end
createHitboxControls(page2)

-- Teleport Parameter Controls: Positioned on the same y-level as the hitbox textbox.
local function createTeleportParamControls(parent)
    local delayBox = Instance.new("TextBox")
    delayBox.Size = UDim2.new(0,50,0,30)
    delayBox.Position = UDim2.new(0,130,0,60)
    delayBox.PlaceholderText = "Delay"
    delayBox.Text = "0"
    delayBox.Parent = parent

    local durationBox = Instance.new("TextBox")
    durationBox.Size = UDim2.new(0,50,0,30)
    durationBox.Position = UDim2.new(0,190,0,60)
    durationBox.PlaceholderText = "Dur."
    durationBox.Text = "0.1"
    durationBox.Parent = parent

    local setBtn = Instance.new("TextButton")
    setBtn.Size = UDim2.new(0,80,0,30)
    setBtn.Position = UDim2.new(0,130,0,95)
    setBtn.Text = "Set Parameters"
    setBtn.BackgroundColor3 = Color3.fromRGB(0,0,255)
    setBtn.TextColor3 = Color3.new(1,1,1)
    setBtn.Font = Enum.Font.SourceSansBold
    setBtn.TextSize = 20
    setBtn.Parent = parent

    setBtn.MouseButton1Click:Connect(function()
        local d = tonumber(delayBox.Text)
        local du = tonumber(durationBox.Text)
        if d then teleportDelay = d end
        if du then teleportDuration = du end
    end)
end
createTeleportParamControls(page2)

local toggleRadiusBtn = Instance.new("TextButton")
toggleRadiusBtn.Size = UDim2.new(1,-20,0,30)
toggleRadiusBtn.Position = UDim2.new(0,10,0,140)
toggleRadiusBtn.Text = "Radius: ON"
toggleRadiusBtn.BackgroundColor3 = Color3.fromRGB(0,255,0)
toggleRadiusBtn.TextColor3 = Color3.new(1,1,1)
toggleRadiusBtn.Font = Enum.Font.SourceSansBold
toggleRadiusBtn.TextSize = 20
toggleRadiusBtn.Parent = page2
toggleRadiusBtn.MouseButton1Click:Connect(function()
    if hitboxIndicator and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
        if radiusToggled then
            hitboxIndicator.CFrame = player.Character.HumanoidRootPart.CFrame * CFrame.new(0,-9999999,0)
            toggleRadiusBtn.Text = "Radius: OFF"
            toggleRadiusBtn.BackgroundColor3 = Color3.fromRGB(255,0,0)
        else
            hitboxIndicator.CFrame = player.Character.HumanoidRootPart.CFrame
            toggleRadiusBtn.Text = "Radius: ON"
            toggleRadiusBtn.BackgroundColor3 = Color3.fromRGB(0,255,0)
        end
        radiusToggled = not radiusToggled
    end
end)

local sidebar = Instance.new("Frame")
sidebar.Size = UDim2.new(0,120,0,250)
sidebar.BackgroundTransparency = 0.5
sidebar.BackgroundColor3 = Color3.new(0,0,0)
sidebar.Parent = screenGui

RunService.Heartbeat:Connect(function()
    if mainFrame and mainFrame.Parent then
        local absPos = mainFrame.AbsolutePosition
        local absSize = mainFrame.AbsoluteSize
        sidebar.Position = UDim2.new(0, absPos.X + absSize.X, 0, absPos.Y)
    else
        if sidebar then sidebar:Destroy() end
    end

    if hitboxIndicator and player.Character and player.Character:FindFirstChild("HumanoidRootPart") and radiusToggled then
        hitboxIndicator.CFrame = player.Character.HumanoidRootPart.CFrame
        sidebar.Visible = true
    else
        sidebar.Visible = false
    end

    local region = Region3.new(hitboxIndicator.Position - hitboxIndicator.Size/2, hitboxIndicator.Position + hitboxIndicator.Size/2)
    local foundParts = workspace:FindPartsInRegion3(region, player.Character, math.huge)
    local validParts = {}
    for _, part in ipairs(foundParts) do
        if part.Name=="HumanoidRootPart" and part.Parent~=player.Character and not part:IsDescendantOf(hitboxIndicator) then
            table.insert(validParts, part)
            if activeTargets[part]==nil then activeTargets[part] = false end
            if not outlinedAdorns[part] then
                local boxAdor = Instance.new("BoxHandleAdornment")
                boxAdor.Adornee = part
                boxAdor.Size = part.Size
                boxAdor.Transparency = 0.5
                boxAdor.AlwaysOnTop = true
                boxAdor.ZIndex = 10
                boxAdor.Color3 = activeTargets[part] and Color3.new(0,1,0) or Color3.new(1,0,0)
                boxAdor.Parent = part
                outlinedAdorns[part] = boxAdor
            else
                outlinedAdorns[part].Color3 = activeTargets[part] and Color3.new(0,1,0) or Color3.new(1,0,0)
            end
        end
    end

    local yOffset = 0
    for _, part in ipairs(validParts) do
        local entry = sidebarEntries[part]
        if not entry then
            entry = Instance.new("Frame")
            entry.Size = UDim2.new(1,0,0,25)
            entry.BackgroundTransparency = 1
            entry.Parent = sidebar

            local toggleBtn = Instance.new("TextButton")
            toggleBtn.Size = UDim2.new(0,25,1,0)
            toggleBtn.Position = UDim2.new(0,0,0,0)
            toggleBtn.Text = activeTargets[part] and "✔" or "X"
            toggleBtn.BackgroundColor3 = activeTargets[part] and Color3.new(0,1,0) or Color3.new(1,0,0)
            toggleBtn.Parent = entry
            toggleBtn.MouseButton1Click:Connect(function()
                activeTargets[part] = not activeTargets[part]
                if activeTargets[part] then
                    toggleBtn.Text = "✔"
                    toggleBtn.BackgroundColor3 = Color3.new(0,1,0)
                else
                    toggleBtn.Text = "X"
                    toggleBtn.BackgroundColor3 = Color3.new(1,0,0)
                end
                if outlinedAdorns[part] then
                    outlinedAdorns[part].Color3 = activeTargets[part] and Color3.new(0,1,0) or Color3.new(1,0,0)
                end
            end)

            local nameLabel = Instance.new("TextLabel")
            nameLabel.Size = UDim2.new(1,-30,1,0)
            nameLabel.Position = UDim2.new(0,30,0,0)
            nameLabel.BackgroundTransparency = 1
            nameLabel.Text = part.Parent.Name
            nameLabel.TextColor3 = Color3.new(1,1,1)
            nameLabel.Font = Enum.Font.SourceSansBold
            nameLabel.TextSize = 16
            nameLabel.Parent = entry

            sidebarEntries[part] = entry
        end
        entry.Position = UDim2.new(0,0,0,yOffset)
        yOffset = yOffset + 25
    end
    sidebar.Size = UDim2.new(0,120,0,yOffset)
    
    for part, entry in pairs(sidebarEntries) do
        local found = false
        for _, v in ipairs(validParts) do
            if v == part then found = true; break end
        end
        if not found then
            entry:Destroy()
            sidebarEntries[part] = nil
            activeTargets[part] = nil
            if outlinedAdorns[part] then
                outlinedAdorns[part]:Destroy()
                outlinedAdorns[part] = nil
            end
        end
    end
end)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        for part, toggled in pairs(activeTargets) do
            if toggled then
                if part and part.Parent then
                    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
                    if hrp then
                        task.wait(teleportDelay)
                        local orig = hrp.CFrame
                        hrp.CFrame = part.CFrame
                        task.wait(teleportDuration)
                        hrp.CFrame = orig
                    end
                end
                break
            end
        end
    end
end)

-----------------------------------------------------------
-- PAGE 3: Inventory, Classic Sword, Restore Jump, Value Override
-----------------------------------------------------------
local page3 = pages[3]

local buttonHeight = 35
local padding = 10

local inventoryBtn = Instance.new("TextButton")
inventoryBtn.Size = UDim2.new(1,-20,0,buttonHeight)
inventoryBtn.Position = UDim2.new(0,10,0,padding)
inventoryBtn.Text = "Show Inventory"
inventoryBtn.BackgroundColor3 = Color3.fromRGB(0,128,255)
inventoryBtn.TextColor3 = Color3.new(1,1,1)
inventoryBtn.Font = Enum.Font.SourceSansBold
inventoryBtn.TextSize = 20
inventoryBtn.Parent = page3
inventoryBtn.MouseButton1Click:Connect(function()
    game:GetService("StarterGui"):SetCoreGuiEnabled(Enum.CoreGuiType.Backpack,true)
end)

local swordBtn = Instance.new("TextButton")
swordBtn.Size = UDim2.new(1,-20,0,buttonHeight)
swordBtn.Position = UDim2.new(0,10,0,padding + buttonHeight + 5)
swordBtn.Text = "Give Classic Sword"
swordBtn.BackgroundColor3 = Color3.fromRGB(0,200,200)
swordBtn.TextColor3 = Color3.new(1,1,1)
swordBtn.Font = Enum.Font.SourceSansBold
swordBtn.TextSize = 20
swordBtn.Parent = page3
swordBtn.MouseButton1Click:Connect(function() giveClassicSword() end)

local restoreJumpBtn = Instance.new("TextButton")
restoreJumpBtn.Size = UDim2.new(1,-20,0,buttonHeight)
restoreJumpBtn.Position = UDim2.new(0,10,0,padding + (buttonHeight+5)*2)
restoreJumpBtn.Text = "Restore Jump"
restoreJumpBtn.BackgroundColor3 = Color3.fromRGB(200,200,0)
restoreJumpBtn.TextColor3 = Color3.new(0,0,0)
restoreJumpBtn.Font = Enum.Font.SourceSansBold
restoreJumpBtn.TextSize = 20
restoreJumpBtn.Parent = page3
restoreJumpBtn.MouseButton1Click:Connect(function()
    if player.Character and player.Character:FindFirstChild("Humanoid") then
        local h = player.Character.Humanoid
        h.JumpPower = 50
        h.UseJumpPower = true
        print("Jump restored!")
    end
end)

local valueOverrideBtn = Instance.new("TextButton")
valueOverrideBtn.Size = UDim2.new(1,-20,0,buttonHeight)
valueOverrideBtn.Position = UDim2.new(0,10,0,padding + (buttonHeight+5)*3)
valueOverrideBtn.Text = "Value Override"
valueOverrideBtn.BackgroundColor3 = Color3.fromRGB(150,0,150)
valueOverrideBtn.TextColor3 = Color3.new(1,1,1)
valueOverrideBtn.Font = Enum.Font.SourceSansBold
valueOverrideBtn.TextSize = 20
valueOverrideBtn.Parent = page3
valueOverrideBtn.MouseButton1Click:Connect(function() openValueOverrideGUI() end)

-- Classic Sword function
function giveClassicSword()
    local backpack = player:WaitForChild("Backpack")
    if backpack:FindFirstChild("Classic Sword") then return end

    local sword = Instance.new("Tool")
    sword.Name = "Classic Sword"
    sword.RequiresHandle = true
    sword.Grip = CFrame.new(0,-0.75,0)
    
    local handle = Instance.new("Part")
    handle.Name = "Handle"
    handle.Size = Vector3.new(0.5,1.5,0.5)
    handle.BrickColor = BrickColor.new("Brown")
    handle.TopSurface = Enum.SurfaceType.Smooth
    handle.BottomSurface = Enum.SurfaceType.Smooth
    handle.Parent = sword
    
    local guard = Instance.new("Part")
    guard.Name = "Guard"
    guard.Size = Vector3.new(1,0.2,1)
    guard.BrickColor = BrickColor.new("Dark stone grey")
    guard.TopSurface = Enum.SurfaceType.Smooth
    guard.BottomSurface = Enum.SurfaceType.Smooth
    guard.Parent = sword
    local gw = Instance.new("WeldConstraint")
    gw.Part0 = handle; gw.Part1 = guard; gw.Parent = handle
    guard.CFrame = handle.CFrame * CFrame.new(0,0.85,0)
    
    local blade = Instance.new("Part")
    blade.Name = "Blade"
    blade.Size = Vector3.new(0.25,4.5,0.5)
    blade.BrickColor = BrickColor.new("Really red")
    blade.TopSurface = Enum.SurfaceType.Smooth
    blade.BottomSurface = Enum.SurfaceType.Smooth
    blade.Parent = sword
    local bw = Instance.new("WeldConstraint")
    bw.Part0 = handle; bw.Part1 = blade; bw.Parent = handle
    blade.CFrame = handle.CFrame * CFrame.new(0,3,0)
    
    local swordScript = Instance.new("LocalScript")
    swordScript.Source = [[
        local tool = script.Parent
        local DAMAGE = 10
        local slashActive = false
        local debounce = false

        tool.Activated:Connect(function()
            slashActive = true
            local character = tool.Parent
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                local anim = Instance.new("Animation")
                anim.AnimationId = "rbxassetid://507770239"
                local track = humanoid:LoadAnimation(anim)
                if track then
                    track:Play()
                else
                    warn("Failed to load slash animation")
                end
            end
            wait(0.3)
            slashActive = false
        end)

        tool.Handle.Touched:Connect(function(hit)
            if slashActive and not debounce then
                local hum = hit.Parent:FindFirstChild("Humanoid")
                if hum and hit.Parent ~= tool.Parent then
                    debounce = true
                    hum.Health = hum.Health - DAMAGE
                    wait(0.2)
                    debounce = false
                end
            end
        end)
    ]]
    swordScript.Parent = sword
    sword.Parent = backpack
end
swordBtn.MouseButton1Click:Connect(function() giveClassicSword() end)

-- Value Override GUI function (unchanged)
function openValueOverrideGUI()
    local overrideGui = Instance.new("ScreenGui")
    overrideGui.Name = "ValueOverrideGUI"
    overrideGui.Parent = player:WaitForChild("PlayerGui")
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0,300,0,400)
    mainFrame.Position = UDim2.new(0.5,-150,0.5,-200)
    mainFrame.BackgroundColor3 = Color3.fromRGB(50,50,50)
    mainFrame.Parent = overrideGui
    
    createDeleteButton(mainFrame)
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1,0,0,30)
    titleLabel.Position = UDim2.new(0,0,0,5)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = "Value Override"
    titleLabel.TextColor3 = Color3.new(1,1,1)
    titleLabel.Font = Enum.Font.SourceSansBold
    titleLabel.TextSize = 20
    titleLabel.Parent = mainFrame

    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1,-20,1,-100)
    scrollFrame.Position = UDim2.new(0,10,0,40)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.ScrollBarThickness = 12
    scrollFrame.Parent = mainFrame

    local contentFrame = Instance.new("Frame")
    contentFrame.Size = UDim2.new(1,0,0,0)
    contentFrame.BackgroundTransparency = 1
    contentFrame.Parent = scrollFrame

    local toggleStates = {}
    local yOffset = 0

    for _, obj in ipairs(game:GetDescendants()) do
        if obj:IsA("NumberValue") then
            local row = Instance.new("Frame")
            row.Size = UDim2.new(1,0,0,25)
            row.Position = UDim2.new(0,0,0,yOffset)
            row.BackgroundTransparency = 1
            row.Parent = contentFrame

            local toggleBtn = Instance.new("TextButton")
            toggleBtn.Size = UDim2.new(0,25,1,0)
            toggleBtn.Position = UDim2.new(0,0,0,0)
            toggleBtn.Text = "X"
            toggleBtn.BackgroundColor3 = Color3.new(1,0,0)
            toggleBtn.Parent = row
            toggleStates[obj] = false
            toggleBtn.MouseButton1Click:Connect(function()
                toggleStates[obj] = not toggleStates[obj]
                if toggleStates[obj] then
                    toggleBtn.Text = "✔"
                    toggleBtn.BackgroundColor3 = Color3.new(0,1,0)
                else
                    toggleBtn.Text = "X"
                    toggleBtn.BackgroundColor3 = Color3.new(1,0,0)
                end
            end)

            local nameLabel = Instance.new("TextLabel")
            nameLabel.Size = UDim2.new(1,-30,1,0)
            nameLabel.Position = UDim2.new(0,30,0,0)
            nameLabel.BackgroundTransparency = 1
            nameLabel.Text = obj.Name .. " (" .. tostring(obj.Value) .. ")"
            nameLabel.TextColor3 = Color3.new(1,1,1)
            nameLabel.Font = Enum.Font.SourceSansBold
            nameLabel.TextSize = 16
            nameLabel.Parent = row

            yOffset = yOffset + 25
        end
    end
    contentFrame.Size = UDim2.new(1,0,0,yOffset)

    local valueBox = Instance.new("TextBox")
    valueBox.Size = UDim2.new(0,150,0,30)
    valueBox.Position = UDim2.new(0,10,1,-40)
    valueBox.PlaceholderText = "Enter new value"
    valueBox.Parent = mainFrame

    local setBtn = Instance.new("TextButton")
    setBtn.Size = UDim2.new(0,100,0,30)
    setBtn.Position = UDim2.new(0,170,1,-40)
    setBtn.Text = "Set"
    setBtn.BackgroundColor3 = Color3.new(0,1,0)
    setBtn.TextColor3 = Color3.new(0,0,0)
    setBtn.Font = Enum.Font.SourceSansBold
    setBtn.TextSize = 20
    setBtn.Parent = mainFrame

    setBtn.MouseButton1Click:Connect(function()
        local newVal = tonumber(valueBox.Text)
        if newVal then
            for obj, selected in pairs(toggleStates) do
                if selected then obj.Value = newVal end
            end
            print("Overridden selected values to " .. newVal)
        else
            warn("Invalid number!")
        end
    end)
end

-- PAGE 4: Highlight Mode & Click Activator
local page4 = pages[4]

-- Highlight Mode
local highlightEnabled = false
local highlightToggle = Instance.new("TextButton")
highlightToggle.Size = UDim2.new(1,-20,0,35)
highlightToggle.Position = UDim2.new(0,10,0,10)
highlightToggle.Text = "Highlight: OFF"
highlightToggle.BackgroundColor3 = Color3.fromRGB(255,0,0)
highlightToggle.TextColor3 = Color3.new(1,1,1)
highlightToggle.Font = Enum.Font.SourceSansBold
highlightToggle.TextSize = 20
highlightToggle.Parent = page4

highlightToggle.MouseButton1Click:Connect(function()
    highlightEnabled = not highlightEnabled
    if highlightEnabled then
        highlightToggle.Text = "Highlight: ON"
        highlightToggle.BackgroundColor3 = Color3.fromRGB(0,255,0)
    else
        highlightToggle.Text = "Highlight: OFF"
        highlightToggle.BackgroundColor3 = Color3.fromRGB(255,0,0)
        if _G.currentHighlight then
            _G.currentHighlight:Destroy()
            _G.currentHighlight = nil
        end
        if _G.selectedPartLabel then
            _G.selectedPartLabel.Text = "No part selected"
        end
    end
end)

local selectionLabel = Instance.new("TextLabel")
selectionLabel.Size = UDim2.new(1,-20,0,30)
selectionLabel.Position = UDim2.new(0,10,0,50)
selectionLabel.BackgroundTransparency = 1
selectionLabel.Text = "No part selected"
selectionLabel.TextColor3 = Color3.new(1,1,1)
selectionLabel.Font = Enum.Font.SourceSansBold
selectionLabel.TextSize = 18
selectionLabel.Parent = page4
_G.selectedPartLabel = selectionLabel

RunService.Heartbeat:Connect(function()
    if highlightEnabled then
        local target = player:GetMouse().Target
        if target and target:IsDescendantOf(workspace) then
            if not _G.currentHighlight or _G.currentHighlight.Adornee ~= target then
                if _G.currentHighlight then _G.currentHighlight:Destroy() end
                _G.currentHighlight = Instance.new("SelectionBox")
                _G.currentHighlight.Adornee = target
                _G.currentHighlight.LineThickness = 0.05
                _G.currentHighlight.Color3 = Color3.new(1,0,0)
                _G.currentHighlight.Parent = target
            end
            _G.selectedPartLabel.Text = target.Name
        else
            if _G.currentHighlight then
                _G.currentHighlight:Destroy()
                _G.currentHighlight = nil
            end
            _G.selectedPartLabel.Text = "No part selected"
        end
    end
end)

-- Click Activator: Opens a new GUI listing all parts with ClickDetectors.
local function openClickOverrideGUI()
    local overrideGui = Instance.new("ScreenGui")
    overrideGui.Name = "ClickActivatorGUI"
    overrideGui.Parent = player:WaitForChild("PlayerGui")
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0,300,0,400)
    mainFrame.Position = UDim2.new(0.5,-150,0.5,-200)
    mainFrame.BackgroundColor3 = Color3.fromRGB(50,50,50)
    mainFrame.Parent = overrideGui
    
    createDeleteButton(mainFrame)
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1,0,0,30)
    titleLabel.Position = UDim2.new(0,0,0,5)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = "Click Activator"
    titleLabel.TextColor3 = Color3.new(1,1,1)
    titleLabel.Font = Enum.Font.SourceSansBold
    titleLabel.TextSize = 20
    titleLabel.Parent = mainFrame

    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1,-20,1,-50)
    scrollFrame.Position = UDim2.new(0,10,0,40)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.ScrollBarThickness = 12
    scrollFrame.Parent = mainFrame

    local contentFrame = Instance.new("Frame")
    contentFrame.Size = UDim2.new(1,0,0,0)
    contentFrame.BackgroundTransparency = 1
    contentFrame.Parent = scrollFrame

    local clickOverrides = {}  -- mapping part -> toggle state
    local yOffset = 0

    for _, obj in ipairs(game:GetDescendants()) do
        if obj:IsA("BasePart") and obj:FindFirstChildOfClass("ClickDetector") then
            local row = Instance.new("Frame")
            row.Size = UDim2.new(1,0,0,25)
            row.Position = UDim2.new(0,0,0,yOffset)
            row.BackgroundTransparency = 1
            row.Parent = contentFrame

            local toggleBtn = Instance.new("TextButton")
            toggleBtn.Size = UDim2.new(0,25,1,0)
            toggleBtn.Position = UDim2.new(0,0,0,0)
            toggleBtn.Text = "X"
            toggleBtn.BackgroundColor3 = Color3.new(1,0,0)
            toggleBtn.Parent = row

            clickOverrides[obj] = false
            toggleBtn.MouseButton1Click:Connect(function()
                clickOverrides[obj] = not clickOverrides[obj]
                if clickOverrides[obj] then
                    toggleBtn.Text = "✔"
                    toggleBtn.BackgroundColor3 = Color3.new(0,1,0)
                else
                    toggleBtn.Text = "X"
                    toggleBtn.BackgroundColor3 = Color3.new(1,0,0)
                end
            end)

            local nameLabel = Instance.new("TextLabel")
            nameLabel.Size = UDim2.new(1,-30,1,0)
            nameLabel.Position = UDim2.new(0,30,0,0)
            nameLabel.BackgroundTransparency = 1
            nameLabel.Text = obj.Name
            nameLabel.TextColor3 = Color3.new(1,1,1)
            nameLabel.Font = Enum.Font.SourceSansBold
            nameLabel.TextSize = 16
            nameLabel.Parent = row

            yOffset = yOffset + 25
        end
    end
    contentFrame.Size = UDim2.new(1,0,0,yOffset)
    
    -- Continuous loop: every heartbeat, fire click detectors on toggled parts.
    local connection
    connection = RunService.Heartbeat:Connect(function()
        if not overrideGui or not overrideGui.Parent then
            connection:Disconnect()
            return
        end
        for obj, state in pairs(clickOverrides) do
            if state then
                for _, cd in ipairs(obj:GetChildren()) do
                    if cd:IsA("ClickDetector") then
                        pcall(function() fireclickdetector(cd) end)
                    end
                end
            end
        end
    end)
end

-- Position Click Activator button so it doesn't overlap the selection label.
local clickOverrideBtn = Instance.new("TextButton")
clickOverrideBtn.Size = UDim2.new(1,-20,0,35)
clickOverrideBtn.Position = UDim2.new(0,10,0,90)
clickOverrideBtn.Text = "Click Activator"
clickOverrideBtn.BackgroundColor3 = Color3.fromRGB(150,0,150)
clickOverrideBtn.TextColor3 = Color3.new(1,1,1)
clickOverrideBtn.Font = Enum.Font.SourceSansBold
clickOverrideBtn.TextSize = 20
clickOverrideBtn.Parent = page4
clickOverrideBtn.MouseButton1Click:Connect(function() openClickOverrideGUI() end)

-----------------------------------------------------------
-- Infinite Health (Page2)
-----------------------------------------------------------
RunService.Heartbeat:Connect(function()
    if infiniteHealth and player.Character and player.Character:FindFirstChild("Humanoid") then
        local h = player.Character.Humanoid
        h.MaxHealth = 1000
        h.Health = h.MaxHealth
        if player.Character:FindFirstChild("HumanoidRootPart") then
            player.Character.HumanoidRootPart:SetNetworkOwner(player)
        end
    end
end)

local infiniteHealthBtn = Instance.new("TextButton")
infiniteHealthBtn.Size = UDim2.new(1,-20,0,40)
infiniteHealthBtn.Position = UDim2.new(0,10,0,10)
infiniteHealthBtn.Text = "Infinite Health: OFF"
infiniteHealthBtn.BackgroundColor3 = Color3.fromRGB(255,0,0)
infiniteHealthBtn.TextColor3 = Color3.new(1,1,1)
infiniteHealthBtn.Font = Enum.Font.SourceSansBold
infiniteHealthBtn.TextSize = 20
infiniteHealthBtn.Parent = pages[2]
infiniteHealthBtn.MouseButton1Click:Connect(function()
    infiniteHealth = not infiniteHealth
    if infiniteHealth then
        infiniteHealthBtn.Text = "Infinite Health: ON"
        infiniteHealthBtn.BackgroundColor3 = Color3.fromRGB(0,255,0)
    else
        infiniteHealthBtn.Text = "Infinite Health: OFF"
        infiniteHealthBtn.BackgroundColor3 = Color3.fromRGB(255,0,0)
    end
end)

-----------------------------------------------------------
-- Teleport Reactivation Bootstrap
-----------------------------------------------------------
if queue_on_teleport then
    queue_on_teleport([[
        repeat wait() until game:GetService("Players").LocalPlayer:FindFirstChild("PlayerGui")
        wait(10)  -- Extra delay to ensure everything is loaded
        if not getgenv().GuiClosed then
            local bootstrapGui = Instance.new("ScreenGui")
            bootstrapGui.Name = "BootstrapGUI"
            bootstrapGui.ResetOnSpawn = false
            bootstrapGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling  -- Ensure proper layering
            bootstrapGui.Parent = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")
            
            local loadButton = Instance.new("TextButton")
            loadButton.Size = UDim2.new(0,200,0,50)
            loadButton.Position = UDim2.new(0,10,0,70)  -- Positioned near the top left
            loadButton.Text = "Load GPT UI"
            loadButton.BackgroundColor3 = Color3.new(0,1,0)
            loadButton.TextColor3 = Color3.new(1,1,1)
            loadButton.Font = Enum.Font.SourceSansBold
            loadButton.TextSize = 24
            loadButton.ZIndex = 10  -- Ensure the button is above other elements
            loadButton.Parent = bootstrapGui
            
            local function loadFullUI()
                bootstrapGui:Destroy()
                local uis = game:GetService("UserInputService")
                uis.MouseBehavior = Enum.MouseBehavior.Default
                uis.MouseIconEnabled = true
                local cam = workspace.CurrentCamera
                if cam then
                    cam.CameraType = Enum.CameraType.Custom
                    cam.MaxZoomDistance = 10000  -- Allow free zooming
                end
                local success, err = pcall(function()
                    loadstring(game:HttpGet("https://raw.githubusercontent.com/workderpidly/gpt-ui/main/README.md", true))()
                end)
                if not success then
                    warn("Error loading full GPT UI: " .. tostring(err))
                end
            end
            
            loadButton.MouseButton1Click:Connect(loadFullUI)
            
            -- Bind the "=" key (and KeypadEquals) to force continuous mouse unlock
            game:GetService("UserInputService").InputBegan:Connect(function(input, gameProcessed)
                if gameProcessed then return end
                if input.KeyCode == Enum.KeyCode.Equals or input.KeyCode == Enum.KeyCode.KeypadEquals then
                    local uis = game:GetService("UserInputService")
                    uis.MouseBehavior = Enum.MouseBehavior.Default
                    uis.MouseIconEnabled = true
                    local cam = workspace.CurrentCamera
                    if cam then
                        cam.CameraType = Enum.CameraType.Custom
                        cam.MaxZoomDistance = 10000
                    end
                    loadFullUI()
                end
            end)
        end
    ]])
else
    player.CharacterAdded:Connect(function(character)
        wait(10)
        if not getgenv().GuiClosed and not player.PlayerGui:FindFirstChild("GPTUI") then
            local success, err = pcall(function()
                loadstring(game:HttpGet("https://github.com/workderpidly/gpt-ui/edit/main/README.md",true))()
            end)
            if not success then
                warn("Error reloading GUI on CharacterAdded: " .. tostring(err))
            end
        end
    end)
end

-- Persistent Forced Mouse Unlock: once "=" is pressed, every frame force unlock the mouse.
local forceMouseUnlock = false
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.Equals or input.KeyCode == Enum.KeyCode.KeypadEquals then
        forceMouseUnlock = true
    end
end)
RunService.Heartbeat:Connect(function()
    if forceMouseUnlock then
        UserInputService.MouseBehavior = Enum.MouseBehavior.Default
        UserInputService.MouseIconEnabled = true
        local cam = workspace.CurrentCamera
        if cam then
            cam.CameraType = Enum.CameraType.Custom
            cam.MaxZoomDistance = 10000
        end
    end
end)
