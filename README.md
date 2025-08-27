--// Serviços
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local PathfindingService = game:GetService("PathfindingService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local Player = Players.LocalPlayer
local Character = Player.Character or Player.CharacterAdded:Wait()
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")
local Humanoid = Character:WaitForChild("Humanoid")

--// Variáveis
local savedPosition = nil
local speed = 60
local isCarryingItem = false
local antiHitEnabled = false

--// AntiHit
local function antiHit()
    if antiHitEnabled and Humanoid then
        Humanoid.HealthChanged:Connect(function()
            Humanoid.Health = Humanoid.MaxHealth
        end)
    end
end

--// Atualiza posição salva
local function updateSavedPosition()
    savedPosition = HumanoidRootPart.Position
end

--// Teleport avançado com TweenService
local function teleportWithPath(targetPos)
    if isCarryingItem then
        FeedbackLabel.Text = "Não é possível teletransportar enquanto carrega algo!"
        task.delay(2, function() FeedbackLabel.Text = "" end)
        return
    end

    local path = PathfindingService:CreatePath({
        AgentRadius = 2,
        AgentHeight = 5,
        AgentCanJump = true,
        AgentJumpHeight = 10,
        AgentMaxSlope = 45,
    })

    path:ComputeAsync(HumanoidRootPart.Position, targetPos)
    local waypoints = path:GetWaypoints()

    for _, waypoint in ipairs(waypoints) do
        local destination = waypoint.Position + Vector3.new(0, HumanoidRootPart.Size.Y/2, 0)
        local currentPos = HumanoidRootPart.Position
        local distance = (destination - currentPos).Magnitude
        local time = math.max(distance / speed, 0.05)

        local tweenInfo = TweenInfo.new(time, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
        local tween = TweenService:Create(HumanoidRootPart, tweenInfo, {CFrame = CFrame.new(destination)})
        tween:Play()

        local completed = false
        tween.Completed:Connect(function() completed = true end)
        while not completed do
            task.wait(0.03)
            local newPos = HumanoidRootPart.Position
            local newDistance = (destination - newPos).Magnitude
            if newDistance > 1 then
                tween:Cancel()
                tween = TweenService:Create(HumanoidRootPart, TweenInfo.new(newDistance / speed, Enum.EasingStyle.Linear), {CFrame = CFrame.new(destination)})
                tween:Play()
            end
        end
    end
end

--// GUI
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BasePanel"
ScreenGui.Parent = Player:WaitForChild("PlayerGui")

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, 180, 0, 200)
Frame.Position = UDim2.new(0.5, -90, 0.5, -100)
Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
Frame.BorderSizePixel = 0
Frame.Visible = false
Frame.Parent = ScreenGui

local FeedbackLabel = Instance.new("TextLabel")
FeedbackLabel.Size = UDim2.new(0, 160, 0, 25)
FeedbackLabel.Position = UDim2.new(0, 10, 0, 160)
FeedbackLabel.Text = ""
FeedbackLabel.TextColor3 = Color3.fromRGB(0,0,0)
FeedbackLabel.BackgroundTransparency = 1
FeedbackLabel.TextScaled = true
FeedbackLabel.Parent = Frame

-- Função RGB animado
local function updateRGB()
    local time = tick() * 2
    local r = (math.sin(time) + 1) / 2
    local g = (math.sin(time + 2) + 1) / 2
    local b = (math.sin(time + 4) + 1) / 2
    return Color3.fromRGB(r*255, g*255, b*255)
end

RunService.RenderStepped:Connect(function()
    for _, button in pairs(Frame:GetChildren()) do
        if button:IsA("TextButton") then
            button.BackgroundColor3 = updateRGB()
        end
    end
end)

-- Criação de botões
local function createButton(text, yPos, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 160, 0, 35)
    btn.Position = UDim2.new(0, 10, 0, yPos)
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(0,0,0)
    btn.BackgroundColor3 = Color3.fromRGB(255,0,0)
    btn.Parent = Frame
    btn.MouseButton1Click:Connect(callback)
    return btn
end

createButton("Salvar Posição", 10, function()
    updateSavedPosition()
    FeedbackLabel.Text = "Posição salva!"
    task.delay(2, function() FeedbackLabel.Text = "" end)
end)

createButton("Ir para Posição Salva", 55, function()
    if savedPosition then
        teleportWithPath(savedPosition)
    else
        FeedbackLabel.Text = "Nenhuma posição salva!"
        task.delay(2, function() FeedbackLabel.Text = "" end)
    end
end)

createButton("Toggle AntiHit", 100, function()
    antiHitEnabled = not antiHitEnabled
    if antiHitEnabled then
        antiHit()
        FeedbackLabel.Text = "AntiHit ativado!"
    else
        FeedbackLabel.Text = "AntiHit desativado!"
    end
    task.delay(2, function() FeedbackLabel.Text = "" end)
end)

createButton("Fechar", 145, function()
    Frame.Visible = false
end)

-- Botão fixo para abrir GUI
local OpenButton = Instance.new("TextButton")
OpenButton.Size = UDim2.new(0, 100, 0, 40)
OpenButton.Position = UDim2.new(0, 10, 0, 10)
OpenButton.Text = "-"
OpenButton.TextColor3 = Color3.fromRGB(0,0,0)
OpenButton.BackgroundColor3 = Color3.fromRGB(100,100,100)
OpenButton.Parent = ScreenGui

OpenButton.MouseButton1Click:Connect(function()
    Frame.Visible = not Frame.Visible
end)

-- Arrastável do menu completo
local dragging = false
local dragInput, mousePos, framePos

Frame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        mousePos = input.Position
        framePos = Frame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

Frame.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        local delta = input.Position - mousePos
        Frame.Position = UDim2.new(framePos.X.Scale, framePos.X.Offset + delta.X, framePos.Y.Scale, framePos.Y.Offset + delta.Y)
    end
end)

-- Toggle GUI com tecla P
UserInputService.InputBegan:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.P then
        Frame.Visible = not Frame.Visible
    end
end)
