-- Tree Auto Farm com Fluent (PalmTree especial + outros com Destroyed/timeout)

-------------------------------------------------
-- LOAD FLUENT
-------------------------------------------------

local Fluent = loadstring(game:HttpGet(
"https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"
))()
local Options = Fluent.Options

-------------------------------------------------
-- WINDOW
-------------------------------------------------

local Window = Fluent:CreateWindow({
    Title = "Tree Farm",
    SubTitle = "TP + Auto Attack",
    TabWidth = 160,
    Size = UDim2.fromOffset(580,460),
    Acrylic = true,
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.LeftControl
})

local Tabs = {
    Main = Window:AddTab({ Title = "Main", Icon = "" })
}

-------------------------------------------------
-- SERVICES
-------------------------------------------------

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local player = Players.LocalPlayer

local clickRemote = ReplicatedStorage
    :WaitForChild("Packages")
    :WaitForChild("Knit")
    :WaitForChild("Services")
    :WaitForChild("ChaosBreakablesService")
    :WaitForChild("RF")
    :WaitForChild("Clicked")

-------------------------------------------------
-- FUNÇÕES
-------------------------------------------------

local function disableTreeCollision(tree)
    for _, part in ipairs(tree:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = false
        end
    end
end

local function teleportToTree(tree)
    local character = player.Character
    if not character then return end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then return end

    root.CanCollide = false
    disableTreeCollision(tree)

    local targetCFrame = nil
    local model = tree:FindFirstChild("Model")
    if model then
        targetCFrame = model:GetPivot()
    elseif tree.PrimaryPart then
        targetCFrame = tree.PrimaryPart.CFrame
    else
        local bounds = tree:FindFirstChild("XZBounds", true)
        if bounds then
            targetCFrame = bounds.CFrame
        end
    end

    if targetCFrame then
        character:PivotTo(targetCFrame * CFrame.new(0, 1, -1))
    end
end

-- pega árvore mais próxima, ignorando destruídas e a atual
local function getNearestSelectedTree(ignoreTree)
    local character = player.Character
    if not character then return nil end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then return nil end

    local selectedType = Options.TargetTree.Value
    if not selectedType then return nil end

    local closestTree, closestDist = nil, math.huge
    for _, tree in ipairs(workspace.GameObjects.ChaosBreakables.Axe:GetChildren()) do
        if tree.Name == selectedType and tree ~= ignoreTree and tree:GetAttribute("Destroyed") ~= true then
            local pos
            local model = tree:FindFirstChild("Model")
            if model then
                pos = model:GetPivot().Position
            elseif tree.PrimaryPart then
                pos = tree.PrimaryPart.Position
            else
                local bounds = tree:FindFirstChild("XZBounds", true)
                if bounds then
                    pos = bounds.Position
                end
            end
            if pos then
                local dist = (root.Position - pos).Magnitude
                if dist < closestDist then
                    closestDist = dist
                    closestTree = tree
                end
            end
        end
    end
    return closestTree
end

-------------------------------------------------
-- LOOPS
-------------------------------------------------

local function autoTPSelectedTrees()
    task.spawn(function()
        while Options.AutoTreeTP.Value do
            local selectedType = Options.TargetTree.Value
            if selectedType == "PalmTree" then
                -- ciclo especial para PalmTree
                local index = 1
                while Options.AutoTreeTP.Value and Options.TargetTree.Value == "PalmTree" do
                    local trees = {}
                    for _, tree in ipairs(workspace.GameObjects.ChaosBreakables.Axe:GetChildren()) do
                        if tree.Name == "PalmTree" then
                            table.insert(trees, tree)
                        end
                    end
                    if #trees > 0 then
                        if index > #trees then
                            index = 1
                        end
                        teleportToTree(trees[index])
                        index = index + 1
                    end
                    task.wait(0.3)
                end
            else
                -- comportamento para Toxic, Infected, Chaos
                local currentTree = getNearestSelectedTree(nil)
                if currentTree then
                    teleportToTree(currentTree)
                    local lastTP = tick()
                    while Options.AutoTreeTP.Value and Options.TargetTree.Value == selectedType do
                        task.wait(0.5)
                        -- troca imediata se destruída
                        if currentTree:GetAttribute("Destroyed") == true then
                            local newTree = getNearestSelectedTree(currentTree)
                            if newTree then
                                currentTree = newTree
                                teleportToTree(currentTree)
                            end
                            lastTP = tick()
                        end
                        -- força troca se já passaram 10s
                        if tick() - lastTP >= 10 then
                            local newTree = getNearestSelectedTree(currentTree)
                            if newTree then
                                currentTree = newTree
                                teleportToTree(currentTree)
                            end
                            lastTP = tick()
                        end
                    end
                else
                    task.wait(1)
                end
            end
        end
    end)
end

local function autoAttackSelectedTrees()
    task.spawn(function()
        while Options.AutoAttack.Value do
            local tree = getNearestSelectedTree(nil)
            if tree then
                clickRemote:InvokeServer()
            end
            task.wait(0.1)
        end
    end)
end

-------------------------------------------------
-- TOGGLES FLUENT
-------------------------------------------------

Tabs.Main:AddToggle("AutoTreeTP", {
    Title = "Auto TP",
    Default = false
})

Options.AutoTreeTP:OnChanged(function(state)
    if state then
        autoTPSelectedTrees()
    end
end)

Tabs.Main:AddToggle("AutoAttack", {
    Title = "Auto Attack",
    Default = false
})

Options.AutoAttack:OnChanged(function(state)
    if state then
        autoAttackSelectedTrees()
    end
end)

Tabs.Main:AddDropdown("TargetTree", {
    Title = "Selecionar Tipo de Árvore",
    Values = {"ToxicTree", "InfectedTree", "PalmTree", "ChaosTree"},
    Default = "ToxicTree"
})

-------------------------------------------------
-- START
-------------------------------------------------

Window:SelectTab(1)
Fluent:Notify({
    Title = "Hub Loaded",
    Content = "Tree Auto Farm Ready (PalmTree ciclo 0.3s, outros ignoram Destroyed e trocam após 10s)",
    Duration = 5
})
