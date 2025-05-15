-- Rayfield Loader
local Rayfield = nil
local ok, result = pcall(function()
    return loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
end)
if ok and typeof(result) == "table" then
    Rayfield = result
else
    warn("Falha ao carregar Rayfield.")
    return
end

-- Services
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local virtual_input = game:GetService("VirtualInputManager")

-- ESP System
local ESPObjects = {}

local function createESP(part, color, text)
    local box = Drawing.new("Text")
    box.Visible = true
    box.Center = true
    box.Outline = true
    box.Size = 16
    box.Color = color
    box.Text = text
    box.Transparency = 1

    ESPObjects[part] = box

    local conn
    conn = RunService.RenderStepped:Connect(function()
        if not part or not part.Parent then
            box.Visible = false
            box:Remove()
            ESPObjects[part] = nil
            if conn then conn:Disconnect() end
            return
        end
        local cam = workspace.CurrentCamera
        local pos, onScreen = cam:WorldToViewportPoint(part.Position)
        if onScreen then
            box.Position = Vector2.new(pos.X, pos.Y)
            box.Visible = true
        else
            box.Visible = false
        end
    end)
end

local function removeAllESP()
    for part, box in pairs(ESPObjects) do
        if box then
            box.Visible = false
            box:Remove()
        end
    end
    ESPObjects = {}
end

local function startESP()
    removeAllESP()
    
    -- Enemies
    local enemiesFolder = workspace:FindFirstChild("enemies")
    if enemiesFolder then
        for _, enemy in ipairs(enemiesFolder:GetChildren()) do
            local head = enemy:FindFirstChild("Head")
            if head then createESP(head, Color3.fromRGB(255,0,0), enemy.Name) end
        end
        
        enemiesFolder.ChildAdded:Connect(function(enemy)
            enemy.ChildAdded:Connect(function(child)
                if child.Name == "Head" then
                    createESP(child, Color3.fromRGB(255,0,0), enemy.Name)
                end
            end)
        end)
    end

    -- Bosses
    local bossFolder = workspace:FindFirstChild("BossFolder")
    if bossFolder then
        for _, boss in ipairs(bossFolder:GetChildren()) do
            local head = boss:FindFirstChild("Head")
            if head then createESP(head, Color3.fromRGB(180,0,255), boss.Name) end
        end
        
        bossFolder.ChildAdded:Connect(function(boss)
            boss.ChildAdded:Connect(function(child)
                if child.Name == "Head" then
                    createESP(child, Color3.fromRGB(180,0,255), boss.Name)
                end
            end)
        end)
    end

    -- Players
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= Players.LocalPlayer then
            if plr.Character and plr.Character:FindFirstChild("Head") then
                createESP(plr.Character.Head, Color3.fromRGB(0,255,0), plr.Name)
            end
            plr.CharacterAdded:Connect(function(char)
                char:WaitForChild("Head", 5)
                if char:FindFirstChild("Head") then
                    createESP(char.Head, Color3.fromRGB(0,255,0), plr.Name)
                end
            end)
        end
    end
    
    Players.PlayerAdded:Connect(function(plr)
        plr.CharacterAdded:Connect(function(char)
            char:WaitForChild("Head", 5)
            if char:FindFirstChild("Head") then
                createESP(char.Head, Color3.fromRGB(0,255,0), plr.Name)
            end
        end)
    end)
end

-- Aimbot System
local Aimbot = {
    Enabled = false,
    Connection = nil
}

local function findNearestTarget()
    local closest = nil
    local closestDistance = math.huge
    local localPlayer = Players.LocalPlayer
    local character = localPlayer.Character
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")

    if not rootPart then return nil end

    local folders = {workspace:FindFirstChild("enemies"), workspace:FindFirstChild("BossFolder")}
    
    for _, folder in ipairs(folders) do
        if folder then
            for _, enemy in ipairs(folder:GetChildren()) do
                local head = enemy:FindFirstChild("Head")
                if head then
                    local distance = (head.Position - rootPart.Position).Magnitude
                    if distance < closestDistance then
                        closest = head
                        closestDistance = distance
                    end
                end
            end
        end
    end
    
    return closest
end

local function aimbotLoop()
    if Aimbot.Enabled then
        local target = findNearestTarget()
        local camera = workspace.CurrentCamera
        
        if target and camera then
            camera.CFrame = CFrame.new(camera.CFrame.Position, target.Position)
        else
            Aimbot.Enabled = false
            if Aimbot.Connection then
                Aimbot.Connection:Disconnect()
                Aimbot.Connection = nil
            end
            Rayfield:Notify({
                Title = "Aimbot",
                Content = "Nenhum alvo encontrado!",
                Duration = 2,
                Image = 4483362458
            })
        end
    end
end

-- Auto-Shoot System
local AutoShoot = {
    Enabled = false,
    ClickDelay = 0,
    Connection = nil,
    Humanizer = Random.new(),
    IsUserClicking = false
}

UserInputService.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        AutoShoot.IsUserClicking = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        AutoShoot.IsUserClicking = false
    end
end)

local function simulateClick()
    if AutoShoot.Enabled and not AutoShoot.IsUserClicking then
        local mouse = Players.LocalPlayer:GetMouse()
        
        if not mouse:IsMouseOverGui() then
            if AutoShoot.ClickDelay == 0 then
                virtual_input:SendMouseButtonEventAbsolute(mouse.X, mouse.Y, 0, true)
            else
                virtual_input:SendMouseButtonEventAbsolute(mouse.X, mouse.Y, 0, true)
                task.wait(AutoShoot.Humanizer:NextNumber(
                    AutoShoot.ClickDelay/1000 * 0.85, 
                    AutoShoot.ClickDelay/1000 * 1.15
                ))
                virtual_input:SendMouseButtonEventAbsolute(mouse.X, mouse.Y, 0, false)
                task.wait(AutoShoot.Humanizer:NextNumber(0.02, 0.08))
            end
        end
    end
end

-- GUI Creation
local Window = Rayfield:CreateWindow({
    Name = "byby Hub",
    LoadingTitle = "Zombie Attack Script",
    LoadingSubtitle = "by edulucas2013",
    ConfigurationSaving = {
        Enabled = false,
        FolderName = "bybyhubzombieattackConfig",
        FileName = "Config"
    },
    Discord = {
        Enabled = false,
        Invite = "noinvitelink",
        RememberJoins = true
    },
    KeySystem = false,
    KeySettings = {
        Title = "Key System",
        Subtitle = "Enter Key",
        Note = "No key needed",
        FileName = "Key",
        SaveKey = false,
        GrabKeyFromSite = false,
        Key = "ABCDEF"
    }
})

-- ESP Tab
local ESPTab = Window:CreateTab("ESP", 4483362458) -- ID da imagem
ESPTab:CreateSection("Visualização de Alvos")
ESPTab:CreateButton({
    Name = "Ativar ESP Completo",
    Callback = function()
        startESP()
        Rayfield:Notify({
            Title = "Sistema ESP",
            Content = "ESP ativado para todos os alvos!",
            Duration = 3,
            Image = 4483362458,
            Actions = {
                Ignore = {
                    Name = "Ok",
                    Callback = function()
                    end
                },
            },
        })
    end
})

-- Aimbot Tab
local AimbotTab = Window:CreateTab("Aimbot", 7733960981) -- Novo ID de imagem
AimbotTab:CreateSection("Controle de Mira")
AimbotTab:CreateToggle({
    Name = "Ativar Aimbot Automático",
    CurrentValue = false,
    Callback = function(Value)
        Aimbot.Enabled = Value
        if Value then
            Aimbot.Connection = RunService.RenderStepped:Connect(aimbotLoop)
            Rayfield:Notify({
                Title = "Aimbot Ativado",
                Content = "Mirando automaticamente nos alvos!",
                Duration = 2,
                Image = 7733960981
            })
        else
            if Aimbot.Connection then
                Aimbot.Connection:Disconnect()
            end
            Rayfield:Notify({
                Title = "Aimbot Desativado",
                Content = "Controle manual ativado",
                Duration = 1.5,
                Image = 7733960981
            })
        end
    end
})

-- Auto Tab
local AutoTab = Window:CreateTab("Auto", 7734053308) -- ID de imagem diferente
AutoTab:CreateSection("Controle de Tiro Automático")
AutoTab:CreateToggle({
    Name = "Ativar Auto-Shoot",
    CurrentValue = false,
    Callback = function(Value)
        AutoShoot.Enabled = Value
        if Value then
            AutoShoot.Connection = RunService.RenderStepped:Connect(function()
                simulateClick()
                if AutoShoot.ClickDelay > 0 then
                    task.wait(AutoShoot.ClickDelay/1000)
                end
            end)
            Rayfield:Notify({
                Title = "Auto-Shoot",
                Content = "Tiro automático ativado!",
                Duration = 1.5,
                Image = 7734053308
            })
        else
            if AutoShoot.Connection then
                AutoShoot.Connection:Disconnect()
                virtual_input:SendMouseButtonEventAbsolute(0, 0, 0, false)
            end
            Rayfield:Notify({
                Title = "Auto-Shoot",
                Content = "Tiro automático desativado!",
                Duration = 1,
                Image = 7734053308
            })
        end
    end
})

AutoTab:CreateSlider({
    Name = "Intervalo de Disparo (ms)",
    Range = {0, 500},
    Increment = 10,
    Suffix = "milissegundos",
    CurrentValue = 0,
    Callback = function(Value)
        AutoShoot.ClickDelay = Value
    end
})

-- Inicialização final
Rayfield:LoadConfiguration()
