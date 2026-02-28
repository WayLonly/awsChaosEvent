-- Tree Auto Farm com Fluent (multiselect via toggles)

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
            -- monta lista de tipos selecionados pelos toggles
            local selectedTypes = {}
            if Options.ToxicTree.Value then table.insert(selectedTypes, "ToxicTree") end
            if Options.InfectedTree.Value then table.insert(selectedTypes, "InfectedTree") end
            if Options.PalmTree.Value then table.insert(selectedTypes, "PalmTree") end
            if Options.ChaosTree.Value then table.insert(selectedTypes, "ChaosTree") end

            local index = 1
            while Options.AutoTreeTP.Value do
                local trees = {}
                for _, tree in ipairs(workspace.GameObjects.ChaosBreakables.Axe:GetChildren()) do
                    if table.find(selectedTypes, tree.Name) and tree:GetAttribute("Destroyed") ~= true then
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

                    index = index + 1
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
            local selectedTypes = {}
            if Options.ToxicTree.Value then table.insert(selectedTypes, "ToxicTree") end
            if Options.InfectedTree.Value then table.insert(selectedTypes, "InfectedTree") end
            if Options.PalmTree.Value then table.insert(selectedTypes, "PalmTree") end
            if Options.ChaosTree.Value then table.insert(selectedTypes, "ChaosTree") end

            local trees = {}
            for _, tree in ipairs(workspace.GameObjects.ChaosBreakables.Axe:GetChildren()) do
                if table.find(selectedTypes, tree.Name) and tree:GetAttribute("Destroyed") ~= true then
                    table.insert(trees, tree)
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

-- Multiselect via toggles
Tabs.Main:AddToggle("ToxicTree", { Title = "Farm ToxicTree", Default = false })
Tabs.Main:AddToggle("InfectedTree", { Title = "Farm InfectedTree", Default = false })
Tabs.Main:AddToggle("PalmTree", { Title = "Farm PalmTree", Default = false })
Tabs.Main:AddToggle("ChaosTree", { Title = "Farm ChaosTree", Default = false })

-------------------------------------------------
-- START
-------------------------------------------------

Window:SelectTab(1)
Fluent:Notify({
    Title = "Hub Loaded",
    Content = "Tree Auto Farm Ready (Multiselect via toggles, ciclo com espera de destruição)",
    Duration = 5
})
