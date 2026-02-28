-- Tree Auto Farm com Fluent (PalmTree especial + outros com Destroyed/timeout + opção "Todos" cíclica com espera)

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

            elseif selectedType == "Todos" then
                -- ciclo por todas as árvores ativas, esperando cada uma ser destruída
                local index = 1
                while Options.AutoTreeTP.Value and Options.TargetTree.Value == "Todos" do
                    local trees = {}
                    for _, tree in ipairs(workspace.GameObjects.ChaosBreakables.Axe:GetChildren()) do
                        if tree:GetAttribute("Destroyed") ~= true then
                            table.insert(trees, tree)
                        end
                    end
                    if #trees > 0 then
                        if index > #trees then
                            index = 1
                        end
                        local currentTree = trees[index]
                        teleportToTree(currentTree)

                        -- espera até ser destruída ou timeout de 10s
                        local startTime = tick()
                        repeat
                            task.wait(0.5)
                        until currentTree:GetAttribute("Destroyed") == true 
                            or tick() - startTime >= 10 
                            or not Options.AutoTreeTP.Value 
                            or Options.TargetTree.Value ~= "Todos"

                        index = index + 1
                    else
                        task.wait(1)
                    end
                end

            else
                -- comportamento para Toxic, Infected, Chaos
                local currentTree = nil
                while Options.AutoTreeTP.Value and Options.TargetTree.Value == selectedType do
                    if not currentTree or currentTree:GetAttribute("Destroyed") == true then
                        local trees = {}
                        for _, tree in ipairs(workspace.GameObjects.ChaosBreakables.Axe:GetChildren()) do
                            if tree.Name == selectedType and tree:GetAttribute("Destroyed") ~= true then
                                table.insert(trees, tree)
                            end
                        end
                        if #trees > 0 then
                            currentTree = trees[1]
                            teleportToTree(currentTree)
                        end
                    end
                    task.wait(0.5)
                end
            end
        end
    end)
end

local function autoAttackSelectedTrees()
    task.spawn(function()
        while Options.AutoAttack.Value do
            local selectedType = Options.TargetTree.Value
            local trees = {}
            for _, tree in ipairs(workspace.GameObjects.ChaosBreakables.Axe:GetChildren()) do
                if selectedType == "Todos" or tree.Name == selectedType then
                    if tree:GetAttribute("Destroyed") ~= true then
                        table.insert(trees, tree)
                    end
                end
            end
            if #trees > 0 then
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
    Values = {"ToxicTree", "InfectedTree", "PalmTree", "ChaosTree", "Todos"},
    Default = "ToxicTree"
})

-------------------------------------------------
-- START
-------------------------------------------------

Window:SelectTab(1)
Fluent:Notify({
    Title = "Hub Loaded",
    Content = "Tree Auto Farm Ready (PalmTree ciclo 0.3s, outros com Destroyed/timeout, Todos espera destruir cada árvore)",
    Duration = 5
})
