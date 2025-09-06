-- PhuLib v1.0 (Full) - Rayfield-style UI library by Phú
-- API: Win, Tab, Btn, Tog, Sld, Box, Drop, Lbl
-- Lightweight, mobile-friendly, draggable, minimize, tab selector.

local PhuLib = {}
PhuLib.__index = PhuLib

-- helpers
local function new(class, props)
    local obj = Instance.new(class)
    if props then
        for k,v in pairs(props) do
            pcall(function() obj[k] = v end)
        end
    end
    return obj
end

local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")

-- safe ScreenGui parent (roblox updates: CoreGui might require check)
local function getScreenParent()
    local parent = CoreGui
    if not parent then parent = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui") end
    return parent
end

-- Basic style
local STYLE = {
    WindowSize = UDim2.new(0, 520, 0, 380),
    BG = Color3.fromRGB(26,26,26),
    Header = Color3.fromRGB(38,38,38),
    Accent = Color3.fromRGB(0,170,255),
    Btn = Color3.fromRGB(52,52,52),
    Text = Color3.fromRGB(240,240,240),
    Corner = UDim.new(0,10),
    Font = Enum.Font.GothamBold,
    SmallFont = Enum.Font.Gotham
}

-- Draggable util (works with touch)
local function makeDraggable(frame, handle)
    local dragging = false
    local dragStart = nil
    local startPos = nil
    local inputType = nil

    handle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            inputType = input.UserInputType
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    handle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == (inputType or Enum.UserInputType.MouseMovement) and dragStart then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- simple tween util
local function tween(obj, props, time, style, dir)
    pcall(function()
        local ts = game:GetService("TweenService")
        local info = TweenInfo.new(time or 0.25, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out)
        ts:Create(obj, info, props):Play()
    end)
end

-- create drop shadow (ImageLabel with slice)
local function createShadow(parent)
    local img = new("ImageLabel", {
        Parent = parent,
        Size = UDim2.new(1, 60, 1, 60),
        Position = UDim2.new(0, -30, 0, -30),
        BackgroundTransparency = 1,
        Image = "rbxassetid://5028857084",
        ScaleType = Enum.ScaleType.Slice,
        SliceCenter = Rect.new(24,24,276,276),
        ImageColor3 = Color3.new(0,0,0),
        ImageTransparency = 0.6,
        ZIndex = 0
    })
    return img
end

-- create base ScreenGui
local function createScreen(name)
    local parent = getScreenParent()
    local sg = parent:FindFirstChild(name)
    if sg then sg:Destroy() end
    sg = new("ScreenGui", {Name = name, ZIndexBehavior = Enum.ZIndexBehavior.Sibling, ResetOnSpawn = false})
    sg.Parent = parent
    return sg
end

-- Window class
local Window = {}
Window.__index = Window

function PhuLib:Win(title, opts)
    opts = opts or {}
    local name = "PhuLib_UI"
    local sg = createScreen(name)

    local container = new("Frame", {
        Parent = sg,
        Size = opts.Size or STYLE.WindowSize,
        Position = opts.Position or UDim2.new(0.25,0,0.18,0),
        BackgroundColor3 = STYLE.BG,
        BorderSizePixel = 0,
        AnchorPoint = Vector2.new(0,0),
        Name = "PhuLib_Window"
    })
    new("UICorner", {Parent = container, CornerRadius = STYLE.Corner})
    createShadow(container)

    -- header
    local header = new("Frame", {Parent = container, Size = UDim2.new(1,0,0,38), BackgroundColor3 = STYLE.Header, BorderSizePixel = 0})
    new("UICorner", {Parent = header, CornerRadius = STYLE.Corner})
    header.ZIndex = 5

    local titleLbl = new("TextLabel", {
        Parent = header,
        Text = title or "PhuLib",
        Size = UDim2.new(0.7, -10, 1, 0),
        Position = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        TextColor3 = STYLE.Text,
        Font = STYLE.Font,
        TextSize = 18,
        TextXAlignment = Enum.TextXAlignment.Left
    })

    -- controls
    local btnMin = new("TextButton", {Parent = header, Size = UDim2.new(0,26,0,26), Position = UDim2.new(1,-62,0.5,-13), Text = "—", BackgroundColor3 = Color3.fromRGB(55,55,55), TextColor3 = STYLE.Text, Font = STYLE.Font, TextSize = 18, BorderSizePixel = 0})
    new("UICorner", {Parent = btnMin, CornerRadius = UDim.new(0,6)})

    local btnClose = new("TextButton", {Parent = header, Size = UDim2.new(0,26,0,26), Position = UDim2.new(1,-30,0.5,-13), Text = "✕", BackgroundColor3 = Color3.fromRGB(55,55,55), TextColor3 = STYLE.Text, Font = STYLE.Font, TextSize = 16, BorderSizePixel = 0})
    new("UICorner", {Parent = btnClose, CornerRadius = UDim.new(0,6)})

    -- body: left tablist, right content
    local left = new("Frame", {Parent = container, Size = UDim2.new(0,140,1,-50), Position = UDim2.new(0,0,0,38), BackgroundTransparency = 1})
    local right = new("Frame", {Parent = container, Size = UDim2.new(1,-150,1,-50), Position = UDim2.new(0,150,0,38), BackgroundTransparency = 1})

    -- tab list
    local tabList = new("ScrollingFrame", {Parent = left, Size = UDim2.new(1,-10,1,0), Position = UDim2.new(0,5,0,0), BackgroundTransparency = 1, ScrollBarImageColor3 = Color3.fromRGB(70,70,70)})
    tabList.CanvasSize = UDim2.new(0,0,0,0)
    tabList.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local tabListLayout = new("UIListLayout", {Parent = tabList, SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0,6)})

    -- content area
    local content = new("ScrollingFrame", {Parent = right, Size = UDim2.new(1,-10,1,0), Position = UDim2.new(0,5,0,0), BackgroundTransparency = 1, ScrollBarImageColor3 = Color3.fromRGB(70,70,70)})
    content.CanvasSize = UDim2.new(0,0,0,0)
    content.AutomaticCanvasSize = Enum.AutomaticSize.Y
    local contentLayout = new("UIListLayout", {Parent = content, SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0,8)})

    -- state
    local win = setmetatable({
        ScreenGui = sg,
        Container = container,
        Header = header,
        TabList = tabList,
        Content = content,
        Tabs = {},
        ActiveTab = nil,
        Minimized = false
    }, Window)

    -- close/minimize actions
    btnClose.MouseButton1Click:Connect(function()
        pcall(function() sg:Destroy() end)
    end)

    btnMin.MouseButton1Click:Connect(function()
        win.Minimized = not win.Minimized
        if win.Minimized then
            tween(container, {Size = UDim2.new(container.Size.X.Scale, container.Size.X.Offset, 0, 38)}, 0.22)
            left.Visible = false
            right.Visible = false
        else
            tween(container, {Size = opts.Size or STYLE.WindowSize}, 0.22)
            left.Visible = true
            right.Visible = true
        end
    end)

    makeDraggable(container, header)

    -- Tab creation
    function win:Tab(name, iconId)
        local tabButton = new("TextButton", {
            Parent = tabList,
            Size = UDim2.new(1,0,0,36),
            BackgroundColor3 = STYLE.Btn,
            Text = "  "..tostring(name),
            TextColor3 = STYLE.Text,
            Font = STYLE.SmallFont,
            TextSize = 14,
            BorderSizePixel = 0,
            AutoButtonColor = true
        })
        new("UICorner", {Parent = tabButton, CornerRadius = UDim.new(0,8)})

        local tabFrame = new("Frame", {Parent = content, Size = UDim2.new(1,0,0,10), BackgroundTransparency = 1})
        local layout = new("UIListLayout", {Parent = tabFrame, SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0,8)})
        tab
