-- Loader Monitor v2 - salva capturas em arquivos
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "LoaderMonitorV2"
ScreenGui.ResetOnSpawn = false
pcall(function() ScreenGui.Parent = gethui and gethui() or game:GetService("CoreGui") end)

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, 420, 0, 220)
Frame.Position = UDim2.new(0, 20, 0, 120)
Frame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
Frame.Active = true
Frame.Draggable = true
Frame.Parent = ScreenGui

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 26)
Title.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
Title.Text = "Loader Monitor v2 (arraste)"
Title.TextColor3 = Color3.new(1, 1, 1)
Title.Parent = Frame

local LogBox = Instance.new("ScrollingFrame")
LogBox.Position = UDim2.new(0, 6, 0, 30)
LogBox.Size = UDim2.new(1, -12, 1, -70)
LogBox.BackgroundColor3 = Color3.fromRGB(10, 10, 14)
LogBox.Parent = Frame

local UIList = Instance.new("UIListLayout")
UIList.SortOrder = Enum.SortOrder.LayoutOrder
UIList.Parent = LogBox

local CopyBtn = Instance.new("TextButton")
CopyBtn.Size = UDim2.new(0, 130, 0, 28)
CopyBtn.Position = UDim2.new(0, 6, 1, -34)
CopyBtn.Text = "Copiar Tudo"
CopyBtn.BackgroundColor3 = Color3.fromRGB(60, 120, 60)
CopyBtn.TextColor3 = Color3.new(1, 1, 1)
CopyBtn.Parent = Frame

local logs = {}
local counter = 0

local function log(txt)
    txt = tostring(txt)
    table.insert(logs, txt)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -8, 0, 20)
    lbl.BackgroundTransparency = 1
    lbl.Text = txt:sub(1, 150)
    lbl.TextColor3 = Color3.fromRGB(0, 255, 120)
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Text = txt
    lbl.Parent = LogBox
end

local function saveCapture(tag, content)
    if not (writefile and isfolder) then return end
    counter += 1
    pcall(function()
        if not isfolder("MonitorCaptures") then makefolder("MonitorCaptures") end
        local path = ("MonitorCaptures/%02d_%s.lua"):format(counter, tag)
        writefile(path, tostring(content))
        log("SALVO: " .. path .. " (" .. #tostring(content) .. " bytes)")
    end)
end

CopyBtn.MouseButton1Click:Connect(function()
    pcall(setclipboard or toclipboard, table.concat(logs, "\n"))
    log(">>> Copiado!")
end)

-- Hook loadstring: captura o código REAL no momento da execução
local oldLoad = loadstring
loadstring = function(src, chunk)
    saveCapture("loadstring", src)
    log("loadstring capturado (" .. #tostring(src) .. " bytes) chunk=" .. tostring(chunk))
    return oldLoad(src, chunk)
end

-- Hook HttpGet genérico (request / syn.request / http_request)
local function hookReq(fn, name)
    return function(...)
        local args = {...}
        local url = type(args[1]) == "table" and args[1].Url or args[1]
        log(name .. " -> " .. tostring(url))
        local ok, res = pcall(fn, ...)
        if ok and type(res) == "table" and res.Body then
            saveCapture("http_" .. tostring(url):match("[%w%.%-]+$") or "resp", res.Body)
        elseif ok and type(res) == "string" then
            saveCapture("http_resp", res)
        end
        if not ok then error(res) end
        return res
    end
end

if http_request then http_request = hookReq(http_request, "http_request") end
if request then request = hookReq(request, "request") end
if syn and syn.request then syn.request = hookReq(syn.request, "syn.request") end

-- Hook game:HttpGet com resposta completa salva
local mt = getrawmetatable(game)
local oldIdx = mt.__index
setreadonly(mt, false)
mt.__index = function(t, k)
    if k == "HttpGet" and t == game then
        return function(self, url, cache)
            log("HttpGet -> " .. tostring(url))
            local ok, res = pcall(oldIdx(t, k), t, url, cache)
            if ok and type(res) == "string" then
                saveCapture("get", res)
            end
            if not ok then error(res) end
            return res
        end
    end
    return oldIdx(t, k)
end
setreadonly(mt, true)

log("Monitor v2 ativo. Rode o loader e complete a key.")
log("Arquivos salvos em: MonitorCaptures/")
