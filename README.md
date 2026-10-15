--!nocheck
--!nolint
--[[
    FusionBombas Hub · Multi-executor (Delta, Solara, Wave, Fluxus, Synapse, etc.)
]]

-- ========= COMPAT LAYER =========
local function safeGetService(name)
    local ok, s = pcall(function()
        return game:GetService(name)
    end)
    return ok and s or nil
end

local Players = safeGetService("Players")
local RS = safeGetService("ReplicatedStorage")
local RunService = safeGetService("RunService")
local UIS = safeGetService("UserInputService")
local TweenService = safeGetService("TweenService")
local Lighting = safeGetService("Lighting")
local CoreGui = safeGetService("CoreGui")
local StarterGui = safeGetService("StarterGui")

local LP = Players and Players.LocalPlayer
if not LP then
    for _ = 1, 50 do
        LP = Players and Players.LocalPlayer
        if LP then break end
        task.wait(0.1)
    end
end
if not LP then
    warn("[FusionBombas] No LocalPlayer")
    return
end

-- getgenv fallback
local genv = (type(getgenv) == "function" and getgenv()) or _G
if genv.FusionBombasLoaded then
    pcall(function()
        if type(genv.FusionBombasDestroy) == "function" then
            genv.FusionBombasDestroy()
        end
    end)
end
genv.FusionBombasLoaded = true

-- newcclosure fallback
local wrap = function(fn)
    if type(newcclosure) == "function" then
        local ok, r = pcall(newcclosure, fn)
        if ok and r then return r end
    end
    return fn
end

-- Parent GUI en cualquier executor
local function protectAndParent(gui)
    pcall(function()
        if type(syn) == "table" and type(syn.protect_gui) == "function" then
            syn.protect_gui(gui)
        end
    end)
    pcall(function()
        if type(protect_gui) == "function" then
            protect_gui(gui)
        end
    end)
    local parented = false
    pcall(function()
        if type(gethui) == "function" then
            gui.Parent = gethui()
            parented = gui.Parent ~= nil
        end
    end)
    if not parented then
        pcall(function()
            if CoreGui then
                gui.Parent = CoreGui
                parented = gui.Parent ~= nil
            end
        end)
    end
    if not parented then
        pcall(function()
            local pg = LP:FindFirstChildOfClass("PlayerGui") or LP:WaitForChild("PlayerGui", 5)
            if pg then
                gui.Parent = pg
            end
        end)
    end
end

local CFG = {
    damageMult = 1,
    selectedBomb = "Nuke",
    bombAmount = 1,
    fullbright = false,
    noFog = false,
    bombEsp = false,
}

local BOMB_LIST = {
    "TNT", "C4", "Dynamite", "Grenade", "Missile", "Rocket",
    "Atomic", "Nuke", "Nuclear", "Hydrogen", "Tsar Bomba", "Tsar",
    "Mega Nuke", "Super Nuke", "Omega Nuke", "God Nuke",
    "Black Hole", "Antimatter", "Singularity", "Quasar",
    "Boss Bomb", "Ultimate", "Max", "Legendary Nuke",
}

local function notify(title, text)
    pcall(function()
        if StarterGui then
            StarterGui:SetCore("SendNotification", {
                Title = tostring(title or "FusionBombas"),
                Text = tostring(text or ""),
                Duration = 3,
            })
        end
    end)
end

local function corner(parent, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r or 8)
    c.Parent = parent
    return c
end

local function stroke(parent, color, thick)
    local s = Instance.new("UIStroke")
    s.Color = color or Color3.fromRGB(255, 80, 40)
    s.Thickness = thick or 1
    s.Transparency = 0.35
    s.Parent = parent
    return s
end

local function pad(parent, l, t, r, b)
    local p = Instance.new("UIPadding")
    p.PaddingLeft = UDim.new(0, l or 8)
    p.PaddingTop = UDim.new(0, t or 8)
    p.PaddingRight = UDim.new(0, r or 8)
    p.PaddingBottom = UDim.new(0, b or 8)
    p.Parent = parent
    return p
end

local foundRemotes = {}

local function scanRemotes()
    table.clear(foundRemotes)
    if not RS then return 0 end
    local keywords = {
        "bomb", "nuke", "spawn", "place", "drop", "merge", "fuse",
        "weapon", "missile", "rocket", "fire", "launch", "buy",
        "purchase", "claim", "give", "add", "create", "item",
    }
    local function check(obj, path)
        if not (obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction")) then
            return
        end
        local n = string.lower(obj.Name)
        local p = string.lower(path)
        for _, kw in ipairs(keywords) do
            if string.find(n, kw, 1, true) or string.find(p, kw, 1, true) then
                table.insert(foundRemotes, { name = obj.Name, remote = obj, path = path })
                break
            end
        end
    end
    local function walk(parent, path, depth)
        if depth > 6 then return end
        local ok, children = pcall(function()
            return parent:GetChildren()
        end)
        if not ok then return end
        for _, ch in ipairs(children) do
            local np = path .. "/" .. ch.Name
            check(ch, np)
            pcall(walk, ch, np, depth + 1)
        end
    end
    pcall(walk, RS, "RS", 0)
    return #foundRemotes
end

local function tryFireRemote(remote, bombName, amount)
    if not remote then return false end
    local argsVariants = {
        { bombName },
        { bombName, amount },
        { amount, bombName },
        { bombName, true },
        { "Spawn", bombName },
        { "SpawnBomb", bombName },
        { "Place", bombName },
        { bombName, amount or 1, true },
        { { Name = bombName, Amount = amount or 1 } },
        { { bomb = bombName, count = amount or 1 } },
    }
    local okAny = false
    for _, args in ipairs(argsVariants) do
        local ok = pcall(function()
            if remote:IsA("RemoteEvent") then
                remote:FireServer(table.unpack(args))
            elseif remote:IsA("RemoteFunction") then
                remote:InvokeServer(table.unpack(args))
            end
        end)
        if ok then okAny = true end
    end
    return okAny
end

local function spawnBomb(bombName, amount)
    amount = math.clamp(tonumber(amount) or 1, 1, 100)
    bombName = tostring(bombName or CFG.selectedBomb)
    if #foundRemotes == 0 then
        scanRemotes()
    end

    local fired = 0
    for _, info in ipairs(foundRemotes) do
        if tryFireRemote(info.remote, bombName, amount) then
            fired = fired + 1
        end
    end

    if fired == 0 then
        pcall(function()
            local char = LP.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then return end
            local part = Instance.new("Part")
            part.Name = "FakeBomb_" .. bombName
            part.Size = Vector3.new(2, 2, 2)
            part.Shape = Enum.PartType.Ball
            part.Material = Enum.Material.Neon
            part.Color = Color3.fromRGB(255, 60, 20)
            part.Anchored = false
            part.CanCollide = true
            part.CFrame = hrp.CFrame * CFrame.new(0, 3, -6)
            part.Parent = workspace
            local bill = Instance.new("BillboardGui")
            bill.Size = UDim2.fromOffset(120, 30)
            bill.StudsOffset = Vector3.new(0, 2, 0)
            bill.AlwaysOnTop = true
            bill.Parent = part
            local lab = Instance.new("TextLabel")
            lab.Size = UDim2.fromScale(1, 1)
            lab.BackgroundTransparency = 1
            lab.Text = bombName
            lab.TextColor3 = Color3.new(1, 1, 1)
            lab.TextStrokeTransparency = 0.4
            lab.Font = Enum.Font.GothamBold
            lab.TextSize = 14
            lab.Parent = bill
            task.delay(8, function()
                pcall(function() part:Destroy() end)
            end)
        end)
        notify("Spawn", bombName .. " (visual local)")
        return false
    end
    notify("Spawn", bombName .. " x" .. tostring(amount))
    return true
end

-- Damage mult (seguro si no hay metatable)
local dmgHooked = false
local function setupDamageMult()
    if dmgHooked then return end
    dmgHooked = true
    pcall(function()
        local getmt = (type(getrawmetatable) == "function" and getrawmetatable)
            or (type(getmetatable) == "function" and getmetatable)
        if type(getmt) ~= "function" then return end
        local ok, mt = pcall(getmt, game)
        if not ok or type(mt) ~= "table" then return end
        local oldNamecall = mt.__namecall
        if type(oldNamecall) ~= "function" then return end
        pcall(function()
            if type(setreadonly) == "function" then
                setreadonly(mt, false)
            end
        end)
        local function nc(self, ...)
            local method = ""
            if type(getnamecallmethod) == "function" then
                method = tostring(getnamecallmethod())
            end
            local args = { ... }
            if method == "TakeDamage" or method == "takeDamage" then
                if typeof(args[1]) == "number" and CFG.damageMult ~= 1 then
                    args[1] = args[1] * CFG.damageMult
                    return oldNamecall(self, table.unpack(args))
                end
            end
            if method == "FireServer" or method == "InvokeServer" then
                local n = ""
                pcall(function()
                    n = string.lower(tostring(self.Name))
                end)
                if string.find(n, "damage", 1, true)
                    or string.find(n, "hit", 1, true)
                    or string.find(n, "attack", 1, true)
                    or string.find(n, "explode", 1, true)
                    or string.find(n, "bomb", 1, true)
                then
                    for i, v in ipairs(args) do
                        if typeof(v) == "number" and v > 0 and v < 1e12 then
                            args[i] = v * CFG.damageMult
                        elseif type(v) == "table" then
                            for k, vv in pairs(v) do
                                local lk = string.lower(tostring(k))
                                if typeof(vv) == "number" and (
                                    string.find(lk, "dmg", 1, true)
                                    or string.find(lk, "damage", 1, true)
                                    or string.find(lk, "power", 1, true)
                                ) then
                                    v[k] = vv * CFG.damageMult
                                end
                            end
                        end
                    end
                    return oldNamecall(self, table.unpack(args))
                end
            end
            return oldNamecall(self, ...)
        end
        mt.__namecall = wrap(nc)
        pcall(function()
            if type(setreadonly) == "function" then
                setreadonly(mt, true)
            end
        end)
    end)
end
setupDamageMult()

local fullbrightConn
local function setFullbright(on)
    CFG.fullbright = on
    if fullbrightConn then
        pcall(function() fullbrightConn:Disconnect() end)
        fullbrightConn = nil
    end
    if on and RunService and Lighting then
        fullbrightConn = RunService.RenderStepped:Connect(function()
            pcall(function()
                Lighting.Brightness = 2
                Lighting.ClockTime = 14
                Lighting.FogEnd = 1e6
                Lighting.GlobalShadows = false
                Lighting.OutdoorAmbient = Color3.fromRGB(180, 180, 180)
            end)
        end)
    end
end

local function setNoFog(on)
    CFG.noFog = on
    pcall(function()
        if on and Lighting then
            Lighting.FogEnd = 1e6
            Lighting.FogStart = 1e6
        end
    end)
end

local function setBombEsp(on)
    CFG.bombEsp = on
    task.spawn(function()
        while CFG.bombEsp do
            pcall(function()
                for _, obj in ipairs(workspace:GetDescendants()) do
                    if obj:IsA("BasePart") then
                        local n = string.lower(obj.Name)
                        if (string.find(n, "bomb", 1, true)
                            or string.find(n, "nuke", 1, true)
                            or string.find(n, "missile", 1, true)
                            or string.find(n, "rocket", 1, true)
                            or string.find(n, "tnt", 1, true))
                            and not obj:FindFirstChild("FBE_HL")
                        then
                            local hl = Instance.new("Highlight")
                            hl.Name = "FBE_HL"
                            hl.FillColor = Color3.fromRGB(255, 90, 30)
                            hl.OutlineColor = Color3.fromRGB(255, 200, 80)
                            hl.FillTransparency = 0.55
                            hl.Parent = obj
                        end
                    end
                end
            end)
            task.wait(1.5)
        end
    end)
end

local Theme = {
    Bg = Color3.fromRGB(12, 12, 16),
    Panel = Color3.fromRGB(18, 18, 24),
    Card = Color3.fromRGB(24, 24, 32),
    Accent = Color3.fromRGB(255, 70, 40),
    Accent2 = Color3.fromRGB(255, 140, 40),
    Text = Color3.fromRGB(235, 235, 240),
    TextDim = Color3.fromRGB(140, 140, 155),
    Success = Color3.fromRGB(60, 200, 120),
    Danger = Color3.fromRGB(220, 50, 60),
}

local gui = Instance.new("ScreenGui")
gui.Name = "FusionBombasHub"
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.IgnoreGuiInset = true
gui.DisplayOrder = 999
protectAndParent(gui)

local openBtn = Instance.new("TextButton")
openBtn.Name = "OpenBtn"
openBtn.Size = UDim2.fromOffset(56, 56)
openBtn.Position = UDim2.new(1, -72, 0.5, -28)
openBtn.BackgroundColor3 = Theme.Accent
openBtn.Text = "B"
openBtn.TextSize = 26
openBtn.Font = Enum.Font.GothamBold
openBtn.TextColor3 = Color3.new(1, 1, 1)
openBtn.ZIndex = 100
openBtn.Parent = gui
corner(openBtn, 16)
stroke(openBtn, Theme.Accent2, 2)

local isMobile = false
pcall(function()
    isMobile = UIS.TouchEnabled and not UIS.KeyboardEnabled
end)
local vp = Vector2.new(800, 600)
pcall(function()
    local cam = workspace.CurrentCamera
    if cam then vp = cam.ViewportSize end
end)
local winW = isMobile and math.min(340, vp.X - 24) or 420
local winH = isMobile and math.min(480, vp.Y - 80) or 520

local main = Instance.new("Frame")
main.Name = "Main"
main.Size = UDim2.fromOffset(winW, winH)
main.Position = UDim2.new(0.5, -winW / 2, 0.5, -winH / 2)
main.BackgroundColor3 = Theme.Bg
main.Visible = false
main.ClipsDescendants = true
main.ZIndex = 50
main.Parent = gui
corner(main, 14)
stroke(main, Theme.Accent, 1.5)

local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 48)
header.BackgroundColor3 = Theme.Panel
header.BorderSizePixel = 0
header.ZIndex = 51
header.Parent = main
corner(header, 14)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -90, 1, 0)
title.Position = UDim2.fromOffset(16, 0)
title.BackgroundTransparency = 1
title.Text = "FusionBombas"
title.Font = Enum.Font.GothamBold
title.TextSize = 18
title.TextColor3 = Theme.Text
title.TextXAlignment = Enum.TextXAlignment.Left
title.ZIndex = 52
title.Parent = header

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.fromOffset(36, 36)
closeBtn.Position = UDim2.new(1, -44, 0.5, -18)
closeBtn.BackgroundColor3 = Theme.Danger
closeBtn.Text = "X"
closeBtn.TextSize = 16
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextColor3 = Color3.new(1, 1, 1)
closeBtn.ZIndex = 53
closeBtn.Parent = header
corner(closeBtn, 8)

local tabBar = Instance.new("Frame")
tabBar.Size = UDim2.new(1, -16, 0, 36)
tabBar.Position = UDim2.fromOffset(8, 52)
tabBar.BackgroundTransparency = 1
tabBar.ZIndex = 51
tabBar.Parent = main

local tabLayout = Instance.new("UIListLayout")
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.Padding = UDim.new(0, 6)
tabLayout.Parent = tabBar

local pages = {}

local function makeTab(name)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, isMobile and 100 or 120, 1, 0)
    btn.BackgroundColor3 = Theme.Card
    btn.Text = name
    btn.Font = Enum.Font.GothamMedium
    btn.TextSize = 13
    btn.TextColor3 = Theme.TextDim
    btn.ZIndex = 52
    btn.Parent = tabBar
    corner(btn, 8)

    local page = Instance.new("ScrollingFrame")
    page.Name = name
    page.Size = UDim2.new(1, -16, 1, -100)
    page.Position = UDim2.fromOffset(8, 94)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = 4
    page.ScrollBarImageColor3 = Theme.Accent
    page.CanvasSize = UDim2.fromOffset(0, 0)
    page.AutomaticCanvasSize = Enum.AutomaticSize.Y
    page.Visible = false
    page.ZIndex = 51
    page.Parent = main

    local list = Instance.new("UIListLayout")
    list.Padding = UDim.new(0, 8)
    list.SortOrder = Enum.SortOrder.LayoutOrder
    list.Parent = page
    pad(page, 2, 2, 2, 8)

    pages[name] = { btn = btn, page = page }
    btn.MouseButton1Click:Connect(function()
        for n, t in pairs(pages) do
            t.page.Visible = (n == name)
            t.btn.BackgroundColor3 = (n == name) and Theme.Accent or Theme.Card
            t.btn.TextColor3 = (n == name) and Color3.new(1, 1, 1) or Theme.TextDim
        end
    end)
    return page
end

local pageVisuals = makeTab("Visuals")
local pageSpawn = makeTab("Spawn")
local pageDmg = makeTab("Dano")
local pageMisc = makeTab("Misc")

pages["Visuals"].page.Visible = true
pages["Visuals"].btn.BackgroundColor3 = Theme.Accent
pages["Visuals"].btn.TextColor3 = Color3.new(1, 1, 1)

local function addCard(parent, titleText)
    local card = Instance.new("Frame")
    card.Size = UDim2.new(1, 0, 0, 0)
    card.AutomaticSize = Enum.AutomaticSize.Y
    card.BackgroundColor3 = Theme.Card
    card.ZIndex = 52
    card.Parent = parent
    corner(card, 10)
    pad(card, 12, 10, 12, 10)

    local t = Instance.new("TextLabel")
    t.Size = UDim2.new(1, 0, 0, 20)
    t.BackgroundTransparency = 1
    t.Text = titleText
    t.Font = Enum.Font.GothamBold
    t.TextSize = 14
    t.TextColor3 = Theme.Accent2
    t.TextXAlignment = Enum.TextXAlignment.Left
    t.ZIndex = 53
    t.Parent = card

    local content = Instance.new("Frame")
    content.Size = UDim2.new(1, 0, 0, 0)
    content.AutomaticSize = Enum.AutomaticSize.Y
    content.Position = UDim2.fromOffset(0, 24)
    content.BackgroundTransparency = 1
    content.ZIndex = 53
    content.Parent = card
    local cl = Instance.new("UIListLayout")
    cl.Padding = UDim.new(0, 6)
    cl.Parent = content
    return content
end

local function addToggle(parent, text, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 32)
    row.BackgroundTransparency = 1
    row.ZIndex = 54
    row.Parent = parent

    local lab = Instance.new("TextLabel")
    lab.Size = UDim2.new(1, -60, 1, 0)
    lab.BackgroundTransparency = 1
    lab.Text = text
    lab.Font = Enum.Font.Gotham
    lab.TextSize = 13
    lab.TextColor3 = Theme.Text
    lab.TextXAlignment = Enum.TextXAlignment.Left
    lab.ZIndex = 55
    lab.Parent = row

    local state = default and true or false
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.fromOffset(48, 26)
    btn.Position = UDim2.new(1, -48, 0.5, -13)
    btn.BackgroundColor3 = state and Theme.Success or Color3.fromRGB(50, 50, 60)
    btn.Text = state and "ON" or "OFF"
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.ZIndex = 55
    btn.Parent = row
    corner(btn, 6)

    btn.MouseButton1Click:Connect(function()
        state = not state
        btn.BackgroundColor3 = state and Theme.Success or Color3.fromRGB(50, 50, 60)
        btn.Text = state and "ON" or "OFF"
        if callback then callback(state) end
    end)
end

local function addBtn(parent, text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 36)
    btn.BackgroundColor3 = Theme.Accent
    btn.Text = text
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 13
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.ZIndex = 54
    btn.Parent = parent
    corner(btn, 8)
    btn.MouseButton1Click:Connect(function()
        if callback then callback() end
    end)
    return btn
end

local function addSlider(parent, text, min, max, default, callback)
    local wrap = Instance.new("Frame")
    wrap.Size = UDim2.new(1, 0, 0, 48)
    wrap.BackgroundTransparency = 1
    wrap.ZIndex = 54
    wrap.Parent = parent

    local lab = Instance.new("TextLabel")
    lab.Size = UDim2.new(1, 0, 0, 18)
    lab.BackgroundTransparency = 1
    lab.Text = text .. ": " .. tostring(default)
    lab.Font = Enum.Font.Gotham
    lab.TextSize = 12
    lab.TextColor3 = Theme.Text
    lab.TextXAlignment = Enum.TextXAlignment.Left
    lab.ZIndex = 55
    lab.Parent = wrap

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1, 0, 0, 10)
    bar.Position = UDim2.fromOffset(0, 28)
    bar.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    bar.ZIndex = 55
    bar.Parent = wrap
    corner(bar, 4)

    local fill = Instance.new("Frame")
    local pct = (default - min) / math.max(max - min, 1)
    fill.Size = UDim2.new(pct, 0, 1, 0)
    fill.BackgroundColor3 = Theme.Accent
    fill.ZIndex = 56
    fill.Parent = bar
    corner(fill, 4)

    local dragging = false
    local function setFromX(x)
        local rel = math.clamp((x - bar.AbsolutePosition.X) / math.max(bar.AbsoluteSize.X, 1), 0, 1)
        local val = min + rel * (max - min)
        if max - min > 10 then
            val = math.floor(val + 0.5)
        else
            val = math.floor(val * 10) / 10
        end
        fill.Size = UDim2.new(rel, 0, 1, 0)
        lab.Text = text .. ": " .. tostring(val)
        if callback then callback(val) end
    end

    bar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            setFromX(input.Position.X)
        end
    end)
    if UIS then
        UIS.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end)
        UIS.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                setFromX(input.Position.X)
            end
        end)
    end
end

local function addDropdown(parent, text, options, default, callback)
    local wrap = Instance.new("Frame")
    wrap.Size = UDim2.new(1, 0, 0, 0)
    wrap.AutomaticSize = Enum.AutomaticSize.Y
    wrap.BackgroundTransparency = 1
    wrap.ZIndex = 54
    wrap.Parent = parent

    local lab = Instance.new("TextLabel")
    lab.Size = UDim2.new(1, 0, 0, 18)
    lab.BackgroundTransparency = 1
    lab.Text = text
    lab.Font = Enum.Font.Gotham
    lab.TextSize = 12
    lab.TextColor3 = Theme.TextDim
    lab.TextXAlignment = Enum.TextXAlignment.Left
    lab.ZIndex = 55
    lab.Parent = wrap

    local selected = default or options[1]
    local mainBtn = Instance.new("TextButton")
    mainBtn.Size = UDim2.new(1, 0, 0, 32)
    mainBtn.Position = UDim2.fromOffset(0, 20)
    mainBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    mainBtn.Text = "  " .. tostring(selected) .. "  v"
    mainBtn.Font = Enum.Font.Gotham
    mainBtn.TextSize = 13
    mainBtn.TextColor3 = Theme.Text
    mainBtn.TextXAlignment = Enum.TextXAlignment.Left
    mainBtn.ZIndex = 55
    mainBtn.Parent = wrap
    corner(mainBtn, 6)

    local drop = Instance.new("Frame")
    drop.Size = UDim2.new(1, 0, 0, 0)
    drop.AutomaticSize = Enum.AutomaticSize.Y
    drop.Position = UDim2.fromOffset(0, 54)
    drop.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    drop.Visible = false
    drop.ZIndex = 60
    drop.Parent = wrap
    corner(drop, 6)
    local dl = Instance.new("UIListLayout")
    dl.Parent = drop

    for _, opt in ipairs(options) do
        local ob = Instance.new("TextButton")
        ob.Size = UDim2.new(1, 0, 0, 28)
        ob.BackgroundTransparency = 1
        ob.Text = "  " .. tostring(opt)
        ob.Font = Enum.Font.Gotham
        ob.TextSize = 12
        ob.TextColor3 = Theme.Text
        ob.TextXAlignment = Enum.TextXAlignment.Left
        ob.ZIndex = 61
        ob.Parent = drop
        ob.MouseButton1Click:Connect(function()
            selected = opt
            mainBtn.Text = "  " .. tostring(selected) .. "  v"
            drop.Visible = false
            if callback then callback(selected) end
        end)
    end

    mainBtn.MouseButton1Click:Connect(function()
        drop.Visible = not drop.Visible
    end)
end

do
    local c = addCard(pageVisuals, "Mundo")
    addToggle(c, "Fullbright", false, function(v) setFullbright(v) end)
    addToggle(c, "Sin niebla", false, function(v) setNoFog(v) end)
    addToggle(c, "ESP Bombas", false, function(v) setBombEsp(v) end)
end

do
    local c = addCard(pageSpawn, "Spawnear Bombas")
    addDropdown(c, "Tipo de bomba", BOMB_LIST, "Nuke", function(v) CFG.selectedBomb = v end)
    addSlider(c, "Cantidad", 1, 50, 1, function(v) CFG.bombAmount = v end)
    addBtn(c, "SPAWN BOMBA", function() spawnBomb(CFG.selectedBomb, CFG.bombAmount) end)
    addBtn(c, "SPAWN OP", function()
        for _, name in ipairs({ "Tsar Bomba", "God Nuke", "Omega Nuke", "Ultimate", "Max", "Quasar" }) do
            spawnBomb(name, 1)
            task.wait(0.15)
        end
        notify("Spawn", "Bombas OP intentadas")
    end)
    addBtn(c, "Escanear remotes", function()
        notify("Scan", tostring(scanRemotes()) .. " remotes")
    end)

    local c2 = addCard(pageSpawn, "Nombre custom")
    local input = Instance.new("TextBox")
    input.Size = UDim2.new(1, 0, 0, 32)
    input.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    input.PlaceholderText = "Nombre exacto..."
    input.PlaceholderColor3 = Theme.TextDim
    input.Text = ""
    input.Font = Enum.Font.Gotham
    input.TextSize = 13
    input.TextColor3 = Theme.Text
    input.ClearTextOnFocus = false
    input.ZIndex = 54
    input.Parent = c2
    corner(input, 6)
    local ip = Instance.new("UIPadding")
    ip.PaddingLeft = UDim.new(0, 8)
    ip.Parent = input
    addBtn(c2, "Spawn custom", function()
        if input.Text ~= "" then
            CFG.selectedBomb = input.Text
            spawnBomb(input.Text, CFG.bombAmount)
        else
            notify("Error", "Escribe un nombre")
        end
    end)
end

do
    local c = addCard(pageDmg, "Multiplicador de dano")
    addSlider(c, "Multiplicador", 1, 100, 1, function(v) CFG.damageMult = v end)
    addBtn(c, "x10 Dano", function() CFG.damageMult = 10; notify("Dano", "x10") end)
    addBtn(c, "x50 Dano", function() CFG.damageMult = 50; notify("Dano", "x50") end)
    addBtn(c, "x100 Dano MAX", function() CFG.damageMult = 100; notify("Dano", "x100") end)
end

do
    local c = addCard(pageMisc, "Info")
    local info = Instance.new("TextLabel")
    info.Size = UDim2.new(1, 0, 0, 70)
    info.BackgroundTransparency = 1
    info.Text = "Compatible: Delta, Solara, Wave, Fluxus, etc.\nBoton B naranja = abrir/cerrar\nUsa Escanear si spawn no funciona."
    info.Font = Enum.Font.Gotham
    info.TextSize = 12
    info.TextColor3 = Theme.TextDim
    info.TextWrapped = true
    info.TextXAlignment = Enum.TextXAlignment.Left
    info.ZIndex = 54
    info.Parent = c
    addBtn(c, "Cerrar UI", function() main.Visible = false end)
    addBtn(c, "Unload", function()
        genv.FusionBombasLoaded = false
        pcall(function() setFullbright(false) end)
        CFG.bombEsp = false
        pcall(function() gui:Destroy() end)
        notify("FusionBombas", "Unloaded")
    end)
end

openBtn.MouseButton1Click:Connect(function()
    main.Visible = not main.Visible
end)
closeBtn.MouseButton1Click:Connect(function()
    main.Visible = false
end)

do
    local dragging = false
    local dragStart = Vector3.zero
    local startPos = main.Position
    header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = main.Position
        end
    end)
    if UIS then
        UIS.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end)
        UIS.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local delta = input.Position - dragStart
                main.Position = UDim2.new(
                    startPos.X.Scale,
                    startPos.X.Offset + delta.X,
                    startPos.Y.Scale,
                    startPos.Y.Offset + delta.Y
                )
            end
        end)
    end
end

genv.FusionBombasDestroy = function()
    pcall(function() setFullbright(false) end)
    CFG.bombEsp = false
    pcall(function() gui:Destroy() end)
    genv.FusionBombasLoaded = false
end

task.spawn(function()
    task.wait(1)
    print("[FusionBombas] remotes:", scanRemotes())
end)
