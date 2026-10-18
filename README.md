-- ============================================================
-- LOADER MONITOR v4 - COMPLETO: hooks + loader + scan automático
-- ============================================================
print("[V4] Iniciando monitor completo...")

-- limpa capturas antigas
if isfolder and isfolder("MonitorCaptures") then
    pcall(delfolder, "MonitorCaptures")
end
if isfolder then pcall(makefolder, "MonitorCaptures") end

local logs = {}
local counter = 0

local function log(t)
    t = tostring(t)
    table.insert(logs, t)
    print("[V4] " .. t)
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

-- ---------- HOOK: require ----------
local oldRequire = require
require = function(a)
    log("require -> " .. tostring(a))
    local ok, res = pcall(oldRequire, a)
    if not ok then error(res) end
    return res
end

log("TODOS os hooks ativos. Executando o loader do Jujutsu em 3 segundos...")

-- ---------- EXECUTA O LOADER ----------
task.delay(3, function()
    local ok, err = pcall(function()
        loadstring(game:HttpGet("https://raw.githubusercontent.com/tricksyfishys/jujutsuer/refs/heads/main/main.lua"))()
    end)
    if not ok then
        log("ERRO ao rodar o loader: " .. tostring(err))
    else
        log("loader executado. Complete a key e espere o menu abrir.")
        log("O auto-scan roda 60 segundos depois. Ou rode scanAll() manualmente.")
    end
end)

-- ---------- SCAN ----------
local function scanAll()
    log("======== SCAN INICIADO ========")
    if not isfolder("MonitorCaptures") then
        log("pasta MonitorCaptures não existe!")
        return
    end
    local keywords = {"jujutsu","Jujutsu","shenanigans","luarmor","Luarmor","Lurarmor",
                      "LPH_","Luraph","luraph","whitelist","hwid",
                      "DomainExpansion","BlackFlash","loadstring","getgenv","key"}
    for _, file in listfiles("MonitorCaptures") do
        local name = tostring(file):match("[^/\\]+$")
        local ok, content = pcall(readfile, file)
        if ok and type(content) == "string" then
            log("=== " .. name .. " (" .. #content .. " bytes) ===")
            local found = {}
            for _, kw in ipairs(keywords) do
                local n = 0
                for _ in content:gmatch(kw) do n += 1 end
                if n > 0 then table.insert(found, kw .. "=" .. n) end
            end
            log("keywords: " .. (#found > 0 and table.concat(found, ", ") or "nenhuma"))
            log("INÍCIO: " .. content:sub(1, 300):gsub("[\r\n]+", " "))
            log("FIM: " .. content:sub(-300):gsub("[\r\n]+", " "))
        end
    end
    log("======== SCAN FINALIZADO ========")
    pcall(function()
        setclipboard(table.concat(logs, "\n"))
        log(">>> TODOS os logs copiados pro clipboard! Cole aqui no chat.")
    end)
end

getgenv().scanAll = scanAll

-- auto-scan 60s depois da execução do loader
task.delay(63, function()
    log("AUTO-SCAN disparado (60s após o loader).")
    scanAll()
end)
