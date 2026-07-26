local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local CoreGui = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local GuiService = game:GetService("GuiService")

local targetParent
pcall(function()
    if CoreGui and not game:GetService("RunService"):IsStudio() then
        targetParent = CoreGui
    end
end)
if not targetParent then targetParent = LocalPlayer:WaitForChild("PlayerGui") end

-- Limpeza de instâncias antigas
if targetParent:FindFirstChild("BatataHubGui") then targetParent.BatataHubGui:Destroy() end
if Lighting:FindFirstChild("BatataFarmBlur") then Lighting.BatataFarmBlur:Destroy() end

------------------------------------------------------------------------
-- 1. SISTEMA DE CAPTURA DO TORNADO REAL
------------------------------------------------------------------------
local function obterMelhorUpdraftEParte()
    local storms = Workspace:FindFirstChild("Storms")
    if not storms then return nil, 0, 0 end

    local melhorUpdraft = nil
    local maiorPontuacao = -1
    local melhorWind = 0
    local melhorRange = 0

    for _, storm in ipairs(storms:GetChildren()) do
        local updraftsFolder = storm:FindFirstChild("Updrafts")
        if updraftsFolder then
            for _, updraft in ipairs(updraftsFolder:GetChildren()) do
                if string.find(updraft.Name, "Updraft") then
                    local mesocyclone = updraft:FindFirstChild("Mesocyclone")
                    local tornado = mesocyclone and mesocyclone:FindFirstChild("Tornado")
                    local tValues = tornado and tornado:FindFirstChild("TornadoValues")
                    
                    local windVal = 0
                    local rangeVal = 0
                    
                    if tValues then
                        local w = tValues:FindFirstChild("Windspeed")
                        local r = tValues:FindFirstChild("Range")
                        windVal = w and tonumber(w.Value) or 0
                        rangeVal = r and tonumber(r.Value) or 0
                    end
                    
                    local score = windVal + rangeVal
                    if score > maiorPontuacao or (melhorUpdraft == nil) then
                        maiorPontuacao = score
                        melhorUpdraft = updraft
                        melhorWind = windVal
                        melhorRange = rangeVal
                    end
                end
            end
        end
    end

    if melhorUpdraft then
        local mesocyclone = melhorUpdraft:FindFirstChild("Mesocyclone")
        if mesocyclone then
            local tornadoPart = mesocyclone:FindFirstChild("Tornado")
            if tornadoPart and tornadoPart:IsA("BasePart") then
                return tornadoPart, melhorWind, melhorRange
            end
            
            local baseRegions = mesocyclone:FindFirstChild("BaseRegions") or melhorUpdraft:FindFirstChild("BaseRegions")
            if baseRegions then
                local mesoBase = baseRegions:FindFirstChild("MesoBaseRegion")
                if mesoBase and mesoBase:IsA("BasePart") then
                    return mesoBase, melhorWind, melhorRange
                end
            end
        end
    end
    return nil, melhorWind, melhorRange
end

------------------------------------------------------------------------
-- 2. CRIAÇÃO DA INTERFACE (GUI)
------------------------------------------------------------------------
local MainGui = Instance.new("ScreenGui")
MainGui.Name = "BatataHubGui"
MainGui.ResetOnSpawn = false
MainGui.Parent = targetParent

-- Tela Preta Inicial (Intro)
local IntroFrame = Instance.new("Frame")
IntroFrame.Size = UDim2.new(1, 0, 1, 0)
IntroFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
IntroFrame.BorderSizePixel = 0
IntroFrame.ZIndex = 10
IntroFrame.Parent = MainGui

local IntroText = Instance.new("TextLabel")
IntroText.Size = UDim2.new(1, 0, 1, 0)
IntroText.BackgroundTransparency = 1
IntroText.Text = "Relaxe enquanto o script carrega..."
IntroText.TextColor3 = Color3.fromRGB(240, 240, 240)
IntroText.Font = Enum.Font.SourceSansItalic
IntroText.TextSize = 24
IntroText.ZIndex = 11
IntroText.Parent = IntroFrame

-- Painel Principal Móvel
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 260, 0, 220)
MainFrame.Position = UDim2.new(0.05, 0, 0.35, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Visible = false
MainFrame.ZIndex = 5
MainFrame.Parent = MainGui

Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 8)
local MainStroke = Instance.new("UIStroke", MainFrame)
MainStroke.Color = Color3.fromRGB(45, 45, 45)
MainStroke.Thickness = 1.5

local Titulo = Instance.new("TextLabel")
Titulo.Size = UDim2.new(1, 0, 0, 35)
Titulo.BackgroundTransparency = 1
Titulo.Text = "⚡ Batata Hub v.2.0"
Titulo.TextColor3 = Color3.fromRGB(255, 170, 0)
Titulo.Font = Enum.Font.SourceSansBold
Titulo.TextSize = 18
Titulo.ZIndex = 6
Titulo.Parent = MainFrame

-- Monitor de Atributos
local MonitorFrame = Instance.new("Frame")
MonitorFrame.Size = UDim2.new(1, -20, 0, 95)
MonitorFrame.Position = UDim2.new(0, 10, 0, 40)
MonitorFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 12)
MonitorFrame.BorderSizePixel = 0
MonitorFrame.ZIndex = 6
MonitorFrame.Parent = MainFrame
Instance.new("UICorner", MonitorFrame).CornerRadius = UDim.new(0, 6)

local function criarLabelStatus(texto, pos)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -10, 0, 20)
    lbl.Position = UDim2.new(0, 10, 0, pos)
    lbl.BackgroundTransparency = 1
    lbl.Text = texto
    lbl.TextColor3 = Color3.fromRGB(180, 180, 180)
    lbl.Font = Enum.Font.SourceSans
    lbl.TextSize = 14
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.ZIndex = 7
    lbl.Parent = MonitorFrame
    return lbl
end

local VentoLabel = criarLabelStatus("Força do Vento: Procurando...", 10)
local RangeLabel = criarLabelStatus("Raio (Range): Procurando...", 35)
local DistanciaLabel = criarLabelStatus("Distância Atual: -- studs", 60)

-- Botão de Ação
local BotaoFarm = Instance.new("TextButton")
BotaoFarm.Size = UDim2.new(1, -20, 0, 40)
BotaoFarm.Position = UDim2.new(0, 10, 0, 150)
BotaoFarm.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
BotaoFarm.Text = "Ativar Auto Farm"
BotaoFarm.TextColor3 = Color3.fromRGB(255, 85, 85)
BotaoFarm.Font = Enum.Font.SourceSansBold
BotaoFarm.TextSize = 16
BotaoFarm.ZIndex = 7
BotaoFarm.Parent = MainFrame
Instance.new("UICorner", BotaoFarm).CornerRadius = UDim.new(0, 6)

-- Letreiro de Farm Ativo (Modo Imersivo)
local ImersivoFrame = Instance.new("Frame")
ImersivoFrame.Size = UDim2.new(1, 0, 1, 0)
ImersivoFrame.BackgroundTransparency = 1
ImersivoFrame.Visible = false
ImersivoFrame.ZIndex = 2
ImersivoFrame.Parent = MainGui

local ImersivoText = Instance.new("TextLabel")
ImersivoText.Size = UDim2.new(1, 0, 0, 50)
ImersivoText.Position = UDim2.new(0, 0, 0.45, -25)
ImersivoText.BackgroundTransparency = 1
ImersivoText.Text = "Você está farmando..."
ImersivoText.TextColor3 = Color3.fromRGB(255, 255, 255)
ImersivoText.Font = Enum.Font.SourceSansLight
ImersivoText.TextSize = 32
ImersivoText.ZIndex = 3
ImersivoText.Parent = ImersivoFrame

-- Efeito de Desfocar a Tela do Jogo
local Blur = Instance.new("BlurEffect")
Blur.Name = "BatataFarmBlur"
Blur.Size = 0
Blur.Parent = Lighting

------------------------------------------------------------------------
-- 3. ANIMAÇÃO DE ENTRADA (FADE INTRO)
------------------------------------------------------------------------
task.spawn(function()
    task.wait(2.5)
    MainFrame.Visible = true
    
    local tweenInfo = TweenInfo.new(1.5, Enum.EasingStyle.Linear, Enum.EasingDirection.Out)
    local fadeIntro = TweenService:Create(IntroFrame, tweenInfo, {BackgroundTransparency = 1})
    local fadeText = TweenService:Create(IntroText, tweenInfo, {TextTransparency = 1})
    
    fadeIntro:Play()
    fadeText:Play()
    fadeIntro.Completed:Connect(function()
        IntroFrame:Destroy()
    end)
end)

------------------------------------------------------------------------
-- 4. LOOP DE ATUALIZAÇÃO DOS STATUS NA TELA
------------------------------------------------------------------------
task.spawn(function()
    while true do
        task.wait(0.2)
        local part, wind, range = obterMelhorUpdraftEParte()
        
        VentoLabel.Text = "Força do Vento: " .. tostring(wind) .. " km/h"
        RangeLabel.Text = "Raio (Range): " .. tostring(range) .. " studs"
        
        local character = LocalPlayer.Character
        local root = character and character:FindFirstChild("HumanoidRootPart")
        
        if part and root then
            local dist = (Vector3.new(root.Position.X, 0, root.Position.Z) - Vector3.new(part.Position.X, 0, part.Position.Z)).Magnitude
            DistanciaLabel.Text = "Distância Atual: " .. string.format("%.1f", dist) .. " studs"
        else
            DistanciaLabel.Text = "Distância Atual: Sem alvos no mapa"
        end
    end
end)

------------------------------------------------------------------------
-- 5. LÓGICA AUXILIAR DA SONDA E COMPRA (CORRIGIDO SEM GETCONNECTIONS)
------------------------------------------------------------------------
local function checarInventarioPorSonda()
    local backpack = LocalPlayer:FindFirstChild("Backpack")
    local character = LocalPlayer.Character
    return (backpack and (backpack:FindFirstChild("Probe") or backpack:FindFirstChild("Sonda")))
        or (character and (character:FindFirstChild("Probe") or character:FindFirstChild("Sonda")))
end

local function comprarSondaNaLoja()
    local pGui = LocalPlayer:FindFirstChild("PlayerGui")
    if pGui then
        local shop = pGui:FindFirstChild("EquipmentShop")
        local eq = shop and shop:FindFirstChild("Equipment")
        local frameEq = eq and eq:FindFirstChild("FrameEquipment")
        local probeFrame = frameEq and frameEq:FindFirstChild("Probe")
        local btn = probeFrame and probeFrame:FindFirstChild("PurchaseButton")
        
        if btn and btn:IsA("GuiButton") then
            -- Método nativo alternativo para forçar clique seguro sem depender do exploit
            btn:Activate()
            pcall(function()
                GuiService.SelectedObject = btn
                VirtualInputManager = game:GetService("VirtualInputManager")
                if VirtualInputManager then
                    VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Return, false, game)
                    VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Return, false, game)
                end
            end)
        end
    end
end

local function usarSonda()
    local character = LocalPlayer.Character
    local backpack = LocalPlayer:FindFirstChild("Backpack")
    if not character then return end
    local probe = (backpack and (backpack:FindFirstChild("Probe") or backpack:FindFirstChild("Sonda"))) or character:FindFirstChild("Probe") or character:FindFirstChild("Sonda")
    if probe then
        probe.Parent = character
        task.wait(0.02)
        if _G.ProbeFarmActive then probe:Activate() end
    end
end

-- Limpeza física absoluta no Respawn
LocalPlayer.CharacterAdded:Connect(function(char)
    if _G.ProbeFarmActive then
        local root = char:WaitForChild("HumanoidRootPart", 5)
        local humanoid = char:WaitForChild("Humanoid", 5)
        if root and humanoid then
            root.Anchored = false
            root.Velocity = Vector3.zero
            root.RotVelocity = Vector3.zero
            humanoid:ChangeState(Enum.HumanoidStateType.Physics)
        end
    end
end)

------------------------------------------------------------------------
-- 6. LOOP PRINCIPAL DO FARM
------------------------------------------------------------------------
_G.ProbeFarmActive = false

local function loopFarm()
    while _G.ProbeFarmActive do
        task.wait(0.05)
        
        if not checarInventarioPorSonda() then
            comprarSondaNaLoja()
            task.wait(0.2)
        end

        local character = LocalPlayer.Character
        local root = character and character:FindFirstChild("HumanoidRootPart")
        local humanoid = character and character:FindFirstChildOfClass("Humanoid")
        
        if root and humanoid and humanoid.Health > 0 and checarInventarioPorSonda() then
            local partAlvo, _, _ = obterMelhorUpdraftEParte()
            if partAlvo and partAlvo.Parent then
                
                -- PASSO 1: FICA TOTALMENTE MOLE
                root.Anchored = false
                root.Velocity = Vector3.zero
                root.RotVelocity = Vector3.zero
                humanoid:ChangeState(Enum.HumanoidStateType.Physics)
                task.wait(0.06) 
                
                -- PASSO 2: TELEPORTA COM O CORPO JÁ MOLE
                local posicaoSoloCalculada = Vector3.new(partAlvo.Position.X + 2, 25, partAlvo.Position.Z)
                if not _G.ProbeFarmActive then break end
                root.CFrame = CFrame.new(posicaoSoloCalculada)
                
                -- Espera o teleporte confirmar e estabilizar
                task.wait(0.15)
                
                -- PASSO 3: SOLTA A PROBE EM MODO MOLE
                usarSonda()
                task.wait(0.1) 
                
                -- PASSO 4: FICA DURO (ANCORA)
                root.Anchored = true
                
                -- PASSO 5: ESPERA DO CICLO (30 SEGUNDOS)
                local tempoGasto = 0
                while tempoGasto < 30 and _G.ProbeFarmActive do
                    task.wait(1)
                    tempoGasto = tempoGasto + 1
                end
                
                -- PASSO 6: DESANCORA E RESETA
                root.Anchored = false
                root.Velocity = Vector3.zero
                humanoid:ChangeState(Enum.HumanoidStateType.Physics)
                task.wait(0.05)
                
                if _G.ProbeFarmActive then
                    humanoid.Health = 0 
                    LocalPlayer.CharacterAdded:Wait()
                    task.wait(0.3)
                end
            end
        end
    end
end

------------------------------------------------------------------------
-- 7. CONTROLE DA ATIVAÇÃO E TRANSIÇÕES VISUAIS DO MODO FARM
------------------------------------------------------------------------
BotaoFarm.MouseButton1Click:Connect(function()
    _G.ProbeFarmActive = not _G.ProbeFarmActive
    
    if _G.ProbeFarmActive then
        BotaoFarm.Text = "Parar Auto Farm"
        BotaoFarm.TextColor3 = Color3.fromRGB(85, 255, 85)
        
        ImersivoFrame.Visible = true
        TweenService:Create(Blur, TweenInfo.new(0.5), {Size = 24}):Play()
        
        task.spawn(loopFarm)
    else
        BotaoFarm.Text = "Ativar Auto Farm"
        BotaoFarm.TextColor3 = Color3.fromRGB(255, 85, 85)
        
        ImersivoFrame.Visible = false
        TweenService:Create(Blur, TweenInfo.new(0.3), {Size = 0}):Play()
        
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            LocalPlayer.Character.HumanoidRootPart.Anchored = false
        end
    end
end)
