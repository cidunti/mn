-- ============================================================
-- MONITOR V4 COMPACTO (mobile-friendly) - CORRIGIDO
-- Interface + captura HTTP + respostas + load/loadstring
--
-- CORREÇÃO: loadstring/load agora CAPTURAM e PERMITEM execução.
-- O código capturado é salvo em arquivo e mostrado no painel,
-- mas a compilação original é repassada ao script.
--
-- Cole e execute este script primeiro.
-- Depois, execute o script que deseja inspecionar.
-- ============================================================

local CAPTURED = {}
local N = 0
local WRITING = false

local MAX_PREVIEW = 6000
local MAX_HTTP_RESPONSE = 2000

local REQUEST_HOOKED = {}
local LOADER_HOOKED = {}
local HTTP_METHOD_HOOKED = {}
local LOADER_FILE_COUNT = 0

local function pack(...)
    return { n = select("#", ...), ... }
end

local unpackFn = table.unpack or unpack

-- ============================================================
-- 1. INTERFACE COMPACTA
-- ============================================================

local parent

if type(gethui) == "function" then
    local ok, result = pcall(gethui)
    if ok and result then parent = result end
end

if not parent then
    local ok, result = pcall(function()
        return game:GetService("CoreGui")
    end)
    if ok then parent = result end
end

assert(parent, "Monitor v4: não foi possível encontrar o parent da GUI.")

local BASE_W, BASE_H = 340, 300

local gui = Instance.new("ScreenGui")
gui.Name = "MonitorV4"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.IgnoreGuiInset = true

local parentOK = pcall(function() gui.Parent = parent end)
if not parentOK then
    error("Monitor v4: não foi possível criar a interface.")
end

local frm = Instance.new("Frame")
frm.Name = "Main"
frm.Size = UDim2.new(0, BASE_W, 0, BASE_H)
frm.Position = UDim2.new(0.5, -BASE_W / 2, 0.5, -BASE_H / 2)
frm.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
frm.BorderSizePixel = 0
frm.Active = true
frm.Draggable = true
frm.Parent = gui

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 8)
frameCorner.Parent = frm

local BAR_H = 30

local bar = Instance.new("Frame")
bar.Name = "TitleBar"
bar.Size = UDim2.new(1, 0, 0, BAR_H)
bar.BackgroundColor3 = Color3.fromRGB(45, 45, 58)
bar.BorderSizePixel = 0
bar.Parent = frm

local barCorner = Instance.new("UICorner")
barCorner.CornerRadius = UDim.new(0, 8)
barCorner.Parent = bar

local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.new(1, -120, 1, 0)
title.Position = UDim2.new(0, 8, 0, 0)
title.BackgroundTransparency = 1
title.Text = "Monitor v4 | 0"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 11
title.TextXAlignment = Enum.TextXAlignment.Left
title.TextTruncate = Enum.TextTruncate.AtEnd
title.Parent = bar

local btnCopy = Instance.new("TextButton")
btnCopy.Name = "Copy"
btnCopy.Size = UDim2.new(0, 58, 0, 20)
btnCopy.Position = UDim2.new(1, -122, 0, 5)
btnCopy.BackgroundColor3 = Color3.fromRGB(60, 130, 210)
btnCopy.Text = "Copiar"
btnCopy.TextColor3 = Color3.new(1, 1, 1)
btnCopy.Font = Enum.Font.GothamBold
btnCopy.TextSize = 10
btnCopy.BorderSizePixel = 0
btnCopy.Parent = bar

local copyCorner = Instance.new("UICorner")
copyCorner.CornerRadius = UDim.new(0, 4)
copyCorner.Parent = btnCopy

local btnMin = Instance.new("TextButton")
btnMin.Name = "Min"
btnMin.Size = UDim2.new(0, 22, 0, 20)
btnMin.Position = UDim2.new(1, -58, 0, 5)
btnMin.BackgroundColor3 = Color3.fromRGB(90, 90, 105)
btnMin.Text = "—"
btnMin.TextColor3 = Color3.new(1, 1, 1)
btnMin.Font = Enum.Font.GothamBold
btnMin.TextSize = 12
btnMin.BorderSizePixel = 0
btnMin.Parent = bar

local minCorner = Instance.new("UICorner")
minCorner.CornerRadius = UDim.new(0, 4)
minCorner.Parent = btnMin

local btnClose = Instance.new("TextButton")
btnClose.Name = "Close"
btnClose.Size = UDim2.new(0, 22, 0, 20)
btnClose.Position = UDim2.new(1, -32, 0, 5)
btnClose.BackgroundColor3 = Color3.fromRGB(180, 60, 60)
btnClose.Text = "×"
btnClose.TextColor3 = Color3.new(1, 1, 1)
btnClose.Font = Enum.Font.GothamBold
btnClose.TextSize = 13
btnClose.BorderSizePixel = 0
btnClose.Parent = bar

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 4)
closeCorner.Parent = btnClose

local scroll = Instance.new("ScrollingFrame")
scroll.Name = "Output"
scroll.Size = UDim2.new(1, -12, 1, -(BAR_H + 10))
scroll.Position = UDim2.new(0, 6, 0, BAR_H + 5)
scroll.BackgroundColor3 = Color3.fromRGB(10, 10, 14)
scroll.BorderSizePixel = 0
scroll.ScrollBarThickness = 4
scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
scroll.Parent = frm

local scrollCorner = Instance.new("UICorner")
scrollCorner.CornerRadius = UDim.new(0, 4)
scrollCorner.Parent = scroll

local txt = Instance.new("TextBox")
txt.Name = "CapturedText"
txt.Size = UDim2.new(1, -8, 0, 24)
txt.Position = UDim2.new(0, 4, 0, 4)
txt.BackgroundTransparency = 1
txt.Text = "(aguardando capturas...)"
txt.TextColor3 = Color3.fromRGB(200, 230, 200)
txt.Font = Enum.Font.Code
txt.TextSize = 10
txt.TextWrapped = true
txt.MultiLine = true
txt.TextXAlignment = Enum.TextXAlignment.Left
txt.TextYAlignment = Enum.TextYAlignment.Top
txt.ClearTextOnFocus = false
txt.Parent = scroll

local TS = game:GetService("TextService")

local function refresh()
    local joined
    if #CAPTURED > 0 then
        joined = table.concat(CAPTURED, "\n\n")
    else
        joined = "(aguardando capturas...)"
    end

    txt.Text = joined
    title.Text = "Monitor v4 | " .. tostring(N)

    local availableWidth = math.max(scroll.AbsoluteSize.X - 20, 200)

    local ok, size = pcall(function()
        return TS:GetTextSize(
            joined,
            10,
            Enum.Font.Code,
            Vector2.new(availableWidth, 1000000)
        )
    end)

    local textHeight = 80
    if ok and size then textHeight = size.Y + 12 end

    txt.Size = UDim2.new(1, -8, 0, textHeight)
    scroll.CanvasSize = UDim2.new(0, 0, 0, textHeight + 12)
end

local function cap(tag, body)
    N = N + 1
    local entry = "[#" .. tostring(N) .. "] " .. tostring(tag)
        .. "\n" .. tostring(body)
    table.insert(CAPTURED, entry)
    pcall(refresh)
    pcall(print, entry)
end

local minimized = false
local lastH = BASE_H

btnMin.MouseButton1Click:Connect(function()
    minimized = not minimized
    if minimized then
        lastH = frm.Size.Y.Offset
        frm.Size = UDim2.new(0, frm.Size.X.Offset, 0, BAR_H)
        scroll.Visible = false
        btnMin.Text = "+"
    else
        frm.Size = UDim2.new(0, frm.Size.X.Offset, 0, lastH)
        scroll.Visible = true
        btnMin.Text = "—"
        pcall(refresh)
    end
end)

btnClose.MouseButton1Click:Connect(function()
    pcall(function() gui:Destroy() end)
end)

btnCopy.MouseButton1Click:Connect(function()
    local allText = table.concat(CAPTURED, "\n\n")
    local success = false

    if type(setclipboard) == "function" then
        success = pcall(setclipboard, allText)
    elseif type(toclipboard) == "function" then
        success = pcall(toclipboard, allText)
    end

    btnCopy.Text = success and "OK!" or "N/D"

    task.delay(2, function()
        if btnCopy and btnCopy.Parent then
            btnCopy.Text = "Copiar"
        end
    end)
end)

pcall(function()
    local cam = workspace.CurrentCamera
    if cam then
        local vs = cam.ViewportSize
        local maxW = math.min(BASE_W, vs.X - 20)
        local maxH = math.min(BASE_H, vs.Y - 40)
        if maxW < BASE_W or maxH < BASE_H then
            frm.Size = UDim2.new(0, maxW, 0, maxH)
            frm.Position = UDim2.new(0.5, -maxW / 2, 0.5, -maxH / 2)
        end
    end
end)

-- ============================================================
-- 2. LOCALIZAR AMBIENTES
-- ============================================================

local ENVS = {}
local SEEN_ENVS = {}

local function addEnv(env, name)
    if type(env) ~= "table" then return end
    if SEEN_ENVS[env] then return end
    SEEN_ENVS[env] = true
    table.insert(ENVS, { env = env, name = name })
end

addEnv(_G, "_G")

pcall(function()
    if type(getfenv) == "function" then addEnv(getfenv(), "getfenv()") end
end)

pcall(function()
    if type(getfenv) == "function" then addEnv(getfenv(0), "getfenv(0)") end
end)

pcall(function()
    if type(getgenv) == "function" then addEnv(getgenv(), "getgenv()") end
end)

pcall(function()
    if type(getrenv) == "function" then addEnv(getrenv(), "getrenv()") end
end)

pcall(function()
    if type(getreg) ~= "function" then return end
    local registry = getreg()
    if type(registry) ~= "table" then return end
    for key, value in pairs(registry) do
        if type(value) == "table" then
            local looksRelevant =
                type(rawget(value, "request")) == "function"
                or type(rawget(value, "http_request")) == "function"
                or type(rawget(value, "loadstring")) == "function"
            if looksRelevant then
                addEnv(value, "getreg()[" .. tostring(key) .. "]")
            end
        end
    end
end)

cap("INIT", "Ambientes encontrados: " .. tostring(#ENVS))

-- ============================================================
-- 3. CAPTURA DE REQUEST / HTTP_REQUEST / SYN.REQUEST / HTTP.REQUEST
-- ============================================================

local function describeRequest(opts)
    local url = "?"
    local body = ""

    if type(opts) == "string" then
        url = opts
    elseif type(opts) == "table" then
        pcall(function()
            url = opts.Url or opts.url or opts.URL or opts[1] or "?"
        end)
        pcall(function()
            body = opts.Body or opts.body or ""
        end)
        if body == "" then
            pcall(function()
                body = game:GetService("HttpService"):JSONEncode(opts)
            end)
        end
    end

    return tostring(url), tostring(body)
end

local function describeResponse(result)
    if type(result) == "table" then
        local responseBody
        pcall(function()
            responseBody = result.Body or result.body
        end)
        if responseBody ~= nil then
            return tostring(responseBody)
        end
        local ok, encoded = pcall(function()
            return game:GetService("HttpService"):JSONEncode(result)
        end)
        if ok then return encoded end
    end
    return tostring(result)
end

local function hookRequest(env, envName, functionName)
    local original = rawget(env, functionName)
    if type(original) ~= "function" then return end
    if REQUEST_HOOKED[original] then return end

    local oldFunction = original

    local function wrapper(...)
        local args = pack(...)
        local opts = args[1]
        local url, body = describeRequest(opts)

        cap(
            envName .. "." .. functionName,
            "URL: " .. url .. "\nBODY: " .. body:sub(1, 800)
        )

        local results = pack(pcall(oldFunction, unpackFn(args, 1, args.n)))

        if results[1] then
            local result = results[2]
            local responseText = describeResponse(result)
            cap(
                envName .. "." .. functionName .. " >> RESPOSTA",
                responseText:sub(1, MAX_HTTP_RESPONSE)
            )
            return unpackFn(results, 2, results.n)
        end

        cap(
            envName .. "." .. functionName .. " >> ERRO",
            tostring(results[2])
        )
        error(results[2], 0)
    end

    if type(hookfunction) == "function" then
        local ok, previous = pcall(hookfunction, original, wrapper)
        if ok then
            oldFunction = previous or original
            REQUEST_HOOKED[original] = true
            cap("HOOK HTTP", envName .. "." .. functionName .. " interceptado via hookfunction.")
            return
        end
        cap(
            "HOOK HTTP AVISO",
            "Falha ao hookar " .. envName .. "." .. functionName
                .. ": " .. tostring(previous)
                .. "\nTentando substituir a referência."
        )
    end

    local ok, err = pcall(function() env[functionName] = wrapper end)
    if ok then
        cap("HOOK HTTP", envName .. "." .. functionName .. " substituído nesse ambiente.")
    else
        cap("HOOK HTTP ERRO", envName .. "." .. functionName .. ": " .. tostring(err))
    end
end

for _, entry in ipairs(ENVS) do
    local env = entry.env
    local envName = entry.name

    for _, functionName in ipairs({
        "request", "http_request", "syn_request", "httpRequest"
    }) do
        pcall(function() hookRequest(env, envName, functionName) end)
    end

    for _, tableName in ipairs({ "syn", "http" }) do
        local requestTable = rawget(env, tableName)
        if type(requestTable) == "table" then
            for _, functionName in ipairs({ "request", "http_request" }) do
                pcall(function()
                    hookRequest(requestTable, envName .. "." .. tableName, functionName)
                end)
            end
        end
    end
end

-- ============================================================
-- 4. CAPTURA GAME:HTTPGET / HTTPGETASYNC
-- ============================================================

if type(hookfunction) == "function" then
    for _, methodName in ipairs({ "HttpGet", "HttpGetAsync" }) do
        pcall(function()
            local original = game[methodName]
            if type(original) ~= "function" then return end
            if HTTP_METHOD_HOOKED[original] then return end

            local oldFunction = original

            local function wrapper(self, url, ...)
                cap("game." .. methodName, "URL: " .. tostring(url))
                local args = pack(...)
                local results = pack(pcall(function()
                    return oldFunction(self, url, unpackFn(args, 1, args.n))
                end))

                if results[1] then
                    local responseText = tostring(results[2])
                    cap(
                        "game." .. methodName .. " >> RESPOSTA",
                        responseText:sub(1, MAX_HTTP_RESPONSE)
                    )
                    return unpackFn(results, 2, results.n)
                end

                cap("game." .. methodName .. " >> ERRO", tostring(results[2]))
                error(results[2], 0)
            end

            local ok, previous = pcall(hookfunction, original, wrapper)
            if ok then
                oldFunction = previous or original
                HTTP_METHOD_HOOKED[original] = true
                cap("HOOK HTTP", "game." .. methodName .. " interceptado.")
            else
                cap(
                    "HOOK HTTP ERRO",
                    "Não foi possível interceptar game." .. methodName
                        .. ": " .. tostring(previous)
                )
            end
        end)
    end
else
    cap(
        "HOOK HTTP AVISO",
        "hookfunction indisponível; HttpGet/HttpGetAsync não foram interceptados."
    )
end

-- ============================================================
-- 5. CAPTURA WRITEFILE
-- ============================================================

local originalWritefile

pcall(function()
    if type(writefile) == "function" then
        originalWritefile = writefile
        writefile = function(path, content)
            if not WRITING then
                cap(
                    "writefile",
                    "PATH: " .. tostring(path)
                        .. "\nCONTEÚDO:\n"
                        .. tostring(content):sub(1, 500)
                )
            end
            return originalWritefile(path, content)
        end
        cap("HOOK FILE", "writefile interceptado.")
    else
        cap("HOOK FILE AVISO", "writefile não está disponível neste ambiente.")
    end
end)

-- ============================================================
-- 6. SALVAR CÓDIGO RECEBIDO PELO LOAD / LOADSTRING
-- ============================================================

local function saveLoaderSource(loaderName, source)
    if type(source) ~= "string" then return end

    local fileWriter = writefile
    if type(fileWriter) ~= "function" then
        cap(
            "LOAD SAVE AVISO",
            "writefile indisponível; o código foi registrado no painel, "
                .. "limitado à prévia apresentada."
        )
        return
    end

    LOADER_FILE_COUNT = LOADER_FILE_COUNT + 1
    local safeName = tostring(loaderName):gsub("[^%w_%-]", "_")
    local path = string.format("MonitorV4_%s_%03d.txt", safeName, LOADER_FILE_COUNT)

    local previousWriting = WRITING
    WRITING = true
    local ok, err = pcall(fileWriter, path, source)
    WRITING = previousWriting

    if ok then
        cap("LOAD SAVE", "Arquivo: " .. path .. "\nTamanho completo: " .. tostring(#source) .. " bytes")
    else
        cap("LOAD SAVE ERRO", "Arquivo: " .. path .. "\nErro: " .. tostring(err))
    end
end

-- ============================================================
-- 7. INTERCEPTAR LOADSTRING / LOAD (CAPTURA + PERMITE EXECUÇÃO)
-- ============================================================

local function hookLoader(env, envName, loaderName)
    local original = rawget(env, loaderName)
    if type(original) ~= "function" then return end

    if type(hookfunction) == "function" and LOADER_HOOKED[original] then
        return
    end

    local oldFunction = original

    local function wrapper(source, chunkname, ...)
        local sourceType = type(source)
        local sourceText = tostring(source)

        -- 1) CAPTURA (painel + arquivo)
        cap(
            envName .. "." .. loaderName,
            "TIPO: " .. sourceType
                .. "\nTAMANHO: " .. tostring(#sourceText) .. " bytes"
                .. "\nCHUNKNAME: " .. tostring(chunkname)
                .. "\nCÓDIGO RECEBIDO:\n"
                .. sourceText:sub(1, MAX_PREVIEW)
                .. (#sourceText > MAX_PREVIEW
                    and "\n[Prévia limitada; consulte o arquivo salvo.]"
                    or "")
        )

        if sourceType == "string" then
            saveLoaderSource(loaderName, source)
        end

        -- 2) PERMITE A EXECUÇÃO: chama o loadstring/load original
        local args = pack(chunkname, ...)
        local results = pack(pcall(oldFunction, source, unpackFn(args, 1, args.n)))

        if results[1] then
            return unpackFn(results, 2, results.n)
        end

        cap(
            envName .. "." .. loaderName .. " >> ERRO ORIGINAL",
            tostring(results[2])
        )
        error(results[2], 0)
    end

    if type(hookfunction) == "function" then
        local ok, result = pcall(hookfunction, original, wrapper)
        if ok then
            oldFunction = result or original
            LOADER_HOOKED[original] = true
            cap(
                "HOOK LOAD",
                envName .. "." .. loaderName
                    .. " interceptado via hookfunction (execução PERMITIDA)."
            )
            return
        end
        cap(
            "HOOK LOAD AVISO",
            "hookfunction falhou em " .. envName .. "." .. loaderName
                .. ": " .. tostring(result)
                .. "\nTentando substituir a referência."
        )
    end

    local ok, err = pcall(function() env[loaderName] = wrapper end)
    if ok then
        cap(
            "HOOK LOAD",
            envName .. "." .. loaderName
                .. " substituído nesse ambiente (execução PERMITIDA)."
        )
    else
        cap(
            "HOOK LOAD ERRO",
            envName .. "." .. loaderName .. ": " .. tostring(err)
        )
    end
end

for _, entry in ipairs(ENVS) do
    for _, loaderName in ipairs({ "loadstring", "load" }) do
        pcall(function()
            hookLoader(entry.env, entry.name, loaderName)
        end)
    end
end

-- ============================================================
-- 8. STATUS FINAL
-- ============================================================

cap(
    "STATUS",
    "Monitor v4 instalado (execução PERMITIDA)."
        .. "\nAguardando chamadas de rede e compiladores."
        .. "\nProcure por: request, HttpGet, loadstring, load."
        .. "\nO código capturado por load/loadstring é salvo e repassado."
        .. "\nA captura depende das APIs que o ambiente permite interceptar."
)
