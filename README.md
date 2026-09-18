local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local StarterGui = game:GetService("StarterGui")

local TRACKER_VERSION = "1.0.10"
local API_URL = "https://steal-an-egg-trackstats.vercel.app/api/update"
local API_KEY = "BatmanSAE_9xK72pQ2026"
local SEND_INTERVAL = 15
local SEND_DATA = true

repeat task.wait() until game:IsLoaded()
local Player = Players.LocalPlayer or Players.PlayerAdded:Wait()

local Request =
    request
    or http_request
    or (http and http.request)
    or (syn and syn.request)
    or (fluxus and fluxus.request)

if not Request then
    warn("SEASHOP: executor does not support request/http_request")
    return
end

local Env = getgenv and getgenv() or _G
local Config = type(Env.SEASHOP_SAE_CONFIG) == "table" and Env.SEASHOP_SAE_CONFIG or {}
Env.SEASHOP_SAE_CONFIG = Config

API_URL = tostring(Config.API_URL or API_URL)
API_KEY = tostring(Config.API_KEY or API_KEY)
SEND_INTERVAL = math.clamp(tonumber(Config.SEND_INTERVAL) or SEND_INTERVAL, 5, 3600)
SEND_DATA = Config.SEND_DATA ~= false

Config.API_URL = API_URL
Config.API_KEY = API_KEY
Config.SEND_INTERVAL = SEND_INTERVAL
Config.SEND_DATA = SEND_DATA

if Env.SEASHOP_SAE_TRACKER_STOP then
    pcall(Env.SEASHOP_SAE_TRACKER_STOP)
end

local Running = true
local Session = HttpService:GenerateGUID(false)

Env.SEASHOP_SAE_TRACKER_STOP = function()
    Running = false
end

local Save
local Assets = {}
local AssetItems
local EggState
local Mutations
local Rarities = {}

pcall(function()
    Save = require(ReplicatedStorage.Shared.Save)
end)

pcall(function()
    Assets = require(ReplicatedStorage.Data.Assets).Directory
end)

pcall(function()
    AssetItems = require(ReplicatedStorage.Shared.Util.AssetItems)
end)

pcall(function()
    EggState = require(ReplicatedStorage.Client.EggState)
end)

pcall(function()
    Mutations = require(ReplicatedStorage.Shared.Modules.Mutations)
end)

pcall(function()
    Rarities = require(ReplicatedStorage.Data.Rarity).Rarities
end)

local RarityById = {}

for _, Rarity in pairs(type(Rarities) == "table" and Rarities or {}) do
    if type(Rarity) == "table" and Rarity._id then
        RarityById[tostring(Rarity._id)] = Rarity
    end
end

local function getAssetId(Value)
    if type(Value) == "number" then
        return math.floor(Value)
    end

    local Text = tostring(Value or "")
    return tonumber(
        Text:match("rbxassetid://(%d+)")
        or Text:match("rbxthumb://.-[?&]id=(%d+)")
        or Text:match("[?&]id=(%d+)")
        or Text:match("^(%d+)$")
    ) or 0
end

local function colorToHex(Color)
    if typeof(Color) ~= "Color3" then
        return ""
    end

    return string.format(
        "#%02X%02X%02X",
        math.floor(Color.R * 255 + 0.5),
        math.floor(Color.G * 255 + 0.5),
        math.floor(Color.B * 255 + 0.5)
    )
end

local function getRawIcon(Value)
    if Value == nil then
        return ""
    end

    local Success, Text = pcall(function()
        return tostring(Value)
    end)

    if not Success then
        return ""
    end

    Text = tostring(Text or "")
    if #Text > 500 then
        Text = Text:sub(1, 500)
    end

    return Text
end

local function getAssetInfo(Category)
    local Data = Assets[Category]

    if type(Data) ~= "table" then
        return {
            name = tostring(Category or "Unknown"),
            eggName = tostring(Category or "Unknown") .. " Egg",
            rarity = "Common",
            rarityNumber = 1,
            rarityColor = "",
            income = 0,
            icon = 0,
            eggIcon = 0,
            iconRaw = "",
            eggIconRaw = "",
            growthTime = 120
        }
    end

    local Rarity = Data.Rarity
    local RarityId = "Common"
    local RarityName = "Common"
    local RarityNumber = 1
    local RarityColor = ""

    if type(Rarity) == "table" then
        RarityId = tostring(Rarity._id or Rarity.DisplayName or "Common")
        RarityName = tostring(Rarity.DisplayName or RarityId)
        RarityNumber = tonumber(Rarity.RarityNumber) or 1
        RarityColor = colorToHex(Rarity.Color)
    elseif Rarity ~= nil then
        RarityId = tostring(Rarity)
        RarityName = RarityId
    end

    local RarityData = RarityById[RarityId]

    if type(RarityData) == "table" then
        RarityName = tostring(RarityData.DisplayName or RarityName)
        RarityNumber = tonumber(RarityData.RarityNumber) or RarityNumber

        if RarityColor == "" then
            RarityColor = colorToHex(RarityData.Color)
        end
    end

    local Egg = type(Data.Egg) == "table" and Data.Egg or {}

    return {
        name = tostring(Data.DisplayName or Category),
        eggName = tostring(Egg.DisplayName or ((Data.DisplayName or Category) .. " Egg")),
        rarity = RarityName,
        rarityNumber = RarityNumber,
        rarityColor = RarityColor,
        income = tonumber(Data.EarningRate) or 0,
        icon = getAssetId(Data.Icon),
        eggIcon = getAssetId(Egg.Icon),
        iconRaw = getRawIcon(Data.Icon),
        eggIconRaw = getRawIcon(Egg.Icon),
        growthTime = tonumber(Egg.GrowthTime) or 120
    }
end

local function getMutations(Value)
    local Result = {}
    local Seen = {}

    local function add(Id)
        if Id == nil then
            return
        end

        local Raw = tostring(Id)
        if Raw == "" or Seen[Raw] then
            return
        end

        if Mutations and Raw == tostring(Mutations.NO_MUTATION) then
            return
        end

        local Name = Raw

        if Mutations and type(Mutations.LabelOf) == "function" then
            pcall(function()
                Name = Mutations.LabelOf(Id) or Raw
            end)
        end

        Name = tostring(Name)

        if not Seen[Name] then
            Seen[Name] = true
            table.insert(Result, Name)
        end
    end

    if type(Value) == "table" then
        local Count = 0

        for Key, Entry in pairs(Value) do
            Count += 1
            if Count > 12 then
                break
            end

            if type(Key) == "number" then
                add(Entry)
            elseif Entry == true then
                add(Key)
            else
                add(Entry)
            end
        end
    else
        add(Value)
    end

    table.sort(Result)
    return Result
end

local function getEggWeight(Category, Scale, Record)
    local Weight = tonumber(Record and (Record.Weight or Record.WeightKg)) or 0

    if AssetItems and type(AssetItems.GetWeightKgForScale) == "function" then
        pcall(function()
            Weight = tonumber(AssetItems.GetWeightKgForScale(Category, tonumber(Scale))) or Weight
        end)
    end

    return Weight
end

local function getPetWeight(Record)
    local Weight = tonumber(Record and (Record.Weight or Record.WeightKg)) or 0

    if AssetItems
        and type(AssetItems.Decode) == "function"
        and type(AssetItems.WeightKg) == "function"
    then
        pcall(function()
            Weight = tonumber(AssetItems.WeightKg(AssetItems.Decode(Record))) or Weight
        end)
    end

    return Weight
end

local function getSaveData()
    if not Save or type(Save.Get) ~= "function" then
        return {}
    end

    local Success, Data = pcall(Save.Get)
    return Success and type(Data) == "table" and Data or {}
end

local function readPets(Data)
    local Pets = {}
    local Equipped = {}
    local TotalIncome = 0

    for _, Uid in ipairs(type(Data.EquippedAssets) == "table" and Data.EquippedAssets or {}) do
        Equipped[tostring(Uid)] = true
    end

    for Uid, Record in pairs(type(Data.Inventory) == "table" and Data.Inventory or {}) do
        if type(Record) == "table" and Record.Category then
            local Category = tostring(Record.Category)
            local Info = getAssetInfo(Category)
            local Income = tonumber(Record.Income or Record.EarningRate or Record.Rate) or Info.income

            TotalIncome += Income

            table.insert(Pets, {
                uid = tostring(Uid),
                category = Category,
                name = Info.name,
                eggName = Info.eggName,
                hatchName = Info.name,
                rarity = Info.rarity,
                rarityNumber = Info.rarityNumber,
                rarityColor = Info.rarityColor,
                weight = getPetWeight(Record),
                income = Income,
                baseIncome = Info.income,
                mutations = getMutations(Record.Mutations or Record.Mutation),
                iconAssetId = Info.icon,
                eggIconAssetId = Info.eggIcon,
                petIconAssetId = Info.icon,
                icon = Info.iconRaw,
                eggIcon = Info.eggIconRaw,
                petIcon = Info.iconRaw,
                equipped = Equipped[tostring(Uid)] == true,
                favorite = Record.IsFavorite == true,
                inFuse = Record.InFuse == true,
                source = "pet"
            })
        end
    end

    table.sort(Pets, function(A, B)
        if A.rarityNumber == B.rarityNumber then
            return A.income > B.income
        end

        return A.rarityNumber > B.rarityNumber
    end)

    return Pets, TotalIncome
end

local function readEggs(Data)
    local Eggs = {}

    for Uid, Record in pairs(type(Data.EggInventory) == "table" and Data.EggInventory or {}) do
        if type(Record) == "table" and Record.AssetCategory then
            local Category = tostring(Record.AssetCategory)
            local Info = getAssetInfo(Category)
            local Placed = type(Record.Placement) == "table"
            local GrowthTime = Info.growthTime
            local Speed = tonumber(Record.GrowthSpeedMultiplier) or 1

            if Speed > 0 then
                GrowthTime /= Speed
            end

            local Remaining = 0

            if Placed then
                local PlacedAt = tonumber(Record.Placement.PlacedAt)

                if PlacedAt then
                    Remaining = math.max(
                        0,
                        PlacedAt + GrowthTime - Workspace:GetServerTimeNow()
                    )
                end
            end

            table.insert(Eggs, {
                uid = tostring(Uid),
                category = Category,
                name = Info.eggName,
                eggName = Info.eggName,
                hatchName = Info.name,
                rarity = Info.rarity,
                rarityNumber = Info.rarityNumber,
                rarityColor = Info.rarityColor,
                weight = getEggWeight(Category, Record.AssetScale, Record),
                income = tonumber(Record.Income or Record.EarningRate) or Info.income,
                baseIncome = Info.income,
                mutations = getMutations(Record.Mutations or Record.Mutation),
                iconAssetId = Info.eggIcon,
                eggIconAssetId = Info.eggIcon,
                petIconAssetId = Info.icon,
                icon = Info.eggIconRaw,
                eggIcon = Info.eggIconRaw,
                petIcon = Info.iconRaw,
                placed = Placed,
                growthRemaining = Remaining,
                growthTotal = GrowthTime,
                area = tostring(Record.AreaId or Record.Area or ""),
                state = Placed and "Growing" or "Bag",
                source = "egg"
            })
        end
    end

    table.sort(Eggs, function(A, B)
        if A.placed ~= B.placed then
            return A.placed
        end

        if A.rarityNumber == B.rarityNumber then
            return A.weight > B.weight
        end

        return A.rarityNumber > B.rarityNumber
    end)

    return Eggs
end

local function readFieldEggs()
    local Eggs = {}

    if not EggState or type(EggState.ReadFieldEggs) ~= "function" then
        return Eggs
    end

    pcall(function()
        local State = EggState.ReadFieldEggs()

        for _, Record in ipairs(type(State) == "table" and State.Records or {}) do
            if type(Record) == "table" then
                local Category = Record.AssetCategory or Record.Category

                if Category then
                    Category = tostring(Category)

                    local Info = getAssetInfo(Category)

                    table.insert(Eggs, {
                        uid = tostring(Record.Uid or Record.UID or ""),
                        category = Category,
                        name = Info.eggName,
                        eggName = Info.eggName,
                        hatchName = Info.name,
                        rarity = tostring(Record.Rarity or Info.rarity),
                        rarityNumber = tonumber(Record.Rank or Record.RarityRank) or Info.rarityNumber,
                        rarityColor = Info.rarityColor,
                        weight = getEggWeight(Category, Record.AssetScale or Record.Scale, Record),
                        income = tonumber(Record.Income or Record.EarningRate) or Info.income,
                        baseIncome = Info.income,
                        mutations = getMutations(Record.Mutations or Record.Mutation),
                        iconAssetId = Info.eggIcon,
                        eggIconAssetId = Info.eggIcon,
                        petIconAssetId = Info.icon,
                        icon = Info.eggIconRaw,
                        eggIcon = Info.eggIconRaw,
                        petIcon = Info.iconRaw,
                        area = tostring(Record.AreaId or Record.Area or ""),
                        state = tostring(Record.State or "Slot"),
                        carrierUserId = tonumber(Record.CarrierUserId or Record.Carrier) or 0,
                        source = "field"
                    })
                end
            end
        end
    end)

    return Eggs
end

local SeenInventory
local RecentField = {}
local StolenHistory = {}

local function updateStolenHistory(Eggs, FieldEggs)
    local Now = os.time() * 1000
    local Current = {}

    for _, Egg in ipairs(FieldEggs) do
        if Egg.uid ~= "" then
            RecentField[Egg.uid] = {
                time = os.clock(),
                data = Egg
            }
        end
    end

    for Uid, Entry in pairs(RecentField) do
        if os.clock() - Entry.time > 180 then
            RecentField[Uid] = nil
        end
    end

    for _, Egg in ipairs(Eggs) do
        Current[Egg.uid] = true

        if SeenInventory and not SeenInventory[Egg.uid] then
            local History = table.clone(Egg)
            local Previous = RecentField[Egg.uid]

            History.timestamp = Now
            History.source = Previous and "field" or "inventory"

            if Previous and History.area == "" then
                History.area = Previous.data.area
            end

            table.insert(StolenHistory, 1, History)

            while #StolenHistory > 100 do
                table.remove(StolenHistory)
            end
        end
    end

    SeenInventory = Current
end

local function buildSnapshot()
    local Data = getSaveData()
    local Pets, PetIncome = readPets(Data)
    local Eggs = readEggs(Data)
    local FieldEggs = readFieldEggs()

    updateStolenHistory(Eggs, FieldEggs)

    local DirectIncome = tonumber(
        Data.MoneyPerSecond
        or Data.IncomePerSecond
        or Data.EarningsPerSecond
        or Data.EarningRate
    )

    local Humanoid = Player.Character and Player.Character:FindFirstChildOfClass("Humanoid")

    local BossTokens =
        tonumber(Data.BossTokens)
        or (type(Data.Currencies) == "table" and tonumber(Data.Currencies.BossTokens))
        or (type(Data.Currency) == "table" and tonumber(Data.Currency.BossTokens))
        or 0

    return {
        username = Player.Name,
        displayName = Player.DisplayName,
        userId = Player.UserId,
        placeId = game.PlaceId,
        jobId = game.JobId,
        gameName = "Steal An Egg",
        placeName = "Steal An Egg",
        session = Session,

        stats = {
            money = tonumber(Data.Money) or 0,
            moneyPerSecond = DirectIncome or PetIncome,
            incomeSource = DirectIncome and "save" or "pet-base-sum",
            speedPower = tonumber(Data.SpeedPower) or 0,
            walkSpeed = Humanoid and Humanoid.WalkSpeed or 0,
            bossTokens = BossTokens,
            treadmillLevel = tonumber(Data.TreadmillUpgradeLevel) or 0,
            eggCount = #Eggs,
            petCount = #Pets
        },

        pets = Pets,
        eggs = Eggs,
        fieldEggs = FieldEggs,
        stolenEggs = StolenHistory,
        catalogVersion = "runtime",
        ts = os.time() * 1000
    }
end

local function sendSnapshot()
    if not SEND_DATA then
        return false
    end

    local Payload = buildSnapshot()

    local Success, Response = pcall(function()
        return Request({
            Url = API_URL,
            Method = "POST",
            Headers = {
                ["Content-Type"] = "application/json",
                ["X-API-Key"] = API_KEY
            },
            Body = HttpService:JSONEncode(Payload),
            Timeout = 15
        })
    end)

    if not Success then
        warn("SEASHOP: request failed:", Response)
        return false
    end

    local StatusCode = tonumber(Response.StatusCode or Response.Status) or 0

    if StatusCode >= 200 and StatusCode < 300 then
        print(
            ("SEASHOP: synced | pets %d | eggs %d | field %d")
            :format(#Payload.pets, #Payload.eggs, #Payload.fieldEggs)
        )
        return true
    end

    local Body = tostring(Response.Body or "")
    warn("SEASHOP: web error:", StatusCode, Body)

    if StatusCode == 503 and Body:find("NOPERM", 1, true) then
        pcall(function()
            StarterGui:SetCore("SendNotification", {
                Title = "SEASHOP Steal An Egg",
                Text = "Redis write denied. Replace UPSTASH_REDIS_REST_TOKEN in Vercel, then redeploy.",
                Duration = 7
            })
        end)
    end

    return false
end

local Connected = sendSnapshot()

pcall(function()
    StarterGui:SetCore("SendNotification", {
        Title = "SEASHOP Steal An Egg",
        Text = Connected and "Web sync connected" or "Web sync failed - check console",
        Duration = 5
    })
end)

task.spawn(function()
    while Running and Player.Parent do
        task.wait(SEND_INTERVAL)

        if Running and SEND_DATA then
            sendSnapshot()
        end
    end
end)

print("SEASHOP: Steal An Egg Tracker v" .. TRACKER_VERSION .. " loaded")
return Env.SEASHOP_SAE_TRACKER_STOP
