local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local RaycastParams = RaycastParams.new()

-- INTERFACE
local gui = Instance.new("ScreenGui")
gui.Name = "AimbotESPGui"
gui.IgnoreGuiInset = true
gui.ResetOnSpawn = false

pcall(function() gui.Parent = game.CoreGui end)
if not gui.Parent then
    gui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 120, 0, 100)
frame.Position = UDim2.new(0, 15, 1, -120) -- canto inferior esquerdo
frame.BackgroundTransparency = 0
frame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
frame.BorderSizePixel = 0
frame.Active = true
frame.Draggable = true
frame.Parent = gui

local aimbotBtn = Instance.new("TextButton")
aimbotBtn.Size = UDim2.new(0, 100, 0, 24)
aimbotBtn.Position = UDim2.new(0, 10, 0, 8)
aimbotBtn.Text = "Aimbot: OFF"
aimbotBtn.BackgroundColor3 = Color3.fromRGB(80, 150, 80)
aimbotBtn.TextColor3 = Color3.fromRGB(255,255,255)
aimbotBtn.Font = Enum.Font.SourceSansBold
aimbotBtn.TextSize = 16
aimbotBtn.Parent = frame

local espBtn = Instance.new("TextButton")
espBtn.Size = UDim2.new(0, 100, 0, 24)
espBtn.Position = UDim2.new(0, 10, 0, 38)
espBtn.Text = "ESP: OFF"
espBtn.BackgroundColor3 = Color3.fromRGB(150, 80, 80)
espBtn.TextColor3 = Color3.fromRGB(255,255,255)
espBtn.Font = Enum.Font.SourceSansBold
espBtn.TextSize = 16
espBtn.Parent = frame

-- Botão de disparo manual para MOBILE
local fireBtn = Instance.new("TextButton")
fireBtn.Size = UDim2.new(0, 100, 0, 24)
fireBtn.Position = UDim2.new(0, 10, 0, 68)
fireBtn.Text = "ATIRAR"
fireBtn.BackgroundColor3 = Color3.fromRGB(80, 80, 150)
fireBtn.TextColor3 = Color3.fromRGB(255,255,255)
fireBtn.Font = Enum.Font.SourceSansBold
fireBtn.TextSize = 16
fireBtn.Visible = UIS.TouchEnabled
fireBtn.Parent = frame

local aimbotOn = false
local espOn = false

aimbotBtn.MouseButton1Click:Connect(function()
    aimbotOn = not aimbotOn
    aimbotBtn.Text = aimbotOn and "Aimbot: ON" or "Aimbot: OFF"
    aimbotBtn.BackgroundColor3 = aimbotOn and Color3.fromRGB(40, 180, 40) or Color3.fromRGB(80,150,80)
end)
espBtn.MouseButton1Click:Connect(function()
    espOn = not espOn
    espBtn.Text = espOn and "ESP: ON" or "ESP: OFF"
    espBtn.BackgroundColor3 = espOn and Color3.fromRGB(180, 40, 40) or Color3.fromRGB(150,80,80)
end)

-- Para mobile, garantir que o Frame seja realmente movível:
if UIS.TouchEnabled then
    local dragging, dragInput, dragStart, startPos
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    frame.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- Função para pegar inimigo vivo mais próximo
local function GetClosestEnemy()
    local minDist = math.huge
    local closest = nil
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Team ~= LocalPlayer.Team and player.Character then
            local hum = player.Character:FindFirstChildOfClass("Humanoid")
            local head = player.Character:FindFirstChild("Head")
            if hum and hum.Health > 0 and head then
                local dist = (LocalPlayer.Character.Head.Position - head.Position).magnitude
                if dist < minDist then
                    minDist = dist
                    closest = player
                end
            end
        end
    end
    return closest
end

-- ESP: mostra TODOS inimigos
local EspBoxes = {}
local function UpdateESP()
    for _, box in ipairs(EspBoxes) do
        box:Destroy()
    end
    EspBoxes = {}

    if not espOn then return end

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Team ~= LocalPlayer.Team and player.Character and player.Character:FindFirstChild("Head") then
            local head = player.Character.Head
            local pos, onScreen = Camera:WorldToViewportPoint(head.Position)
            if onScreen then
                local box = Instance.new("Frame", gui)
                box.Size = UDim2.new(0, 28, 0, 28)
                box.Position = UDim2.new(0, pos.X-14, 0, pos.Y-14)
                box.BackgroundTransparency = 0.5
                box.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
                box.BorderSizePixel = 0
                table.insert(EspBoxes, box)
            end
        end
    end
end

-- Checa se inimigo está visível
local function IsEnemyVisible(head)
    RaycastParams.FilterDescendantsInstances = {LocalPlayer.Character}
    RaycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    RaycastParams.IgnoreWater = true
    local result = workspace:Raycast(Camera.CFrame.Position, (head.Position-Camera.CFrame.Position).unit * (head.Position-Camera.CFrame.Position).Magnitude, RaycastParams)
    if result then
        return result.Instance:IsDescendantOf(head.Parent)
    end
    return false
end

-- Função de disparo
local function ShootAt(head)
    -- PC: simula clique do mouse esquerdo
    if not UIS.TouchEnabled then
        local VirtualInputManager = game:GetService("VirtualInputManager")
        VirtualInputManager:SendMouseButtonEvent(UIS:GetMouseLocation().X, UIS:GetMouseLocation().Y, 0, true, game, 0)
        VirtualInputManager:SendMouseButtonEvent(UIS:GetMouseLocation().X, UIS:GetMouseLocation().Y, 0, false, game, 0)
    else
        -- MOBILE: simula toque no botão de disparo do jogo (você pode precisar ajustar isso para seu sistema de armas)
        -- Alternativamente, você pode acionar a função do botão "ATIRAR" (fireBtn)
        if fireBtn.Visible then
            fireBtn.BackgroundColor3 = Color3.fromRGB(50,50,200)
            fireBtn.Text = "DISPARANDO!"
            wait(0.12)
            fireBtn.BackgroundColor3 = Color3.fromRGB(80,80,150)
            fireBtn.Text = "ATIRAR"
        end
        -- Adapte aqui se seu jogo usa outra função!
    end
end

-- Disparo manual para mobile
fireBtn.MouseButton1Click:Connect(function()
    if aimbotOn then
        local enemy = GetClosestEnemy()
        if enemy and enemy.Character and enemy.Character:FindFirstChild("Head") and IsEnemyVisible(enemy.Character.Head) then
            Camera.CFrame = CFrame.new(Camera.CFrame.Position, enemy.Character.Head.Position)
            ShootAt(enemy.Character.Head)
        end
    end
end)

local lastShot = 0
local shotInterval = 0.3

-- Loop principal
RunService.RenderStepped:Connect(function()
    UpdateESP()
    if aimbotOn then
        local enemy = GetClosestEnemy()
        if enemy and enemy.Character and enemy.Character:FindFirstChild("Head") then
            local head = enemy.Character.Head
            if IsEnemyVisible(head) then
                Camera.CFrame = CFrame.new(Camera.CFrame.Position, head.Position)
                if tick() - lastShot > shotInterval then
                    ShootAt(head)
                    lastShot = tick()
                end
            end
        end
    end
end)
