-- ============================================================
-- MONITOR DELTA - Captura Loadstring Luraph V15 (Ultra Leve)
-- ============================================================

local Players = game:GetService("Players")
local playerGui = Players.LocalPlayer:WaitForChild("PlayerGui")

-- Variáveis
local CAPTURED_CODE = nil
local STATUS = "Aguardando o script Luraph..."

-- ===== GUI Mínima =====
local gui = Instance.new("ScreenGui")
gui.Name = "MonitorDelta"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.Parent = playerGui

local frm = Instance.new("Frame")
frm.Size = UDim2.new(0, 300, 0, 150)
frm.Position = UDim2.new(0.5, -150, 0.5, -75)
frm.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
frm.BorderSizePixel = 0
frm.Active = true
frm.Draggable = true
frm.Parent = gui
Instance.new("UICorner", frm).CornerRadius = UDim.new(0, 6)

local bar = Instance.new("Frame")
bar.Size = UDim2.new(1, 0, 0, 28)
bar.BackgroundColor3 = Color3.fromRGB(45, 45, 58)
bar.BorderSizePixel = 0
bar.Parent = frm
Instance.new("UICorner", bar).CornerRadius = UDim.new(0, 6)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -10, 1, 0)
title.Position = UDim2.new(0, 8, 0, 0)
title.BackgroundTransparency = 1
title.Text = "Monitor Luraph"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 11
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = bar

local statusLabel = Instance.new("TextLabel")
statusLabel.Size = UDim2.new(1, -20, 0, 40)
statusLabel.Position = UDim2.new(0, 10, 0, 35)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = STATUS
statusLabel.TextColor3 = Color3.fromRGB(200, 230, 200)
statusLabel.Font = Enum.Font.Code
statusLabel.TextSize = 10
statusLabel.TextWrapped = true
statusLabel.TextXAlignment = Enum.TextXAlignment.Left
statusLabel.TextYAlignment = Enum.TextYAlignment.Top
statusLabel.Parent = frm

local btnCopy = Instance.new("TextButton")
btnCopy.Size = UDim2.new(1, -20, 0, 32)
btnCopy.Position = UDim2.new(0, 10, 1, -42)
btnCopy.BackgroundColor3 = Color3.fromRGB(60, 130, 210)
btnCopy.Text = "Copiar Código (Se capturado)"
btnCopy.TextColor3 = Color3.new(1, 1, 1)
btnCopy.Font = Enum.Font.GothamBold
btnCopy.TextSize = 11
btnCopy.BorderSizePixel = 0
btnCopy.Parent = frm
Instance.new("UICorner", btnCopy).CornerRadius = UDim.new(0, 4)

-- ===== Funções =====
local function atualizarStatus(txt, cor)
    statusLabel.Text = txt
    if cor then statusLabel.TextColor3 = cor end
end

btnCopy.MouseButton1Click:Connect(function()
    if CAPTURED_CODE and #CAPTURED_CODE > 0 then
        if type(setclipboard) == "function" then
            pcall(setclipboard, CAPTURED_CODE)
            btnCopy.Text = "Copiado com sucesso!"
            task.delay(2, function() if btnCopy.Parent then btnCopy.Text = "Copiar Código (Se capturado)" end end)
        else
            btnCopy.Text = "Clipboard indisponível"
        end
    else
        btnCopy.Text = "Nada capturado ainda!"
        task.delay(2, function() if btnCopy.Parent then btnCopy.Text = "Copiar Código (Se capturado)" end end)
    end
end)

-- ===== O Hook =====
local oldLoadstring = loadstring
local function hookLoadstring(source, chunkname)
    -- Se o código que o Luraph está tentando rodar for grande (o código real)
    if type(source) == "string" and #source > 5000 then
        CAPTURED_CODE = source
        atualizarStatus("✅ CÓDIGO CAPTURADO!\nTamanho: " .. #source .. " bytes.\nClique em Copiar e cole aqui!", Color3.fromRGB(100, 255, 100))
        
        -- Copia automaticamente pra facilitar
        if type(setclipboard) == "function" then
            pcall(setclipboard, source)
        end
    end
    
    -- IMPORTANTE: Executa o código original para o Luraph não detectar e o jogo rodar
    if oldLoadstring then
        return oldLoadstring(source, chunkname)
    end
end

-- Aplica o hook usando hookfunction (invisível para o Luraph)
if type(hookfunction) == "function" then
    hookfunction(oldLoadstring, hookLoadstring)
    atualizarStatus("Hook aplicado com sucesso.\nAgora execute o script Luraph no Delta.", Color3.fromRGB(255, 255, 100))
else
    -- Fallback se o Delta não tiver hookfunction
    loadstring = hookLoadstring
    atualizarStatus("Hook via substituição aplicado.\nExecute o script Luraph.", Color3.fromRGB(255, 255, 100))
end
