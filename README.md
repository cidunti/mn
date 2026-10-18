-- ============================================================
-- MONITOR DE URLS (JNKIE/LURAPH) - Com menu e Copiar Tudo
-- ============================================================

local Players = game:GetService("Players")
local playerGui = Players.LocalPlayer:WaitForChild("PlayerGui")

-- Variáveis
local LOGS = {}
local URLS_RAW = {}
local CAPTURED = false

-- ===== GUI =====
local gui = Instance.new("ScreenGui")
gui.Name = "UrlMonitor"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.IgnoreGuiInset = true
gui.Parent = playerGui

local frm = Instance.new("Frame")
frm.Size = UDim2.new(0, 360, 0, 240)
frm.Position = UDim2.new(0.5, -180, 0.5, -120)
frm.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
frm.BorderSizePixel = 0
frm.Active = true
frm.Draggable = true
frm.Parent = gui
Instance.new("UICorner", frm).CornerRadius = UDim.new(0, 8)

local bar = Instance.new("Frame")
bar.Size = UDim2.new(1, 0, 0, 32)
bar.BackgroundColor3 = Color3.fromRGB(45, 45, 58)
bar.BorderSizePixel = 0
bar.Parent = frm
Instance.new("UICorner", bar).CornerRadius = UDim.new(0, 8)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -40, 1, 0)
title.Position = UDim2.new(0, 10, 0, 0)
title.BackgroundTransparency = 1
title.Text = "Monitor de URLs (Aguardando...)"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 12
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = bar

local btnClose = Instance.new("TextButton")
btnClose.Size = UDim2.new(0, 22, 0, 22)
btnClose.Position = UDim2.new(1, -28, 0, 5)
btnClose.BackgroundColor3 = Color3.fromRGB(180, 60, 60)
btnClose.Text = "X"
btnClose.TextColor3 = Color3.new(1, 1, 1)
btnClose.Font = Enum.Font.GothamBold
btnClose.TextSize = 11
btnClose.BorderSizePixel = 0
btnClose.Parent = bar
Instance.new("UICorner", btnClose).CornerRadius = UDim.new(0, 4)

local scroll = Instance.new("ScrollingFrame")
scroll.Size = UDim2.new(1, -12, 1, -80)
scroll.Position = UDim2.new(0, 6, 0, 38)
scroll.BackgroundColor3 = Color3.fromRGB(10, 10, 14)
scroll.BorderSizePixel = 0
scroll.ScrollBarThickness = 4
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
scroll.Parent = frm
Instance.new("UICorner", scroll).CornerRadius = UDim.new(0, 4)

local txt = Instance.new("TextBox")
txt.Size = UDim2.new(1, -8, 0, 24)
txt.Position = UDim2.new(0, 4, 0, 4)
txt.BackgroundTransparency = 1
txt.Text = "Nenhuma URL capturada ainda...\n\nExecute o script Luraph agora."
txt.TextColor3 = Color3.fromRGB(0, 255, 0)
txt.Font = Enum.Font.Code
txt.TextSize = 10
txt.TextWrapped = true
txt.MultiLine = true
txt.TextXAlignment = Enum.TextXAlignment.Left
txt.TextYAlignment = Enum.TextYAlignment.Top
txt.ClearTextOnFocus = false
txt.Parent = scroll

local btnCopy = Instance.new("TextButton")
btnCopy.Size = UDim2.new(1, -12, 0, 32)
btnCopy.Position = UDim2.new(0, 6, 1, -38)
btnCopy.BackgroundColor3 = Color3.fromRGB(60, 130, 210)
btnCopy.Text = "Copiar Tudo"
btnCopy.TextColor3 = Color3.new(1, 1, 1)
btnCopy.Font = Enum.Font.GothamBold
btnCopy.TextSize = 12
btnCopy.BorderSizePixel = 0
btnCopy.Parent = frm
Instance.new("UICorner", btnCopy).CornerRadius = UDim.new(0, 4)

-- ===== Funções de UI =====
local function atualizarUI()
    if #LOGS == 0 then return end
    
    txt.Text = table.concat(LOGS, "\n\n")
    title.Text = "Monitor de URLs (" .. #LOGS .. " capturadas)"
    
    -- Ajusta o tamanho para o Scroll funcionar bem
    local size = 24
    for _, log in ipairs(LOGS) do
        size = size + 20 -- Estimativa de altura por linha
    end
    txt.Size = UDim2.new(1, -8, 0, size)
end

btnCopy.MouseButton1Click:Connect(function()
    if #URLS_RAW == 0 then
        btnCopy.Text = "Nada capturado!"
    else
        local textoFinal = table.concat(URLS_RAW, "\n")
        local sucesso = false
        if type(setclipboard) == "function" then
            sucesso = pcall(setclipboard, textoFinal)
        end
        
        btnCopy.Text = sucesso and "Copiado!" or "Erro ao copiar"
    end
    
    task.delay(2, function()
        if btnCopy.Parent then btnCopy.Text = "Copiar Tudo" end
    end)
end)

btnClose.MouseButton1Click:Connect(function()
    gui:Destroy()
end)

-- ===== Hook e Captura =====
local function registrarMetodo(metodo, url)
    if type(url) ~= "string" then return end
    
    -- Só registra se parecer com uma URL ou link de API
    if string.match(url, "^https?://") or string.match(url, "%.com") or string.match(url, "%.lua") then
        
        -- Evita duplicar a mesma URL repetidamente na tela
        local duplicado = false
        for _, u in ipairs(URLS_RAW) do
            if u == url then duplicado = true break end
        end
        
        if not duplicado then
            table.insert(URLS_RAW, url)
            table.insert(LOGS, "[" .. metodo .. "]\n" .. url)
            atualizarUI()
        end
    end
end

-- Hook game.HttpGet
local oldHttpGet = game.HttpGet
game.HttpGet = function(self, url, ...)
    registrarMetodo("HttpGet", tostring(url))
    return oldHttpGet(self, url, ...)
end

-- Hook game.HttpGetAsync
local oldHttpGetAsync = game.HttpGetAsync
game.HttpGetAsync = function(self, url, ...)
    registrarMetodo("HttpGetAsync", tostring(url))
    return oldHttpGetAsync(self, url, ...)
end

-- Hook request
if type(request) == "function" then
    local oldReq = request
    request = function(opts)
        if type(opts) == "table" and opts.Url then
            registrarMetodo("request", tostring(opts.Url))
        end
        return oldReq(opts)
    end
end

-- Hook http_request
if type(http_request) == "function" then
    local oldHttpReq = http_request
    http_request = function(opts)
        if type(opts) == "table" and opts.Url then
            registrarMetodo("http_request", tostring(opts.Url))
        end
        return oldHttpReq(opts)
    end
end

-- Hook syn.request
if type(syn) == "table" and type(syn.request) == "function" then
    local oldSynReq = syn.request
    syn.request = function(opts)
        if type(opts) == "table" and opts.Url then
            registrarMetodo("syn.request", tostring(opts.Url))
        end
        return oldSynReq(opts)
    end
end
