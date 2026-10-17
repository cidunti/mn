-- ============================================================
-- MONITOR 100KB/s - loadstring/load (mobile-safe)
-- Salva em pedaços a 100 KB por segundo e mostra % no título.
-- Botões: Copiar (última captura) e Salvar (força gravação).
-- Permite a execução do loadstring/load original.
-- ============================================================

local CAPTURED = {}
local PREVIEWS = {}
local LAST_CAPTURE = nil   -- última string capturada (código completo)
local N = 0
local FILE_COUNT = 0
local MAX_PREVIEW = 1500
local HOOKED = {}

local CHUNK_BYTES = 100 * 1024   -- 100 KB por segundo

local unpackFn = table.unpack or unpack

-- ---------- GUI ----------
local parent = (type(gethui) == "function" and select(2, pcall(gethui)))
    or game:GetService("CoreGui")

local gui = Instance.new("ScreenGui")
gui.Name = "Monitor100KB"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.IgnoreGuiInset = true
gui.Parent = parent

local frm = Instance.new("Frame")
frm.Size = UDim2.new(0, 320, 0, 240)
frm.Position = UDim2.new(0.5, -160, 0.5, -120)
frm.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
frm.BorderSizePixel = 0
frm.Active = true
frm.Draggable = true
frm.Parent = gui
Instance.new("UICorner", frm).CornerRadius = UDim.new(0, 6)

local bar = Instance.new("Frame")
bar.Size = UDim2.new(1, 0, 0, 26)
bar.BackgroundColor3 = Color3.fromRGB(45, 45, 58)
bar.BorderSizePixel = 0
bar.Parent = frm
Instance.new("UICorner", bar).CornerRadius = UDim.new(0, 6)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -120, 1, 0)
title.Position = UDim2.new(0, 6, 0, 0)
title.BackgroundTransparency = 1
title.Text = "Monitor | 0%"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 11
title.TextXAlignment = Enum.TextXAlignment.Left
title.TextTruncate = Enum.TextTruncate.AtEnd
title.Parent = bar

local btnCopy = Instance.new("TextButton")
btnCopy.Size = UDim2.new(0, 52, 0, 18)
btnCopy.Position = UDim2.new(1, -80, 0, 4)
btnCopy.BackgroundColor3 = Color3.fromRGB(60, 130, 210)
btnCopy.Text = "Copiar"
btnCopy.TextColor3 = Color3.new(1, 1, 1)
btnCopy.Font = Enum.Font.GothamBold
btnCopy.TextSize = 9
btnCopy.BorderSizePixel = 0
btnCopy.Parent = bar
Instance.new("UICorner", btnCopy).CornerRadius = UDim.new(0, 3)

local btnClose = Instance.new("TextButton")
btnClose.Size = UDim2.new(0, 18, 0, 18)
btnClose.Position = UDim2.new(1, -24, 0, 4)
btnClose.BackgroundColor3 = Color3.fromRGB(180, 60, 60)
btnClose.Text = "×"
btnClose.TextColor3 = Color3.new(1, 1, 1)
btnClose.Font = Enum.Font.GothamBold
btnClose.TextSize = 11
btnClose.BorderSizePixel = 0
btnClose.Parent = bar
Instance.new("UICorner", btnClose).CornerRadius = UDim.new(0, 3)

local progBg = Instance.new("Frame")
progBg.Size = UDim2.new(1, -8, 0, 4)
progBg.Position = UDim2.new(0, 4, 0, 27)
progBg.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
progBg.BorderSizePixel = 0
progBg.Parent = frm
Instance.new("UICorner", progBg).CornerRadius = UDim.new(0, 2)

local progFill = Instance.new("Frame")
progFill.Size = UDim2.new(0, 0, 1, 0)
progFill.BackgroundColor3 = Color3.fromRGB(60, 170, 90)
progFill.BorderSizePixel = 0
progFill.Parent = progBg
Instance.new("UICorner", progFill).CornerRadius = UDim.new(0, 2)

local scroll = Instance.new("ScrollingFrame")
scroll.Size = UDim2.new(1, -8, 1, -38)
scroll.Position = UDim2.new(0, 4, 0, 34)
scroll.BackgroundColor3 = Color3.fromRGB(10, 10, 14)
scroll.BorderSizePixel = 0
scroll.ScrollBarThickness = 4
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
scroll.Parent = frm
Instance.new("UICorner", scroll).CornerRadius = UDim.new(0, 3)

local txt = Instance.new("TextBox")
txt.Size = UDim2.new(1, -4, 1, 0)
txt.Position = UDim2.new(0, 2, 0, 0)
txt.BackgroundTransparency = 1
txt.Text = "(aguardando...)"
txt.TextColor3 = Color3.fromRGB(200, 230, 200)
txt.Font = Enum.Font.Code
txt.TextSize = 10
txt.TextWrapped = true
txt.MultiLine = true
txt.TextXAlignment = Enum.TextXAlignment.Left
txt.TextYAlignment = Enum.TextYAlignment.Top
txt.ClearTextOnFocus = false
txt.Parent = scroll

-- ---------- progresso ----------
local PROG = { total = 0, done = 0, lastShown = -1, name = "" }

local function setProgress(pct)
    pct = math.clamp(math.floor(pct), 0, 100)
    if pct == PROG.lastShown then return end
    PROG.lastShown = pct
    title.Text = "Monitor | " .. PROG.name .. " " .. pct .. "%"
    progFill.Size = UDim2.new(pct / 100, 0, 1, 0)
end

local function doRefresh()
    local lines = {}
    for _, entry in ipairs(CAPTURED) do lines[#lines + 1] = entry end
    for _, p in ipairs(PREVIEWS) do lines[#lines + 1] = p end
    txt.Text = (#lines > 0) and table.concat(lines, "\n\n") or "(aguardando...)"
end

-- ---------- escrita com ritmo 100 KB/s ----------
local function openFile(name)
    FILE_COUNT = FILE_COUNT + 1
    return string.format("Monitor100KB_%s_%03d.txt",
        tostring(name):gsub("[^%w_%-]", "_"), FILE_COUNT)
end

local function writeChunk(path, chunk, isFirst)
    local wf, af, rf = writefile, appendfile, readfile
    if type(wf) ~= "function" then return false, "writefile indisponível" end

    if isFirst then
        return pcall(wf, path, chunk)
    end
    if type(af) == "function" then
        return pcall(af, path, chunk)
    end
    if type(rf) == "function" then
        local ok, prev = pcall(rf, path)
        if ok and type(prev) == "string" then
            return pcall(wf, path, prev .. chunk)
        end
    end
    return pcall(wf, path, chunk)
end

-- Salva o código em pedaços de 100 KB, esperando 1s entre eles
-- (ritmo de 100 KB por segundo).
local function saveAt100KBs(name, src)
    local total = #src
    PROG.name = name
    PROG.total = total
    PROG.done = 0
    PROG.lastShown = -1
    setProgress(0)

    local path = openFile(name)
    local i = 1
    local first = true
    local saved = 0

    while i <= total do
        local j = math.min(i + CHUNK_BYTES - 1, total)
        local chunk = src:sub(i, j)

        local ok, err = writeChunk(path, chunk, first)
        first = false

        if not ok then
            CAPTURED[#CAPTURED + 1] = "SAVE ERRO: " .. tostring(err)
            return path, saved
        end

        saved = saved + #chunk
        PROG.done = j
        setProgress((PROG.done / total) * 100)

        -- espera 1 segundo entre os pedaços (100 KB/s)
        if j < total and task and task.wait then
            task.wait(1)
        end

        i = j + 1
    end

    setProgress(100)
    CAPTURED[#CAPTURED + 1] = "SALVO: " .. path .. " (" .. total .. " bytes)"
    return path, saved
end

-- ---------- hook ----------
local function hook(env, envName, fname)
    local orig = rawget(env, fname)
    if type(orig) ~= "function" then return end
    if HOOKED[orig] then return end
    HOOKED[orig] = true

    local function wrapper(source, ...)
        local st = type(source)
        local s = tostring(source)

        N = N + 1
        CAPTURED[#CAPTURED + 1] = "[#" .. N .. "] " .. envName .. "." .. fname
            .. " | " .. st .. " | " .. #s .. " bytes"

        if st == "string" and #s > 0 then
            -- guarda o código completo para o botão Copiar
            LAST_CAPTURE = s

            -- salva em pedaços de 100 KB/s
            local path = saveAt100KBs(fname, s)

            -- prévia curta no painel
            PREVIEWS[1] = "[ÚLTIMA PRÉVIA] " .. fname
                .. " | " .. #s .. " bytes\n"
                .. s:sub(1, MAX_PREVIEW)
                .. (#s > MAX_PREVIEW and "\n[arquivo completo: " .. tostring(path) .. "]" or "")
            pcall(doRefresh)
        end

        -- executa o original normalmente
        local args = { ... }
        local n = select("#", ...)
        local results = { n = n, pcall(orig, source, unpackFn(args, 1, n)) }

        if results[1] then
            return unpackFn(results, 2, results.n + 1)
        end
        error(results[2], 0)
    end

    if type(hookfunction) == "function" then
        local ok = pcall(hookfunction, orig, wrapper)
        if ok then
            CAPTURED[#CAPTURED + 1] = "HOOK: " .. envName .. "." .. fname .. " (permitido)"
            return
        end
    end
    local ok = pcall(function() env[fname] = wrapper end)
    CAPTURED[#CAPTURED + 1] = "HOOK: " .. envName .. "." .. fname
        .. (ok and " (permitido)" or " (falhou)")
end

-- ---------- ambientes ----------
local ENVS = {}
local seen = {}
local function addEnv(e, n)
    if type(e) == "table" and not seen[e] then
        seen[e] = true
        ENVS[#ENVS + 1] = { env = e, name = n }
    end
end

addEnv(_G, "_G")
pcall(function() if type(getgenv) == "function" then addEnv(getgenv(), "getgenv") end end)
pcall(function() if type(getfenv) == "function" then addEnv(getfenv(), "getfenv") end end)

CAPTURED[#CAPTURED + 1] = "INIT: " .. #ENVS .. " ambiente(s)"

for _, e in ipairs(ENVS) do
    hook(e.env, e.name, "loadstring")
    hook(e.env, e.name, "load")
end

CAPTURED[#CAPTURED + 1] = "STATUS: pronto. Execute o script alvo."
doRefresh()

-- ---------- botões ----------
btnCopy.MouseButton1Click:Connect(function()
    if type(LAST_CAPTURE) ~= "string" then
        btnCopy.Text = "Vazio"
        task.delay(1.5, function()
            if btnCopy.Parent then btnCopy.Text = "Copiar" end
        end)
        return
    end

    local ok = false
    if type(setclipboard) == "function" then
        ok = pcall(setclipboard, LAST_CAPTURE)
    elseif type(toclipboard) == "function" then
        ok = pcall(toclipboard, LAST_CAPTURE)
    end

    btnCopy.Text = ok and "OK!" or "N/D"
    task.delay(1.5, function()
        if btnCopy.Parent then btnCopy.Text = "Copiar" end
    end)
end)

btnClose.MouseButton1Click:Connect(function() gui:Destroy() end)
