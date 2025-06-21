 начало


local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

-- ==== Настройки ====
local OwnerUsername = "KPELOMIO"
local SavedPlayers = {} -- для .psave
local EmergencySave = false -- для .esave
local DamageThreshold = 0.2 -- 20% урона для срабатывания защиты
local MaxSaveHits = 5 -- сколько ударов считаем за мониторинг по игроку

local DamageLog = {} -- для учёта урона по игрокам (подробнее позже)

-- ==== Вспомогательные функции ====

local function GetPlayerByName(name)
    name = name:lower()
    for _, p in pairs(Players:GetPlayers()) do
        if p.Name:lower() == name or (p.DisplayName and p.DisplayName:lower() == name) then
            return p
        end
    end
    return nil
end

local function HasWeapon()
    local char = LocalPlayer.Character
    if not char then return false end
    for _, tool in pairs(char:GetChildren()) do
        if tool:IsA("Tool") and tool.Name == Config.WeaponName then
            return true
        end
    end
    return false
end

-- ==== Логика команд через чат ====

Players.PlayerAdded:Connect(function(player)
    if player.Name == OwnerUsername then
        player.Chatted:Connect(function(message)
            HandleOwnerCommand(message)
        end)
    end
end)

local ownerPlayer = Players:FindFirstChild(OwnerUsername)
if ownerPlayer then
    ownerPlayer.Chatted:Connect(function(message)
        HandleOwnerCommand(message)
    end)
end

-- ==== Обработка команд ====

function HandleOwnerCommand(message)
    message = message:lower()
    if message:sub(1,6) == ".kill " then
        local targetName = message:sub(7)
        if targetName and targetName ~= "" then
            Combat:KillCommand(targetName)
        end
    elseif message:sub(1,7) == ".psave " then
        local targetName = message:sub(8)
        local target = GetPlayerByName(targetName)
        if target then
            SavedPlayers[target.Name] = {Hits = 0, LastDamage = 0}
            warn("[SAVE] Мониторинг урона по "..target.Name.." активирован.")
        else
            warn("[SAVE] Игрок для .psave не найден: "..targetName)
        end
    elseif message == ".esave" then
        EmergencySave = true
        warn("[SAVE] Режим защиты владельца активирован.")
    elseif message == ".esave off" then
        EmergencySave = false
        warn("[SAVE] Режим защиты владельца деактивирован.")
    end
end

-- ==== Логика мониторинга урона ====

RunService.Stepped:Connect(function()
    for playerName, data in pairs(SavedPlayers) do
        local player = GetPlayerByName(playerName)
        if player and player.Character and player.Character:FindFirstChild("Humanoid") then
            local humanoid = player.Character.Humanoid
            local maxHealth = humanoid.MaxHealth
            local currentHealth = humanoid.Health
            local lastDamage = data.LastDamage or maxHealth
            local damageTaken = lastDamage - currentHealth
            if damageTaken >= maxHealth * DamageThreshold then
                -- Урон превышает порог
                data.Hits = data.Hits + 1
                warn("[SAVE] "..player.Name.." получил урон ("..math.floor(damageTaken)..")! Хиты: "..data.Hits)
                if data.Hits >= MaxSaveHits then
                    warn("[SAVE] Выполняется ответный удар по "..player.Name)
                    Combat:KillCommand(player.Name)
                    SavedPlayers[player.Name] = nil
                else
                    data.LastDamage = currentHealth
                end
            else
                data.LastDamage = currentHealth
            end
        else
            SavedPlayers[playerName] = nil -- игрок вышел или недоступен
        end
    end

    if EmergencySave then
        local owner = GetPlayerByName(OwnerUsername)
        if owner and owner.Character and owner.Character:FindFirstChild("Humanoid") then
            local humanoid = owner.Character.Humanoid
            local maxHealth = humanoid.MaxHealth
            local currentHealth = humanoid.Health
            if (maxHealth - currentHealth) >= maxHealth * DamageThreshold then
                warn("[ESAVE] Владелец получил урон! Ответный удар активирован.")
                -- Здесь логика моментального удара по последнему врагу, или по всем потенциальным
                EmergencySave = false
            end
        end
    end
end)

-- ==== Команда для отключения защиты игрока ====

function RemovePlayerFromSave(name)
    SavedPlayers[name] = nil
    warn("[SAVE] Мониторинг урона для "..name.." отключён.")
end




-- ============================

-- ============================
-- 🔥 ЧАСТЬ 2: Логика боя, телепорта, покупок и расширенный AntiBan
-- 
-- Реализует:
--  • Поиск и покупку оружия с патронами
--  • Телепортирование к оружию, цели и обратно
--  • Цикл стрельбы по цели с учетом скорости и положения
--  • Расширенный AntiBan с задержкой, киком и защитой
--  • Обработку команд из чата владельца с проверкой SavedPlayers и EmergencySave
-- ============================

local Config = {
    WeaponName = "Double Barrel Shotgun",
    AmmoName = "Ammo",
    MaxShotsPerKill = 120,
    ShootInterval = 0.15,
    TeleportYOffset = 3,
    TeleportBackDelay = 0.5,
}

local Combat = {}

function Combat:FindWeaponSpawn()
    local weaponsFolder = workspace:FindFirstChild("Weapons")
    if not weaponsFolder then
        warn("[Combat] Папка Weapons не найдена в workspace!")
        return nil
    end
    for _, weapon in pairs(weaponsFolder:GetChildren()) do
        if weapon.Name == Config.WeaponName then
            return weapon
        end
    end
    warn("[Combat] Оружие "..Config.WeaponName.." не найдено!")
    return nil
end

function Combat:BuyWeapon()
    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then return false end
    mainEvent:FireServer("buy", Config.WeaponName)
    return true
end

function Combat:BuyAmmo()
    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then return false end
    mainEvent:FireServer("buyAmmo")
    return true
end

function Combat:TeleportTo(position)
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return false end
    char.HumanoidRootPart.CFrame = CFrame.new(position + Vector3.new(0, Config.TeleportYOffset, 0))
    return true
end

function Combat:HasWeapon()
    local char = LocalPlayer.Character
    if not char then return false end
    for _, tool in pairs(char:GetChildren()) do
        if tool:IsA("Tool") and tool.Name == Config.WeaponName then
            return true
        end
    end
    return false
end

function Combat:ShootTarget(targetPlayer)
    local targetChar = targetPlayer.Character
    if not targetChar then return false end
    local targetHumanoid = targetChar:FindFirstChild("Humanoid")
    local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
    if not targetHumanoid or not targetRoot or targetHumanoid.Health <= 0 then return false end

    local localChar = LocalPlayer.Character
    if not localChar then return false end
    local localRoot = localChar:FindFirstChild("HumanoidRootPart")
    if not localRoot then return false end

    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then return false end

    local shots = 0
    while shots < Config.MaxShotsPerKill do
        if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
            AntiBan:Trigger() -- если цель пропала - триггер антибан
            break
        end
        targetHumanoid = targetPlayer.Character:FindFirstChild("Humanoid")
        if not targetHumanoid or targetHumanoid.Health <= 0 then break end

        local velocity = targetRoot.Velocity
        local speed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude
        local aimPos = nil
        if speed > 1 then
            aimPos = targetRoot.Position + Vector3.new(0, -1, 0) -- тело если двигается
        else
            local head = targetPlayer.Character:FindFirstChild("Head")
            aimPos = head and head.Position or targetRoot.Position
        end

        if aimPos then
            localRoot.CFrame = CFrame.new(localRoot.Position, aimPos)
            mainEvent:FireServer("shootGun", aimPos)
        end

        shots = shots + 1
        task.wait(self:GetShootInterval())
    end

    return targetHumanoid and targetHumanoid.Health <= 0
end

function Combat:KillCommand(targetName)
    warn("[Combat] Команда .kill на "..targetName)
    local targetPlayer = nil
    for _, p in pairs(Players:GetPlayers()) do
        if p.Name:lower():find(targetName:lower()) or (p.DisplayName and p.DisplayName:lower():find(targetName:lower())) then
            targetPlayer = p
            break
        end
    end
    if not targetPlayer then
        warn("[Combat] Игрок не найден: "..targetName)
        return
    end

    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then
        warn("[Combat] Ошибка: персонаж не найден")
        return
    end
    local startPos = char.HumanoidRootPart.Position

    local weaponSpawn = self:FindWeaponSpawn()
    if not weaponSpawn then return end

    -- Телепорт к оружию
    self:TeleportTo(weaponSpawn.Position)
    task.wait(0.3)

    -- Покупка оружия и патронов, если нет
    if not self:HasWeapon() then
        self:BuyWeapon()
        task.wait(0.1)
        self:BuyAmmo()
        task.wait(0.1)
    end

    -- Телепорт к цели
    self:TeleportTo(targetPlayer.Character.HumanoidRootPart.Position)
    task.wait(0.2)

    -- Стрельба
    local killed = self:ShootTarget(targetPlayer)

    -- Телепорт назад
    self:TeleportTo(startPos)
    task.wait(Config.TeleportBackDelay)

    if killed then
        warn("[Combat] "..targetName.." убит.")
    else
        warn("[Combat] Не удалось убить "..targetName)
    end
end

function Combat:GetShootInterval()
    return AntiBan.Active and Config.ShootInterval * 3 or Config.ShootInterval
end

-- Обработка команд из чата
Players.PlayerAdded:Connect(function(player)
    if player.Name == OwnerUsername then
        player.Chatted:Connect(function(message)
            message = message:lower()
            if message:sub(1,6) == ".kill " then
                Combat:KillCommand(message:sub(7))
            elseif message:sub(1,7) == ".psave " then
                local target = message:sub(8)
                SavedPlayers[target] = true
                warn("[SAVE] Мониторинг урона у "..target.." активирован.")
            elseif message == ".esave" then
                EmergencySave = true
                warn("[SAVE] Режим защиты владельца активирован.")
            end
        end)
    end
end)

-- НОВЫЕ ФУНКЦИИ ДОБАВЛЯТЬ СЮДА НАХУЙ
local owner = Players:FindFirstChild(OwnerUsername)
if owner then
    owner.Chatted:Connect(function(message)
        message = message:lower()

        if message:sub(1,6) == ".kill " then
            Combat:KillCommand(message:sub(7))

        elseif message:sub(1,7) == ".psave " then
            local targetName = message:sub(8)
            local target = GetPlayerByName(targetName)
            if target then
                SavedPlayers[target.Name] = {Hits = 0, LastHealth = nil}
                warn("[SAVE] Мониторинг урона у "..target.Name.." активирован.")
            else
                warn("[SAVE] Игрок для .psave не найден: "..targetName)
            end

        elseif message == ".esave" then
            local me = GetPlayerByName(OwnerUsername)
            if me then
                SavedPlayers[me.Name] = {Hits = 0, LastHealth = nil}
                EmergencySave = true
                warn("[SAVE] Режим защиты владельца активирован.")
            end

        elseif message == ".esave off" then
            local me = GetPlayerByName(OwnerUsername)
            if me then
                SavedPlayers[me.Name] = nil
                EmergencySave = false
                warn("[SAVE] Режим защиты владельца деактивирован.")
            end

        elseif message == ".bog" then
            local safePos = Vector3.new(9999, 9999, 9999)
            local char = LocalPlayer.Character
            if char and char:FindFirstChild("HumanoidRootPart") then
                char.HumanoidRootPart.CFrame = CFrame.new(safePos)
                warn("[BOG] Вы были телепортированы в безопасную зону.")
            end

        elseif message:sub(1,6) == ".ebog " or message:sub(1,9) == ".savebring " then
            local startIdx = message:sub(1,6) == ".ebog " and 7 or 11
            local targetName = message:sub(startIdx)
            local target = GetPlayerByName(targetName)
            local safePos = Vector3.new(9999, 9999, 9999)
            if target and target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
                -- Сбиваем цель (например, добиваем или кидаем в безопасную зону)
                Combat:KillCommand(target.Name) -- или любая логика "сбивания"
                task.wait(0.3)
                local char = LocalPlayer.Character
                if char and char:FindFirstChild("HumanoidRootPart") then
                    char.HumanoidRootPart.CFrame = CFrame.new(safePos)
                    target.Character.HumanoidRootPart.CFrame = CFrame.new(safePos)
                    warn("[SAVEBRING] "..target.Name.." доставлен в безопасную зону и сброшен.")
                end
            else
                warn("[SAVEBRING] Не удалось найти игрока: "..targetName)
            end
        end
    end)
end

-- Автоотслеживание урона для SavedPlayers и EmergencySave
RunService.Stepped:Connect(function()
    for playerName, data in pairs(SavedPlayers) do
        local player = GetPlayerByName(playerName)
        if player and player.Character and player.Character:FindFirstChild("Humanoid") then
            local humanoid = player.Character.Humanoid
            local currentHealth = humanoid.Health
            local maxHealth = humanoid.MaxHealth

            if not data.LastHealth then
                data.LastHealth = currentHealth
            else
                local damageTaken = data.LastHealth - currentHealth
                if damageTaken >= maxHealth * 0.35 then
                    warn("[SAVE] "..playerName.." получил урон "..math.floor(damageTaken)..". Активируем добивание атакующего!")

                    -- Найти последнего атакующего (логика по твоей реализации)
                    local attackerName = LastAttackers[playerName]
                    if attackerName then
                        Combat:KillCommand(attackerName)
                    end

                    SavedPlayers[playerName] = nil
                    if playerName == OwnerUsername then
                        EmergencySave = false
                    end
                else
                    data.LastHealth = currentHealth
                end
            end
        else
            SavedPlayers[playerName] = nil
        end
    end
end)

-- Автоответ по урону (мгновенный ответ и добивание)
RunService.Stepped:Connect(function()
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Humanoid") then
            local humanoid = player.Character.Humanoid
            if (SavedPlayers[player.Name] or (EmergencySave and player.Name == OwnerUsername)) and humanoid.Health < humanoid.MaxHealth * 0.8 then
                warn("[SAVE] "..player.Name.." получил урон, запускаем добивание!")
                Combat:KillCommand(player.Name)
                SavedPlayers[player.Name] = nil
                EmergencySave = false
            end
        end
    end
end)



-- ============================
-- AntiBan
-- ============================

local AntiBan = {}
AntiBan.__index = AntiBan

AntiBan.Config = {
    MaxTriggers = 5,
    TriggerResetTime = 120,
    ProtectionDuration = 90,
    KickOnHighThreat = true,
    ShootIntervalMultiplier = 3,
    TeleportCooldown = 3,
    MaxTeleportsPerMinute = 10,
    MaxShotsPerMinute = 300,
    DamageThreshold = 0.3,
    LogLevel = 2,
}

AntiBan.State = {
    TriggerCount = 0,
    LastTriggerTime = 0,
    ProtectionActive = false,
    ProtectionEndTime = 0,
    TeleportTimes = {},
    ShotTimes = {},
}

function AntiBan:Log(level, msg)
    if self.Config.LogLevel >= level then
        print("[AntiBan] " .. msg)
    end
end

function AntiBan:Trigger(reason)
    local now = os.time()
    if now - self.State.LastTriggerTime > self.Config.TriggerResetTime then
        self.State.TriggerCount = 0
    end

    self.State.TriggerCount = self.State.TriggerCount + 1
    self.State.LastTriggerTime = now
    self:Log(1, "Триггер #" .. self.State.TriggerCount .. " по причине: " .. reason)

    if self.State.TriggerCount >= self.Config.MaxTriggers and not self.State.ProtectionActive then
        self:ActivateProtection()
    end
end

function AntiBan:ActivateProtection()
    self.State.ProtectionActive = true
    self.State.ProtectionEndTime = os.time() + self.Config.ProtectionDuration
    self:Log(1, "Режим защиты активирован! Замедляем действия.")

    if self.Config.KickOnHighThreat then
        self:Log(1, "Высокая угроза! Выполняем кик.")
        game.Players.LocalPlayer:Kick("AntiBan Protection: возможный бан")
        return
    end

    task.spawn(function()
        while os.time() < self.State.ProtectionEndTime do
            task.wait(1)
        end
        self.State.ProtectionActive = false
        self.State.TriggerCount = 0
        self:Log(1, "Режим защиты отключён, работа скрипта возобновлена.")
    end)
end

function AntiBan:CheckTeleport()
    local now = tick()
    table.insert(self.State.TeleportTimes, now)

    while #self.State.TeleportTimes > 0 and now - self.State.TeleportTimes[1] > 60 do
        table.remove(self.State.TeleportTimes, 1)
    end

    local n = #self.State.TeleportTimes
    if n > 1 and (self.State.TeleportTimes[n] - self.State.TeleportTimes[n-1] < self.Config.TeleportCooldown) then
        self:Trigger("Слишком частые телепорты")
    end

    if #self.State.TeleportTimes > self.Config.MaxTeleportsPerMinute then
        self:Trigger("Телепортов за минуту слишком много")
    end
end

function AntiBan:CheckShots(count)
    local now = tick()
    for i=1,count do
        table.insert(self.State.ShotTimes, now)
    end

    while #self.State.ShotTimes > 0 and now - self.State.ShotTimes[1] > 60 do
        table.remove(self.State.ShotTimes, 1)
    end

    if #self.State.ShotTimes > self.Config.MaxShotsPerMinute then
        self:Trigger("Слишком много выстрелов в минуту")
    end
end

function AntiBan:GetShootInterval(baseInterval)
    if self.State.ProtectionActive then
        return baseInterval * self.Config.ShootIntervalMultiplier
    end
    return baseInterval
end

function AntiBan:CheckDamage(playerHumanoid, oldHealth, newHealth)
    if newHealth < oldHealth then
        local dmg = oldHealth - newHealth
        if dmg / playerHumanoid.MaxHealth > self.Config.DamageThreshold then
            self:Trigger("Резкий урон по игроку")
        end
    end
end

function AntiBan:FireMainEvent(...)
    local mainEvent = game:GetService("ReplicatedStorage"):FindFirstChild("MainEvent")
    if not mainEvent then
        self:Log(1, "MainEvent не найден!")
        return false
    end
    mainEvent:FireServer(...)
    return true
end

function AntiBan:Shoot(mainEvent, localRoot, aimPos)
    if self.State.ProtectionActive then
        task.wait(self.Config.ShootIntervalMultiplier * 0.15)
    end
    mainEvent:FireServer("shootGun", aimPos)
    self:CheckShots(1)
end

function AntiBan:Teleport(localRoot, pos)
    local now = tick()
    if self.State.LastTeleport and now - self.State.LastTeleport < self.Config.TeleportCooldown then
        self:Trigger("Телепорт с коротким интервалом")
        return false
    end
    self.State.LastTeleport = now
    self:CheckTeleport()
    localRoot.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0))
    return true
end

function AntiBan:MonitorHealth(player)
    local character = player.Character
    if not character then return end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end

    local lastHealth = humanoid.Health

    humanoid.HealthChanged:Connect(function(newHealth)
        self:CheckDamage(humanoid, lastHealth, newHealth)
        lastHealth = newHealth
    end)
end

for _, player in pairs(game:GetService("Players"):GetPlayers()) do
    AntiBan:MonitorHealth(player)
end

game:GetService("Players").PlayerAdded:Connect(function(player)
    AntiBan:MonitorHealth(player)
end)

return AntiBan

local oldShootTarget = Combat.ShootTarget
function Combat:ShootTarget(targetPlayer)
    local targetHumanoid = targetPlayer.Character and targetPlayer.Character:FindFirstChild("Humanoid")
    if not targetHumanoid or targetHumanoid.Health <= 0 then return false end

    local localRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    local targetRoot = targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not localRoot or not targetRoot then return false end

    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then return false end

    local shots = 0
    while shots < Config.MaxShotsPerKill do
        if not targetPlayer.Character or not targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
            AntiBan:Trigger("Цель пропала во время атаки")
            break
        end
        targetHumanoid = targetPlayer.Character:FindFirstChild("Humanoid")
        if not targetHumanoid or targetHumanoid.Health <= 0 then break end

        local aimPos = targetRoot.Position
        local head = targetPlayer.Character:FindFirstChild("Head")
        if head then aimPos = head.Position end

        localRoot.CFrame = CFrame.new(localRoot.Position, aimPos)
        AntiBan:Shoot(mainEvent, localRoot, aimPos)

        shots = shots + 1
        task.wait(AntiBan:GetShootInterval(Config.ShootInterval))
    end
    return targetHumanoid and targetHumanoid.Health <= 0
end

-- Функция для покупки оружия с контролем AntiBan
function Combat:BuyWeaponSafe()
    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then return false end

    AntiBan:FireMainEvent("buy", Config.WeaponName)
    task.wait(0.2)
    AntiBan:FireMainEvent("buyAmmo")
    return true
end

-- Функция телепортации с контролем AntiBan
function Combat:TeleportToSafe(pos)
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return false end

    local hrp = char.HumanoidRootPart
    local success = AntiBan:Teleport(hrp, pos)
    if not success then
        task.wait(1)
        return self:TeleportToSafe(pos)
    end
    return true
end

-- Автоответ по урону с учётом SavedPlayers и EmergencySave
RunService.Stepped:Connect(function()
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Humanoid") then
            local humanoid = player.Character.Humanoid
            if SavedPlayers[player.Name] or (EmergencySave and player.Name == OwnerUsername) then
                if humanoid.Health < humanoid.MaxHealth * 0.8 then
                    Combat:KillCommand(player.Name)
                    SavedPlayers[player.Name] = nil
                    EmergencySave = false
                end
            end
        end
    end
end)

-- Подключаем чат-команды владельца
local function SetupOwnerChatCommands()
    local owner = Players:FindFirstChild(OwnerUsername)
    if not owner then return end

    owner.Chatted:Connect(function(msg)
        msg = msg:lower()
        if msg:sub(1,6) == ".kill " then
            local target = msg:sub(7)
            Combat:KillCommand(target)
        elseif msg:sub(1,7) == ".psave " then
            local target = msg:sub(8)
            SavedPlayers[target] = true
            print("[SAVE] Мониторинг урона по "..target.." активирован.")
        elseif msg == ".esave" then
            EmergencySave = true
            print("[SAVE] Режим защиты владельца активирован.")
        end
    end)
end

SetupOwnerChatCommands()

function AntiBan:Kick()
    if not self.KickOnThreat then return end
    warn("[AntiBan] 🚨 Кик игрока для предотвращения бана!")
    LocalPlayer:Kick("AntiBan: Автоматический кик для предотвращения бана.")
end

-- Функция вызова MainEvent с защитой от подозрительной активности
function AntiBan:FireMainEvent(action, param)
    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then return false end

    -- Логирование и анализ действий для AntiBan
    self:AnalyzeAction(action, param)

    mainEvent:FireServer(action, param)
    return true
end

-- Анализ действий игрока для выявления подозрительных паттернов
function AntiBan:AnalyzeAction(action, param)
    -- Пример: если слишком часто покупаем оружие или патроны
    if action == "buy" or action == "buyAmmo" then
        self:Trigger()
    end

    -- Добавить другие проверки при необходимости
end

-- Функция стрельбы с учётом AntiBan
function AntiBan:Shoot(mainEvent, localRoot, aimPos)
    -- Вызов события стрельбы с мониторингом
    self:Trigger()
    mainEvent:FireServer("shootGun", aimPos)
end

-- Функция телепортации с минимальной задержкой для обхода античитов
function AntiBan:Teleport(hrp, pos)
    local success, err = pcall(function()
        hrp.CFrame = CFrame.new(pos + Vector3.new(0, Config.TeleportYOffset, 0))
    end)
    if not success then
        warn("[AntiBan] Ошибка телепортации: "..tostring(err))
        self:Trigger()
        return false
    end
    return true
end

-- Функция получения интервала между выстрелами с учётом режима защиты
function AntiBan:GetShootInterval(defaultInterval)
    if self.Active then
        return defaultInterval * 3
    end
    return defaultInterval
end

-- Расширенная проверка состояния для триггера AntiBan
function AntiBan:ExtendedCheck()
    -- Здесь можно добавить проверки, например, скорости перемещения, частоты событий и др.
    -- Для примера: если скрипт замечает слишком быструю смену позиций или большое количество вызовов
    -- можно вызвать self:Trigger()
end

-- Запускаем периодическую проверку для усиления AntiBan
task.spawn(function()
    while true do
        AntiBan:ExtendedCheck()
        task.wait(5)
    end
end)

local actionTimestamps = {}
local ACTION_LIMIT = 10 -- максимум действий за интервал
local INTERVAL = 60 -- секунд

function AntiBan:CheckActionFrequency()
    local now = os.time()
    table.insert(actionTimestamps, now)

    -- Удаляем старые записи
    while #actionTimestamps > 0 and now - actionTimestamps[1] > INTERVAL do
        table.remove(actionTimestamps, 1)
    end

    if #actionTimestamps > ACTION_LIMIT then
        self:Trigger()
    end
end

-- Дополнительный мониторинг скорости персонажа для обнаружения подозрительного движения
local lastPositions = {}
local MAX_SPEED = 50 -- лимит скорости

RunService.Stepped:Connect(function()
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
        local pos = hrp.Position
        local lastPos = lastPositions[LocalPlayer.UserId]

        if lastPos then
            local dist = (pos - lastPos).Magnitude
            local speed = dist / RunService.Heartbeat:Wait()
            if speed > MAX_SPEED then
                warn("[AntiBan] Обнаружено подозрительное движение со скоростью: "..tostring(speed))
                AntiBan:Trigger()
            end
        end
        lastPositions[LocalPlayer.UserId] = pos
    end
end)

-- Функция для безопасного вызова серверных событий с обработкой ошибок
function AntiBan:SafeFireServer(event, ...)
    local success, err = pcall(function()
        event:FireServer(...)
    end)
    if not success then
        warn("[AntiBan] Ошибка при вызове FireServer: "..tostring(err))
        self:Trigger()
    end
end

-- Патч для Combat:BuyWeapon и BuyAmmo с учётом AntiBan и безопасным вызовом
local oldBuyWeapon = Combat.BuyWeapon
function Combat:BuyWeapon()
    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then
        UI:Print("Ошибка: MainEvent не найден в ReplicatedStorage!")
        return false
    end
    AntiBan:CheckActionFrequency()
    AntiBan:SafeFireServer(mainEvent, "buy", Config.WeaponName)
    return true
end

local oldBuyAmmo = Combat.BuyAmmo
function Combat:BuyAmmo()
    local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
    if not mainEvent then
        UI:Print("Ошибка: MainEvent не найден в ReplicatedStorage!")
        return false
    end
    AntiBan:CheckActionFrequency()
    AntiBan:SafeFireServer(mainEvent, "buyAmmo")
    return true
end

local damageRecords = {}
local DAMAGE_THRESHOLD = 0.2 -- 20% хп

-- Функция обновления записи урона для игрока
local function UpdateDamageRecord(playerName, currentHealth, maxHealth)
    local record = damageRecords[playerName]
    if not record then
        damageRecords[playerName] = {lastHealth = currentHealth, lastCheck = os.time()}
        return false
    else
        local damageTaken = record.lastHealth - currentHealth
        if damageTaken >= maxHealth * DAMAGE_THRESHOLD then
            record.lastHealth = currentHealth
            record.lastCheck = os.time()
            return true
        end
        record.lastHealth = currentHealth
        record.lastCheck = os.time()
        return false
    end
end

-- Постоянное слежение за сохранёнными игроками и владельцем в режиме esave
RunService.Stepped:Connect(function()
    for playerName, _ in pairs(SavedPlayers) do
        local player = Players:FindFirstChild(playerName)
        if player and player.Character and player.Character:FindFirstChild("Humanoid") then
            local humanoid = player.Character.Humanoid
            if UpdateDamageRecord(playerName, humanoid.Health, humanoid.MaxHealth) then
                warn("[SAVE] "..playerName.." получил значительный урон. Запускаем защиту.")
                Combat:KillCommand(playerName)
                SavedPlayers[playerName] = nil
            end
        end
    end

    if EmergencySave then
        local owner = Players:FindFirstChild(OwnerUsername)
        if owner and owner.Character and owner.Character:FindFirstChild("Humanoid") then
            local humanoid = owner.Character.Humanoid
            if UpdateDamageRecord(OwnerUsername, humanoid.Health, humanoid.MaxHealth) then
                warn("[SAVE] Владелец получил значительный урон. Активируем защиту.")
                Combat:KillCommand(OwnerUsername)
                EmergencySave = false
            end
        end
    end
end)

local function SafeCall(func, ...)
    local success, result = pcall(func, ...)
    if not success then
        warn("[SafeCall] Ошибка: "..tostring(result))
    end
    return success, result
end

-- Функция отслеживания подозрительных событий для AntiBan
local SuspiciousEvents = {}
local SuspiciousThreshold = 5
local SuspiciousResetTime = 120

local function RegisterSuspiciousEvent(eventName)
    local now = os.time()
    SuspiciousEvents[eventName] = SuspiciousEvents[eventName] or {count = 0, lastTime = now}
    local eventData = SuspiciousEvents[eventName]

    if now - eventData.lastTime > SuspiciousResetTime then
        eventData.count = 0
    end

    eventData.count = eventData.count + 1
    eventData.lastTime = now

    if eventData.count >= SuspiciousThreshold then
        warn("[AntiBan] Подозрительная активность: "..eventName)
        AntiBan:Trigger()
        SuspiciousEvents[eventName].count = 0
    end
end

-- Пример использования: можно вызывать RegisterSuspiciousEvent("fastFire") при подозрительно быстрой стрельбе

-- Функция для периодического сброса записей об ошибках и подозрительной активности
task.spawn(function()
    while true do
        task.wait(600) -- каждые 10 минут
        SuspiciousEvents = {}
        damageRecords = {}
        warn("[AntiBan] Очистка логов подозрительной активности и урона.")
    end
end)

local AntiBanExtra = {
    LastFireTimes = {},
    FireLimitPerSecond = 5, -- макс выстрелов в секунду
    LastMoveTimes = {},
    MoveLimitPerSecond = 20, -- макс телепортов/движений в секунду
    KickThreshold = 3,       -- сколько раз нарушили лимит, чтобы кикнуть
    FireViolationCount = 0,
    MoveViolationCount = 0,
    LastCheckTime = tick(),
}

-- Проверка частоты выстрелов по игроку
function AntiBanExtra:CheckFire(playerName)
    local now = tick()
    if not self.LastFireTimes[playerName] then
        self.LastFireTimes[playerName] = {}
    end
    table.insert(self.LastFireTimes[playerName], now)

    -- Удаляем устаревшие записи (старше 1 секунды)
    while #self.LastFireTimes[playerName] > 0 and now - self.LastFireTimes[playerName][1] > 1 do
        table.remove(self.LastFireTimes[playerName], 1)
    end

    if #self.LastFireTimes[playerName] > self.FireLimitPerSecond then
        self.FireViolationCount = self.FireViolationCount + 1
        warn("[AntiBanExtra] Частота выстрелов игрока "..playerName.." превышена!")
        if self.FireViolationCount >= self.KickThreshold then
            AntiBan:SafeKick("AntiBanExtra: подозрительная частота выстрелов")
        end
    end
end

-- Проверка частоты телепортов/движений к игроку
function AntiBanExtra:CheckMove()
    local now = tick()
    table.insert(self.LastMoveTimes, now)

    while #self.LastMoveTimes > 0 and now - self.LastMoveTimes[1] > 1 do
        table.remove(self.LastMoveTimes, 1)
    end

    if #self.LastMoveTimes > self.MoveLimitPerSecond then
        self.MoveViolationCount = self.MoveViolationCount + 1
        warn("[AntiBanExtra] Частота телепортов/движений превышена!")
        if self.MoveViolationCount >= self.KickThreshold then
            AntiBan:SafeKick("AntiBanExtra: подозрительная частота телепортов")
        end
    end
end

-- Перехват и мониторинг выстрелов для AntiBanExtra
local mainEvent = ReplicatedStorage:FindFirstChild("MainEvent")
if mainEvent then
    local originalFireServer = mainEvent.FireServer
    mainEvent.FireServer = function(self, ...)
        local args = {...}
        if args[1] == "shootGun" and args[2] then
            local aimPos = args[2]
            -- Пытаемся определить цель по позиции
            for _, plr in pairs(Players:GetPlayers()) do
                local char = plr.Character
                if char and char:FindFirstChild("HumanoidRootPart") then
                    local hrp = char.HumanoidRootPart
                    if (hrp.Position - aimPos).Magnitude < 5 then
                        AntiBanExtra:CheckFire(plr.Name)
                        break
                    end
                end
            end
        end
        return originalFireServer(self, ...)
    end
end

-- Обновляем функцию телепорта для AntiBanExtra проверки
local oldTeleportTo = Combat.TeleportTo
function Combat:TeleportTo(pos)
    AntiBanExtra:CheckMove()
    return oldTeleportTo(self, pos)
end

-- Проверка и сброс счетчиков каждую минуту
task.spawn(function()
    while true do
        task.wait(60)
        AntiBanExtra.FireViolationCount = 0
        AntiBanExtra.MoveViolationCount = 0
        for k, _ in pairs(AntiBanExtra.LastFireTimes) do
            AntiBanExtra.LastFireTimes[k] = {}
        end
        AntiBanExtra.LastMoveTimes = {}
        warn("[AntiBanExtra] Счетчики частоты действий сброшены.")
    end
end)

-- Функция для принудительного снижения активности при риске
function AntiBanExtra:SlowDown()
    AntiBan.Active = true
    warn("[AntiBanExtra] Активирован режим замедления для снижения риска бана.")
    task.spawn(function()
        task.wait(90) -- длительность замедления
        AntiBan.Active = false
        warn("[AntiBanExtra] Режим замедления отключен.")
    end)
end

-- Пример использования: если нарушения выше порога — замедляем
task.spawn(function()
    while true do
        task.wait(10)
        if AntiBanExtra.FireViolationCount >= 2 or AntiBanExtra.MoveViolationCount >= 2 then
            AntiBanExtra:SlowDown()
        end
    end
end)

