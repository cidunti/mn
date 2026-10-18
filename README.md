-- ============================================================
-- LOADER MONITOR v3 - tudo em um (hooks + salvamento + scan)
-- ============================================================
print("[V3] Iniciando monitor...")

if not isfolder("MonitorCaptures") then makefolder("MonitorCaptures") end

local logs = {}
local counter = 0

local function log(t)
    t = tostring(t)
    table.insert(logs, t)
    print("[V3] " .. t)
end

local function save(tag, content)
    if not writefile then return end
    counter += 1
    local name = ("MonitorCaptures/capture_%02d_%s.lua"):format(counter, tag)
    pcall(writefile, name, tostring(content))
    log("SALVO: " .. name .. " (" .. #tostring(content) .. " bytes)")
    return name
end

-- ---------- HOOK: loadstring ----------
local oldLoad = loadstring
loadstring = function(src, chunk)
    log(">>> loadstring capturado! chunk=" .. tostring(chunk) .. " tamanho=" .. #tostring(src))
    save("loadstring", src)
    return oldLoad(src, chunk)
end

-- ---------- HOOK: getgenv (pra detectar configs do loader) ----------
local oldGetgenv = getgenv
if oldGetgenv then
    getgenv = function(...)
        return oldGetgenv(...)
    end
end

-- ---------- HOOK: game:HttpGet / HttpGetAsync ----------
local mt = getrawmetatable(game)
local oldIdx = mt.__index
setreadonly(mt, false)
mt.__index = function(t, k)
    if t == game and (k == "HttpGet" or k == "HttpGetAsync") then
        local real = oldIdx(t, k)
        return function(self, url, ...)
            log(k .. " -> " .. tostring(url))
            local ok, res = pcall(real, self, url, ...)
            if ok and type(res) == "string" then
                save("get", res)
                log("resposta (" .. #res .. " bytes) salva")
            end
            if not ok then error(res) end
            return res
        end
    end
    return oldIdx(t, k)
end
setreadonly(mt, true)

-- ---------- HOOK: http_request / request / syn.request ----------
local function hookReq(fn, name)
    return function(req)
        local url = type(req) == "table" and req.Url or req
        log(name .. " -> " .. tostring(url))
        local ok, res = pcall(fn, req)
        if ok then
            local body = type(res) == "table" and res.Body or res
            if type(body) == "string" then
                log("resposta: " .. body:sub(1, 400))
                save("req_" .. tostring(url):gsub("[^%w]", "_"):sub(-30), body)
            end
        end
        if not ok then error(res) end
        return res
    end
end

if http_request then http_request = hookReq(http_request, "http_request") end
if request then request = hookReq(request, "request") end
if syn and syn.request then syn.request = hookReq(syn.request, "syn.request") end

-- ---------- HOOK: require (alguns loaders usam asset require) ----------
local oldRequire = require
require = function(a)
    log("require -> " .. tostring(a))
    local ok, res = pcall(oldRequire, a)
    if not ok then error(res) end
    return res
end

log("TODOS os hooks ativos. Rode o loader do Jujutsu AGORA.")
log("Depois de completar a key e o menu abrir, rode o SCAN abaixo.")

-- ============================================================
-- SCAN: cole e rode esta parte DEPOIS que o menu abrir
-- (ou execute tudo junto e rode o scan por último)
-- ============================================================
task.delay(30, function() -- auto-scan após 30s de cada captura
end)

local function scanAll()
    log("======== SCAN INICIADO ========")
    if not isfolder("MonitorCaptures") then return end
    local keywords = {"jujutsu", "Jujutsu", "shenanigans", "luarmor", "Luarmor",
                      "LPH_", "Luraph", "luraph", "key", "whitelist", "hwid",
                      "DomainExpansion", "BlackFlash", "loadstring", "getgenv"}
    for _, file in listfiles("MonitorCaptures") do
        local name = tostring(file):match("[^/\\]+$")
        local ok, content = pcall(readfile, file)
        if ok and type(content) == "string" then
            log("--- " .. name .. " (" .. #content .. " bytes) ---")
            local found = {}
            for _, kw in ipairs(keywords) do
                local n = 0
                for _ in content:gmatch(kw) do n += 1 end
                if n > 0 then table.insert(found, kw .. "=" .. n) end
            end
            if #found > 0 then
                log("keywords: " .. table.concat(found, ", "))
            else
                log("nenhuma keyword relevante")
            end
            log("INÍCIO: " .. content:sub(1, 300):gsub("[\r\n]+", " "))
            log("FIM: " .. content:sub(-300):gsub("[\r\n]+", " "))
        end
    end
    log("======== SCAN FINALIZADO ========")
    log(">>> Agora copie TUDO do console e envie pro chat.")
    pcall(function()
        setclipboard(table.concat(logs, "\n"))
        log("logs completos copiados pro clipboard também")
    end)
end

-- registra comando global: rode scanAll() no console quando quiser
getgenv().scanAll = scanAll

log("Quando o menu abrir, execute: scanAll()")
