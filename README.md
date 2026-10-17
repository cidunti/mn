-- ============================================================
-- MONITOR V4 (Xeno compatible) + captura Luraph multi-estágio
-- Menu do v4 + hook de loadstring/load escondido do Luraph
-- ============================================================

local CAPTURED = {}
local N = 0
local WRITING = false
local FILE_COUNT = 0
local MAX_PREVIEW = 2000
local LAST_FULL = nil
local HOOKED = {}

-- ===== 1. GUI (PlayerGui para Xeno) =====
local parent
do
    if type(gethui) == "function" then
        local ok, r = pcall(gethui)
        if ok and r then parent = r end
    end
    if not parent then
        local ok, r = pcall(function()
            return game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui", 5)
        end)
        if ok and r then parent = r end
    end
    if not parent then
        local ok, r = pcall(function()
            return game:GetService("CoreGui")
        end)
        if ok and r then parent = r end
    end
end

if not parent then
    warn("[Monitor] sem parent de GUI")
    return
end

local gui = Instance.new("ScreenGui")
gui.Name = "MonitorV4"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.IgnoreGuiInset = true
gui.Parent = parent

local frm = Instance.new("Frame")
frm.Size = UDim2.new(0, 480, 0, 360)
frm.Position = UDim2.new(0.5, -240, 0.5, -180)
frm.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
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
title.Size = UDim2.new(1, -130, 1, 0)
title.Position = UDim2.new(0, 10, 0, 0)
title.BackgroundTransparency = 1
title.Text = "Monitor v4 | 0"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 12
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = bar

local btnCopy = Instance.new("TextButton")
btnCopy.Size = UDim2.new(0, 90, 0, 22)
btnCopy.Position = UDim2.new(1, -118, 0, 5)
btnCopy.BackgroundColor3 = Color3.fromRGB(60, 130, 210)
btnCopy.Text = "Copiar"
btnCopy.TextColor3 = Color3.new(1, 1, 1)
btnCopy.Font = Enum.Font.GothamBold
btnCopy.TextSize = 11
btnCopy.BorderSizePixel = 0
btnCopy.Parent = bar
Instance.new("UICorner", btnCopy).CornerRadius = UDim.new(0, 4)

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
scroll.Size = UDim2.new(1, -12, 1, -44)
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

-- ===== 2. helpers =====
local function refresh()
    txt.Text = (#CAPTURED > 0) and table.concat(CAPTURED, "\n\n") or "(aguardando...)"
    title.Text = "Monitor v4 | " .. N
end

local function cap(tag, body)
    N = N + 1
    table.insert(CAPTURED, "[#" .. N .. "] " .. tag .. "\n" .. tostring(body))
    pcall(refresh)
    pcall(print, "[#" .. N .. "] " .. tag)
end

btnCopy.MouseButton1Click:Connect(function()
    local target = LAST_FULL or table.concat(CAPTURED, "\n\n")
    local ok = false
    if type(setclipboard) == "function" then
        ok = pcall(setclipboard, target)
    elseif type(toclipboard) == "function" then
        ok = pcall(toclipboard, target)
    end
    btnCopy.Text = ok and "OK!" or "N/D"
    task.delay(1.5, function()
        if btnCopy.Parent then btnCopy.Text = "Copiar" end
    end)
end)

btnClose.MouseButton1Click:Connect(function()
    pcall(function() gui:Destroy() end)
end)

-- ===== 3. HTTP hooks (v4) =====
local function describeRequest(opts)
    local url, body = "?", ""
    if type(opts) == "string" then
        url = opts
    elseif type(opts) == "table" then
        pcall(function() url = opts.Url or opts.url or opts.URL or opts[1] or "?" end)
        pcall(function() body = opts.Body or opts.body or "" end)
        if body == "" then
            pcall(function() body = game:GetService("HttpService"):JSONEncode(opts) end)
        end
    end
    return tostring(url), tostring(body)
end

local function describeResponse(result)
    if type(result) == "table" then
        local b
        pcall(function() b = result.Body or result.body end)
        if b ~= nil then return tostring(b) end
        local ok, enc = pcall(function()
            return game:GetService("HttpService"):JSONEncode(result)
        end)
        if ok then return enc end
    end
    return tostring(result)
end

local function hookRequest(env, envName, fname)
    local orig = rawget(env, fname)
    if type(orig) ~= "function" then return end
    if HOOKED[orig] then return end
    HOOKED[orig] = true

    local function wrapper(...)
        local url, body = describeRequest(select(1, ...))
        cap(envName .. "." .. fname, "URL: " .. url .. "\nBODY: " .. body:sub(1, 800))

        local ok, result = pcall(orig, ...)
        if ok then
            cap(envName .. "." .. fname .. " >> RESP", describeResponse(result):sub(1, 2000))
            return result
        end
        cap(envName .. "." .. fname .. " >> ERRO", tostring(result))
        error(result, 0)
    end

    if type(hookfunction) == "function" then
        local ok = pcall(hookfunction, orig, wrapper)
        if ok then return end
    end
    pcall(function() env[fname] = wrapper end)
end

local ENVS, SEEN = {}, {}
local function addEnv(e, n)
    if type(e) == "table" and not SEEN[e] then
        SEEN[e] = true
        ENVS[#ENVS + 1] = { env = e, name = n }
    end
end

addEnv(_G, "_G")
pcall(function() if type(getfenv) == "function" then addEnv(getfenv(), "getfenv") end end)
pcall(function() if type(getgenv) == "function" then addEnv(getgenv(), "getgenv") end end)

for _, e in ipairs(ENVS) do
    for _, fname in ipairs({"request", "http_request", "syn_request", "httpRequest"}) do
        pcall(hookRequest, e.env, e.name, fname)
    end
    for _, tname in ipairs({"syn", "http"}) do
        local t = rawget(e.env, tname)
        if type(t) == "table" then
            for _, fname in ipairs({"request", "http_request"}) do
                pcall(hookRequest, t, e.name .. "." .. tname, fname)
            end
        end
    end
end

if type(hookfunction) == "function" then
    for _, m in ipairs({"HttpGet", "HttpGetAsync"}) do
        pcall(function()
            local orig = game[m]
            if type(orig) ~= "function" or HOOKED[orig] then return end
            HOOKED[orig] = true
            hookfunction(orig, function(self, url, ...)
                cap("game." .. m, "URL: " .. tostring(url))
                local ok, res = pcall(orig, self, url, ...)
                if ok then
                    cap("game." .. m .. " >> RESP", tostring(res):sub(1, 2000))
                    return res
                end
                cap("game." .. m .. " >> ERRO", tostring(res))
                error(res, 0)
            end)
        end)
    end
end

-- ===== 4. writefile =====
if type(writefile) == "function" then
    local orig = writefile
    writefile = function(path, content)
        if not WRITING then
            cap("writefile", "PATH: " .. tostring(path) .. "\nCONTEUDO: " .. tostring(content):sub(1, 500))
        end
        return orig(path, content)
    end
end

-- ===== 5. loadstring / load multi-estágio (Luraph) =====
local function saveStage(name, src)
    if type(src) ~= "string" or #src == 0 then return nil end
    if type(writefile) ~= "function" then return nil end
    FILE_COUNT = FILE_COUNT + 1
    local safe = tostring(name):gsub("[^%w_%-]", "_")
    local path = string.format("MonitorV4_%s_%02d_%d.txt", safe, FILE_COUNT, os.time())
    local prev = WRITING
    WRITING = true
    local ok = pcall(writefile, path, src)
    WRITING = prev
    return ok and path or nil
end

local LOADER_HOOKED = {}

local function hookLoader(env, envName, fname)
    local orig = rawget(env, fname)
    if type(orig) ~= "function" then return end
    if LOADER_HOOKED[orig] then return end
    LOADER_HOOKED[orig] = true

    local function wrapper(source, ...)
        if type(source) == "string" and #source > 0 then
            LAST_FULL = source
            local path = saveStage(fname, source)
            cap(
                envName .. "." .. fname,
                "TAMANHO: " .. #source .. " bytes"
                .. (path and ("\nSALVO: " .. path) or "\nSAVE FALHOU (writefile off)")
                .. "\nPRÉVIA:\n" .. source:sub(1, MAX_PREVIEW)
            )
        end
        return orig(source, ...)
    end

    if type(hookfunction) == "function" then
        local ok = pcall(hookfunction, orig, wrapper)
        if ok then
            cap("HOOK", envName .. "." .. fname .. " via hookfunction")
            return
        end
    end
    local ok = pcall(function() env[fname] = wrapper end)
    cap("HOOK", envName .. "." .. fname .. (ok and " via substituição" or " FALHOU"))
end

for _, e in ipairs(ENVS) do
    hookLoader(e.env, e.name, "loadstring")
    hookLoader(e.env, e.name, "load")
end

-- ===== 6. esconde o hook do Luraph =====
if type(debug) == "table" and type(debug.getinfo) == "function"
    and type(hookfunction) == "function" then
    pcall(function()
        local realGetInfo = debug.getinfo
        hookfunction(realGetInfo, function(level, what)
            local info = realGetInfo(level, what)
            if type(info) == "table" and info.func == loadstring then
                info.what = "C"
                info.source = "=[C]"
                info.short_src = "[C]"
            end
            return info
        end)
        cap("HIDE", "debug.getinfo hookado — Luraph não detecta loadstring")
    end)
end

cap("STATUS", "Pronto. Execute o script alvo agora.")
refresh()
