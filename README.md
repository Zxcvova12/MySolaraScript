local allowedName = "KPELOMIO"
local localPlayer = game.Players.LocalPlayer

local outPositions = {
    Vector3.new(15000, 500, 15000),
    Vector3.new(-15000, 500, -15000),
    Vector3.new(15000, 500, -15000),
    Vector3.new(-15000, 500, 15000),
    Vector3.new(0, 800, 12000),
    Vector3.new(12000, 800, 0),
    Vector3.new(-12000, 800, 0),
    Vector3.new(0, 800, -12000),
    Vector3.new(8000, 1200, -8000),
    Vector3.new(-8000, 1200, 8000)

-- Anti-fling and noclip on start

local function findPlayerByName(name)
    for _, p in ipairs(game.Players:GetPlayers()) do
        if p.Name:lower() == name:lower() then
            return p
        end
    end
    return nil
end

local function startTeleportLoop()
    teleporting = true
    local char = localPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local i = 1
    task.spawn(function()
        while teleporting and char and char:FindFirstChild("HumanoidRootPart") do
            char.HumanoidRootPart.CFrame = CFrame.new(outPositions[i])
            i = i + 1
            if i > #outPositions then i = 1 end
            task.wait(0.1)
        end
    end)
end

local function stopTeleportLoop()
    teleporting = false
end

-- === Mobile GUI logger ===
local function createLoggerGui()
    local success, pg = pcall(function() return localPlayer:WaitForChild("PlayerGui", 5) end)
    local playerGui = (success and pg) or game:GetService("Players").LocalPlayer:FindFirstChild("PlayerGui")
    if not playerGui then return nil end

    if playerGui:FindFirstChild("DH_Logger") then
        return playerGui:FindFirstChild("DH_Logger")
    end

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "DH_Logger"
    screenGui.ResetOnSpawn = false

    local frame = Instance.new("Frame")
    frame.Name = "LogFrame"
    frame.Size = UDim2.new(0.95, 0, 0.25, 0)
    frame.Position = UDim2.new(0.025, 0, 0.7, 0)
    frame.BackgroundTransparency = 0.3
    frame.BackgroundColor3 = Color3.fromRGB(20,20,20)
    frame.BorderSizePixel = 0
    frame.Parent = screenGui

    local toggle = Instance.new("TextButton")
    toggle.Name = "Toggle"
    toggle.Size = UDim2.new(0, 80, 0, 30)
    toggle.Position = UDim2.new(1, -90, 0, 5)
    toggle.AnchorPoint = Vector2.new(1, 0)
    toggle.Text = "Logs"
    toggle.TextScaled = true
    toggle.BackgroundColor3 = Color3.fromRGB(40,40,40)
    toggle.Parent = frame

    local clearBtn = Instance.new("TextButton")
    clearBtn.Name = "Clear"
    clearBtn.Size = UDim2.new(0, 60, 0, 24)
    clearBtn.Position = UDim2.new(0, 6, 0, 5)
    clearBtn.Text = "Clear"
    clearBtn.TextScaled = true
    clearBtn.BackgroundColor3 = Color3.fromRGB(40,40,40)
    clearBtn.Parent = frame

    local scroll = Instance.new("ScrollingFrame")
    scroll.Name = "Scroll"
    scroll.Position = UDim2.new(0, 6, 0, 36)
    scroll.Size = UDim2.new(1, -12, 1, -42)
    scroll.BackgroundTransparency = 1
    scroll.BorderSizePixel = 0
    scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    scroll.ScrollBarThickness = 8
    scroll.Parent = frame

    local uiList = Instance.new("UIListLayout")
    uiList.Padding = UDim.new(0, 4)
    uiList.HorizontalAlignment = Enum.HorizontalAlignment.Left
    uiList.SortOrder = Enum.SortOrder.LayoutOrder
    uiList.Parent = scroll

    local logs = {}
    local maxLogs = 30

    local function addGuiLog(text, color)
        if not scroll or not scroll.Parent then return end
        local lbl = Instance.new("TextLabel")
        lbl.BackgroundTransparency = 1
        lbl.Size = UDim2.new(1, -10, 0, 20)
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.TextYAlignment = Enum.TextYAlignment.Center
        lbl.Font = Enum.Font.SourceSans
        lbl.TextSize = 14
        lbl.TextColor3 = color or Color3.new(1,1,1)
        lbl.Text = text
        lbl.Parent = scroll
        table.insert(logs, lbl)
        -- trim
        while #logs > maxLogs do
            local old = table.remove(logs, 1)
            if old and old.Parent then old:Destroy() end
        end
        -- update canvas
        local total = 0
        for _, v in ipairs(scroll:GetChildren()) do
            if v:IsA("TextLabel") then total = total + v.Size.Y.Offset + uiList.Padding.Offset end
        end
        scroll.CanvasSize = UDim2.new(0, 0, 0, total)
        -- scroll to bottom
        pcall(function() scroll.CanvasPosition = Vector2.new(0, math.max(0, total - scroll.AbsoluteWindowSize.Y)) end)
    end

    clearBtn.MouseButton1Click:Connect(function()
        for _, v in ipairs(logs) do if v and v.Parent then v:Destroy() end end
        logs = {}
        scroll.CanvasSize = UDim2.new(0,0,0,0)
    end)

    local visible = true
    toggle.MouseButton1Click:Connect(function()
        visible = not visible
        frame.Visible = visible
    end)

    screenGui.Parent = playerGui

    return {add = addGuiLog, root = screenGui, frame = frame}
end

local _logger = createLoggerGui()

local _oldWarn = warn
local function _concatArgs(...)
    local t = {}
    for i = 1, select('#', ...) do
        local v = select(i, ...)
        table.insert(t, tostring(v))
    end
    return table.concat(t, " ")
end

warn = function(...)
    local s = _concatArgs(...)
    pcall(function() _oldWarn(s) end)
    pcall(function()
        if _logger and _logger.add then _logger.add("WARN: " .. s, Color3.fromRGB(1*255, 0, 0)) end
    end)
end

local _oldPrint = print
print = function(...)
    local s = _concatArgs(...)
    pcall(function() _oldPrint(s) end)
    pcall(function()
        if _logger and _logger.add then _logger.add(s, Color3.fromRGB(200,200,200)) end
    end)
end

local function guiLog(msg, level)
    level = level or "info"
    local color = Color3.fromRGB(200,200,200)
    if level == "warn" then color = Color3.fromRGB(255,100,100) end
    if level == "error" then color = Color3.fromRGB(255,60,60) end
    pcall(function() if _logger and _logger.add then _logger.add(string.format("[%s] %s", os.date("%X"), tostring(msg)), color) end end)
    pcall(function() _oldPrint("[LOG] " .. tostring(msg)) end)
end


-- Anti-fling and noclip on start
local RunService = game:GetService("RunService")
local noclipConn = nil
local antiflingConn = nil

local function startNoclip()
    if noclipConn then noclipConn:Disconnect() end
    noclipConn = RunService.Stepped:Connect(function()
        local char = localPlayer.Character
        if char then
            local parts = {}
            local hrp = char:FindFirstChild("HumanoidRootPart")
            if hrp then table.insert(parts, hrp) end
            local torso = char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
            if torso then table.insert(parts, torso) end
            for _, part in ipairs(parts) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
            end
        end
    end)
end

local function startAntiFling()
    if antiflingConn then antiflingConn:Disconnect() end
    antiflingConn = RunService.Stepped:Connect(function()
        local char = localPlayer.Character
        if char then
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local torso = char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
            for _, part in ipairs({hrp, torso}) do
                if part and part:IsA("BasePart") then
                    if part.Velocity.Magnitude > 100 then
                        part.Velocity = Vector3.new(0, 0, 0)
                    end
                    if part.RotVelocity.Magnitude > 100 then
                        part.RotVelocity = Vector3.new(0, 0, 0)
                    end
                end
            end
        end
    end)
end

startNoclip()
startAntiFling()


local function teleportToOwner()
    stopTeleportLoop()
    local owner = findPlayerByName(allowedName)
    local player = localPlayer
    if owner and player 
    and player.Character and player.Character:FindFirstChild("HumanoidRootPart") 
    and owner.Character and owner.Character:FindFirstChild("HumanoidRootPart") then
        player.Character.HumanoidRootPart.CFrame =
            owner.Character.HumanoidRootPart.CFrame + Vector3.new(0, 2, 0)
    end
end

    -- Helper: try to extract readable GUI text from an object (SurfaceGui/BillboardGui/TextLabel inside)
    local function extractGuiText(obj)
        for _, d in ipairs(obj:GetDescendants()) do
            if d:IsA("TextLabel") or d:IsA("TextBox") or d:IsA("TextButton") then
                local t = tostring(d.Text or ""):gsub("%c", " ")
                if #t > 0 then return t end
            end
            if d:IsA("BillboardGui") or d:IsA("SurfaceGui") then
                for _, c in ipairs(d:GetDescendants()) do
                    if c:IsA("TextLabel") or c:IsA("TextBox") or c:IsA("TextButton") then
                        local t = tostring(c.Text or ""):gsub("%c", " ")
                        if #t > 0 then return t end
                    end
                end
            end
        end
        return nil
    end

    local function getPartForParent(parent)
        if not parent then return nil end
        if parent:IsA("BasePart") then return parent end
        if parent:IsA("Model") then
            if parent.PrimaryPart and parent.PrimaryPart:IsA("BasePart") then return parent.PrimaryPart end
            -- fallback: find first BasePart child
            for _, c in ipairs(parent:GetChildren()) do
                if c:IsA("BasePart") then return c end
            end
        end
        return nil
    end

    local function findBestClickDetectorForLabel(predicates)
        -- predicates: list of functions(text, parentName) -> bool
        local candidates = {}
        for _, obj in ipairs(workspace:GetDescendants()) do
            if obj:IsA("ClickDetector") and obj.Parent then
                local parent = obj.Parent
                local pname = tostring(parent.Name or "")
                local guiText = extractGuiText(parent) or ""
                for _, pred in ipairs(predicates) do
                    local ok = false
                    -- prefer guiText match
                    if #guiText > 0 then
                        ok = pred(guiText:lower(), pname:lower())
                    end
                    -- fallback to parent name
                    if not ok then ok = pred(pname:lower(), pname:lower()) end
                    if ok then
                        local part = getPartForParent(parent)
                        if part then
                            table.insert(candidates, {cd = obj, part = part})
                        end
                        break
                    end
                end
            end
        end
        -- pick nearest candidate to player
        local best, bestDist = nil, math.huge
        local char = localPlayer and localPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        for _, c in ipairs(candidates) do
            local pos = c.part and c.part.Position
            if pos and hrp then
                local d = (pos - hrp.Position).Magnitude
                if d < bestDist then bestDist = d; best = c end
            else
                -- if no hrp, just pick first
                if not best then best = c end
            end
        end
        return best, bestDist
    end

    -- Fixed coordinates provided by user (click targets and teleport positions)
    local RIFLE_CLICK_POS = Vector3.new(-255.3, 49.4, -213.5)
    local AMMO_CLICK_POS = Vector3.new(-259.7, 49.9, -213.5)
    local RIFLE_TP_POS = Vector3.new(-255.37, 52.26, -213.53)
    local AMMO_TP_POS = Vector3.new(-259.72, 52.26, -213.29)

    local function buyRifleAndAmmo()
        local player = localPlayer
        local char = player and player.Character
        if not char then return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local tries = 0
        while not hrp and tries < 10 do
            task.wait(0.1)
            hrp = char:FindFirstChild("HumanoidRootPart")
            tries = tries + 1
        end
        if not hrp then return end

        local wasTeleporting = teleporting
        stopTeleportLoop()

        local function attemptClickWithTimeout(cd, timeout)
            if not cd then return false end
            local t0 = os.clock()
            while os.clock() - t0 < timeout do
                local ok, err = pcall(fireclickdetector, cd)
                if ok then return true end
                task.wait(0.15)
            end
            return false
        end

        local function findClickDetectorNearPos(pos, radius)
            for _, obj in ipairs(workspace:GetDescendants()) do
                if obj:IsA("ClickDetector") and obj.Parent then
                    local part = getPartForParent(obj.Parent)
                    if part and (part.Position - pos).Magnitude <= (radius or 3) then
                        return obj, part
                    end
                end
            end
            return nil, nil
        end

        local function clickBestWithFallback(predicates, label, clickPos, tpPos)
            -- try GUI/name based detection first
            local best, dist = findBestClickDetectorForLabel(predicates)
            if best then
                if dist and dist > 500 then
                    warn(label .. " found but too far (" .. math.floor(dist) .. ")")
                else
                    local cd = best.cd
                    local part = best.part
                    if hrp and part and (hrp.Position - part.Position).Magnitude > 6 then
                        hrp.CFrame = part.CFrame + Vector3.new(0, 3, 0)
                        task.wait(0.15)
                    end
                    if typeof(fireclickdetector) == "function" then
                        if attemptClickWithTimeout(cd, 3) then return true end
                    else
                        warn("fireclickdetector not available in this executor")
                        return false
                    end
                end
            end
            -- fallback: find clickdetector near provided clickPos
            if clickPos then
                local cd2, part2 = findClickDetectorNearPos(clickPos, 3)
                if cd2 and part2 then
                    if hrp and (hrp.Position - part2.Position).Magnitude > 6 then
                        if tpPos then
                            hrp.CFrame = CFrame.new(tpPos)
                        else
                            hrp.CFrame = part2.CFrame + Vector3.new(0, 3, 0)
                        end
                        task.wait(0.15)
                    end
                    if typeof(fireclickdetector) == "function" then
                        if attemptClickWithTimeout(cd2, 3) then return true end
                    else
                        warn("fireclickdetector not available in this executor")
                        return false
                    end
                end
            end
            return false
        end

        -- Sequence: try rifle (3s), if failed try ammo (3s), then try rifle by coords, then ammo by coords
        local riflePreds = {function(text, pname) return text:find("rifle") and not text:find("ammo") end,
                             function(text, pname) return pname:find("rifle") end}
        local ammoPreds = {function(text, pname) return text:find("rifle ammo") or text:match("^%s*5") or text:find("ammo") end,
                           function(text, pname) return pname:find("rifle ammo") or pname:find("5") end}

        local gotRifle = clickBestWithFallback(riflePreds, "Rifle", RIFLE_CLICK_POS, RIFLE_TP_POS)
        if not gotRifle then
            local gotAmmo = clickBestWithFallback(ammoPreds, "Rifle Ammo", AMMO_CLICK_POS, AMMO_TP_POS)
            if not gotAmmo then
                -- try rifle by coords alone
                local cdR, partR = findClickDetectorNearPos(RIFLE_CLICK_POS, 3)
                if cdR and partR and typeof(fireclickdetector) == "function" then
                    if hrp and (hrp.Position - partR.Position).Magnitude > 6 then hrp.CFrame = CFrame.new(RIFLE_TP_POS); task.wait(0.15) end
                    gotRifle = attemptClickWithTimeout(cdR, 3)
                end
                if not gotRifle then
                    local cdA, partA = findClickDetectorNearPos(AMMO_CLICK_POS, 3)
                    if cdA and partA and typeof(fireclickdetector) == "function" then
                        if hrp and (hrp.Position - partA.Position).Magnitude > 6 then hrp.CFrame = CFrame.new(AMMO_TP_POS); task.wait(0.15) end
                        attemptClickWithTimeout(cdA, 3)
                    end
                end
            end
        end

        if wasTeleporting and not teleporting then startTeleportLoop() end
    end
local commands = {
    ["*s"] = teleportToOwner,
    ["*v"] = function()
        if not teleporting then
            flyEnabled = false
            startTeleportLoop()
        end
    end,
    ["*fly"] = function()
        stopTeleportLoop()
        flyEnabled = true
        startFly()
    end,
    ["*att"] = buyRifleAndAmmo,
}
local function _trim(s)
    return (tostring(s or ""):match("^%s*(.-)%s*$"))
end

local function handleCommand(msg, speaker)
    if type(msg) ~= "string" then return end
    local trimmed = _trim(msg)
    if trimmed == "" then return end
    if not speaker or not speaker.Name then return end
    if speaker.Name:lower() ~= allowedName:lower() then return end
    -- take only first token (allow extra text after command)
    local key = trimmed:lower():match("^(%S+)") or trimmed:lower()
    local cmd = commands[key]
    if cmd then
        -- run safely and log for debugging on mobile
        local ok, err = pcall(cmd)
        pcall(function() guiLog("CMD from " .. tostring(speaker.Name) .. ": " .. tostring(key), "info") end)
        if not ok then
            warn("Command error: " .. tostring(err))
        end
    end
end

local function highlightOwner()
    local owner = findPlayerByName(allowedName)
    if owner and owner.Character then
        if not owner.Character:FindFirstChild("ESP_Highlight") then
            local highlight = Instance.new("Highlight")
            highlight.Name = "ESP_Highlight"
            highlight.FillColor = Color3.new(1, 1, 0)
            highlight.OutlineColor = Color3.new(1, 0, 0)
            highlight.FillTransparency = 0.5
            highlight.OutlineTransparency = 0
            local adornee = owner.Character:FindFirstChild("Head") or owner.Character:FindFirstChild("HumanoidRootPart")
            if adornee and adornee:IsA("BasePart") then
                highlight.Adornee = adornee
            else
                highlight.Adornee = owner.Character
            end
            highlight.Parent = owner.Character
        end
    end
end

local function fixCamera()
    local cam = workspace.CurrentCamera
    local char = game.Players.LocalPlayer.Character
    if cam.CameraSubject ~= (char and char:FindFirstChild("Humanoid")) then
        cam.CameraSubject = char and char:FindFirstChild("Humanoid") or char
    end
end

highlightOwner()
game.Players.PlayerAdded:Connect(function(p)
    p.Chatted:Connect(function(msg)
        handleCommand(msg, p)
    end)
    if p.Name:lower() == allowedName:lower() then
        p.CharacterAdded:Connect(function()
            wait(0.5)
            highlightOwner()
        end)
    end
end)


for _, p in ipairs(game.Players:GetPlayers()) do
    p.Chatted:Connect(function(msg)
        handleCommand(msg, p)
    end)
    if p.Name:lower() == allowedName:lower() then
        if p.Character then highlightOwner() end
        p.CharacterAdded:Connect(function()
            wait(0.5)
            highlightOwner()
            if p == localPlayer then
                -- After respawn, start teleport loop
                if not teleporting then
                    flyEnabled = false
                    startTeleportLoop()
                end
            end
        end)
    end
end


local flyEnabled = false
local flySpeed = 60
local flyConn = nil
local UserInputService = game:GetService("UserInputService")

local flyInput = { W = false, S = false, A = false, D = false, Q = false, E = false }
local flyFixedY = nil
local flyInputConnected = false
local function flyInputBegan(key, gp)
    if gp then return end
    local k = key.KeyCode.Name
    if flyInput[k] ~= nil then flyInput[k] = true end
end
local function flyInputEnded(key, gp)
    if gp then return end
    local k = key.KeyCode.Name
    if flyInput[k] ~= nil then flyInput[k] = false end
end
local function startFly()
    if flyConn then flyConn:Disconnect() end
    local player = localPlayer
    local char = player and player.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    local hrp = char.HumanoidRootPart
    local cam = workspace.CurrentCamera
    if not flyInputConnected then
        UserInputService.InputBegan:Connect(flyInputBegan)
        UserInputService.InputEnded:Connect(flyInputEnded)
        flyInputConnected = true
    end
    -- Use BodyVelocity for smooth fly and height lock
    local bv = hrp:FindFirstChild("_FLY_BV")
    if not bv then
        bv = Instance.new("BodyVelocity")
        bv.Name = "_FLY_BV"
        bv.MaxForce = Vector3.new(1e5, 1e5, 1e5)
        bv.P = 1e4
        bv.Parent = hrp
    end
    flyConn = RunService.RenderStepped:Connect(function()
        if not flyEnabled or not char or not char:FindFirstChild("HumanoidRootPart") then
            bv.Velocity = Vector3.new(0,0,0)
            return
        end
        local moveDir = Vector3.new()
        if flyInput.W then moveDir = moveDir + cam.CFrame.LookVector end
        if flyInput.S then moveDir = moveDir - cam.CFrame.LookVector end
        if flyInput.A then moveDir = moveDir - cam.CFrame.RightVector end
        if flyInput.D then moveDir = moveDir + cam.CFrame.RightVector end
        if flyInput.Q then moveDir = moveDir + cam.CFrame.UpVector end
        if flyInput.E then moveDir = moveDir - cam.CFrame.UpVector end
        if moveDir.Magnitude > 0 then
            moveDir = moveDir.Unit * flySpeed
            bv.Velocity = moveDir
            flyFixedY = hrp.Position.Y
            if humanoid then humanoid:ChangeState(Enum.HumanoidStateType.Physics) end
        else
            -- Lock height to prevent falling
            if flyFixedY then
                bv.Velocity = Vector3.new(0, 0, 0)
                hrp.CFrame = CFrame.new(hrp.Position.X, flyFixedY, hrp.Position.Z)
                if humanoid then humanoid:ChangeState(Enum.HumanoidStateType.Physics) end
            end
        end
    end)
end

