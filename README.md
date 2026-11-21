-- Troca todo texto com horario (hh:mm) por 10:20 e imprime o caminho completo de cada um.
-- NAO TESTADO. So muda a SUA tela (propriedade Text local). Use o caminho impresso para achar no Dex.

local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local StarterGui = game:GetService("StarterGui")
local CoreGui = game:GetService("CoreGui")
local playerGui = Players.LocalPlayer:WaitForChild("PlayerGui")

local NOVO = "10:20"
local PADRAO = "%d%d?:%d%d"   -- 10:13, 9:05... (troca o primeiro horario dentro do texto)
local DEBUG = true
local BIND = "UCAR_HoraTroca"

-- Reexecucao: desliga a instancia anterior
local genv = (getgenv and getgenv()) or _G
if genv.UCAR_HoraCleanup then pcall(genv.UCAR_HoraCleanup) end

local TEXTOS = { TextLabel = true, TextButton = true, TextBox = true }

local ativo = true
local escrevendo = false   -- true so enquanto ESTE script escreve (o hook deixa passar)
local alvos = {}           -- [objeto de texto] = true
local vistos = {}          -- copias do StarterGui ja listadas
local conexoes = {}

-- Hook: descarta escritas do JOGO em .Text dos alvos (senao o relogio volta pro horario real)
local wrapC = newcclosure or function(f) return f end
if type(hookmetamethod) == "function" and type(checkcaller) == "function" then
    local oldNewIndex
    oldNewIndex = hookmetamethod(game, "__newindex", wrapC(function(self, key, value)
        if ativo and not escrevendo and key == "Text" and alvos[self] and not checkcaller() then
            return
        end
        return oldNewIndex(self, key, value)
    end))
elseif DEBUG then
    warn("[Hora] hookmetamethod/checkcaller indisponivel: so a reaplicacao por frame sera usada")
end

local function examinar(d)
    if not TEXTOS[d.ClassName] or alvos[d] or vistos[d] then return end
    local ok, t = pcall(function() return d.Text end)
    if not ok or type(t) ~= "string" or not t:find(PADRAO) then return end

    if d:IsDescendantOf(StarterGui) then
        vistos[d] = true
        if DEBUG then
            print(string.format("[Hora] (copia no StarterGui, so referencia de caminho) %s | texto: %q", d:GetFullName(), t))
        end
        return
    end

    alvos[d] = true
    if DEBUG then
        print(string.format("[Hora] achei: %s | %s | texto: %q | Visible: %s",
            d:GetFullName(), d.ClassName, t, tostring(d.Visible)))
    end
end

local function varrer(raiz)
    local ok, lista = pcall(function() return raiz:GetDescendants() end)
    if not ok then return end
    for _, d in ipairs(lista) do examinar(d) end
end

-- Aplica a troca (as escritas do proprio script passam pelo hook por causa da flag)
local function aplicar(obj)
    local t = obj.Text
    local novo, n = t:gsub(PADRAO, NOVO, 1)
    if n > 0 and novo ~= t then
        escrevendo = true
        pcall(function() obj.Text = novo end)
        escrevendo = false
    end
end

pcall(function() RunService:UnbindFromRenderStep(BIND) end)
RunService:BindToRenderStep(BIND, Enum.RenderPriority.Last.Value, function()
    if not ativo then return end
    for obj in pairs(alvos) do
        if obj.Parent then
            aplicar(obj)
        else
            alvos[obj] = nil
        end
    end
end)

table.insert(conexoes, playerGui.DescendantAdded:Connect(examinar))

-- Se nada for achado: lista textos com ":" e numeros soltos de 2 digitos (relogio pode estar dividido em varios labels)
local function diagnostico()
    print("[Hora] nenhum texto no formato hh:mm. Textos com ':' ou so 2 digitos (ate 30):")
    local n = 0
    local ok, lista = pcall(function() return game:GetDescendants() end)
    if not ok then return end
    for _, d in ipairs(lista) do
        if TEXTOS[d.ClassName] and not d:IsDescendantOf(StarterGui) then
            local okT, t = pcall(function() return d.Text end)
            if okT and type(t) == "string" and (t:find(":", 1, true) or t:match("^%s*%d%d?%s*$")) then
                print(string.format("  %s | %q", d:GetFullName(), t))
                n = n + 1
                if n >= 30 then return end
            end
        end
    end
end

task.spawn(function()
    varrer(game) -- varredura inicial no jogo inteiro (PlayerGui, CoreGui, SurfaceGuis...)
    local ciclos = 0
    local diagnosticou = false
    while ativo do
        varrer(playerGui)
        pcall(function() varrer(CoreGui) end)
        ciclos = ciclos + 1
        if DEBUG and not diagnosticou and ciclos >= 2 and next(alvos) == nil then
            diagnosticou = true
            diagnostico()
        end
        task.wait(2)
    end
end)

genv.UCAR_HoraCleanup = function()
    ativo = false -- o hook fica instalado, mas so repassa quando ativo = false
    pcall(function() RunService:UnbindFromRenderStep(BIND) end)
    for _, c in ipairs(conexoes) do pcall(function() c:Disconnect() end) end
end
