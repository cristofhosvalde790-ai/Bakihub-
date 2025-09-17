-- AutomationBot.lua
local AutomationBot = {}
AutomationBot.__index = AutomationBot

local RunService = game:GetService("RunService")
local States = {
  Idle = "Idle",
  Attack = "Attack",
  AcceptQuest = "AcceptQuest",
  Teleport = "Teleport"
}

function AutomationBot.new(api)
  local self = setmetatable({}, AutomationBot)
  self.api = api                   -- API externa (retorna canAttack/attack, etc.)
  self.state = States.Idle
  self.nextActionTime = 0
  self.running = false
  return self
end

local function sleep(seconds)
  local t0 = os.clock()
  while os.clock() - t0 < seconds do
    coroutine.yield()
  end
end

function AutomationBot:canAct(now)
  return (not now) or (now >= self.nextActionTime)
end

function AutomationBot:tick(now)
  if not self:canAct(now) then
    return
  end

  local api = self.api

  if self.state == States.Idle then
    if api and api.canAcceptQuest and api:canAcceptQuest() then
      self.state = States.AcceptQuest
    elseif api and api.canAttack and api:canAttack() then
      self.state = States.Attack
    elseif api and api.canTeleport and api:canTeleport() then
      self.state = States.Teleport
    else
      local tNow = now or os.clock()
      self.nextActionTime = tNow + 0.2
    end

  elseif self.state == States.Attack then
    local ok, err = pcall(function() api:attack() end)
    if not ok and api.log then
      api:log("Erro em Attack: " .. tostring(err))
    end
    self.state = States.Idle
    self.nextActionTime = (now or os.clock()) + 0.5

  elseif self.state == States.AcceptQuest then
    local ok, err = pcall(function() api:acceptQuest() end)
    if not ok and api.log then
      api:log("Erro em AcceptQuest: " .. tostring(err))
    end
    self.state = States.Idle
    self.nextActionTime = (now or os.clock()) + 0.5

  elseif self.state == States.Teleport then
    local ok, err = pcall(function() api:teleport() end)
    if not ok and api.log then
      api:log("Erro em Teleport: " .. tostring(err))
    end
    self.state = States.Idle
    self.nextActionTime = (now or os.clock()) + 0.5
  end
end

function AutomationBot:Start()
  if self.running then return end
  self.running = true
  spawn(function()
    while self.running do
      local t = tick()
      self:tick(t)
      RunService.Heartbeat:Wait()
    end
  end)
end

function AutomationBot:Stop()
  self.running = false
end

return AutomationBot

-- RobloxBotUsageClient.lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local AutomationBotModule = require(ReplicatedStorage:WaitForChild("AutomationBot"))

local attackEvent = ReplicatedStorage:WaitForChild("BotAttack")       -- RemoteEvent
local acceptEvent = ReplicatedStorage:WaitForChild("BotAcceptQuest")  -- RemoteEvent
local teleportEvent = ReplicatedStorage:WaitForChild("BotTeleport")   -- RemoteEvent

-- API para Roblox (cliente)
local botApi = {}

function botApi:canAttack()
  local target = workspace:FindFirstChild("TrainingDummy")
  if not target then return false end
  local hrp = (game:GetService("Players").LocalPlayer.Character or nil).HumanoidRootPart
  if not hrp or not target.PrimaryPart then return false end
  local dist = (hrp.Position - target.PrimaryPart.Position).Magnitude
  return dist <= 25
end

function botApi:attack()
  local target = workspace:FindFirstChild("TrainingDummy")
  if target and attackEvent then
    attackEvent:FireServer(target.Name)
  end
  self:log("Ataque solicitado ao servidor")
end

function botApi:canAcceptQuest()
  local npc = workspace:FindFirstChild("QuestGiver")
  if not npc or not npc.PrimaryPart then return false end
  local hrp = game.Players.LocalPlayer.Character and game.Players.LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
  if not hrp then return false end
  local dist = (hrp.Position - npc.PrimaryPart.Position).Magnitude
  return dist <= 20
end

function botApi:acceptQuest()
  local npc = workspace:FindFirstChild("QuestGiver")
  if npc and acceptEvent then
    acceptEvent:FireServer(npc.Name)
  end
  self:log("Missão solicitada ao servidor")
end

function botApi:canTeleport()
  return true
end

function botApi:teleport()
  local dest = workspace:FindFirstChild("TeleportDest1") or workspace:FindFirstChild("SafeZone")
  if dest and teleportEvent then
    teleportEvent:FireServer(dest.Name)
  end
  self:log("Solicitação de teleport acionada")
end

function botApi:log(msg)
  print("[Bot] " .. tostring(msg))
end

-- Instancia o Bot
local bot = AutomationBotModule.new(botApi)
bot:Start()
