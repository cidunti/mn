-- ============================================================
-- MONITOR ANTI-CRASH FINAL (Captura + Bloqueia Fechamento)
-- ============================================================
local Players = game:GetService("Players")
local playerGui = Players.LocalPlayer:WaitForChild("PlayerGui")

local CAPTURED = false

-- ===== GUI =====
local gui = Instance.new("ScreenGui", playerGui)
gui.Name = "AntiCrashMonitor"
gui.DisplayOrder = 999
gui.IgnoreGuiInset = true

local frm = Instance.new("Frame", gui)
frm.Size = UDim2.new(0, 320, 0, 150)
frm.Position = UDim2.new(0.5, -160, 0.5, -75)
frm.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
frm.BorderSizePixel = 0
frm.Active = true
frm.Draggable = true
Instance.new("UICorner", frm).CornerRadius = UDim.new(0, 8)

local title = Instance.new("TextLabel", frm)
title.Size = UDim2.new(1, -20, 0, 30)
title.Position = UDim2.new(0, 10, 0, 5)
title.BackgroundTransparency = 1
title.Text = "Anti-Crash Ativo"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 12
title.TextXAlignment = Enum.TextXAlignment.Left

local info = Instance.new("TextLabel", frm)
info.Size = UDim2.new(1, -20, 0, 50)
info.Position = UDim2.new(0, 10, 0, 35)
info.BackgroundTransparency = 1
info.Text = "Proteção contra 'game:Shutdown' ligada.\nExecute o script Luraph agora."
info.TextColor3 = Color3.fromRGB(200, 230, 200)
info.Font = Enum.Font.Code
info.TextSize = 10
info.TextWrapped = true
info.TextXAlignment = Enum.TextXAlignment.Left

local btnCopy = Instance.new("TextButton", frm)
btnCopy.Size = UDim2.new(1, -20, 0, 30)
btnCopy.Position = UDim2.new(0, 10, 1, -40)
btnCopy.BackgroundColor3 = Color3.fromRGB(60, 170, 90)
btnCopy.Text = "Copiar Código (Se capturado)"
btnCopy.TextColor3 = Color3.new(1, 1, 1)
btnCopy.Font = Enum.Font.GothamBold
btnCopy.TextSize = 11
Instance.new("UICorner", btnCopy).CornerRadius = UDim.new(0, 4)

btnCopy.MouseButton1Click:Connect(function()
    if CAPTURED then
        btnCopy.Text = "Já copiado automaticamente!"
    else
        btnCopy.Text = "Nada capturado ainda."
    end
    task.delay(2, function() if btnCopy.Parent then btnCopy.Text = "Copiar Código (Se capturado)" end end)
end)

-- ===== 1. ANTI-CRASH =====
local function hookCrashMethod(obj, methodName)
    if type(obj) == "userdata" or type(obj) == "table" then
        pcall(function()
            local old = obj[methodName]
            if type(old) == "function" then
                obj[methodName] = function(...) 
                    info.Text = "🚫 Tentativa de crash bloqueada: " .. methodName
                    info.TextColor3 = Color3.fromRGB(255, 100, 100)
                end
            end
        end)
    end
end

-- Bloqueia os métodos comuns de fechar o jogo
hookCrashMethod(game, "Shutdown")
hookCrashMethod(game, "Destroy")
pcall(function() os.exit = function() end end)
pcall(function() game:GetService("Players").LocalPlayer:Kick = function() end end)

-- ===== 2. CAPTURA DO LOADSTRING =====
local oldLoad = loadstring
loadstring = function(src, chunkname)
    if type(src) == "string" and #src > 5000 then
        CAPTURED = true
        if type(setclipboard) == "function" then
            pcall(setclipboard, src)
        end
        info.Text = "✅ CÓDIGO CAPTURADO! Tamanho: " .. #src .. " bytes.\nJÁ ESTÁ NO SEU CTRL+V!"
        info.TextColor3 = Color3.fromRGB(0, 255, 0)
    end
    return oldLoad(src, chunkname)
end

-- ===== 3. BLOQUEIO DE LOOP INFINITO =====
local oldWait = task.wait
task.wait = function(t)
    return oldWait(math.min(t or 0.03, 0.03)) -- Força o yield para não travar o Roblox
end
