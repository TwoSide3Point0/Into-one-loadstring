assert(Drawing, 'exploit not supported')

local UserInputService = game:GetService'UserInputService';
local HttpService = game:GetService'HttpService';
local GUIService = game:GetService'GuiService';
local RunService = game:GetService'RunService';
local Players = game:GetService'Players';
local LocalPlayer = Players.LocalPlayer;
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse();
local Menu = {};
local MouseHeld = false;
local LastRefresh = 0;
local OptionsFile = 'IC3_ESP_SETTINGS.dat';
local Binding = false;
local BindedKey = nil;
local OIndex = 0;
local LineBox = {};
local UIButtons = {};
local Sliders = {};
local Dragging = false;
local DraggingUI = false;
local DragOffset = Vector2.new();
local DraggingWhat = nil;
local OldData = {};
local IgnoreList = {};
local Red = Color3.new(1, 0, 0);
local Green = Color3.new(0, 1, 0);
local MenuLoaded = false;

shared.MenuDrawingData = shared.MenuDrawingData or { Instances = {} };
shared.PlayerData = shared.PlayerData or {};
shared.RSName = shared.RSName or ('UnnamedESP_by_ic3-' .. HttpService:GenerateGUID(false));

local GetDataName = shared.RSName .. '-GetData';
local UpdateName = shared.RSName .. '-Update';

local Debounce = setmetatable({}, {
__index = function(t, i)
return rawget(t, i) or false
end;
});

pcall(function() shared.InputBeganCon:disconnect() end);
pcall(function() shared.InputEndedCon:disconnect() end);

function GetMouseLocation()
return UserInputService:GetMouseLocation();
end

function MouseHoveringOver(Values)
local X1, Y1, X2, Y2 = Values[1], Values[2], Values[3], Values[4]
local MLocation = GetMouseLocation();
return (MLocation.x >= X1 and MLocation.x <= (X1 + (X2 - X1))) and (MLocation.y >= Y1 and MLocation.y <= (Y1 + (Y2 - Y1)));
end

function GetTableData(t) -- basically table.foreach i dont even know why i made this
if typeof(t) ~= 'table' then return end
return setmetatable(t, {
__call = function(t, func)
if typeof(func) ~= 'function' then return end;
for i, v in pairs(t) do
pcall(func, i, v);
end
end;
});
end
local function Format(format, ...)
return string.format(format, ...);
end
function CalculateValue(Min, Max, Percent)
return Min + math.floor(((Max - Min) * Percent) + .5);
end

local Options = setmetatable({}, {
__call = function(t, ...)
local Arguments = {...};
local Name = Arguments[1];
OIndex = OIndex + 1; -- (typeof(Arguments[3]) == 'boolean' and 1 or 0);
rawset(t, Name, setmetatable({
Name = Arguments[1];
Text = Arguments[2];
Value = Arguments[3];
DefaultValue = Arguments[3];
AllArgs = Arguments;
Index = OIndex;
}, {
__call = function(t, v)
if typeof(t.Value) == 'function' then
t.Value();
elseif typeof(t.Value) == 'EnumItem' then
local BT = Menu:GetInstance(Format('%s_BindText', t.Name));
Binding = true;
local Val = 0
while Binding do
wait();
Val = (Val + 1) % 17;
BT.Text = Val <= 8 and '|' or '';
end
t.Value = BindedKey;
BT.Text = tostring(t.Value):match'%w+%.%w+%.(.+)';
BT.Position = t.BasePosition + Vector2.new(t.BaseSize.X - BT.TextBounds.X - 20, -10);
else
local NewValue = v;
if NewValue == nil then NewValue = not t.Value; end
rawset(t, 'Value', NewValue);
if Arguments[2] ~= nil then
if typeof(Arguments[3]) == 'number' then
local AMT = Menu:GetInstance(Format('%s_AmountText', t.Name));
AMT.Text = tostring(t.Value);
AMT.Position = t.BasePosition + Vector2.new(t.BaseSize.X - AMT.TextBounds.X - 10, -10);
else
local Inner = Menu:GetInstance(Format('%s_InnerCircle', t.Name));
Inner.Visible = t.Value;
end
end
end
end;
}));
end;
})

function Load()
local _, Result = pcall(readfile, OptionsFile);
if _ then -- extremely ugly code yea i know but i dont care p.s. i hate pcall
local _, Table = pcall(HttpService.JSONDecode, HttpService, Result);
if _ then
for i, v in pairs(Table) do
if Options[i] ~= nil and Options[i].Value ~= nil and (typeof(Options[i].Value) == 'boolean' or typeof(Options[i].Value) == 'number') then
Options[i].Value = v.Value;
pcall(Options[i], v.Value);
end
end
end
end
end

Options('Enabled', 'ESP Enabled', true);
Options('ShowTeam', 'Show Team', false);
Options('ShowName', 'Show Names', true);
Options('ShowDistance', 'Show Distance', true);
Options('ShowHealth', 'Show Health', true);
Options('ShowBoxes', 'Show Boxes', true);
Options('ShowTracers', 'Show Tracers', true);
Options('ShowDot', 'Show Head Dot', false);
Options('VisCheck', 'Visibility Check', false);
Options('Crosshair', 'Crosshair', false);
Options('TextOutline', 'Text Outline', true);
Options('TextSize', 'Text Size', syn and 18 or 14, 10, 24); -- cuz synapse fonts look weird???
Options('MaxDistance', 'Max Distance', 2500, 100, 5000);
Options('RefreshRate', 'Refresh Rate (ms)', 5, 1, 200);
Options('MenuKey', 'Menu Key', Enum.KeyCode.F4, 1);
Options('ResetSettings', 'Reset Settings', function()
for i, v in pairs(Options) do
if Options[i] ~= nil and Options[i].Value ~= nil and Options[i].Text ~= nil and (typeof(Options[i].Value) == 'boolean' or typeof(Options[i].Value) == 'number') then
Options[i](Options[i].DefaultValue);
end
end
end, 4);
Options('LoadSettings', 'Load Settings', Load, 3);
Options('SaveSettings', 'Save Settings', function()
writefile(OptionsFile, HttpService:JSONEncode(Options));
end, 2)
-- Options.SaveSettings.Value();

Load();

Options('MenuOpen', nil, true);

local function Set(t, i, v)
t[i] = v;
end
local function Combine(...)
local Output = {};
for i, v in pairs{...} do
if typeof(v) == 'table' then
table.foreach(v, function(i, v)
Output[i] = v;
end)
end
end
return Output
end
function IsStringEmpty(String)
if type(String) == 'string' then
return String:match'^%s+$' ~= nil or #String == 0 or String == '' or false;
end
return false
end

function NewDrawing(InstanceName)
local Instance = Drawing.new(InstanceName);
return (function(Properties)
for i, v in pairs(Properties) do
pcall(Set, Instance, i, v);
end
return Instance;
end)
end

function Menu:AddMenuInstace(Name, Instance)
if shared.MenuDrawingData.Instances[Name] ~= nil then
shared.MenuDrawingData.Instances[Name]:Remove();
end
shared.MenuDrawingData.Instances[Name] = Instance;
return Instance;
end
function Menu:UpdateMenuInstance(Name)
local Instance = shared.MenuDrawingData.Instances[Name];
if Instance ~= nil then
return (function(Properties)
for i, v in pairs(Properties) do
-- print(Format('%s %s -> %s', Name, tostring(i), tostring(v)));
pcall(Set, Instance, i, v);
end
return Instance;
end)
end
end
function Menu:GetInstance(Name)
return shared.MenuDrawingData.Instances[Name];
end

function LineBox:Create(Properties)
local Box = { Visible = true }; -- prevent errors not really though dont worry bout the Visible = true thing

local Properties = Combine({
Transparency = 1;
Thickness = 1;
Visible = true;
}, Properties);

Box['TopLeft'] = NewDrawing'Line'(Properties);
Box['TopRight'] = NewDrawing'Line'(Properties);
Box['BottomLeft'] = NewDrawing'Line'(Properties);
Box['BottomRight'] = NewDrawing'Line'(Properties);

function Box:Update(CF, Size, Color, Properties)
if not CF or not Size then return end

local TLPos, Visible1 = Camera:WorldToViewportPoint((CF * CFrame.new( Size.X,  Size.Y, 0)).p);
local TRPos, Visible2 = Camera:WorldToViewportPoint((CF * CFrame.new(-Size.X,  Size.Y, 0)).p);
local BLPos, Visible3 = Camera:WorldToViewportPoint((CF * CFrame.new( Size.X, -Size.Y, 0)).p);
local BRPos, Visible4 = Camera:WorldToViewportPoint((CF * CFrame.new(-Size.X, -Size.Y, 0)).p);
-- ## BEGIN UGLY CODE
if Visible1 then
Box['TopLeft'].Visible = true;
Box['TopLeft'].Color = Color;
Box['TopLeft'].From = Vector2.new(TLPos.X, TLPos.Y);
Box['TopLeft'].To = Vector2.new(TRPos.X, TRPos.Y);
else
Box['TopLeft'].Visible = false;
end
if Visible2 then
Box['TopRight'].Visible = true;
Box['TopRight'].Color = Color;
Box['TopRight'].From = Vector2.new(TRPos.X, TRPos.Y);
Box['TopRight'].To = Vector2.new(BRPos.X, BRPos.Y);
else
Box['TopRight'].Visible = false;
end
if Visible3 then
Box['BottomLeft'].Visible = true;
Box['BottomLeft'].Color = Color;
Box['BottomLeft'].From = Vector2.new(BLPos.X, BLPos.Y);
Box['BottomLeft'].To = Vector2.new(TLPos.X, TLPos.Y);
else
Box['BottomLeft'].Visible = false;
end
if Visible4 then
Box['BottomRight'].Visible = true;
Box['BottomRight'].Color = Color;
Box['BottomRight'].From = Vector2.new(BRPos.X, BRPos.Y);
Box['BottomRight'].To = Vector2.new(BLPos.X, BLPos.Y);
else
Box['BottomRight'].Visible = false;
end
-- ## END UGLY CODE
if Properties then
GetTableData(Properties)(function(i, v)
pcall(Set, Box['TopLeft'], i, v);
pcall(Set, Box['TopRight'], i, v);
pcall(Set, Box['BottomLeft'], i, v);
pcall(Set, Box['BottomRight'], i, v);
end)
end
end
function Box:SetVisible(bool)
pcall(Set, Box['TopLeft'], 'Visible', bool);
pcall(Set, Box['TopRight'], 'Visible', bool);
pcall(Set, Box['BottomLeft'], 'Visible', bool);
pcall(Set, Box['BottomRight'], 'Visible', bool);
end
function Box:Remove()
self:SetVisible(false);
Box['TopLeft']:Remove();
Box['TopRight']:Remove();
Box['BottomLeft']:Remove();
Box['BottomRight']:Remove();
end

return Box;
end

function CreateMenu(NewPosition) -- Create Menu
local function FromHex(HEX)
HEX = HEX:gsub('#', '');
return Color3.fromRGB(tonumber('0x' .. HEX:sub(1, 2)), tonumber('0x' .. HEX:sub(3, 4)), tonumber('0x' .. HEX:sub(5, 6)));
end

local Colors = {
Primary = {
Main = FromHex'424242';
Light = FromHex'6d6d6d';
Dark = FromHex'1b1b1b';
};
Secondary = {
Main = FromHex'e0e0e0';
Light = FromHex'ffffff';
Dark = FromHex'aeaeae';
};
};

MenuLoaded = false;

GetTableData(UIButtons)(function(i, v)
v.Instance.Visible = false;
v.Instance:Remove();
end)
GetTableData(Sliders)(function(i, v)
v.Instance.Visible = false;
v.Instance:Remove();
end)

UIButtons = {};
Sliders = {};

local BaseSize = Vector2.new(300, 580);
local BasePosition = NewPosition or Vector2.new(Camera.ViewportSize.X / 8 - (BaseSize.X / 2), Camera.ViewportSize.Y / 2 - (BaseSize.Y / 2));

Menu:AddMenuInstace('CrosshairX', NewDrawing'Line'{
Visible = false;
Color = Color3.new(0, 1, 0);
Transparency = 1;
Thickness = 1;
});
Menu:AddMenuInstace('CrosshairY', NewDrawing'Line'{
Visible = false;
Color = Color3.new(0, 1, 0);
Transparency = 1;
Thickness = 1;
});

delay(.025, function() -- since zindex doesnt exist
Menu:AddMenuInstace('Main', NewDrawing'Square'{
Size = BaseSize;
Position = BasePosition;
Filled = false;
Color = Colors.Primary.Main;
Thickness = 3;
Visible = true;
});
end);
Menu:AddMenuInstace('TopBar', NewDrawing'Square'{
Position = BasePosition;
Size = Vector2.new(BaseSize.X, 25);
Color = Colors.Primary.Dark;
Filled = true;
Visible = true;
});
Menu:AddMenuInstace('TopBarTwo', NewDrawing'Square'{
Position = BasePosition + Vector2.new(0, 25);
Size = Vector2.new(BaseSize.X, 60);
Color = Colors.Primary.Main;
Filled = true;
Visible = true;
});
Menu:AddMenuInstace('TopBarText', NewDrawing'Text'{
Size = 25;
Position = shared.MenuDrawingData.Instances.TopBarTwo.Position + Vector2.new(25, 15);
Text = 'Unnamed ESP';
Color = Colors.Secondary.Light;
Visible = true;
});
Menu:AddMenuInstace('TopBarTextBR', NewDrawing'Text'{
Size = 15;
Position = shared.MenuDrawingData.Instances.TopBarTwo.Position + Vector2.new(BaseSize.X - 65, 40);
Text = 'by ic3w0lf';
Color = Colors.Secondary.Dark;
Visible = true;
});
Menu:AddMenuInstace('Filling', NewDrawing'Square'{
Size = BaseSize - Vector2.new(0, 85);
Position = BasePosition + Vector2.new(0, 85);
Filled = true;
Color = Colors.Secondary.Main;
Transparency= .5;
Visible = true;
});

local CPos = 0;

GetTableData(Options)(function(i, v)
if typeof(v.Value) == 'boolean' and not IsStringEmpty(v.Text) and v.Text ~= nil then
CPos = CPos + 25;
local BaseSize = Vector2.new(BaseSize.X, 30);
local BasePosition = shared.MenuDrawingData.Instances.Filling.Position + Vector2.new(30, v.Index * 25 - 10);
UIButtons[#UIButtons + 1] = {
Option = v;
Instance = Menu:AddMenuInstace(Format('%s_Hitbox', v.Name), NewDrawing'Square'{
Position = BasePosition - Vector2.new(30, 15);
Size = BaseSize;
Visible = false;
});
};
Menu:AddMenuInstace(Format('%s_OuterCircle', v.Name), NewDrawing'Circle'{
Radius = 10;
Position = BasePosition;
Color = Colors.Secondary.Light;
Filled = true;
Visible = true;
});
Menu:AddMenuInstace(Format('%s_InnerCircle', v.Name), NewDrawing'Circle'{
Radius = 7;
Position = BasePosition;
Color = Colors.Secondary.Dark;
Filled = true;
Visible = v.Value;
});
Menu:AddMenuInstace(Format('%s_Text', v.Name), NewDrawing'Text'{
Text = v.Text;
Size = 20;
Position = BasePosition + Vector2.new(20, -10);
Visible = true;
Color = Colors.Primary.Dark;
});
end
end)
GetTableData(Options)(function(i, v) -- just to make sure certain things are drawn before or after others, too lazy to actually sort table
if typeof(v.Value) == 'number' then
CPos = CPos + 25;

local BaseSize = Vector2.new(BaseSize.X, 30);
local BasePosition = shared.MenuDrawingData.Instances.Filling.Position + Vector2.new(0, CPos - 10);

local Text = Menu:AddMenuInstace(Format('%s_Text', v.Name), NewDrawing'Text'{
Text = v.Text;
Size = 20;
Position = BasePosition + Vector2.new(20, -10);
Visible = true;
Color = Colors.Primary.Dark;
});
local AMT = Menu:AddMenuInstace(Format('%s_AmountText', v.Name), NewDrawing'Text'{
Text = tostring(v.Value);
Size = 20;
Position = BasePosition;
Visible = true;
Color = Colors.Primary.Dark;
});
local Line = Menu:AddMenuInstace(Format('%s_SliderLine', v.Name), NewDrawing'Line'{
Transparency = 1;
Color = Colors.Primary.Dark;
Thickness = 3;
Visible = true;
From = BasePosition + Vector2.new(20, 20);
To = BasePosition + Vector2.new(BaseSize.X - 10, 20);
});
CPos = CPos + 10;
local Slider = Menu:AddMenuInstace(Format('%s_Slider', v.Name), NewDrawing'Circle'{
Visible = true;
Filled = true;
Radius = 6;
Color = Colors.Secondary.Dark;
Position = BasePosition + Vector2.new(35, 20);
})

local CSlider = {Slider = Slider; Line = Line; Min = v.AllArgs[4]; Max = v.AllArgs[5]; Option = v};
Sliders[#Sliders + 1] = CSlider;

-- local Percent = (v.Value / CSlider.Max) * 100;
-- local Size = math.abs(Line.From.X - Line.To.X);
-- local Value = Size * (Percent / 100); -- this shit's inaccurate but fuck it i'm not even gonna bother fixing it

Slider.Position = BasePosition + Vector2.new(40, 20);

v.BaseSize = BaseSize;
v.BasePosition = BasePosition;
AMT.Position = BasePosition + Vector2.new(BaseSize.X - AMT.TextBounds.X - 10, -10)
end
end)
GetTableData(Options)(function(i, v) -- just to make sure certain things are drawn before or after others, too lazy to actually sort table
if typeof(v.Value) == 'EnumItem' then
CPos = CPos + 30;

local BaseSize = Vector2.new(BaseSize.X, 30);
local BasePosition = shared.MenuDrawingData.Instances.Filling.Position + Vector2.new(0, CPos - 10);

UIButtons[#UIButtons + 1] = {
Option = v;
Instance = Menu:AddMenuInstace(Format('%s_Hitbox', v.Name), NewDrawing'Square'{
Size = Vector2.new(BaseSize.X, 20) - Vector2.new(30, 0);
Visible = true;
Transparency= .5;
Position = BasePosition + Vector2.new(15, -10);
Color = Colors.Secondary.Light;
Filled = true;
});
};
local Text = Menu:AddMenuInstace(Format('%s_Text', v.Name), NewDrawing'Text'{
Text = v.Text;
Size = 20;
Position = BasePosition + Vector2.new(20, -10);
Visible = true;
Color = Colors.Primary.Dark;
});
local BindText = Menu:AddMenuInstace(Format('%s_BindText', v.Name), NewDrawing'Text'{
Text = tostring(v.Value):match'%w+%.%w+%.(.+)';
Size = 20;
Position = BasePosition;
Visible = true;
Color = Colors.Primary.Dark;
});

Options[i].BaseSize = BaseSize;
Options[i].BasePosition = BasePosition;
BindText.Position = BasePosition + Vector2.new(BaseSize.X - BindText.TextBounds.X - 20, -10);
end
end)
GetTableData(Options)(function(i, v) -- just to make sure certain things are drawn before or after others, too lazy to actually sort table
if typeof(v.Value) == 'function' then
local BaseSize = Vector2.new(BaseSize.X, 30);
local BasePosition = shared.MenuDrawingData.Instances.Filling.Position + Vector2.new(0, CPos + (25 * v.AllArgs[4]) - 35);

UIButtons[#UIButtons + 1] = {
Option = v;
Instance = Menu:AddMenuInstace(Format('%s_Hitbox', v.Name), NewDrawing'Square'{
Size = Vector2.new(BaseSize.X, 20) - Vector2.new(30, 0);
Visible = true;
Transparency= .5;
Position = BasePosition + Vector2.new(15, -10);
Color = Colors.Secondary.Light;
Filled = true;
});
};
local Text = Menu:AddMenuInstace(Format('%s_Text', v.Name), NewDrawing'Text'{
Text = v.Text;
Size = 20;
Position = BasePosition + Vector2.new(20, -10);
Visible = true;
Color = Colors.Primary.Dark;
});

-- BindText.Position = BasePosition + Vector2.new(BaseSize.X - BindText.TextBounds.X - 10, -10);
end
end)

delay(.1, function()
MenuLoaded = true;
end);

-- this has to be at the bottom cuz proto drawing api doesnt have zindex :triumph:
Menu:AddMenuInstace('Cursor1', NewDrawing'Line'{
Visible = false;
Color = Color3.new(1, 0, 0);
Transparency = 1;
Thickness = 2;
});
Menu:AddMenuInstace('Cursor2', NewDrawing'Line'{
Visible = false;
Color = Color3.new(1, 0, 0);
Transparency = 1;
Thickness = 2;
});
Menu:AddMenuInstace('Cursor3', NewDrawing'Line'{
Visible = false;
Color = Color3.new(1, 0, 0);
Transparency = 1;
Thickness = 2;
});
end

CreateMenu();

shared.InputBeganCon = UserInputService.InputBegan:connect(function(input)
if input.UserInputType.Name == 'MouseButton1' and Options.MenuOpen.Value then
MouseHeld = true;
local Bar = Menu:GetInstance'TopBar';
local Values = {
Bar.Position.X;
Bar.Position.Y;
Bar.Position.X + Bar.Size.X;
Bar.Position.Y + Bar.Size.Y;
}
if MouseHoveringOver(Values) and not syn then -- disable dragging for synapse cuz idk why it breaks
DraggingUI = false; -- also breaks for other exploits
DragOffset = Menu:GetInstance'Main'.Position - GetMouseLocation();
else
for i, v in pairs(Sliders) do
local Values = {
v.Line.From.X - (v.Slider.Radius);
v.Line.From.Y - (v.Slider.Radius);
v.Line.To.X + (v.Slider.Radius);
v.Line.To.Y + (v.Slider.Radius);
};
if MouseHoveringOver(Values) then
DraggingWhat = v;
Dragging = true;
break
end
end
end
end
end)
shared.InputEndedCon = UserInputService.InputEnded:connect(function(input)
if input.UserInputType.Name == 'MouseButton1' and Options.MenuOpen.Value then
MouseHeld = false;
for i, v in pairs(UIButtons) do
local Values = {
v.Instance.Position.X;
v.Instance.Position.Y;
v.Instance.Position.X + v.Instance.Size.X;
v.Instance.Position.Y + v.Instance.Size.Y;
};
if MouseHoveringOver(Values) then
v.Option();
break -- prevent clicking 2 options
end
end
elseif input.UserInputType.Name == 'Keyboard' then
if Binding then
BindedKey = input.KeyCode;
Binding = false;
elseif input.KeyCode == Options.MenuKey.Value or (input.KeyCode == Enum.KeyCode.Home and UserInputService:IsKeyDown(Enum.KeyCode.LeftControl)) then
Options.MenuOpen();
end
end
end)

function ToggleMenu()
if Options.MenuOpen.Value then
GetTableData(shared.MenuDrawingData.Instances)(function(i, v)
if OldData[v] then
pcall(Set, v, 'Visible', true);
end
end)
else
-- GUIService:SetMenuIsOpen(false);
GetTableData(shared.MenuDrawingData.Instances)(function(i, v)
if v.Visible == true then
OldData[v] = true;
pcall(Set, v, 'Visible', false);
end
end)
end
end

function CheckRay(Player, Distance, Position, Unit)
local Pass = true;

if Distance > 999 then return false; end

local _Ray = Ray.new(Position, Unit * Distance);

local List = {LocalPlayer.Character, Camera, Mouse.TargetFilter};

for i,v in pairs(IgnoreList) do table.insert(List, v); end;

local Hit = workspace:FindPartOnRayWithIgnoreList(_Ray, List);
if Hit and not Hit:IsDescendantOf(Player.Character) then
Pass = false;
if Hit.Transparency >= .3 or not Hit.CanCollide and Hit.ClassName ~= Terrain then -- Detect invisible walls
IgnoreList[#IgnoreList + 1] = Hit;
end
end

return Pass;
end

function CheckPlayer(Player)
if not Options.Enabled.Value then return false end

local Pass = true;
local Distance = 0;

if Player ~= LocalPlayer and Player.Character then
if not Options.ShowTeam.Value and Player.TeamColor == LocalPlayer.TeamColor then
Pass = false;
end

local Head = Player.Character:FindFirstChild'Head';

if Pass and Player.Character and Head then
Distance = (Camera.CFrame.p - Head.Position).magnitude;
if Options.VisCheck.Value then
Pass = CheckRay(Player, Distance, Camera.CFrame.p, (Head.Position - Camera.CFrame.p).unit);
end
if Distance > Options.MaxDistance.Value then
Pass = false;
end
end
else
Pass = false;
end

return Pass, Distance;
end

function UpdatePlayerData()
if (tick() - LastRefresh) > (Options.RefreshRate.Value / 1000) then
LastRefresh = tick();
for i, v in pairs(Players:GetPlayers()) do
local Data = shared.PlayerData[v.Name] or { Instances = {} };

Data.Instances['Box'] = Data.Instances['Box'] or LineBox:Create{Thickness = 3};
Data.Instances['Tracer'] = Data.Instances['Tracer'] or NewDrawing'Line'{
Transparency = 1;
Thickness = 2;
}
Data.Instances['HeadDot'] = Data.Instances['HeadDot'] or NewDrawing'Circle'{
Filled = true;
NumSides = 30;
}
Data.Instances['NameTag'] = Data.Instances['NameTag'] or NewDrawing'Text'{
Size = Options.TextSize.Value;
Center = true;
Outline = Options.TextOutline.Value;
Visible = true;
};
Data.Instances['DistanceHealthTag'] = Data.Instances['DistanceHealthTag'] or NewDrawing'Text'{
Size = Options.TextSize.Value - 1;
Center = true;
Outline = Options.TextOutline.Value;
Visible = true;
};

local NameTag = Data.Instances['NameTag'];
local DistanceTag = Data.Instances['DistanceHealthTag'];
local Tracer = Data.Instances['Tracer'];
local HeadDot = Data.Instances['HeadDot'];
local Box = Data.Instances['Box'];

local Pass, Distance = CheckPlayer(v);

if Pass and v.Character then
Data.LastUpdate = tick();
local Humanoid = v.Character:FindFirstChildOfClass'Humanoid';
local Head = v.Character:FindFirstChild'Head';
local HumanoidRootPart = v.Character:FindFirstChild'HumanoidRootPart';
if v.Character ~= nil and Head then
local ScreenPosition, Vis = Camera:WorldToViewportPoint(Head.Position);
if Vis then
local Color = v.TeamColor == LocalPlayer.TeamColor and Green or Red;

local ScreenPositionUpper = Camera:WorldToViewportPoint(Head.CFrame * CFrame.new(0, Head.Size.Y, 0).p);
local Scale = Head.Size.Y / 2;

if Options.ShowName.Value then
NameTag.Visible = true;
NameTag.Text = v.Name;
NameTag.Size = Options.TextSize.Value;
NameTag.Outline = Options.TextOutline.Value;
NameTag.Position = Vector2.new(ScreenPositionUpper.X, ScreenPositionUpper.Y);
NameTag.Color = Color;
if Drawing.Fonts then -- CURRENTLY SYNAPSE ONLY :MEGAHOLY:
NameTag.Font = Drawing.Fonts.UI;
end
else
NameTag.Visible = false;
end
if Options.ShowDistance.Value or Options.ShowHealth.Value then
DistanceTag.Visible = true;
DistanceTag.Size = Options.TextSize.Value - 1;
DistanceTag.Outline = Options.TextOutline.Value;
DistanceTag.Color = Color3.new(1, 1, 1);
if Drawing.Fonts then -- CURRENTLY SYNAPSE ONLY :MEGAHOLY:
NameTag.Font = Drawing.Fonts.UI;
end

local Str = '';

if Options.ShowDistance.Value then
Str = Str .. Format('[%d] ', Distance);
end
if Options.ShowHealth.Value and Humanoid then
Str = Str .. Format('[%d/100]', Humanoid.Health / Humanoid.MaxHealth * 100);
end

DistanceTag.Text = Str;
DistanceTag.Position = Vector2.new(ScreenPositionUpper.X, ScreenPositionUpper.Y) + Vector2.new(0, NameTag.Size);
else
DistanceTag.Visible = false;
end
if Options.ShowDot.Value then
local Top = Camera:WorldToViewportPoint((Head.CFrame * CFrame.new(0, Scale, 0)).p);
local Bottom = Camera:WorldToViewportPoint((Head.CFrame * CFrame.new(0, -Scale, 0)).p);
local Radius = (Top - Bottom).y;

HeadDot.Visible = true;
HeadDot.Color = Color;
HeadDot.Position = Vector2.new(ScreenPosition.X, ScreenPosition.Y);
HeadDot.Radius = Radius;
else
HeadDot.Visible = false;
end
if Options.ShowTracers.Value then
Tracer.Visible = true;
Tracer.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y);
Tracer.To = Vector2.new(ScreenPosition.X, ScreenPosition.Y);
Tracer.Color = Color;
else
Tracer.Visible = false;
end
if Options.ShowBoxes.Value and HumanoidRootPart then
Box:Update(HumanoidRootPart.CFrame, Vector3.new(2, 3, 0) * (Scale * 2), Color);
else
Box:SetVisible(false);
end
else
NameTag.Visible = false;
DistanceTag.Visible = false;
Tracer.Visible = false;
HeadDot.Visible = false;

Box:SetVisible(false);
end
end
else
NameTag.Visible = false;
DistanceTag.Visible = false;
Tracer.Visible = false;
HeadDot.Visible = false;

Box:SetVisible(false);
end

shared.PlayerData[v.Name] = Data;
end
end
end

function Update()
for i, v in pairs(shared.PlayerData) do
if not Players:FindFirstChild(tostring(i)) then
GetTableData(v.Instances)(function(i, obj)
obj.Visible = false;
obj:Remove();
v.Instances[i] = nil;
end)
shared.PlayerData[i] = nil;
end
end

local CX = Menu:GetInstance'CrosshairX';
local CY = Menu:GetInstance'CrosshairY';
if Options.Crosshair.Value then
CX.Visible = true;
CY.Visible = true;

CX.To = Vector2.new((Camera.ViewportSize.X / 2) - 8, (Camera.ViewportSize.Y / 2));
CX.From = Vector2.new((Camera.ViewportSize.X / 2) + 8, (Camera.ViewportSize.Y / 2));
CY.To = Vector2.new((Camera.ViewportSize.X / 2), (Camera.ViewportSize.Y / 2) - 8);
CY.From = Vector2.new((Camera.ViewportSize.X / 2), (Camera.ViewportSize.Y / 2) + 8);
else
CX.Visible = false;
CY.Visible = false;
end

if Options.MenuOpen.Value and MenuLoaded then
local MLocation = GetMouseLocation();
shared.MenuDrawingData.Instances.Main.Color = Color3.fromHSV(tick() * 24 % 255/255, 1, 1);
local MainInstance = Menu:GetInstance'Main';
local Values = {
MainInstance.Position.X;
MainInstance.Position.Y;
MainInstance.Position.X + MainInstance.Size.X;
MainInstance.Position.Y + MainInstance.Size.Y;
};
if MainInstance and MouseHoveringOver(Values) then
Debounce.CursorVis = true;
-- GUIService:SetMenuIsOpen(true);
Menu:UpdateMenuInstance'Cursor1'{
Visible = true;
From = Vector2.new(MLocation.x, MLocation.y);
To = Vector2.new(MLocation.x + 5, MLocation.y + 6);
}
Menu:UpdateMenuInstance'Cursor2'{
Visible = true;
From = Vector2.new(MLocation.x, MLocation.y);
To = Vector2.new(MLocation.x, MLocation.y + 8);
}
Menu:UpdateMenuInstance'Cursor3'{
Visible = true;
From = Vector2.new(MLocation.x, MLocation.y + 6);
To = Vector2.new(MLocation.x + 5, MLocation.y + 5);
}
else
if Debounce.CursorVis then
Debounce.CursorVis = false;
-- GUIService:SetMenuIsOpen(false);
Menu:UpdateMenuInstance'Cursor1'{Visible = false};
Menu:UpdateMenuInstance'Cursor2'{Visible = false};
Menu:UpdateMenuInstance'Cursor3'{Visible = false};
end
end
if MouseHeld then
if Dragging then
DraggingWhat.Slider.Position = Vector2.new(math.clamp(MLocation.X, DraggingWhat.Line.From.X, DraggingWhat.Line.To.X), DraggingWhat.Slider.Position.Y);
local Percent = (DraggingWhat.Slider.Position.X - DraggingWhat.Line.From.X) / ((DraggingWhat.Line.To.X - DraggingWhat.Line.From.X));
local Value = CalculateValue(DraggingWhat.Min, DraggingWhat.Max, Percent);
DraggingWhat.Option(Value);
elseif DraggingUI then
Debounce.UIDrag = true;
local Main = Menu:GetInstance'Main';
local MousePos = GetMouseLocation();
Main.Position = MousePos + DragOffset;
end
else
Dragging = false;
if DraggingUI and Debounce.UIDrag then
Debounce.UIDrag = false;
DraggingUI = false;
CreateMenu(Menu:GetInstance'Main'.Position);
end
end
if not Debounce.Menu then
Debounce.Menu = true;
ToggleMenu();
end
elseif Debounce.Menu and not Options.MenuOpen.Value then
Debounce.Menu = false;
ToggleMenu();
end
end

RunService:UnbindFromRenderStep(GetDataName);
RunService:UnbindFromRenderStep(UpdateName);

RunService:BindToRenderStep(GetDataName, 1, UpdatePlayerData);
RunService:BindToRenderStep(UpdateName, 1, Update);
_8p13NG5DfjygebnQ, Protected_by_MoonSecV2, Discord = 'discord.gg/gQEH2uZxUk'


,nil,nil;(function() _msec=(function(_,__,_l)local _ll__l=__[((0x1440/48)+-#"Wanna hear a joke? RainX")];local __l__l=_l[_[(663+-0x29)]][_[(107325/0x87)]];local ____l=(-#"3 month of free private access be like"+(0x11e2/109))/(((((-0x6e+4704)+-#'the private version should be paid, like seriously, I get 0 fps drops!')/52)+-#'weirdo above')-0x49)local _lll=((0xd2ba/(0x4731/75))/0x6f)-(50-0x31)local ___lll=_l[_[(0x181-250)]][_[(232+-#{'nil';(function()return('BoMOi0'):find("\77")end)();44,nil;(function(_)return _ end)(),(function()return('iAOOIR'):find("\6")end)(),(function(_)return _ end)()})]];local _ll__=(0x2e-((342-0xdc)-0x4d))+(106-0x68)local __l_ll=_l[_[((38352+-0x48)/0x42)]][_[(-#"person below gay asf"+(76804/0x5b))]]local _ll=(0x2b+-41)-((-54+(333-0xcd))/74)local __l_l=(0x25+(10-(((-#'the private version should be paid, like seriously, I get 0 fps drops!'+(-103+0x24fc))/0x8f)+-#[[Best ironbrew remake]])))local _l_l=((-38+((-6880/(91+-#{1;47;1,(function()return('M8liR1'):find("\108")end)();(function()return{','}end)()}))+161))+-#'RainX more like IeatyourfuckingIPaddressX')local _l__=(0x2c-(-99+(0x178-(488-0xfd))))local __ll=(4+-#{'}';79;(function(_)return _ end)(),nil})local __l_=((-#[[Your exploit crashed and now youre crying? <a:HAHAHAHA:762019852327190598>]]+(0xfe4c/((8025+-0x4b)/0x35)))/180)local ___l=(0x83-(((26821035/0x53)/0xa7)/15))local _l___=(128-(0x144-(229+(0xf+((63-0x2e)+-#"when the obfuscator is too secure that the script doesnt work")))))local __ll_=(((-#[[RainX is a RAT, it does not utilize SxLib like it claims and instead completely takes over your pc]]+((713+((-0x49+22)+-#"oh you use protocrasher? name every crash you got"))-337))-0x84)+-#"how to deobfuscate this code: you cant lol")local _____=((0x6db/(-#'uwu suck it harder daddy omg im going to cum!11!!'+(0xc54c/(-#{'}';'nil';(function()return('BMO0MI'):find("\79")end)(),'}'}+211))))+-#"balls")local _l_l_=(0x49-((-#"Wanna hear a joke? RainX"+(((5679062/0xc7)+-#"hi skid")-14293))/0xce))local _lll_=(-#'745595618944221236'+((((34880-0x4434)-0x2228)-4386)/0xc3))local ___l_=(81+((5-(0x9c-(210+-0x55)))+-#'This file was obfuscated using PSU Obfuscator 4.0.A'))local _l_ll=(((458-((-0x29+340)+-#"how bored do you have to be to read this"))+-94)-102)local ____=(((0x14b+((-0x6e8/(-#[[Send dudes]]+(0x98-(10125/0x51))))+-#[[RainX winning]]))+-#[[the private version should be paid, like seriously, I get 0 fps drops!]])/0x30)local _ll_=(-#[[RainX is a RAT, it does not utilize SxLib like it claims and instead completely takes over your pc]]+((((((0x5a+-81)+-#"when you tried to skid a moonsec obfuscated script")+-0x42)+78)+-#"balls")+135))local _llll=(-#"Send dudes"+((0x4380/(-#{(function()return('RBBCl1'):find("\66")end)();{};190;190,",",(function()return('RRHBN1'):find("\5")end)()}+133))-122))local _ll_l=((-#[[why are you wasting your time here? go outside and get a partner]]+(((714+(-#"oh you use krnl? name everything that broke last update."+(0x97-121)))-405)-0xb8))-0x20)local ___ll=(0x111/(-#{{};'}',145,'nil';(function()return('IoNNNN'):find("\14")end)()}+95))local _l_l_l=_[(-#'dn method'+((-0x36+2884)-1471))];local _lll_l=_l[_[((-#'ur retarded'+(90024/0xf8))-217)]][_[(15505/0x23)]];local ____ll=_l[(function(_)return type(_):sub(1,1)..'\101\116'end)('��������')..'\109\101'..('\116\97'or'�🤣�🤣��').._[(-#[[can you find what you want here?]]+(37332/0x3d))]];local _l__l=_l[_[(705+-0x7d)]][_[(919+-#{",",(function()return('MAIMHN'):find("\73")end)(),(function()return{','}end)();109,",";nil;kernel32dII})]];local __llll=(36-0x22)-(-#'person below gay asf'+(((0x293-358)-0xd5)-66))local _lllll=(((-0x61+511)+-66)/174)-(0x15+-19)local _l__ll=_l[_[(350-0xd7)]][_[(-#[[no he wasnt]]+(0x1b7+-110))]];local __l=function(_l,_)return _l.._ end local _l_lll=(0x15c/87)*(-#"lua beautifier is gay"+(-102+(((0x3a1a0/8)/0xc9)+-#'cock and ball torture')))local __ll_l=_l[_[(-#{'}';87,'}',(function()return('Ao0EH0'):find("\48")end)(),87,(function(_)return _ end)();nil}+1168)]];local _l_=(0x7a-120)*(0x153-(((-#'ur retarded'+(-96+0xab9b))/166)+-#'just boost moonsec and get an even better obfuscator!'))local _ll_ll=((2181-(2392-0x4e4))+-#'baxoplenty is fat')*(72+((-0x68+74)-40))local ___l_l=(-0x27+((((-108+0x6b)+-#"mahmds bbc is kinda big no cap")+143)+-#[[cock and ball torture]]))local __l__=(((478-0x103)-0xa9)+-0x30)*(0x14a/165)local __lll=_l[_[((1166+-0x59)+-#"lua beautifier is gay")]]or _l[_[((0x1ee00/208)+-#[[Imagine using Public version]])]][_[((1166+-0x59)+-#"lua beautifier is gay")]];local ___=(-88+(((-0x36+857)+-#"Allah is a")-0x1c1))local _=_l[_[(1332+-0x64)]];local __l_ll=(function(_l_)local ___,__=4,0x10 local _l={j={},v={}}local __l=-_ll local _=__+_lll while true do _l[_l_:sub(_,(function()_=___+_ return _-_lll end)())]=(function()__l=__l+_ll return __l end)()if __l==(_l_lll-_ll)then __l=""__=__llll break end end local __l=#_l_ while _<__l+_lll do _l.v[__]=_l_:sub(_,(function()_=___+_ return _-_lll end)())__=__+_ll if __%____l==__llll then __=_lllll _l__l(_l.j,(_l__ll((_l[_l.v[_lllll]]*_l_lll)+_l[_l.v[_ll]])))end end return __l_ll(_l.j)end)("..:::MoonSec::..😀😃😄😁😆😅🤣😂🙂🙃😉😊😇😝🤪😜😝😅😅😉😃😅😊😅😅😅😝😉🙃🙃😁😅😝😅😂😅😂😇😃🤣😇😄😅😉😆😝😆😅😝😅😂😅😃😅😀🙃😉😇😜😄😉😝😅🙃🤪😃😆🙂😊😝🙂😄😄😀😊😉😜😇🙃🙃😄😂😊🤣😆🙂😜😅🤣😀😜😅🙃😅😁😅😁🙂😝😝😂😉😃🙂😇😀😅🙂😜🤣😉😃😆😃😝😉🙂😂😆😜😅🤪😜😅🙃😅😁😅😄😅🤪😃😂🤣😄🤪😊😉🤣😂😝😇😉😉😁🤪😉🤣😃😇😊😅😅😅😜😅😜😝😉🙃😆🙃🤪😅😆😜🤪😆🙂😆😃🤣🙃😇😁😅😝😅😂😅😆😉😇🤣😁😁😉😃😜🤣😅😊😇🤣😄😝😃🤪😊😅😅😅😜😅😝🙂🙃😝😁🤣🤪😂😂🤣😃🙂😇🙃😅😉😀😂🙂😁🤪😊😂😅😃😅😊😅😂😆😀🤣😂😁🤪🙂😆🙃😊😂😁🙂🙃🙃😀🙂😁😆😜😊😅😝😝🙂😁😇🤣😆😀😃🙃😁😀😁😂😄😇😂😆🙃😊😂😁😂😊😇😅😅😜😅🙃😅😂😂😁😇😇😅😅😉😜😅🙃🤣😄😊😇🤣😁😇😝😅😂😅😃😅😀🙂😂😂😊😂😀😉😅😝😊😊😀😝🙃😀😊😊😅😅😜😅🙃😅😄🙂😁🤪😃😀😊😂😁😁😀😁😆😁🙃🙂😁😅😝😅😂😅😆🙂😇🙃😁😃😃🤣🙃😊😁😅😝😅😂😅😝😊😃😁😃🙃😂😊😝😝🙃😅😀😄😂😅😃😅😊😅😅😅😜😅😅😅🙂😊😃😅😅😉😃🙂😊😅😅😅😜😅😀😁🙃😉😆😇😉🤣😃😝😊😅😅😅😜😅😝🤣😉😄😁😜😂😜😆😜😀😀🙃🙃😄😜😉😆😁😅😝😅😂😅😃😅😊😅😝😅😁🙂😝😅😆🙂😝🙂😂😅😃😅😊😅🙃😇😅😉😀🙃😁🙂😝😜😂😅😃😅😊😅🤣😊😜🙃😂🙂😇😂😆🙃😇🤣😁😉🙃😝😀😂😂🙃😉🙂😁😇😝😅😂😅😃😅😃😊😇😂🤣😆😀😄🙂😂😃😇😊😂😅😅😊🤪😅😅😜😅🙃😅🙂😄😁🤣😝😇🙂😁😃🤪😇🙃🤣😉😜🙃🙃😉😅😜😂😊😃😅😊😅😅😅🤪😝😁😜😜😇😝😇😝😀😉😃😀😆😅🤪😜😅🙃😅😁😅😁😇😝🤣🙂😄😃😉😇🙂😅😉😀😁😉🙂😁😉😉😄😃😅😊😅😅😅😜😅🙃😅😁😅😄😁😊😅😅😃😊😅😅😅😜😅🙃😅😁😅🙃😅😇🙂😅😅😀😜😅😅😜😅🙃😅😁😅😝😅😜😅😅😉😜😅😁😉😜🤣🙃😅😁😅😝😅🤪😅🤪😊😇😃😅😅😜😅🙃😅😉😄😀🤣😅🤪😇😄😃😜🙃😀😜😅😆😝😇😃😃😊😂😂🤪🙂😆😜😅🤪😜😅🙃😅😁😅😆😇🤪😆🙂😂😄😀😇🙂🤣😅😜🤣🙃🙂😁😉🙂🙂😃😝😊😅😅😅😜😅😝🙂🙃🤣😆😃🤪😃😂😂😃🤣😊🙂🤣😀😉😂😁😊😝😅😂😅😃😅🙃😂😇😆😝🤣😝🤣🙂😂😄😁😄😇😃😊😊😅😅😅😜😅😄🙂😀😁😜😅😉😆😂🙃😊😄😉😜😜😅🙃😅😁😅😝😅😂😅🙃😅😜😜🙃😅😀😂🙃😊😁😅😝😅😂😅🙂🤪😉😂😄😀🤣😆😂😇😂🤣🤣😀😂😉😃😅😊😅😅😅😁🤣🤪😁🤣😇😝🙃🤣😂😂😁😊🤪😅😅😜😅🙃😅🙃🙂😁😝😝🤣🙂😂😃🤣😊🙂🤣🙃😜😉😉😂😅😃😂🙃😃😅😊😅😅😅😅😇😜🤣😉😄😁😉😀😊😃😅😊😅😅😅😜😅🙃😅😂😅😄🙂😊😅😊🤪😊😜😅😅😜😅🙃😅😂😇😁😉🤪🙃🤣🙂😃😉😇😂🤣😊😜🤪🙃🙂😁😉😝😄😃😊😊😅😅😅😜😅😝😂😆🙂😊🤣😃😝🙃😄😜😊😊😄😜😜🙃😅😁😅😝😅😊😀😄😁🙂😅😜🙂😂😂🤪🙂😅🤣😊😝😁😄🙃😊😇😂🤣😇😜😅🙃😅😁😅😀😅🙂😆😜😆🤣😀😝😝😃😜😊😄😄😁🙂😂😜😜🤣🤣😝😅😆😂😊😜😀🙂🙃😝😃🤣😂🙃🤪😂🤣🙂😇😝😆😁😊😄😜😊😝😊😂😅😃😅😊😅😜😅😀😝😂😂😇😊😀😇😊😁😜😉😇😇😅😅😜😅🙃😅😂🙃😜😇🤣🤪😊😁😅😆🤪🤣😅😆😝🙃😁😝😇😁😁😜😂😅😝😀🙂🙃😜😊🙂🤣🤪😝😅😝🙃😃😃😉🙂😉😜😝😂😂🙃🙂😁😊😝😅😂😅😃😅😜😆🤪😝😉😉😅😊🙃😝🤪🙂😆😂😃😅😊😅😅😅😜😅🙃😅😊😅😄🤣😊😅😉😀😊😜😅😅😜😅🙃😅🙂🤣😄😇😉🙂😃😀😉😅😁😅😊😇😁😄😉😜😁😆😊😆😃🙂😊😅😅😅😜😅😝😄😁🙃😊🙂😜😇😃😅😊😅😅😅😜😅🙃😅😁😅😝😝😊😅🤣😀😊😅😅😅😜😅🙃😅😁😅😝😅😊🙂😅😅🙃🙃😅😊😜😅🙃😅😁😅😆😂😉🙂😄🤣😉😂🙃😂🙃🙂😇🤣😁😊😝😅😂😅😃😅😄😝😇😅😅🙂😜🤣😉😃😆😃🤣😅😃😅😊😅😅😅😜😅🙃😅😊😅😃🤪😊😅😉😜😊🙃😅😅😜😅🙃😅🙃🤣🤪😅😆😂🙃😂😊🙃🤣😝😜😅🙃😅😁😅😆😅😝🤣🙂🙂😄🙃😊😉😅😂😜🤪😉😁😀😁😝🙂🙂😆😄😄🙂😆🤣😂😜🤣😉😇😀😆🤪😉🤣😀🤪🙂🙂😇😁😊😜😜😂😉😁😁😂😊😃😅😊😅😅😅😅😂😊🤣😀🙂😅😀😉😃😀😂😃😄😅😊😜😅🙃😅😁😅😂😉🙃🤪😅🙂🤣😉😁😄😉😂🙃🙂😆😉😝😅😂😅😃😅🤪🙂😅😝😝😂😁😂😊😁😄🤣🙂🙂😇😄🤣😊😝😊😃😁😊😊😃😅😂🙃🤪😇😆🤣😜🙃😆🤪😝😇😄🙃😉😜😄😆😂😉😃😅😊😅😅😅😁🙂😜😝🙃😉😆🙂🤪🙃🤪😊😊🤣😅😅😜😅🙃😅😂😂😀😄😂😅😃😅😊😅😅😅😜😅🙃😅😂😄😃😅😅🙃😃😊😊😅😅😅😜😅😊🙃🙃😆🤣😊😝🙃🙃😃😜😝😜🤪😜😜🙃😅😁😅😝😅😇🙂🙂😉😄😂😊😊😅🤣😜🙂🙃😉😃😇🤪😉😂🤪😇😂😅🤪😜😅🙃😅😁😅😜😊😂😅😜😀😅😜😊😂😆😆😉🙃😃🙃🙂🙂🙂🙂😄😆😊😅😅😅😜😅😝🤣🙃🙃😁🙃😊😉🙂😊😃😉😇😁🤣🙃😝😝🙃🤣😆😁😝🙃🙂😃😃😉😇😂😊😇🙃😊😁😅😝😅😂😅🙂🙂😃😝😊🤣😅🙃😀😆😉😇😊🙃😂😇😃😅😊😅😅😅😀😇😉🙂😃🙂🙂😆😇😊😅🙃😝🙂🤣😂😜😊🙃😅😁😅😝😅😀🤪🤪😅😅😊😅🙃😜😃😝😆😇😜😝🙃😂😅😃😅😊😅🙃😜🤣🙂😀😆😉😁😅😃😂🙃😃😅😊😅😅😅😁🙂😜🤣😉🙂😁😝😃😝😃😊😊😅😅😅😜😅😊🙂😆😁😉😊😃😊🙃😀😜🙃😝😃😜🙃🙃😅😁😅😝😅🤪😅😂🤣😄😂😇🙃😅😄😉😄😁😅😝😅😂😅😄😜😇😄😁😁🙃😂😀😜😂🤣🤪😅😅😂😇😜😄🙃😊🙂😃😝🙃😀😇😜🤪😄😂😅😃😅😊😅😊🤣🤣😅😜🤪🤣😆😆😇😝😉😂😂😃😝😇😆🤣😆😀😀😉🙂😀😆😄😝😃🤪😊😅😅😅😜😅🤪🙂🙂😝😃😝😊😝😆😂🤪😀😂😃😀😁🙃😜😅😁😝😅😂😅😃😅😊😅😅😅😂😅😝😅😂😅🙃😇😂🤪😃😅😊😅😅😅😆😃😉😜😃😃😂😀😉😜😄😉🙂😅😝😀😄😉😀😄😆😀😝😅😂😅😃😅😜🤣😊🙃🤣😄😜🤪😉😁😄😂😝😉🙂😄😄😆😇🙃😅😉😊😁😁😅😝😅😂😅😃😅😊😅😅😅😆🙃😝😅😉😄😝😉😂😅😃😅😊😅😉😂😅🤪😜😇🙃😝😆🙃😁🙂😃😅😊😅😅😅😜😅🙃😅😁😅😀🤣😊😅😊😝😊😊😅😅😜😅🙃😅😀😅😇😝😁😜😅😜😇🤣😇😝😆😀🙃😅😁😅😝😅😂😅😃😅😊😅😉😃😁😅🤪😁😁😇😝😅😂😅😃😅😜🤣😅🙃😝😅😆🙃😉😝😄🤣🙂🙃😁😁😊😅😅😅😜😅🙃😅😁😅😃😅😇😅😅😅🙃😉🤣😆😜😅🙃😅😁😅😁🙃🤪🙂😂😊😝😅😊🤪🤣🙂😊😅🙃😝😁🤣🤪😂🙂😄😄😃😊😉🤣🙂😀🙂🤪😇😝😊😂😅😃😅😊😅🤪😄😁🙃🤣😆😀😇🤪😀😅😊😝😇😊🙂😅😅😜😅🙃😅😂😉😄🙂😇😅😄😂😇😄😅😅😜😅🙃😅😅🤣😝🙃😅😄😊🙂😄🙃😉🤣😜😂🙂🙂🤪🙃🤣😆😝😆😁🙃😉🙂🙂🤣😀😃🙃😅😁😅😝😅😊🤣😂🙃😃🙃🙃😉🤣😝😀🙃😉😂😁🤣😊🙂😂🤣😄🙂😊😝😜🤣😜🤪😁😅😝😅😂😅😃😅😊😅😅😅😆😅😝😅😂😅🤪😀😂😅😃😅😊😅😇😃😅😉😜🤣🙃🙃😁😉🤪😂🙂🙂😄🙃😊🤣🤣🙃😀🙂😜😊😝😊😂😅😃😅😊😅🤣😂😀😁😂😀🤪😁😅🤣🙂😂😆😅😊😊😅😅😜😅🙃😅😜😇😆😉😝🤪😆😉🙃😆😃😉🤣😝🙃😅😁😅😝😅😂😅😃😅😜😅😉😇😁😅😀😄😆😁😝😅😂😅😃😅😃😇😊🤣🤣😄😜😉😉🙂😁😉🤪😁🙂🙂😃😉🙂😆😅😝😀🙃😉🙃😆😅😃🤪😃😝😊😅😅😅😜😅🤪😄😁🙂🙂😂😇😄😇😀😄🙂😂🙃😊🙃🤣😉😁😇😝😅😂😅😃😅🤪😊😅🙃😝🙂😁😉😉🙃😄😄🙃😉😅😆😊😊😅😅😜😅🙃😅😅😂😂😉🙃🤪😄😇😇😁😄🙃🤪🙃🙃🤪😁😅😝😅😂😅🙂😇😃😉😊😂😅😝😀😆😉😆😆😀😊🤪😅🙃😁😃🤣😁😜😅🙃😅😁😅😁😜😇🙃😅😃😇😇😅😊😝🙃😅😜🤪🙂😂😂😇😀🙂😜😃😅🙃😃😃😊😄😜😁😉😝😅😂😅😃😅😜😂🙃😝😃😆🙃😃😀🤪😝🙂🙂🤣😃😅😊😅😅😅😄😁🤣😇😃😆😆😅😊😂😆😄😉😊😀🙃🙂🙂😃😉🙂🤪😀😝🤣😊😝🙃😆🤣😇😄😃😅😄😜😁🤪😝😅😂😅😃😅😄😇😊🤣🤣😃😀😀😉🙂😆😅😝😉😂😉😃🙃😊🤪😜😅🙃😅😁😅😝😅😂😅😅😅😀😉🙃😅😊😊🙃😊😁😅😝😅😂😅😁😊😊😝😁😄🙃🙂😃😁🙂😊😁😁😂😂😃😅😊😅😅😅🤣😆😀🙂😃🤣😝😅😂😅😃😅😊😅😅😅😜😅🤪😀😂😅😉😊😂😝😃😅😊😅😅😅😁😝😀😉😉😄😁🤣🤪😁🙂😆😃🤪😊🙃🙂😜🙃🙃😁😅😝😅😂😅🤣😅😃🤣😇😂🤣🙃🙂🙃😁😉😝😅😂😅😃😅😜🙂😇😃🤣😆😀😇😉😁😉😊😂🙃😃😅😊😅😅😅🤣🙃😜🤪😉😄😁😉😅😉😃😝😊😅😅😅😜😅😜😂🙃🤪😆🙃😉😁😂😂😄😝😇😆🤣😂😉🙂😁😜😝😅😂😅😃😅😀😀😊🤪😅🙂😀😀🙂😅😆😃😝🤣🙂🤪😃😉😇😂😂😃🙃😝😁😅😝😅😂😅🤣🙂😃🙂😇😂😅😉😀😁😂😇😆😉😝🤪🙃😇😊😅😅😅😜😅🙃😅😁😅😝😅😊😜😅😅😃😁🤣😀😜😅🙃😅😁😅😃🙂😝🤪🙂🙃😃🤪😇😜😅😉😀😁😉🙂😁😝😝🤪🙂😅🙃🤣😅😅😜😅🙃😅😁😅😝😅😂😅😀😅😜😆🙂🤣😜🤪🙃😅😁😅😝😅😊😂🙂😆😃🙃😇🤪😁😊😀😆😉😂😁🙂😝😉😃🙃😊😝😅😅😜😅🙃😅🙂🙂😁😉🤪🙃🤣🙂😃🙂😇😆🤣😂😜😉😆🙂😝😇😂😅😃😅😊😅🙃🙂🤣😆😀😂🙃😉😃😇🤪😉😂🤪😉🙃🤣😀😜😅🙃😅😁😅😄😃😝😉😂🤣😃🙃😊😉🤣😂😜😂😉😆😁🤣🤪😂😂🙃😄😃😅😊😜😅🙃😅😁😅😜😊🙂😜😃😉😆😇😂😃😇🤣😀😝😁😅😝😅😂😅😃😅😊😅😅😅😆😊😝😅😂😅😝😉😂😅😃😅😊😅😉🙂😅🙂😀😆😉😂😁😉😉😄😃😅😊😅😅😅😜😅🙃😅😃😅😆😆😊😅😉😀😊🙃😅😅😜😅🙃😅🙃😂🤪😄😄😃😂🙂😄😄😅🤪😜😅🙃😅😁😅😄😇😝🤣🙂😃😄😀😇🙂🤣😅😜😉🙃😉😁🙃😉🤣😁🤣😊😅😅😅😜😅😜😇😅😀🙃🙂😝😇😄😂😁🤣😂😁😊😊😆😆😂😝😝😂😄😀😅🙃🙃🤪🤪😝😁😉😆😄😊😜😃😁😅🙃🤣😁🤪🤪😄😅🙂😁🙂😁😜🤪😃😉😉😄😉😉😀😄🤣🤣😊😝😀😂🤪😝😝😝😂😅😃😅😊😅🙃😂🤣😆😜🙃😉🤪😃😇🤪🤪🙂😂😄😆🤣🙂😜🙃🙃😅😁😅😝😅😊😝😂😉😃🤣😊🙃🙂😀🙃😊😁😅😝😅😂😅🤣😄😄😆😇😁🤣😀😜😉😉🤪😊😉😂🙃😃😅😊😅😅😅😁😝😜😉🙃🤣😁🙃🤪😄😄😄😊😅😅😅😜😅😝🤣😂😀🤪🤣😅😀😝😃😅🙃😝😃😄😄😊😊😁😃😉😅😃🤣😉😃😀😁😅😝😜😅🙃😅😁😅🤪😇🙂😁🤪😜😅😇😊🙃😁🤪😉🙃😃😅🤪🙂😂😉😃😅😊😅😅😅😁😊😀😆😉😂😁🙂😝😉😅😅😊😇😅😅😜😅🙃😅😂🙂😁🤪🤪🙃😂🤪😄😜😊😉🤣😁😆😝😆😃😝😅😂😅😃😅😜🤣🙂😂😜😆😆😅😊😂😇🙂😅😄😝🙃😆😀😉😆😀😃🤣😆🤪😇😝🙂😂😅😃😅😊😅🙃😁😀🙃😂🙂😆🙂🤪😀😂😅😃😅😊😅😊😁😃😊😉😊😁😁🙂😅😃😄🙃🙂😀😆🙂😁😜😉😂😄🙃😄😝😜😂😅😃😅😊😅🤣😅😀🙃😂🙂🤪😁😁😆😊😝😄😂🙃😜😜🙃🙂😆😂🙃😁😊😝😅😂😅😃😅😉😇😊😆😉😜😁🙃😜😉🤪🤪😉😅😃😊😊😅😅😅😜😅😁😄😅😄😆😄😜😃😇😄😂😀🤪😊😜😅🙃😅😁😅😝😅😂😅🙃😅😜🙃🙃😅😀😂🙃😊😁😅😝😅😂😅😄🤣😇🙃😁🙂😉🙂😃😁🙂😄🤣🤪😂😊😃😅😊😅😅😅😆😄🤣😆🙂😄🙃🤣😊😉😄😇😀😁😅😊😜😅🙃😅😁😅🤪😂🙃😃😄😁😆😀😜😉😄😆😆😇😁🙂😝😅😂😅😃😅😝🙂😅😅😇🤣😁😝😁🙃😝😅😂😅😃😅🤪😊😅😅😝😃😁🙃😃🙃😝😊😂😅😃😅😊😅😜😝😇😝😝😇😄😆😜😊🙃🤪😃🙂😊😇😅😅😜😅🙃😅😆😂🤪😁😅😄😇😄😄🙃🙃😂😃🙂😅😇😁😜😝😅😂😅😃😅😜😜😇🙂🤣😆😀😁😂🙃😁😉😝🙂🙂😆😃🙃😊😉🙂😜😉🤪😁😅😝😅😂😅😅😅😀🙂🙂🤣😀😀😁😀😝😜😂😂😜😀😂😂😉😊😅😜😜🙃😁😀😝😉🤣😊😜😄🤣😝😉😁😆😆😜😁🤣😜🤪😉🤣😁🤪😊😄😅😜😅🤣😁😜😅🙃😅😁😅😄😁😝😉🙂😇😄🙃😇😆🤣😁😀🙂😉😆😁😊🤪🙃😅😜😀🙂😉😆😆😁🤣😊😁🙃😝😅😂😅😃😅😃🤣🙂😝🤪🙂😁🙃😊😀😝😊😂😅😃😅😊😅😂🙂🤣😜😉🤪😉🙃🤣😀😂😁😁😂😊😉😅😅😜😅🙃😅🙂😉😃🙃😝🤪🙂😄🤪😂🤪😀😜😇🙃😅😁😅😝😅😇😊😂😉😃🙂😇🙃🤣😆😀😂🤣🙂😁😄😂😊😃😅😊😅😅😅😄🙂🙃😅😃😃😂😅🤪😊😅🙃🤪🤣🤣🙃😜😅🙃😅😁😅😄🙂😝😉🙂🙃😀😅😇😂😅🤪😀😄🙃🤣😆😂🤪🤪🤣😅😃🤣😇😂🤣🙃😝🙂😂😊😆😂😝🤣🙂😄😃😉🙂🤣😜😊🙃😅😁😅😝😅😝😝😄😂😂😃😉🙂😜😉😆😃😃🙃🤪🤣😂😅😃😅😊😅😂🤣😜🙃😂😆🤪😀😆😝😊😂😄😅😉🙂😀🙃😂🙂😝😂🤣🙂😝😁😆🤣😉😅😃😊🙂🙃🙃🤪🙃🙂😁😅😝😅😂😅😂🤣😄😅😊🤪😃🤣🙃😜😁😅😝😅😂😅😆😁😊😁😁😂🙃🙃😀😂🙂🙃😜🤣😅🙃😇🙃😆😅🤪😀😜🙂🙃😅😁😅😝😅😊🤣🤣😅😜🤪😊😄😜😊🙃😅😁😅😝😅😄😄😂😄😃🤣🙃🤪😉😜😉😊😄🙃😝🤪😂😅😃😅😊😅🙃😂🤣😆😜🙃😉🤪😃😊🤪😆🙂😂😃🙂😊😉🤣😃🙃😊😁😅😝😅😂😅🤣😇😉😅😝🤪🙃😅🤪🤪🤪😂🙃😇🙂😃😃😅😊😅😅😅😆🙃😀😂🙃🤪😁😇😝😇😂😉😄😂🙃😉🤣😊😜😉😉😁😆🙃🤪😃😄😃😊😅😅😅😜😅😝😊😃🤪😉😂😄😃🤣🙂🤪🤪🤣🙂😝🤣😅😁😊🤣😁😀🙂😅🤣😆😊😝😅😅😜😅🙃😅😂🤪😆😁🤪🙂🙂🙃😃🤣😇😁😅🙂😜😉😝😉😝😅😂😅😃😅😊😅😅😅😜😅😇🤪😂😅😜😂😂😊😃😅😊😅😅😅😁🤣😀😄😉😆😆😉🤪😁🙂🙃😇🙂🤣😅😜😅🙃😅😁😅😃😝🤪😉🙂😄😃🤣😇😁🤣😆😜🤪🙃🙃😄😂🤪😆🙂😆😄🙃😉😅😅🤣😀😂😉🙃🤣😜😂🙃😃😅😊😅😅😅😆😃🙃😇😀😅😅😆😉😄😃😅😊😅😅😅😜😅🙃😅😊😅😃😇😊😅😝😇😊🤣😅😅😜😅🙃😅🙃🤣😉😉😂😊😃😅😊😅😅😅🤪😅🤪😃😄🙃😝🤪😇😉😉🙂😄😃🤣😀😜😅🙃😅😁😅🤪😂🙂😁😜😀🤣😁😝🤣😆😆😉😝😃😂🙂😜😜🙃😂🤣😆😜😅🙂😜😅🙃😅😁😅😄😃😝😉😂😇🙃🙃🤣😃😜😅🙃😅😁😅😜😊😂😅🤪😝🤣🙂😉😉😆😁😊🤣😜😂🙂😇😜😝😂😀😝🙂😄😊😜😉🙃😅😁😅😝😅😊😝🙂😆😄😁😊😉🤣🤪😆🙃😁😅😝😅😂😅😃😅😊😅😅😅😁😅😝😅🙂😆😝😜😂😅😃😅😊😅😊🤪🤣🙂😀😃🙃🙂😆😃🤪😆🙂🙂😄😉😇😂😅😉😁🤪😆😀😝😅😂😅😃😅😀😃😇😆😅🙂😜🤣😉😃😄😅🤪😃😂🤣😄🤪😊😉🤣😂😝😅😁😝😝😅😂😅😃😅😜😝😇🙃🤣🙃😀😅🙂😅😆😆🤪🙂🙂🙃😉🙂😅😉😜😅🙃😅😁😅😃😆😂😂🤪😅🤣😀😝😀🙂😜🙃😇😁😅😝😅😂😅😅😝😄🙃😇🙃🤣😅😝😇🙃😉😆🙃😀😊😃😅😊😅😅😅😜😅🙃😅😊😅😄😊😊😅😝🙂😊😅😅😅😜😅🙃😅😁😅😝😅🙃😝😅😅🙃😉🤣😄😜😅🙃😅😁😅😆🤣😉😀😄😀🙂🤣🤪😜😆😅😊🙃😄🙂🙂😂😜😉🤣😝😇🤣😁😃😉😝🙃🤪😁😅😝😅😂😅🤣😊🤪😃😅🙂😇😁😃😂😉😀😀😃🤣😝😝🙂😀😆🤣😁😜😅🙃😅😁😅😁😜🙂😝😝🙃😃🙂🤣😊😉😝😜😂😆😆🙃😂😉😆😁😂🙂😃😇😃😀😜😄😀😆🤣😝😅😂😅😃😅😀😂😅😉😉😇🤪🤣😁😀😂😂🙂😀😉🙃😃😂🙂😆😇😆😜🤪😜😊😆😇🙃😆😝🙃😃😉😄😉😜😊🙃😅😁😅😝😅😊😅🙂😝😝😜😇😄🙃🙃😇🤣😉😃😝😊😂😅😃😅😊😅😇🙃😂😀😝🤣🙂🙃😝😜🙂😅😆😅😊😊😅😅😜😅🙃😅😅😜😁😃😝😆😆😊😁🙃😄😉😀🙂😉🙃😁😅😝😅😂😅😅😝🤪😉😅😂😇😀😃😅🙂😉😜🙂😀😇🙃🤣😄🤪🙂🙃🤪😝😅😜😂😇😀🤣🙃🙂😀😁😅😜😇😜😁😜😉🙂😁😉😝😅😂😅😃😅😀😄😇😆🤣😁😜😉😉🤪😂🤪🙂😁😃😅😊😅😅😅😁😅😝😁😆😉😊😀😝🙂😂😆😀🙂😂😉😝😜😅🙂😇😀😁😀😉😁😃😇😆😜🤣🤣😜😅🙃😅😁😅😄😂😝😉🙂😅😄😃😊🤪😅🙂😜🤣😉🙃😁😉😝🙃🤣🙂😄🙃😇😆🤣😂😜🤣🙃😇😁😉😆😉😃😉😊😅😅😅😜😅😝🙂😁😅😉🤣😄😀🙂🙃🙃😉😅😜😜😅🙃😅😁😅😃😊😝🤪🙂😂😃😉😉🙂😅😉😀😂😉😊😁😉🤪😂😆🤣😇😃😅😅😜😅🙃😅😂🤪😆😁🤪😊🙂😆😄😀😊😉😆🙂😜😉😉😂😆😊😝😉🙂😂😁🤪😅😅😅😁🙃😊😁😅😝😅😂😅🤣😄😜😀😅😃😇😂😁😃😉😅🤣😊😂😅😃😅😊😅😅😅😜😅🙃😅😂🤣😃😅😜😃😃😊😊😅😅😅😜😅🤪😅😉😃😁🤣🤪🤪😂😉😄😂😜😝😜😊🙃😅😁😅😝😅😆🤪😝😊🤪😆😅🤣😁😀😄🙃😝🤪🤪😄😂😅😃😅😊😅🙃🤣😁😂😊😆😄😅😉😝😁😇😊😊😃😁😊😁😁😀😉😊😄😁😊😅😄😇😄😃😊😅😅😅😜😅😉😝😆😄😊😂😄🙂🙂😅😀😄😂🙂😊🤣😅🤣😊🙃😄😅🙃😜🤣😁😇😁😅😅😜😅🙃😅🤣😅🤪😆😅😆😇😀😁😝😂😜😃😄🙂😁🤪😂😅😜😇🤣😁😅😉😂😃😜😀🙂🙃😊😁😅😝😅😂😅🤣😂😃😉😇😄🤣😆😀🙃🙃😉🙂😇😂😊😃😅😊😅😅😅😜🙃😂😝😆😃🙃😆🙃😇😂🙂😀😀😅😅😜😅🙃😅😁😅😝😅😂😅😁🙃😜😅😊😄😜😊🙃😅😁😅😝😅🙂😄🤪🤣😝😁😁🤪😃😃🙂😜🤣🤣😝😇😂😅😃😅😊😅😊🙃😅🤪😀🙂🙃🙂😆😆🤪😂😂🙃😃😁😅🙃😜😅🙃😅😁😅😄😃😝😉😂😊😄🙃😇😝😜😅🙃😅😁😅😝😅😂😅😃😅😝😃🙃😅😁😃🙃😅😁😅😝😅😂😅😃😅😁😅😉😁😁😅😀😄😁😂😝😅😂😅😃😅😀🙃😉🙂😅😄🙃😊😁😅😝😅😂😅😂🤣😇😃🙃😜😇😀🙂🙂🤪😜😉😊😂🤪😃😅😊😅😅😅😆😇😀😆😉😂😆😀🤪🙂🙂😅😃🤣😊🙂😅😉🙂🙂😁🙂😝😅😂😅😃😅😇🙃😆😂😊😆😄😀😁😝😝😅😂😅😃😅😜😂😇😆😅🙃😀🤪😂😇😆🤪🤪😂🙂😆😆🤪😅😉😜😅🙃😅😁😅😜😀😂😅🤪🤣😅🙃😝😀😆😆🙃🙃😁😅😝😅😂😅😅😝😃😉😊🤣😅🙃😄😜🤣😜😝😅😂😅😃😅😃😇🙃😁😜🤪🤣😅😝😁😀😅🤣😅😝😀😂🤪😝🙂😅🙃😂😊😃😜🙂😇😀😄🤣😃😝🙃😁😃😉😜😃🙂😂😅🤪😝😆🙃😇😀😄😉🙃😊😜😂🤣😊😝😇😀😃😉😃😃🙂🙂😃😉🤪🤣😀😊😉😁😀😅😇😇🤣😁😆🙃🤪😀🤣🙃🙂😃😄🙂🙂😝🤣🤣😄😇🤪😄😉🙃😅😇🙃😂😄😝😃😆😆😊😆🤪😀😂😄😝😁😂😃🙃🤣😁😅😝😅😂😂😉😆😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😄😂😝😂😅😅😜😅🙃😅😝😉😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁🤣😝😅😂😅😃😅😊😅😅😅😀😂🙃😅😁😅😝🤣😂😅😇😀😊😅😅😅😜😅😉😂😁😅😝😅😂🤣😃😅😃😝😅😅😜😅🙃😅😅😊😝😅😂😅😃🤣😊😅😅🤪😜😅🙃🤣😁😅😀😜😂😅😃🙃😊🤣😅😅😜😅🙃😅😁🤪😝😅😂🤣😃😅😊🙃😅🤣😜😅🙃😅😁😅😝😜😂😅😃🤣😊😅😅😅😀😄🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😂😄😆😁😅😝🤣😂😅😄😀😊😅😅😅😜😅😉😂🙃😀😝😅😂🤣😃😅😁😝😅😅😜😅🙃😅😁😅😀😂😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅🤪😂😊🙂😃😅😊😂😅😅😜😝🙃😅😁😅😝😅😂😂😉😆😊😅😅🙂😜😅😉😂😁😅😝😅😂😅😃😅😜😜😅😅😜🙃🙃😅😁🤣😝😅😂😅😃😅😊😅😅😅😜😅🙃😉😁😅😝🙂😂😅😃😅😊😅🤣😂😜😅🙃😅😁😊😝😅🙃🤪😃😅😊😅😅😅😜😅🙃😅😁😅😝😉😂😅😃😂😊😅😅😂😜😅🙃😊😁😅😝😅😂🙃😃😅😃😁😅😅😜🤣🙃😅😁😉😝😅😂🙃😃🤣😊😅😅😅😜😅😜😁😁😅😝🤣😂😅😃😅😀😅😅😅😜😉🙃😅😁🙂😝😅😂😅😃😅😇😂🙂🙂😜😅🙃😊😁😅😉😀😂😅😃😅😊😅🤣😂😜😅🙃😅😁😇😝😅😜😝😃😅😊😅😅😅😀😂🙃😅😁😅😝😝😂😅😆😝😊😅😅😅😜😅😉😂😁😅😝😅😂🤪😃😅🙃😆😅😅😜😅🙃😅😆😂😝😅😂😅😃😜😊😅😂😊😜😅🙃😅😁😅😝😅😂😅😃😅😊😇😅😅😜😜🙃😅😁😂😝😅😂😅😃😅😊😅😅😊😜😅🙃😊😁😅😝😇😂😅😃😅😊😅😅😅😜😉🙃😅😁😂😝😅😂😂😃😅😊😊😅😅😜😅🙃🙃😁😅😁😁😂😅😃🤣😊😅😅😉😜😅🙃🙃😁🤣😝😅😂😅😃😅😃😁😅😅😜🤣🙃😅😁😅😄😅😂😅😃😉😊😅😅🙂😜😅🙃😅😁😅🤪😂😉🙂😃😅😊😊😅😅😇😀🙃😅😁😅😝😅🙂😂😃😅😊😅😅😇😜😅😃😝😁😅😝😅😂😅😄😂😊😅😅😅😜😝🙃😅😝😂😝😅😂😅😃😅😇😂😅😅😜😅🙃🤪😁😅😃🙃😂😅😃😅😊😅🤣😂😜😅🙃😅😁😜😝😅🙂😁😃😅😊😅😅😅😜😅🙃😅😁😅😝😇😂😅😃😜😊😅😅😂😜😅🙃😅😁😅😝😅😂😊😃😅😊😊😅😅😜😇🙃😅😁😅😝😅😂😅😃😉😊😅😅😂😜😅🙃😂😁😅😝😊😂😅😃😅😊🙃😅😅😅😁🙃😅😁🤣😝😅😂😉😃😅😊🙃😅🤣😜😅🙃😅😁😅😁😁😂😅😃🤣😊😅😅😅😆😅🙃😅😁😉😝😅😂🙂😃😅😊😅😅😅😜😂😄😆😁😅😝😊😂😅😃😜😊😅😅😅😜😅🙃😅🙃🙂😝😅😂😉😃😅😊😂😅😅😜😂🙃😅😁😊😀😀😂😅😃😉😊😅🙂🤣😜😅🙃🤣😁😅😝🤣😂😅😃🙃😊🤣😅😅😜😅🙃😅🤣🤣😝😅😂🤣😃😅😊🙃😅🤣😜😅🙃😅😁😅😁😄😂😅😃🤣😊😅😅😅😆😅🙃😅😁😉😝😅😂🙂😃😅😊😅😅😅😀😂🤪😇😁😅😝😊😂😅😁😝😊😅😅😅😜😅😉😂😁😅😝😅😂😇😃😅😇😃😅😅😜😅🙃😅😆😂😝😅😂😅😃😝😊😅😂😀😜😅🙃😅😁😅😝😅😂😅😃😅😊😇😅😅😜😇🙃😅😁😝😝😅😂😊😃😅😊😅😅😇😜😅😇🤪😁😅😝🤣😂😅😃😅😊😅😅🙃😜🤣🙃😅😁😅😝😅😉🤪😃😅😊🤣😅😅😜🙃🙃🤣😁😅😝😅😂😅😅😁😊😅😅🤣😜😅😉😂😆😇😝😅😂😇😃😅😁😝😅😅😜😅🙃😅😆😂😃🙂😂😅😃😝😊😅🙂😊😜😅🙃😅😁😅🤪😂😊🙂😃😅😊🤪😅😅😃😂🙃😅😁😅😝😅🙂😂😅🙂😊😅😅😜😜😅🤪😆😁😅😝😅😂😅😃😅😁🤪😅😅😜😇🙃😅😁😜😝😅😂😂😃😅😊😅😝😆😜😅🙃😊😁😅😝😊😂😅😃😇😊😅😅😊😁😄🙃😅😁😊😝😅😊😂😃😅😊🤣😅😅😜😅🙃😅😁🙃😝🤣😂😅😃😅😊😅🙃😂😜😅🙃🤣😁😅😝🙃😂🤣😃😅😊😅😅😅😁🙂🙃😅😁🤣😝😅🙂😂😄😇😊😅😅😊😜😅😜🙂😁😅😝😅😂😅😃😅😃🙂😅😅😜😉🙃😅😁😂😝😅😂😂😃😅😊😊🤪😀😜😅🙃🙃😁😅😁😁😂😅😃🤣😊😅😅😉😜😅🙃🙃😁🤣😝😅😂😅😃😅😃😁😅😅😜🤣🙃😅😁😅😄😅😂😅😃😉😊😅😅🙂😜😅🙃😅😁😅🤪😂😉🙂😃😅😊😊😅😅😃😝🙃😅😁😅😝😅🙂😂😃😅😊😅😅😇😜😅😃😝😁😅😝😅😂😅😄😂😊😅😅😅😜😝🙃😅😊😅😝😅😂😅😃😅😇😂😅😅😜😅🙃🤪😁😅😀🙂😂😅😃😅😊😅🤣😂😜😅🙃😅😁😜😝😅😁😁😃😅😊😅😅😅😜😅🙃😅😁😅😝😇😂😅😃😜😊😅😅😂😜😅🙃😅😁😅😝😅😂😊😃😅😊😊😅😅😜😇🙃😅😁😅😝😅😂😅😃😉😊😅😅😂😜😅🙃😂😁😅😝😊😂😅😃😅😊🙃😅😅😅😁🙃😅😁🤣😝😅😂😉😃😅😊🙃😅🤣😜😅🙃😅😁😅😁😁😂😅😃🤣😊😅😅😅😆😅🙃😅😁😉😝😅😂🙂😃😅😊😅😅😅😀😂🤪😀😁😅😝😊😂😅😁😝😊😅😅😅😜😅😉😂😁😅😝😅😂😇😃😅😁😝😅😅😜😅🙃😅😆😂😝😅😂😅😃😝😊😅😜😃😜😅🙃😅😁😅🤪😂😂😅😃😅😊🤪😅😅😊🙃🙃😅😁😅😝😅🙂😂😃😅😊😅😅😜😜😅😜😃😁😅😝😅😂😅😃😅😊😅😅😅😜😇🙃😅😁😜😝😅😂😂😃😅😊😅😅😅😜😅🙃😊😁😅😝😊😂😅😃😇😊😅😅😅😜😅🙃😅😁😉😝😅😂😂😃😅😊😂😅😅😜😅🙃😅😁😅😝🙃😂😅😃😉😊😅😅😅😜😅🙃🙃😁😅😝😅😂😅😃😅😃😁😅😅😜🤣🙃😅😁😅😜🤪😂😅😃🙃😊😅😅😅😜😅🙃😅😁😅😝😅😉😂😃😅😊🙃😅😅😜🤣🙃😅😁😅😝😅😂😊😆😀😊😅😅🙃😜😅😜🤣😁😅😝🤣😂😅😃🤣😊😅😅🙃😜🤣🙃😅😁😅😝😅😝🤣😃😅😊🤣😅😅😜🙃🙃🤣😁😅😝😅😂😅😂🤪😊😅😅🤣😜😅😇😅😆😂😝😅😂😉😃😅🤪🤣😅😅😅😀🙃😅😆😂😁😀😂😅😃😉😊😅😀😂😜😅🙃😅😁😅😝😅🙂😄😃😅😊😅😅😅😜🤣🙃😅😁😅😝😅🙂😂😄😇😊😅😅😉😜😅😅😃😁😅😝😅😂😅😃😊😊😇😅😅😜😊🙃😅😆😆😝😅😂🤣😃😅😊🤣😅😅😜😅😄😊😁😅😝😅😂😅😃😂😊😅😅😅😜😅🙃😅😂😉😝😅😂😉😃😅😊😂😅😅😜🤣🙃😅😁🙃😝🤣😂😅😃😅😊😅😊🙃😜😅🙃🤣😁😅😝😅🙂🙃😃😅😊😉😅😅😜😅🙃😅😁🙂😝😅😉😅🙃😅😊😅😅😉😜😅😝😅😁😅😉😁😂😅😆😅😊😅😅😅😜😉🙃😅😇😁😝😅😆😁😃😅🤪😅😅😅😜😅🙃😉😁😅😉🙃😂😅🤪😁😊😅😅😅😜😅🙃😅😁😊😝😅😂😅😃😅😊😂😅😅😀😂🙃😅😁😅😝😇😂😅🤣🙃😊😅😅😅😜😅😉😂😁😅😝😅😂😝😃😅😆😊😅😅😜😅🙃😅😅😅😝😅😂😅😃😝😊😅😅😝😜😅😉🙂😁😅🤪😂😂😅😃😅😊😜😅😅🙃🙂🙃😅😁😅😝😅😂😅😃😅😊😅😅😝😜😅🙃😜😁😅😝😂😂😅😁😅😊😅😅😅😜😝🙃😅😁😝😝😅😅😁😃😅😝😅😅😅😜😅🙃😝😁😅😝😝😂😅😊😅😊😅🤣😂😜😅🙃😅😁🤪😝😅😜😝😃😅😊😅😅😅😀😂🙃😅😁😅😝😜😂😅😝😂😊😅😅😅😜😅😉😂😁😅😝😅🙂😀😃😅🙂🤪😅😅😜😅🙃😅😆😂😝😅😂😅😄😃😊😅😊😜😜😅🙃😅😁😅😝😅😂😅😃😅😊🤪😅😅😀😃🙃😅😁😂😝😅😂😅😃😅😊😅😅😝😜😅🙃😝😁😅😝🤪😂😅😃😅😊😅😅😅😜😇🙃😅😁😂😝😅😂😂😃😅😝😅😅😅😜😅🙃😇😁😅😝😇😂😅😅🤪😊😅😂😅😜😅🙃😅😁😇😝😅😂😇😃😅😝😜😅😅😀😅🙃😅😁😅😝😊😂😅😁😜😊😅😅😇😜😅😉😂😁😅😝😅😂😇😃😅😀🙃😅😅😜😅🙃😅😆😂😝😅😂😅😃😝😊😅🤪😊😜😅🙃😅😁😅😜😅😂😅😃😅😊😝😅😅😜😝🙃😅😆🙂😝😅🙂😂😃😅😊😅😅😜😜😅😁🙂😁😅😝😅😂😅😃😅😊😅😅😅😜😝🙃😅😁😜😝😅😂😂😃😅😝😅😅😅😜😅🙃😝😁😅😝😝😂😅😜😁😊😅😂😅😜😅🙃😅😁😝😝😅😂😝😃😅😅😅😅😅😀😂🙃😅😁😅😝🤪😂😅🙃😝😊😅😅😅😜😅😉😂😁😅😝😅😂😜😃😅😃😊😅😅😜😅🙃😅😆😂😝😅😂😅😄😀😊😅😂😆😜😅🙃😅😁😅🤪😂😂😅😃😅😇😃😅😅😉😄🙃😅😁😅😝😅😂😅😃😅😊😅😅🤪😜😅😉😃😁😅😝😂😂😅😃😅😊😅😅😅😜😝🙃😅😁😝😝😅😂🤪😃😅😊😅😅😅😜😅🙃😇😁😅😝😂😂😅😃😂😊😅😂😅😜😅🙃😅😁😇😝😅😂😇😃😅😜🤪😅😅😃😅🙃😅😁😅😝😇😂😅😃😇😊😅😄😉😜😅😉😅😁😅😝😅😂😊😃😅🙂😉😅😅😜😇🙃😅😆😂😝😅😂😅😃😇😊😅😂🤪😜😅🙃😅😁😅🤪😂😂😅😃😅😊😝😅😅🙂😊🙃😅😁😅😝😅🙃😅😃😅😊😅😅😝😜😅🙃😝😁😅😃🙂😂😅😄😂😊😅😅😅😜😜🙃😅😊😝😝😅😂😅😃😅😇😂😅😅😜😅😉😀😁😅😊😉😂😅😃😅😊😅🤣😂😜😅🙃😅😆😃😝😅😀😅😃😅😊😅😅😅😀😂🙃😅😁😅🤪😄😂😅😁🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😜😃😅😇😄😅😅😜😂🙃😅😁😅😝😅😂😅😃😝😊😅😅😜😜😅🙃😅😁😅😝😅😂😅😃😅😊😇😅😅😜😅🙃😅😁😂😝😅😂😅😃😅😊😅😅😇😜😅🙃🤣😁😅😝😂😂😅😄😂😊😅😅😅😜😝🙃😅😅🤪😝😅😂😅😃😅😇😂😅😅😜😅🙃🤪😁😅🤣😊😂😅😃😅😊😅😂😅😜😅🙃😅😁🤪😝😅😂🤪😃😅😜🙂😅😅😀😂🙃😅😁😅🤪😀😂😅🙃😝😊😅😅😅😜😅😉😂😁😅😝😅🙂😃😃😅😁😂😅😅😜😅🙃😅😆😂😝😅😂😅😄😄😊😅🤪😇😜😅🙃😅😁😅🤪😂😂😅😃😅😇😁😅😅😂😄🙃😅😁😅😝😅😂😅😃😅😊😅🤣😀😜😅😉😁😁😅😝😅😂😅😃😅😊😅😅😅😜🤪🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😝😁😅😝😅😂😅😃😂😊😅😅😅😜😅🙃😅😁😝😝😅😂🤣😃😅😊😂😅😅😃😅🙃😅😁😅😝🤪😂😅😃😇😊😅😇😀😜😅😉😂😁😅😝😅🙂😀😃😅😁😝😅😅😜😅🙃😅😆😂😝😅😂😅😄😃😊😅😄🙂😜😅🙃😅😁😅🤪😂😂😅😃😅😇😄😅😅😂😃🙃😅😁😅😝😅🙂😂😃😅😊😅🤣😁😜😅🤣😅😁😅😝😅😂😅😃😅😊😅😅😅😀😀🙃😅😆😁😝😅😂😂😃😅😇😂😅😅😜😅😉😃😁😅😅😝😂😅😃😅😊😅🤣😂😜😅🙃😅😆😄😝😅😜😇😃😅😊😅😅😅😀😂🙃😅😁😅🤪😁😂😅😇🙂😊😅😅😅😜😅😉😂😁😅😝😅🙂😆😃😅😆🤣😅😅😜😅🙃😅😁😅😝😅😂😅😄😃😊😅🤣😆😜😅🙃😂😁😅🤪😂😂😅😃😅😇😄😅😅😁😃🙃😅😁😅😝😅😂😅😃😅😊😅😅🤪😜😅😉😄😁😅😝😂😂😅😄😂😊😅😅😅😜😜🙃😅😇😊😝😅😂😅😃😅😝😅😅😅😜😅🙃😜😁😅😝😜😂😅😊🙂😊😅😂😅😜😅🙃😅😁😜😝😅😂😜😃😅🤣😊😅😅😀😂🙃😅😁😅🤪😃😂😅🙃😝😊😅😅😅😜😅😉😂😁😅😝😅🙂😄😃😅😂😅😅😅😜😅🙃😅😆😂😝😅😂😅😄😁😊😅😂😅😜😅🙃😅😁😅🤪😂😂😅😃😅😇😆😅😅🤣😄🙃😅😁😅😝😅😂😅🙃🤪😊😅🤣😃😜😅😉😆😁😅😝😂😂😅😃😅😝😜😅😅😜😜🙃😅😆😃😝😅😂😂😃😅😇😂😅😅😜😅😉😀😁😅😅😝😂😅😃😅😊😅🤣😂😜😅🙃😅😆😃😝😅🙃😃😃😅😊😅😅😅😀😂🙃😅😁😅🤪😄😂😅😝😊😊😅😅😅😜😅😉😂😁😅😝😅🙂😁😃😅🙃😄😅😅😜😅🙃😅😁😅😝😅😂😅😄😀😊😅🤣😁😜😅🙃😂😁😅😝😅😂😅😃😅😊😜😅😅😜😜🙃😅😆😀😝😅🙃😅😃😅😊😅😅😜😜😅🙃😜😁😅😀😅😂😅😁😅😊😅😅😅😜😜🙃😅😁😜😝😅😀😃😃😅😇😂😅😅😜😅😉😀😁😅😅😝😂😅😃😅😊😅🤣😂😜😅🙃😅😆😃😝😅😀😄😃😅😊😅😅😅😀😂🙃😅😁😅🤪😄😂😅😂😆😊😅😅😅😜😅😉😂😁😅😝😅🙂😁😃😅😇🤣😅😅😜😅🙃😅😁😅😝😅😂😅😄😀😊😅🤣😁😜😅🙃😂😁😅😝😅😂😅😃😅😊😜😅😅😜😜🙃😅😆😀😝😅🙃😅😃😅😊😅🤣😀😜😅🙃🤪😁😅🤣😉😂😅😄😂😊😅😅😅😀😄🙃😅😊😝😝😅😂😅😃😅😇😂😅😅😜😅😉😁😁😅😄😄😂😅😃😅😊😅🤣😂😜😅🙃😅😆😆😝😅😃😄😃😅😊😅😅😅😀😂🙃😅😁😅🤪😅😂😅😆😁😊😅😅😅😜😅🙃😅😁😅😝😅🙂😄😃😅😇😅😅😅😜😂🙃😅😁😅😝😅😂😅😄😀😊😅🤣😄😜😅🙃😂😁😅😜😅😂😅😃😅😇😃😅😅😜🤪🙃😅😇😉😝😅🙂😂😃😅😊😅🤣😁😜😅😃😝😁😅😝😅😂😅😄😂😊😅😅😅😀😆🙃😅🤪😅😝😅😂😅😃😅😇😂😅😅😜😅😉😅😁😅😀😀😂😅😃😅😊😅🤣😂😜😅🙃😅😆🤣😝😅😇😉😃😅😊😅😅😅😜😅🙃😅😁😅🤪😁😂😅😄🤣😊😅😅😂😜😅🙃😅😁😅😝😅🙂😃😃😅😇😁😅😅😜😂🙃😅😅😅😝😅😂😅😄😄😊😅😅🤪😜😅😄😉😁😅🤪😂😂😅😃😅😇😆😅😅😝🤣🙃😅😁😅😝😅😂😅😃😅😊😅🤣😄😜😅😉😆😁😅😝😂😂😅😁😅😊😅😅😅😀😁🙃😅😁🤪😝😅😀😉😃😅😇😂😅😅😜😅😉😅😁😅😅😝😂😅😃😅😊😅🤣😂😜😅🙃😅😆🤣😝😅😂🤪😃😅😊😅😅😅😀😂🙃😅😁😅🤪😂😂😅🤪😂😊😅😅😅😜😅😉😂😁😅😝😅🙂🙂😃😅🙂😝😅😅😜😅🙃😅😁😅😝😅😂😅😄😅😊😅🤣🙂😜😅🙃😂😁😅😝😅😂😅😃😅😇😁😅😅😀😅🙃😅😁😂😝😅🙃😅😃😅😊😅🤣😆😜😅😉😁😁😅😊🙂😂😅😄😂😊😅😅😅😀🤣🙃😅😊😝😝😅😂😅😃😅😇😂😅😅😜😅😉😂😁😅🙃🙂😂😅😃😅😊😅🤣😂😜😅🙃😅😆🙂😝😅😇😇😃😅😊😅😅😅😀😂🙃😅😁😅🤪🙃😂😅😂😅😊😅😅😅😜😅🙃😅😁😅😝😅🙂🤣😃😅😇🙃😅😅😜😂🙃😅😁😅😝😅😂😅😄😆😊😅🤣🤣😜😅🙃🤣😁😅😜😅😂😅😃😅😇😆😅😅😀😁🙃😅😃🙂😝😅🙂😂😃😅😊😅🤣🤣😜😅😃😝😁😅😝😅😂😅😄😂😊😅😅😅😀😂🙃😅😁😂😝😅😂😅😃😅😇😂😅😅😜😅😉🙂😁😅😀🤪😂😅😃😅😊😅🤣😂😜😅🙃😅😆🙃😝😅😇🤣😃😅😊😅😅😅😜😅🙃😅😁😅🤪🤣😂😅😄🙃😊😅😅😂😜😅🙃😅😁😅😝😅🙂😆😃😅😇🤣😅😅😜🤣🙃😅😅😅😝😅😂😅😄😆😊😅🤣😁😜😅😂🙂😁😅🤪😂😂😅😃😅😇🤣😅😅😂😝🙃😅😁😅😝😅🙂😂😃😅😊😅🤣😂😜😅😇😃😁😅😝😅😂😅😄😂😊😅😅😅😀🙂🙃😅🙃🙃😝😅😂😅😃😅😇😂😅😅😜😅😉🙃😁😅🤪😜😂😅😃😅😊😅😅😅😜😅🙃😅😆🤣😝😅🙂🙃😃😅😊😂😅😅😜😅🙃😅😁😅🤪😆😂😅😄🤣😊😅😅🤣😜😅😊😅😁😅😝😅🙂😆😃😅😇😀😅😅😁😇🙃😅😆😂😝😅😂😅😄🤣😊😅😝😝😜😅🙃😅😁😅🤪😂😂😅😃😅😇😂😅😅😊🤣🙃😅😁😅😝😅🙂😂😃😅😊😅🤣🙂😜😅😄🙃😁😅😝😅😂😅😄😂😊😅😅😅😀🙃🙃😅😉😉😝😅😂😅😃😅😊😅😅😅😜😅😉🤣😁😅🤪🙃😂😅😃😂😊😅😅😊😜😇🙃😅😆😂😝😅😂😅😃😅😊🤣😅😅😜🤣🙃😅😁😅🤣😊😂😅😃😅😊😅😅😉😜😅🙃😅😁😅😝😅🙃🤣😃😅😇😆😅😅😀😂🙃😅😁🤣😝😅🙃😅😃😅😊😅🤣😆😜😅😉😀😁😅😃😇😂😅😄😂😊😅😅😅😀🤣🙃😅😊😝😝😅😂😅😃😅😇😂😅😅😜😅😉😂😁😅🙃😇😂😅😃😅😊😅🤣😂😜😅🙃😅😆🙂😝😅😆😇😃😅😊😅😅😅😀😂🙃😅😁😅🤪🙃😂😅🤣🙂😊😅😅😅😜😅🙃😅😁😅😝😅🙂🤣😃😅😇🙃😅😅😜😂🙃😅😁😊😝😇😂😅😄😂😊😅😅🙃😜😅🙃🤣😁😅😝🤣😂😅😃😅😆😊😅😅😜😅🙃😅😁😉😝😅😂😅😃😅😊😅😂🤣😜😅😉😆😁😅🤪😂😂😅😃🤣😊😅😂😅😜😅🙃😅😆😆😝😅🙂😀😃😅😜😇😅😅😀😂🙃😅😁😅🤪🤣😂😅🙃😝😊😅😅😅😜😅😉😂😁😅😝😅🙂😂😃😅😂🤪😅😅😜😅🙃😅😆😂😝😅😂😅😄🙂😊😅🤪😆😜😅🙃😅😁😅🤪😂😂😅😃😅😇🙃😅😅😉🤣🙃😅😁😅😝😅😂😅😃😅😊😅🤣🤣😜😅😉🙃😁😅😝😂😂😅😃😊😊😇😅😅😀😂🙃😅😆🤣😝😅😂🤣😃😅😊🤣😅😅😜😅😄😊😁😅😝😅😂😅😃😊😊😅😅😅😜😅🙃😅😅🤣😝😅🙂😆😃😅😇😂😅😅😜🤣🙃😅😅😅😝😅😂😅😄😆😊😅🤣😀😜😅😝😇😁😅🤪😂😂😅😃😅😇🤣😅😅😂😝🙃😅😁😅😝😅🙂😂😃😅😊😅🤣😂😜😅😄😜😁😅😝😅😂😅😄😂😊😅😅😅😀🙂🙃😅😝😇😝😅😂😅😃😅😇😂😅😅😜😅😉🙃😁😅😄😃😂😅😃😅😊😅😅😅😜😅🙃😅😆🤣😝😅🙂🙃😃😅😊😂😅😅😜😊🙃😇😁😅🤪😂😂😅😃😇😊😅😅🤣😜😅🙃🤣😁😅😝😅😀😊😃😅😊😅😅😅😜😉🙃😅😁😅😝😅😂😅😁🤣😊😅🤣😆😜😅😉😂😁😅😝🤣😂😅😁😅😊😅😅😅😀😆🙃😅😆😀😝😅😀😂😃😅😇😂😅😅😜😅😉🤣😁😅😅😝😂😅😃😅😊😅🤣😂😜😅🙃😅😆😂😝😅🙂😅😃😅😊😅😅😅😀😂🙃😅😁😅🤪🙂😂😅😊🤪😊😅😅😅😜😅😉😂😁😅😝😅🙂🙃😃😅🙃😃😅😅😜😅🙃😅😁😅😝😅😂😅😄🤣😊😅🤣🙃😜😅🙃😂😁😅😝😊😂😇😃😅😇😂😅😅😜😊🙃😅😁🤣😝😅😂🤣😃😅😊😅🤪😊😜😅🙃😅😁😅😝😜😂😅😃😅😊😅😅😅😁😀🙃😅😆😆😝😅🙂😂😃😅😊🤣😅😅😀😂🤪😄😁😅🤪😆😂😅😉😊😊😅😅😅😜😅😊😅😁😅😝😅🙂😆😃😅😇😆😅😅😀🙂🙃😅😆😂😝😅😂😅😄🤣😊😅😝😝😜😅🙃😅😁😅🤪😂😂😅😃😅😇😂😅😅😄😆🙃😅😁😅😝😅🙂😂😃😅😊😅🤣🙂😜😅🤪😂😁😅😝😅😂😅😄😂😊😅😅😅😀🙃🙃😅😂😄😝😅😂😅😃😅😊😅😅😅😜😅😉🤣😁😅🤪🙃😂😅😃😂😊😅😅😅😜😅🙃😅😆😆😝😅🙂🤣😃😅😊😂😅😅😃😅🙃😅😁😅🤪😆😂😅😄😆😊😅😉🤪😜😅😊😅😁😅😝😅🙂😆😃😅😇😆😅😅🙂😝🙃😅😁😊😝😇😂😅😄🤣😊😅😅🤪😜😅🙃🤣😁😅😝😂😂😅😃😅😆😊😅😅😜😅🙃😅😁😉😝😅😂😅😃😅😊😅🤪😊😜😅🙃😅😁😅😝😜😂😅😃😅😊😅😅😅😃🤣🙃😅😆😆😝😅🙂🤣😃😅😊🤣😅😅😃😅🙃😅😁😅🤪😆😂😅😄😃😊😅🙃😇😜😅😉😂😁😅😝😅🙂🤣😃😅😁😝😅😅😜😅🙃😅😆😂😝😅😂😅😄😂😊😅😄😆😜😅🙃😅😁😅🤪😂😂😅😃😅😇🙂😅😅😀😄🙃😅😁😅😝😅🙂😂😃😅😊😅🤣🙃😜😅😇😉😁😅😝😅😂😅😃😅😊😅😅😅😀🤣🙃😅😆🙃😝😅😂😂😃😅😊😂🤪😆😜😅😉😂😁😅😝😂😂😅😃😅😊😅😅😅😁😀🙃😅😆😆😝😅🙂😂😃😅😊🤣😅😅😃😅🤪🤣😁😅🤪😆😂😅😄😄😊😅🙃😇😜😅😉😂😂🙂😝😅🙂🤣😃😅🙃🤣😅😅😜😅🙃😅😁😊😝😇😂😅😄😂😊😅🤣😃😜😅🙃🤣😁😅😝🤣😂😅😃😅😆😊😅😅😜😅🙃😅😁😝😝😅😂😅😃😅😊😅😂🤣😜😅😉😆😁😅🤪😂😂😅😃🤣😊😅😂😅😜😅🙃😅😆😆😝😅🙂😄😃😅😜😇😅😅😀😂🙃😅😁😅🤪🤣😂😅🙃😝😊😅😅😅😜😅😉😂😁😅😝😅🙂😂😃😅😀😀😅😅😜😅🙃😅😆😂😝😅😂😅😄🙂😊😅🙃😁😜😅🙃😅😁😅🤪😂😂😅😃😅😇🙃😅😅😊😆🙃😅😁😅😝😅😂😅😃😅😊😅🤣🤣😜😅😉🙃😁😅😝😂😂😅😃😊😊😇😅😅😀😂🙃😅😆😅😝😅😂🤣😃😅😊🤣😅😅😜😅😄😊😁😅😝😅😂😅😃😝😊😅😅😅😜😅🙃😅😅🤣😝😅🙂😆😃😅😇😂😅😅😜🤣🙃😅😅😅😝😅😂😅😄😆😊😅🤣😄😜😅😝😇😁😅🤪😂😂😅😃😅😇🤣😅😅😂😝🙃😅😁😅😝😅🙂😂😃😅😊😅🤣😂😜😅😊🙂😁😅😝😅😂😅😄😂😊😅😅😅😀🙂🙃😅😆😀😝😅😂😅😃😅😇😂😅😅😜😅😉🙃😁😅🙂😝😂😅😃😅😊😅😅😅😜😅🙃😅😆🤣😝😅🙂🙃😃😅😊😂😅😅😜😊🙃😇😁😅🤪😂😂😅😄😁😊😅😅🤣😜😅🙃🤣😁😅😝😅😀😊😃😅😊😅😅😅😜😝🙃😅😁😅😝😅😂😅😁🤣😊😅🤣😆😜😅😉😂😁😅😝🤣😂😅😁😅😊😅😅😅😀😆🙃😅😆😄😝😅😊😇😃😅😇😂😅😅😜😅😉🤣😁😅😅😝😂😅😃😅😊😅🤣😂😜😅🙃😅😆😂😝😅😀😀😃😅😊😅😅😅😀😂🙃😅😁😅🤪🙂😂😅😄😊😊😅😅😅😜😅😉😂😁😅😝😅🙂🙃😃😅😇😝😅😅😜😅🙃😅😁😅😝😅😂😅😄🤣😊😅🤣🙃😜😅🙃😂😁😅😝😊😂😇😃😅😇😂😅😅😜😝🙃😅😁🤣😝😅😂🤣😃😅😊😅🤪😊😜😅🙃😅😁😅😝😝😂😅😃😅😊😅😅😅😃🤣🙃😅😆😆😝😅🙂😂😃😅😊🤣😅😅😃😅🙃😅😁😅🤪😆😂😅😄😄😊😅🙃😇😜😅😉😂😁😅😝😅🙂🤣😃😅😁😝😅😅😜😅🙃😅😆😂😝😅😂😅😄😂😊😅😅😜😜😅🙃😅😁😅🤪😂😂😅😃😅😇🙂😅😅🙂🙂🙃😅😁😅😝😅🙂😂😃😅😊😅🤣🙃😜😅😉🤪😁😅😝😅😂😅😃😅😊😅😅😅😀🤣🙃😅😆🙃😝😅😂😂😃😅😊😊😅😇😜😅😉😂😁😅🤪😄😂😅😃🤣😊😅😅🤣😜😅🙃😅😇😊😝😅😂😅😃😅😊😝😅😅😜😅🙃😅😁😅😜🤣😂😅😄😆😊😅🤣😂😜😅🙃🤣😁😅😜😅😂😅😃😅😇😆😅😅😀😄🙃😅😂😇😝😅🙂😂😃😅😊😅🤣🤣😜😅😃😝😁😅😝😅😂😅😄😂😊😅😅😅😀😂🙃😅🙃😂😝😅😂😅😃😅😇😂😅😅😜😅😉🙂😁😅🙃😉😂😅😃😅😊😅🤣😂😜😅🙃😅😆🙃😝😅😝😅😃😅😊😅😅😅😜😅🙃😅😁😅🤪🤣😂😅😄🙃😊😅😅😂😜😅🙃😊😁😇😝😅🙂😂😃😅😊😉😅😅😜🤣🙃😅😁🤣😝😅😂😅😉😊😊😅😅😅😜😅🙃😝😁😅😝😅😂😅😃😅😝🤣😅😅😀😆🙃😅😆😂😝😅😂🤣😃😅😝😅😅😅😜😅😉😆😁😅🤪😄😂😅😇🙃😊😅🤣😂😜😅🙃😅😆🤣😝😅😜😝😃😅😊😅😅😅😀😂🙃😅😁😅🤪😂😂😅😂😇😊😅😅😅😜😅😉😂😁😅😝😅🙂🙂😃😅🤪😄😅😅😜😅🙃😅😆😂😝😅😂😅😄🙃😊😅🙂😂😜😅🙃😅😁😅😝😅😂😅😃😅😇🤣😅😅😀🙃🙃😅😁😂😝😅🙂😂😄😇😊😅🤣😂😜😅😜😁😁😅😝😅😂😅😁😅😝🙃😅😅😀😂🙃😅😆😂😝😅😂😊😃😅😇😂🙃🙂😜😅😉🙂😁😅😅😆😂😅😃😅😊😅🤣😂😁🙂🙃😅😆🙃😝😅🤪😇😃😅😊😅😅😅😀😂😝🙂😁😅🤪😉😂😅🙂😇😊😅😅😅😜😅🙃😅😊🤪😝😅🙂😂😃😅😇😉😅😅😜😂🙃😅😁😊😝😇😂😅😄🙂😊😅😅🙂😜😅🙃🤣😁😅😝🤣😂😅😃😅😆😊😅😅😜😅🙃😅😁😝😝😅😂😅😃😅😊😅🙃😀😜😅😉😆😁😅🤪🙂😂😅😃🤣😊😅😅😅😀😄🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😀🙂🙃😅😁😅😝😅😂🤣😃🙃😊😅😅😅😜😅😝😆😁😊😝😅😂😅😃😅😁😄😄🤪🙃😂😂🙂😂😆😂😆🙃🤣😃😝😊😅😅😅😜😅😉😇😆😁😉😜😃😇😂🙃😜🤪🤣🙃😝😅🙂🙃😁🤪😝😅😂😅😃😅😀😊😅😝😊😀😜😂😄😂🙂😇😇🙃😃😃😅🙂😜😉😜😅🙃😅😁😅😝😅😂😅😃😅😀😊🙃😅😜😝🙃😅😁😅😝😅😂😅🙂🙃😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😄😂😊😅😅😅😜😂🙃😅😁😂😝😅😂😅😃😅😇😂😅😅😜😅🙃🙂😁😅😝🙂😂😅😃😅😊😅🤣😂😜😅🙃😅😁🙃😝😅😂🤣😃😅😊😅😅😅😀😂🙃😅😁😅😝😉😂😅😃🙃😊😅😅😅😜😅🙃😅😁😅😝😅😂😂😃😅😊😉😅😅😜😂🙃😅😁😅😝😅😂😅😃🤣😊😅😅😂😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜🤣🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😂🙃😂😁😅😝😅😂😅😝🙂😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅🤣😇😅😉😜😅🙃😅😁😅😃😆😂😂🤪😅🤣😀😝😀😃😄🙃😅😁😅😝😅🙂😂😅🙂😊😅😅😂😜😅🙃🤣😁😅😝😅😂😅😄😂😜🙂😅😅😜🙂🙃😅😁🤣😝😅😂😅😃😅😇😂🤣😇😜😅🙃🙃😁😅😝😂😂😅😃😅😊😅😅😊😜😇🙃😅😁😉😝😅😂😅😃😅😊🤣😅😅😜🙃🙃😅😁😅🤣😊😂😅😃😅😊😅😅😂😜😅🙃😅😁😅😝😅😀😊😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😉😊😊😅😅😅😜😅🙃🙂😁😅😝😅😂😅😃😅😆😊😅😅😜😅🙃😅😁🤣😝😅😂😅😃😅😊😅🙃😉😜😅🙃🙃😁😅😝😂😂😅😃🤣😊😅😅😊😄😄🙃😅😁😅😝😅🙂😃😃😅😊🤣😅😅😜🤣🙃😅😁🙃😝🤣😂😅😃😅😊😅🤣😃😜😅🙃🤣😁😅😝🙃😂🤣😃😅😊😅😅😅😀🤪🙃😅😁🤣😝😅😂😊😆😄😊😅😅😅😜😅😉😆😁😅😝🤣😂😅😃🤣😊😅😅🙃😜🤣🙃😅😁😅😝😅🙂😆😃😅😊🤣😅😅😜🙃🙃🤣😁😅😝😅😂😅😄🤪😊😅😅🤣😜😅🙃😊🙂😂😝😅😂🤣😃😅😇🤪😅😅😜🤣🙃😅😁😅😝😅😂🙃😃🤣😊😅😅😅😜😅😉🤪😁😅😝🤣😂😅😃😅😝😇😅😅😜🙃🙃😅😁😂😝😅😂🙂😃😅😊😅😂😇😜😅🙃😉😁😅😝🙂😂😅😃😂😊😅😅😊😆😂🙃😅😁🙃😝😅🙂🤪😃😅😊🤣😅😅😜😉🙃😅😁🙃😝🤣😂😅😃😅😊😅🤣🤪😜😅🙃🤣😁😅😝😅🙃😇😃😅😊🙃😅😅😜😂🙃😅😁🙂😝😅😂😅😁😇😊😅😅😉😜😅🙃🙂😁😅😝😂😂😅😃😊🤪😄😅😅😜🙃🙃😅😆😜😝😅😂🤣😃😅😊😉😅😅😜🙃🙃🤣😁😅😝😅😂😅😄😜😊😅😅🤣😜😅🙃😅😅🤪😝😅😂🙃😃😅😊😅😅😅😜😅🙃😅😁😅😀😂😂😅😃🙃😊😅😅🤣😜😅🙃😅😁😅😝😅😂😂😃😅😊🙃😅😅😜😂🙃😅😁😅😝😅😂😅😄😄😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😇😅😅😜😅🙃😅😁😅😅😇😂😅😃😅😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁🙂😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😂😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😝😅😅😜😅🙃😅😄🙂😝😜😂😅😃😅😊😅🤣😊😜🙃😂🙂😇😂😆🙃😇🤣😁😉🙃😝😀😂😂🙃🙃🙃😁😜😝😅😂😅😃😅😀🤣🙂😉🤪😆😄😉🙃😝😀😊😂😀😇😆😃😜🙂😄😝🙃🙃😇😁😅😝😅😂😅😆😂😇😁😁😄😉😄😀🙃😂😂😜🙂😜😀😃😝😊😅😅😅😜😅😉😇😆😁😉😜😃😇😂🙃😜🤪🤣🙃😝😅😆🙃😁😅😝😅😂😅😃😅😊😅😝😅😆😆😝😅😂😆😝😊😂😅😃😅😊😅😝🙂😄😁😊🙂🙂😃🤣😝🙂😁😊😝😊😇😅😅😜😅🙃😅😅😂🤪🙂😆🙃😇😆😁😆🙃🙃😀🙂😀😃😁🙃😝😅😂😅😃😅🤪😊😅😅😝😃😁🙃😆😁😝😅😂😅😃😅😇😂😉😄😜😅🙃🤣😁😅😝😝😂😅😃😅😊😅😂😅😜😅🙃😅😁🤣😝😅😂🤣😃😅😊🤣😅😅😀😂🙃😅😁😅😝🙂😂😅😃🙃😊😅😅😅😜😅😉😂😁😅😝😅😂🙃😃😅😊😂😅😅😜😅🙃😅😆😂😝😅😂😅😃😉😊😅😅😊😜😅🙃😅😁😅🤪😂😂😅😃😅😊😊😅😅😜😉🙃😅😁😅😝😅😂😅😃😅😊😅😅🙂😜😅🙃😊😁😅😝😂😂😅😃😅😊😅😅😅😜🤣🙃😅😁🙂😝😅😂😂😃😅😝😅😅😅😜😅🙃🤣😁😅😝🤣😂😅😃😇😊😅😂😅😜😅🙃😅😁🤣😝😅😂🤣😃😅😊🙂😅😅😜😊🙃😇😁😅😝🙂😂😅😃😅😊😅😅🤣😜😅🙃🤣😁😅😝😅😀😊😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😅😀😊😅😅🤣😜😅🙃🙂😁😅😝🤣😂😅😃😅😇😄😅😅😜😅🙃😅😁🤣😝😅😂😅😃😅😊🤣😅😅😜😅🙃😅😁😅😝🤪😂😅😃😅😊😅😆🙂😜🙃🙃😅😁😅😝😅🙂😇😃🙃🙂😅😜🙂😝🙃🙃😜😁😅😝😅😂😅😁😀😇😁😁😊🙃🙃😃🤣🤣🙂😜😁🤣🤣😝😂😆😁🤣😂😜😇🙃😅😁😅😝😅🙃😆😄😀🙂😅😀😝🤣🙃🤪🤣😅😂🤣🤪😂😜😃😅😊😅😅😅😃🙃😉😆😃😆😂🙃😜🤣😆🙂😝😁😆🤣😊😂😄😁😇🤪😝🙃😂😅😃😅😊😅🙂😊😜😅😂😃😝🙃🤣🤪🙂😅😃😅😊😅😅😅😀😇😉🙃😃😃😂😅😜😄🤣😁😇😝😁🙂🙃🤣😄😁🙃😁😀🙂😅😆😝😅😅🤣😇🙂😄😅🙃😜😁😅😝😅😂😅😄😂😊😅😁😄😂😂😃😁🙂😀😜😀😅😝😇🙂😁🙃😝😀😜🤪🙃😅😁😅😝😅🙂😂😃😇🙂😅😀🤣🤣😅😝😂😅🙂😊🙃😁🤣😅😆😇😀😅😅😜😅🙃😅😅😀🤪😁😆😂😊😅😁😀🙂😆😃😀😂😅😜😝😅🙃😝🤣😄😝😊😅😅😅😜😅😉😂🙂😃😝😅😂😅😃😅😊😉😅😅😜😅🙃😅😅😅😝😅😂😅😃😅😊😅😅😅😜😅🙃🙂😁😅😜😅😂😅😃😅😊😅😅😅😜😅🙃😅😁🤪😝😅🙃😅😃😅😊😅😅🤣😜😅🙃😅😁😅😝😝😂😅😁😅😊😅😅😅😜🤣🙃😅😁🤣😝😅😂😊😃😅😊😅😅😅😜😅🙃😂😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😂😝😅😂😂😃😅😊😅😅😅😀😅🙃😅😁😅😝🤣😂😅😃😇😊😅😅😂😜😅😊😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😝🙃😅😅😅😝😅😂😅😃🤣😊😅😅🤣😜😅🙃🤣😁😅😝😅😂😅😃😅😊😂😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😂😜😅🙃😂😁😅😝😅😂😅😄😅😊😅😅😅😜🤣🙃😅😁😇😝😅😂😂😃😅😝😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃😝😊😅😂😅😜😅🙃😅😁🤣😝😅😂🤣😃😅😊😂😅😅😜😅🙃😅😁😅😝😂😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😂😃😅😊😂😅😅😜😅🙃😅😆😅😝😅😂😅😃🤣😊😅😅😇😜😅🙃😂😁😅😜😅😂😅😃😅😊🤣😅😅😜😅🙃😅😁😝😝😅🙃😅😃😅😊😅😅🤣😜😅🙃🤣😁😅😝🙃😂😅😃😅😊😅😅😅😜😂🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😂😁😅😝😂😂😅😃😅😊😅🤣😅😜😅🙃😅😁🤣😝😅😂😇😃😅😊😂😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊🤣😅😅😜😅🙃😅😀😉🤪😀😂😅😃😅😊😅😂😂😜🙃😂🙂😊🙃😁😂😉😆😀😂😉😁😃😀🙂😁😜🤣😁😉😝😅😂😅😃😅😊😅😝😁😜😅🙃🤣😁😅😝😅😂😅😃😅😊😅😂😅😜😅🙃😅😁🤣😝😅😂🤣😃😅😊🤣😅😅😜😅🙃😅😁😅😝🙂😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😅😊🙂😅😅😜🤣🙃😅😁😅😝😅😂😅😃😅😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅🤣😀😁🙃😅😁😅😝😅😀🤪😃🤪😊😅😅😅😜😅😝😊😆😁😊🤣😃😜🙃😂😀😆🤣😅😝😂😆🙃😜😊🤪😃😂😅😃😅😊😅😂😂😜🙃😂🙂😊😅😅🙂😇🙂😁🤣🙃😝😀🤣🙂🙃😜🙂😅🙃😃😆😂😊😃😅😊😅😅😅😊😄😝😄🙂😅🤣🤪😃😀🙃🙃😜😅🤣😃😜😅🙃😅😁😅😃😝🙂😅😇😝😃😃😆😃🙃😃😝😉🤪😊😅🤪🙃😆😝😁😄😀😅🙂😜🙂🙃😅😁😅😝😅🙃😃😃😅🙃😆😆😅😜😅🙃😅😁😅😝😅😂😅😃😅😀🤣🙃😅🙃😝😉😀😁😅😝😅😂😅😄😊😊🙃😁🙂😂😂😀😇😂😝😜😀😅🙂😝🤣😁🙃😊😄🤪🙂🙃😊😁😅😝😅😂😅😄🤣😇😀😁😁🙃😂😀😜🙂😂🤪🙂😂😊😃😅😊😅😅😅😁😇😊😅😀😝😂😆😝🙂😄🙂😉🙂😅😝😜😅🙃😅😁😅🤪😇🙂😁🤪😜😅😇😊🙃😁🤪😉🙃😃😅😃😃😂😅😃😅😊😅😅😅😜😅🙃😅🙂🤪😃😅😜😀😃😊😊😅😅😅😜😅🙂😆😝🙃😄🤪🤪😃🤪😁😀😅😀😇😜😉🙃😅😁😅😝😅😊😆😃😅🙂😝😀🤣😂😂🙃😅😁😅😝😅😂😅😃😅😊😅🙃😅😆😇😝😅😅😁😝😅😂😅😃😅😊😅🙃😁😜😅🙃🤣😁😅😝😅😂😅😃😅😊😅🤣😂😜😅🙃😅😁😂😝😅😂😜😃😅😊😅😅😅😀😂🙃😅😁😅😝🙂😂😅😃🙃😊😅😅😅😜😅😉😂😁😅😝😅😂🙃😃😅😊🙂😅😅😜😅🙃😅😆😂😝😅😂😅😃😉😊😅🤣😁😜😅🙃😅😁😅😝😅😂😅😃😅😊😂😅😅😜😉🙃😅😁😂😝😅😂😅😃😅😊😅😅🤣😜😅🙃😂😁😅😝😅😂😅😃😊😊😅😅😅😜😅🙃😅😁😜😝😅😂🤣😃😅😊🤣😅😅😜🙃🙃🤣😁😅😝😅😂😅😃😜😊😅😅🤣😜😅🙃🙃😁🤣😝😅😂😅😃😅😇😀😅😅😜🤣🙃😅😁🙃😝🤣😂😅😃😅😊😅😂😄😜😅🙃🤣😁😅🤪😂🙂😇😃😅😊🤣😅😅😀😄🙃😅😁😅😝😅🙂😂😃😊😊😅😅😂😜😅🙃🤣😁😅😝😅😂😅😁😅😊😅😅😅😜😂🙃😅😁😂😝😅😂😉😃😅😝😅😅😅😜😅🙃😂😁😅😝😂😂😅😃😝😊😅😂😅😜😅🙃😅😁😂😝😅😂😂😃😅😊😇😅😅😜😅🙃😅😁😅😝😂😂😅😃🙂😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜🙂🙃😅😁🙃😝😅😂😅😃😅😊😅😂😀😜😅🙃🤣😁😅😜😅😇🤣😃😅😊😊😅😅😜😉🙃😅😁😂😝😅🙂😂😉😃😊😅😅😝😜😅🙃😜😁😅😝😅😂😅😄😂😊😅😅😅😜🤪🙃😅😁🤪😝😅😂😅😃😅😇😂😅😅😜😅🙃😜😁😅🤪😃😂😅😃😅😊😅🤣😂😜😅🙃😅😆😀😝😅😂😊😃😅😊😅😅😅😜😅🙃😅😁😅😝😝😂😅😄😀😊😅😅😂😜😅😉😂😁😅😝😅😂🤪😃😅😇😀😅😅😜😅🙃😅😁😅😝😅😂😅😃😊😊😅😅🤪😜😅🙃🤣😁😅😝😊🤪😊😃😅😊🤣😅😅😀🙂🙃😅😁🤣😝😅😂😂😃😅😊🙃😅🤣😜😅🙃😅😁😅🤪🙂😂😅😃🤣😊😅😅😅😀😄🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃🤣😊😅😅😅😜😅😃😜😆😄😝😅😂😅😃😅😝😄😅😅😝😃😁🙃🙂🙃😀😂😂😆🤪🙂😂😁😝😊😆😊😇😀😄🙃😝😉😂😅😃😅😊😅😅😅😂😁🙃😅😁🤣😝😅😂😅😃😅😊😅😅😅😃😅🙃😅😁😅😝🤣😂😅😃🤣😊😅😅🤣😜😅🙃😅😁😅😝😅😂🙂😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅🙂😜😅🙃🤣😁😅😝😅😂😅😃😅😊😅😅😅😜🤣🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅😉😅😁😅😝😅😂😅🤣😆😊🙃😅😅😜😅🙃😅🙃😝😀😊🤣😝😝😃🤣😇😅😉😜😅🙃😅😁😅😃😆🙂🤣🤪😝😅😂😇🙃😆😁🙃😊😁😅😝😅😂😅😇😊🤪🤣😀🙃🙂😄🙂🤣😁🙃😆😃😂😊😃😅😊😅😅😅😇😄😆😁🤣😀🙂😝😆😜🙂😝😅😝🤣🙂😜😅🙃😅😁😅🤪🤣😂🙃🤪🙂🤣😊😇😅😆🤣😊😂😀😆🙃🙃😀🤣🤣😂😝😇😆😅😇😂😄🙃😂😝😃🙂😂🙃😜😃😅😉😅😅😜😅🙃😅😁😅😝😅😜😅🤣😄😜😅🙂😅😜😊🙃😅😁😅😝😅😉😅😄😃🙃😁😀🙃😂😄🤪🙂😊😇😝😅😂😅😃😅😊😅😅😅😜😅😇😅😂😅😝🙂😂😝😃😅😊😅😅😅😀😇😉😁😀😜😂😇😝🙃😅🤪😇🙃😁😅😅😁🙃😜😁😅😝😅😂😅🙂😊😜😝😂😂🤪🤪😅😊🤪😝😅🙂😝😁😅🙂😇😁😆🙂😜😝🙃😅😁😅😝😅😉😝😄🙂🙂🙃😀😃😅🙂🤪😝😅😆😊🙃😅🤣😃😅😊😅😅😅😜😅🙃😅😁😅😃🙂😊😅😜🤣😊😅😅😅😜😅🙃😅😁😅😝😅🙃😅😅😅😉🙂🤣😃😜😅🙃😅😁😅🤪😝🙂😄😜😉🤣😁😇😜😁🙃🙃😂😃🙃🙃🤣😀😉🤣🙃🤪🤣😜😝😜😝🙃😅😁😅😝😅😉😂😄🙃🙃🤣😀🤣🤣🙃🤪😄😆😂😇😝🤪😃😃😝😊😅😅😅😜😅😝😂😁😇😊😁😄😆🤣😝😀🙂🤣🙃🤪😃😉😊😁😅😝😅😂😅😃😅😝🙂😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😝😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😉😊😅😂😅😜😅🙃😅😁😅😝😅😂😅😃😅😇😁😅😅😜😅🙃😅😁😅😝😂😂😅😃😅😊😅😅🤣😜😅🙃😅😁😅😝😅😂🙂😃😅😊😅😅😅😜🙃🙃😅🤣😅😝😅😂😅😃🙂😊😅😅😂😜😅😉😄😁😅🤪😂😂😅😃😅😊🙃😅😅😜🤪🙃😅😁😅😝😅🙂😂😃😅😊😅😅😉😜😅🙃🤣😁😅😝😅😂😅😄😂😊😅😅😅😜😊🙃😅😁🙃😝😅😂😅😃😅😇😂😅😅😜😅🙃😇😁😅😝😊😂😅😃😅😊😅😅😅😜😅🙃😅😁🙃😝😅😂😇😃😅😊😂😅😅😀😅🙃😅😁😅😝🙂😂😅😄😆😊😅😅🙃😜😅😉😂😁😅😝😅😂🙃😃😅😊🤪😅😅😜😅🙃😅😆😂😝😅😂😅😃😉😊😅😅😜😜😅🙃😅😁😅🤪😂😂😅😃😅😊😊😅😅😜🙂🙃😅😁😅😝😅🙂😂😃😅😊😅😅😇😜😅😉😃😁😅😝😅😂😅😃😅😊😅😅😅😜🙃🙃😅😁😇😝😅😂😂😃😅😇😅😅😅😜😅🙃🙂😁😅🤪😀😂😅😃🙃😊😅🙂😅😜😅🙃😅😁🙂😝😅😂😇😃😅😊😝😅😅😀😅🙃😅😁😅😝😂😂😅😄😅😊😅😅🙂😜😅🙃😅😁😅😝😅😂😅😃😅😊😂😅😅😜🤣🙃😅😁😅😝😅😂😅😃😅😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅🤣😜🙃🙃😅😁😅😝😅😁😊😃😝😊😅😅😅😜😅😉😇😆😁😉😜😃😇😂🙃😜🤪🤣🙃😝😅🤣😉😁😊😝😅😂😅😃😅😊🙂😁😝😝😊😆😆🙃😄😇😜🙂😆😃😅😊😅😅😅😜😅🙃😅😁😅😄😀😊😅😄🙂😊😜😅😅😜😅🙃😅😂😅😃😁😂😇🤪😃😄😇😉😜😃😁🙂😅😝🙃😅🤪😂😝😃😅😊😅😅😅😜😅😀🙃😁😅😝🤣😂😅😃😅😊😅😅😅😜😅😉😂😁😅😝😅😂😂😃😅😊🤣😅😅😜😅🙃😅😆😂😝😅😂😅😃🙂😊😅😅🙃😜😅🙃😅😁😅🤪😂😂😅😃😅😊🙃😅😅😜😂🙃😅😁😅😝😅🙂😂😃😅😊😅😅😉😜😅🙃🙂😁😅😝😅😂😅😃😅😊😅😅😅😜😂🙃😅😁😉😝😅😂😂😃😅😊😅😅😅😜😅🙃🤣😁😅😝😂😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😂😊😅😅😅😜😅🙂🙂😆😆😝😅😂😅😃😅😇🤣🤣😁😝😇😃😉😉😝😄😀🙃😀😜🙃🤣🙂😇🙂😅😁😊😊😄😊😉😀😀🙃🙃😀😇😃😅😅😜😅🙃😅😆🤣🤪😁😅😇🙃🙃😃😂🙂😆😜🙂🙂😁🤪😊😅😊😝😀😁🙃😇😄😅😅😜😅🙃😅😁😅🤣🤪😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😜😅😂😅😃😅😊🤣😅😅😜🤣🙃😅😁😂😝😅😂😅😃😅😊😅😅🙂😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜🤣🙃😅😁🙂😝😅😂🤣😃😅😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃😅😊😅😂😅😜😅🙃😅😁🤣😝😅😂🤣😃😅😊🤣😅😅😜😅🙃😅😁😅😝🙂😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😅😊🙂😅😅😜🤣🙃😅😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😜😅😂😅😃😅😊🤣😅😅😜🤣🙃😅😁😂😝😅😂😅😃😅😊😅😅🙂😜😅🙃🤣😁😅😝😅😂😅😃😅😊😅😅😅😜🤣🙃😅😁🙂😝😅😂🤣😃😅😊😅😅😅😜😅🙃😅😁😅😝🤣😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅🙃🤪😃😅😊😅😅😅😉😇🙃😉😁😅😝😅😂😅😅😆😊😅😄😝😉🤣😃😂😁🙃😝🤪😂😅😃😅😊😅😉😊😆🙂🤪🤣😂🙂😜😁😉😁😄😊😊🤪😅😀🤪😅🙃😅😁😅😝😅😂😅😃😅😊😅😂🙃😁😅😅😊😁😊😝😅😂😅😃😅😁😊😄😅🙂😅😆😀😁🙃😂😂😄😇😃🤪😊😅😅😅😜😅😉😂😁😇😉😅😄🤣🙂😅😜😂😂🙂😝🙃😅🤣😅🤣😝😇😂😅😃😅😊😅😂😆😀😀🤣😅🤪😝😆🙃😇🤣😁😂😅😝😅😝😜😅🙃😅😁😅🤪😇🙂🙃😜😃😅😅😝😄😆😁😉😝😃🙂😜😂😂😜😃😅😊😅😅😅😁😅😜😆😂🤪😀😆😂😀😃😆🙃🙃😄😇😉😃😆😂😉😃😝😊😂😅😃😅😊😅😂😉🤪🤪😁😃😁🙂😉😄🤣😁😆😅😇😅😅😅😜😅🙃😅😆😇🤪🙃😅😃😊😅😁😄😉😁😀😝😂🙂😝🤣🤣😁😝😁😆🙂🙃😆😃😅🙃🤣😀🙂😆😂😝🙃😂😅😃😅😊😅🙃😃😜😅😂🙂😝😇😝🙂😂😝😃😅😊😅😅😅😃😆😉😁😃😂😂😝😜🙂😅😝😝😁😆😄😁😆🙃🤪😁😅😝😅😂😅😆😂😇😁😁🤣😉😁😃🙃🙂🙂🤪😝🤣😄😇🙃🙂😉😅😊😜😅🙃😅😁😅🤪🤣🙂😀😜😁😅😂😇😜😆😂😂🙃😁🙂😝😅😂😅😃😅😝😃😅😅😝😆🙃🙂😁🤪😝😅😂😅😃😅😝😊😅😅😝😀😁😜🙃😂😄😆🙂🙃😜🙃🤣🙂🤪🤪😀😃🙃😅😁😅😝😅🙃😂😃🙃🙃🙂😝😅😂🙂🤪🙂😅🤣😊😝😄🤣😉🙃😃🙂😂🙃😄😅🙃😊😁😅😝😅😂😅🙃😁🙂🙃😅😅😃😁😆😄😁🤪😁😄😂🙃😃😅😊😅😅😅😄😊🙃😅😃😃😂🙃😊😆😃🤪😊😅😅😅😜😅😝😃😁😅😉😊😄😄🙂😝😀🙂😂🙃😝🙂😆🙃😂😃😝😅😂😅😃😅😊😅😅😅😜😅😝😁😂😅😉😂😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😜😅😄🙃😊😅😄😆😊😅😅😅😜😅🙃😅😁😅😝😅😂😝😅😅🤣😝😅🤣😜😅🙃😅😁😅😄😜😂🙂😃🙃😊😅😅😅😜😅😇😂😁🙃😉😝😄😀😂😄😊🤪😅😅😜😅🙃😅😆😝🤪🙂😆🙃😇😃😀🙂😉🤣😃😁🙂😆😜😂😅😜😂😊😃😅😊😅😅😅😀😇🙃🙃😀😅🙂😀😜🙂😅😇😄😃😅😝😜😅🙃😅😁😅🤪😇🙂😁🤪😜😅😇😊🙃😁🤪😉🙃😃😅🤣😜🙂😃😃😅😊😅😅😅😁😝😉😜😀😃😆😜🙂🙃🤪😁😁🤣😅😃😇🤪😀🤪😅😂😉🤪😀😜😃🤣😊😅😅😅😜😅🤪🤪😃😉😝😊😂😅😃😅😊😅🙃😇😁😊😊🙃😆🤣😝😀😆🤣😉🤪😊🤪😅😅😜😅🙃😅😂😊🤪😁😅🤣😊😜😁😂😉😆😀😅😂😂🤪🙃😉😂😂😅😃😅😊😅😅😅😜😅🙃😅😅😜😃😅😂🙂😃🙃😊😅😅😅😜😅😝😊😆🤣😉😅😄😆🤣😁😇😀😅😅😜😅🙃😅😅😀🤪😁😆😂😊😅😁😀🙂😆😃😀😂😅😜😝😅🙃😝🤣😂🤪😊😅😅😅😜😅🙃😅😁😅😝😅😉😉😅😅🙂😊😅🤣😜😅🙃😅😁😅😄😝😅🙂😃🤣😊🙂🤣😀😜😅🙃😅😁😅🤪😊😂🙃😜🙂😁😂😇😇😁😝😊😀😃🙂🙃🤣😜🙃😂😄🙂😉😅😊😜😅🙃😅😁😅😂😀😅😃😀🙃😃😉😄😀🙂😊😀😜😁😅😝😅😂😅😃😅😜😃😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😇😂😅😅😜😅🙃🤣😁😅😜😄😂😅😃😅😊😅🤣😂😜😅🙃😅😁😂😝😅😂😂😃😅😊😅😅😅😀😂🙃😅😁😅😝🙂😂😅😃🤪😊😅😅😅😜😅😉😂😁😅😝😅😂🙃😃😅😝😂😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅🙃😜😅🙃😂😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁🤣😝😅🙃😊😃😅😊😅😅😅😜😅🙃😜😁😅😝🤣😂😅😁😇😊😅😅🙃😜🤣🙃😅😁😅😝😅😂😜😃😅😊🤣😅😅😜🙃🙃🤣😁😅😝😅😂😅😁😁😊😅😅🤣😜😅😉😂😆😇😝😅😂😅😃😅😇🙂😅😅😜😅🙃😅😅😅😜🙃😂😅😃😅😊😅😅😅😜😅🙃😊😁😅😜😅🙃🙃😃😅😊😅😅😅😜😅🙃😅😅🙃😝😅🙃😅😁🙃😊😅😅😅😜😅🙃😅😁😅😝😉😂😅😃😊😜😄😅😅😜😅🙃😅😅😁😝😅😂🤣😃😅😊😅😅😅😜🙃🙃🤣😁😅😝😅😂😅😁😁😊😅😅🤣😜😅😉😂😆😇😝😅😂😅😃😅😇🙂😅😅😜😅🙃😅😅😅😜😉😂😅😃😅😊😅😅😅😜😅🙃😊😁😅😜😅😂😅😃😅😊😅😅😅😜😅🙃😅😅🙃😝😅🙃😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😉😂😅😁😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😇😃😅😝😅😅😅😜😅🙃😅😁😅😝😅😂😅😁😃😊😅🤣😁😜😅🙃😅😆😊😝😅🙃😁😃😅😊🤣😅😅😜😅🙃😅😁🙃😝🤣😂😅😃😅😊😅😂😁😜😅🙃🤣😁😅🤪😂🙂😇😃😅😊😅😅😅😀🙂🙃😅😁😅😝😅🙃😅😁😁😊😅😅😅😜😅🙃😅😁😅😝😊😂😅😁😅😊😅😅😅😜😅🙃😅😁😅😝😅🙃🙃😃😅😝😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😉😊😅😂😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😇😅😅😄😅🙃😅😁😅😝😅😂😅😄😅😊😅🤣😉😜😅🙃😅🤣🤪😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😆😂🤪😊😂😅😃🤣😊😅😂😄😜😅🙃😅😁😅🤪😂😂😅😃😅😊😂😅😅😜😝🙃😅😁😅😝😅🙂😂😃😅😊😅😅🙂😜😅😊🤪😁😅😝😅😂😅😄😂😊😅😅😅😜🙃🙃😅😅😉😝😅😂😅😃😅😊😅😅😅😜😅🙃🤣😁😅😝🙃😂😅😃😂😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊🤣😅😅😃😊🙃😅😁😅😝😅😂😅😁😝😊😅😅🤣😜😅😊😇😁😅😝🙃😂🤣😃😅😊😅😅😅😃😝🙃😅😁🤣😝😅😂🙃😃🤣😊😅😅😅😜😅😜😄😁😅😝🤣😂😅😄😂😇😇😅😅😜😅🙃😅😆🙂😝😅😂😅😃😅😝😅😂😀😜😅🙃😅😁😅😝😅😂😅😃😊😊😅😂😅😜😅🙃😅😁😅😝😅😂😅😃😅😝🙃😅😅😃😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😉😜😅😉😂😁😅😝😅😂🤣😃😅😊🤣😅😅😜😅🙃😅😆😂😝😅😂😅😃😂😊😅😂🤣😜😅🙃😅😁😅😜😅😂😅😃😅😊😂😅😅😜😂🙃😅😅😀😝😅🙃😅😃😅😊😅😅😂😜😅🙃😂😁😅😜😝😂😅😃😅😊😅😅😅😜😂🙃😅😁🙂😝😅😂😅😃😅😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃🙂😊😅😅🙃😜😅🙃😅😁😅😝😅😝😀😃😅😊🤣😅😅😜😊😝😄😁😅😝😅😂😅🤣😜😊😅😅🤣😜😅🙃😅😁😅😝🙃😂🤣😃😅😊😅😅😅😆😜🙃😅😁🤣😝😅🙃😅😁🙃😊😅😅😊😜😅🙃😅😁😅😝😇😂😅😁😅😝🙃😅😅😜😊🙃😅😁😊😝😅🙃😃😃😅😇😁🙂🤣😜😅😉😊😁😅😄😜😂😅😃🤣😊😅😅😊😜😅🙃🙃😁🤣😝😅😂😅😃😅😀😜😅😅😜🤣🙃😅😅😅😜🙃😂😅😃😊😊😅😅😉😜😅😉😃😁😅😜😅🤪😀😃😅😊😇😅😅😜😅🙃😅😁😜😝😅🙃😅😃😅😊😅😅😇😜😅🙃😇😁😅🤪😃😂😅😃😅😊😅😅😅😜😊🙃😅😁😊😝😅😂😇😃😅😝😅😅😅😜😅🙃😊😁😅😝😊😂😅😄🙃😊😅😂😊😜😅🙃😅😁😊😝😅😊🤣😃😅😊🤣😅😅😜🙂🙃😅😁🙃😝🤣😂😅😃😅😊😅🙃🤣😜😅🙃🤣😁😅😝🙃😂🤣😃😅😊😅😅😅😆😜🙃😅😁🤣😝😅🙂😂😄😇😊😅😅😊😜😅😉😀😁😅😝😅😂😅😁😅😁😄😅😅😜😊🙃😅😁😊😝😅🙂😜😃😅😝😅😅😅😜😅🙃😇😁😅😝😉😂😅😄😃😊😅😂😅😜😅🙃😅😁😇😝😅😂😇😃😅😝😊😅😅😃😅🙃😅😁😅😝😇😂😅😃😇😊😅🤣😝😜😅🙃😅😁😅😝😅😂😊😃😅😊😂😅😅😜😂🙃😅😆😂😝😅😂😅😃😇😊😅🤣😀😜😅🙃😅😁😅😜😅😂😅😃😅😊😇😅😅😜😇🙃😅😆😜😝😅🙃😅😃😅😊😅😅😝😜😅🙃😉😁😅🤪😃😂😅😁😅😊😅😅😅😜😝🙃😅😁😝😝😅🙃😆😃😅😝😅😅😅😜😅🙃😝😁😅😝😝😂😅😄😝😊😅😅😅😜😅🙃😅😁😇😝😅😂😂😃😅😊😂😅😅😀😂🙃😅😁😅😝😝😂😅😄😀😊😅😅😅😜😅😊😅😁😅😝😅😂😝😃😅😊😝😅😅😀😜🙃😅😅😅😝😅😂😅😃🤪😊😅😅😉😜😅😉😃😁😅😜😅😂😅😃😅😊🤪😅😅😜🤪🙃😅😆🤪😝😅🙃😅😃😅😊😅😅🤪😜😅🙃🤪😁😅🤪😝😂😅😃😅😊😅😅😅😜😝🙃😅😁😂😝😅😂😂😃😅😇😂😅😅😜😅🙃🤪😁😅🤪😄😂😅😃😅😊😅😂😅😜😅🙃😅😁🤪😝😅😂🤪😃😅😝🙂😅😅😜😊🙃😇😁😅😝😜😂😅😃😅😊😅😅🤣😜😅🙃😂😁😅😝😅🙂😀😃😅😊😅😅😅😜🤣🙃😅😁😅😝😅😂😅😉😊😊😅😅😅😜😅🙃😉😁😅😝😅😂😅😃😅😃🙂😅😅😜🤪🙃😅😁😂😝😅😂😂😃😅😊😅🙂😊😜😅🙃🤪😁😅😝🤣😂😅😃🤣😊😅😅😅😄😇🙃😅😁🙃😝😅😂😅😃😅😊😅😅😅😜😊😀😊😁😅😝🤣😂😅😆🙂😊😅😅🤣😜😅🙃😂😁😅😝🙃😂🤣😃😅😊😅😅😅😄🙂🙃😅😁🤣😝😅😂😅😆🤪😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😄😂😇😊😅😅😜🤣🙃😅😅😄😝😅😂😅😃😅😇😂😅😅😜😅🙃😂😁😅😜😁😂😅😃😅😊😅🤣😂😜😅🙃😅😁🙂😝😅🙂😂😃😅😊😅😅😅😀😂🙃😅😁😅😝🙃😂😅😄😇😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😅😊🙃😅😅😜😂🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃🤣😁😅😜😊😂😅😃😅😊😅😅😅😅😇🙃😅😁🤣😝😅🙃😇😃😅😊🙃😅🤣😜😅🙃😅😁😅😁😇😂😅😃🤣😊😅😅🙃😜🤣🙃😅😁😅😝😅🤪🤪😃😅😊🤣😅😅😀😂😉😇😁😅😝😅😂😅😃🤣😊😅😅😅😜😅😉😂😁😊😝😅😂🤣😃😅😝🤣😅😅😜😅🙃😅😅😅😝😅😂😅😃🤣😊😅😅🤣😜😅😉😆😁😅😜😅😂😅😃😅😊🤣😅😅😜🤣🙃😅😆😁😝😅🙃😅😃😅😊😅😅🤣😜😅🙃🤣😁😅😜😝😂😅😃😅😊😅😅😅😜🤣🙃😅😁😂😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😂😊😅😅🙃😜😅🙃😅😁😅😝😅🤪😇😃😅😊🤣😅😅😃😅🤪🤣😁😅😝😉😂😅😃🙃😊😅🤣🤣😜😅😉😂😇😃😝😅😂😇😃😅😝😄😅😅😜😅🙃😅😆😂😝😅😂😅😃😝😊😅😂😅😜😅🙃😅😁😅🤪😂😂😅😃😅😊🤪😅😅😜🙃🙃😅😁😅😝😅🙂😂😃😅😊😅😅😜😜😅😉😉😁😅😝😅😂😅😃😅😊😅😅😅😜😇🙃😅😁😜😝😅😂😂😃😅😇😂😅😅😜😅🙃😝😁😅🤪😊😂😅😃😅😊😅😅😅😜😅🙃😅😁😉😝😅😂😝😃😅😊🤣😅😅😜😊😀😊😁😅😝😅😂😅🙂😆😊😅😅🤣😜😅🙃😂😁😅😝🙃😂🤣😃😅😊😅😅😅🤣😆🙃😅😁🤣😝😅😂😅😄😄😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁🙂😝😅😂😅😃😅🙂😉😅😝😜😅🙃😅😁😅😀😝🙂🙂🤪🙃🤣😃😉🙂😆🤣😊😁😄😆😇🙂🙂😃😃😅😊😅😅😅😀😝😉😄😃😉🙂😁🤪😜😅🙃😊😂😁🙃😊🤣😄😉🙂🙃😀🤣😁😊😄😁😊😅😅😅😜😅😊😆😁😝😉😂😃😜🙃🙃😀😆😆😝🤪🙂😆🙃😇😃😀🙂😉🤣😃😁🙂😆😅😝😜😅🙃😅😁😅😝😅😇😝😃😅😊😅😅😅😜😅🙃😅😁😅😝😅🙃😅😃😅😊😅😅😅😜😅🙃😅😁😅😝🙂😂😅😁😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😂😃😅😊😅😅😅😜😅🙃😂😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁🙂😝😅😂🤣😃😅😊😅😅😅😀😅🙃😅😁😅😝😂😂😅😃🤣😊😅😅🙂😜😅🙃😅😁😅😝😅😂😅😃😅😊😂😅😅😜🤣🙃😅😁😅😝😅😂😅😃😅😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😅😉😄😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🙂😊😝😅😅😜😅🙃😅😜😊😝😊😂😅😃😅😊😅🙃😂😀🙂😂🤣😝😝😅😄😊😊😜🙃😊🙃😅😅😜😅🙃😅🤣😂😝😇😆😅😇🤣😅😉😅😅😜😅🙃😅😁😅😝😅😂😅🙂😅😜😅😄😂😜😅🙃😅😁😅😝😅😂😅😃😅😉😅🙃😆😂😀🙃🙃😁😅😝😅😂😅😆🤣😇😝😁🙂🙃🙃😆😝😁😅😝😅😂😅😃😅😜😆😅🙂😜😅🙃😅😁😅😃😂🙂🙃🤪🤣🙃🙃🤣😃😜😅🙃😅😁😅😃😊😅🤪🤪😂🤣😃😉🙂😄🤪😉🙂😃🤣🙃😁😜🤣😂😀😇😅😂🤣😜😅🙃😅😁😅🤪😂🙂😇😃😅😊🙂😅😅😜😝🙃😅😁😅😝😅😂😅🙃😆😊😅😅🙂😜😅🙃🙂😁😅😝🤣😂😅😃😊😜😄😅😅😜🙂🙃😅😁😇😝😅😂🤣😃😅😊😅😅😅😜🙃🙃🤣😁😅😝😅😂😅😃😇😊😅😅🤣😜😅😉😂😆😇😝😅😂🙂😃😅😊😝😅😅😜😅🙃😅😁😅😅😆😂😅😃🙂😊😅😅🙂😜😅🙃🤣😁😅😝😅😂😂😃😅😊🙂😅😅😜😂🙃😅😁😅😝😅🙂😂😅🙂😊😅😅🙂😜😅🙃🙃😁😅😝😅😂😅😄😂😜🤣😅😅😜🙃🙃😅😁😊😝😅😂😅😃😅😇😂😅😅😜😅🙃😉😁😅😝🙃😂😅😃😅😊😅😅😅😜😅🙃😅😁😊😝😅😂😅😃😅😊😅😅😅😀😂🙃😅😁😅😝😇😂😅😃🙃😊😅😅😅😜😅🙃🙃😁😅😝😅😂😉😃😅😝😄😅😅😜🤣🙃😅😁😅😄😅😂😅😃🤪😊😅😅🙃😜😅🙃😅😁😅🤪😂😉😁😃😅😊😜😅😅😜🤣🙃😅😁😅😝😅🙃😅😃😅😊😅😅😜😜😅🙃😜😁😅😝😂😂😅😁😅😊😅😅😅😀😀🙃😅😁😅😝😅😂😇😃😅😊😅😅😅😜😅😉😄😁😅😝😝😂😅😃😅😊😅😅😅😜😅🙃😅😆😁😝😅😂😝😃😅😊😅😅😅😜😅🙃😅😁😅🤪😀😂😅😄😁😊😅😅😂😜😅😊😅😁😅😝😅🙂😀😃😅😇😀😅😅😜😉🙃😅😁😅😝😅😂😅😄😀😊😅😅😂😜😅🙃😂😁😅😝😅😂😅😃😅😇😀😅😅😀😀🙃😅😁🙂😝😅🙃😅😃😅😊😅🤣😀😜😅😉😀😁😅😝🙂😂😅😃😅😊😅😅😅😜😜🙃😅😁😂😝😅😂😂😃😅😊😅😅😅😜😅🙃🙃😁😅😝🤪😂😅😃😜😊😅😅😅😜😅🙃😅😁🤪😝😅😂🙂😃😅😊😂😅😅😃😅🙃😅😁😅😝🙂😂😅😃🤪😊😅😅🙂😜😅🙃🙃😊🙃😝😅😂😉😃😅😇😄😅😅😜🤣🙃😅😆😂🤪😇😂😅😃😉😊😅😅😝😜😅🙃😅😁😅😝😅😇😆😃😅😊😉😅😅😜🤣🙃😅😁🙃😝😅😂😅😃😂😊😅😅🙃😜😅🙃😂😁😅😝😅😂😅😃😅😇😄😅😅😜😅🙃😅😁🤣😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁🤣😝🤣😂😅😃😅😊😅😀😇😀😃🙃😅😁😅😝😅🙂🤣😄😁🙃😇😝🙃😅😂😇😆😁🙂😇😁😄😊🙃😊😃😀😂🙃😜😉🙃😅😁😅😝😅😂😅🙃😁😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😁😅😊😅😅😅😜🤣🙃😅😁🤣😝😅😂🤣😃😅😊😅😅😅😜😅🙃🙂😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁🤣😝😅😂🙂😃😅😊🤣😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊🤣😅😅😜😅🙃😅😝😝🤪😄😂😅😃😅😊😅😂🙂😀🤣🤣😅😝😂😆🙃😇🤣😁😂🙂🙂😃😁😂😊🤪😊🤣😀😇🙃😂😉😃😅😊😅😅😅😜😅😃😁😁😅😝🤣😂😅😃😅😊😅😅😅😜😅😊😅😁😅😝😅😂🤣😃😅😊🤣😅😅😜🤣🙃😅😁😅😝😅😂😅😃🙂😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊🤣😅😅😜🙂🙃😅😁🤣😝😅😂😅😃😅😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃🤣😁🤣😝😅😂😅😃😅🤪😅🤣😃😜😅🙃😅😁😅🤪🤣🙂😁😜😇😁🙃😊😂😄😆🙃🙂😄😁🙂😊😜😊😂😀😝🙃😅😉😜😅🙃😅😁😅😝😅😜😁😃😅😊🤣😅😅😜😅🙃😅😁😅😝😅🙃😅😃😅😊😅😅🤣😜😅🙃🤣😁😅😝🤣😂😅😃😅😊😅😅😅😜🙂🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃🤣😁😅😝🙂😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🤣😉😁😉😝😅😂😅😃😅😜😆🤣🤣😇😝😆😄😊🙂😝🙃😂😅😃😅😊😅🤣😂😀😇🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😅😇🤪😁😅😝🤣😂😅😃😅😊😅😅😅😜😅🙃😅😂😉😝😅😂😅😃😅😊😂😅😅😜🤣🙃😅😁😅🤪😄😂😅😃😅😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅🤣😜🤣🙃😅😁😅😝😅🤣🙂😄😆😊😅😅😅😜😅😊🙂😁🙃😉😅😄😃🙂😂😜😇🤣🙃😝😂😆😜😉🙂😁😁🙃😊😀😊🙂😀🤪🙃😜😉🙃😅😁😅😝😅😂😅🙃😁😊😅😅🤣😜😅🙃😅😁😅😝😅😂😅😁😅😊😅😅😅😜🤣🙃😅😁🤣😝😅😂🤣😃😅😊😅😅😅😜😅🙃🙂😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁🤣😝😅😂🙂😃😅😊🤣😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😇😅😅😅😜😅🙃😅😃😉🤪😁😂😅😃😅😊😅😊😜😅😅🤪🙃😂😀😀😊🙃😅😄😂😊😇😅😂😉😇🙂😂😃🙃😉😃😁😂😃🙃😊😂😅😅😜😅🙃😅🙂🙃😄🙂😄😇😃😝😊😅😅😅😜😅😉😇😆😁😉😜😃😇😂🙃😜🤪🤣🙃😝😅😅😊😁😇😝😅😂😅😃😅😜🤣😅🙃😝😅😆🙃😉😝😄🤣🙂🙃😁🤣😊🙃😅😅😜😅🙃😅🤣😊😝😅😅😃😊🙃🤣😇😅😊😜😅🙃😅😁😅😀🙂😂😅😜😃😅😅😇😊😁🙃🤣😂😁😅😝😅😂😅😃😅😊😅😅😅😃😜😝😅😊😜🤪😇😂😅😃😅😊😅🙂😅😀😆😂😆🤪😀😅😝🙃😜😁😄😉😁😀😂😂😜🤪🤣😅😅😇😂😁😜🙂🙂😃😝🙃🤣😜🙃🤣😂🤪🙂😆😝😇😁😁😄🤪😅🤣😁😜😅🙃😅😁😅😀😅🙂😆😜😆🤣😀😝😝😃😜😊😄😄😁🙂😂😜😜🤣🤣😝😅😆😂😊😜😀😃😁😜😝😅😂😅😃😅😇😊😅🙃😝🙂😄😂😉🙃😄🤣🙃😉😜😝🤣😂😝🙃🤣😃🙃😊😁😅😝😅😂😅😉😃😂😃😃😊😃😊😃😅😉😁😉😂😂😅😃😅😊😅😅😅😜😅🙃😅🤣🤣😃😅😀🤪😄🤣😊😅😅😅😜😅😊🤣😁🙃😊😆😄😀🙂😝😜😂🤣😅🤪🙂😆🙃😊🙂😃😂😉🙂😃😁🙂🤣🤪😅😅😊😇🙃😇😜🤪😁😂😅😃😅😊😅😊😜😅😃🤪😃🤣😆😜😊🙂😃😀😜😉😀😁😂🙂🙂😅😜🤪😝😂😃😀😁😇😇😊😊😅😅😜😅🙃😅😅😆😃😆😊🤣😜😃😝😂😀😝😁😆😉😄😁😅😝😅😂😅😄😜😇😄😁😁🙃😂😀😜😂🤣🤪😅😅😂😇😜😄🙃😊🙂😃😝🙃😀🤣😃😝😅😂😅😃😅😊😊🙃😄😜😅🙃😅😁😅🤪😊😂😅😃🤣😊😅😅😅😜😅🙃🙃😁🤣😝😅😂😅😃😅😇😊😅😅😜🤣🙃😅😆😂🤪😇😂😅😃🤣😊😅😅🙃😜😅🙃😅😁😅🤪😂🙃😝😃😅😊😂😅😅😜😉🙃😅😁😅😝😅🙃😅😃😅😊😅😅😂😜😅🙃😂😁😅😝😜😂😅😄😂😊😅😅😅😜🙃🙃😅😆😄😝😅😂😅😃😅😊😅😅😅😜😅🙃😂😁😅😝🙃😂😅😃😂😊😅😂😅😜😅🙃😅😁😂😝😅😂😂😃😅😊😂😅😅😃😅🙃😅😁😅😝😂😂😅😃😂😊😅😅😊😜😅😉😂😁😅😝😅😂🙂😃😅😊🙂😅😅😜😅🙃😅😆😂😝😅😂😅😃🙃😊😅🤣😁😜😅🙃😅😁😅🤪😂😂😅😃😅😊😉😅😅😀😆🙃😅😁😅😝😅🙂😂😃😅😊😅😅😊😜😅😉😃😁😅😝😅😂😅😃😅😊😅😅😅😜🙂🙃😅😁😊😝😅😂😂😃😅😊😅😅😅😜😅🙃😂😁😅😝😂😂😅😃🙂😊😅😅😅😜😅🙃😅😁🤣😝😅😂😂😃😅😊😂😅😅😃😅🙃😅😁😅😝🤣😂😅😃🤣😊😅🤣😅😜😅🙃😂😇😆😝😅😂😂😃😅😊😅😅😅😜😅🙃😅😆😅🤪😜😂😅😃🤣😊😅😅😝😜😅🙃😂😁😅😝😂😀😆😃😅😊😂😅😅😜🤣🙃😅😁😅😝😅🙂😅😄😜😊😅😅🤣😜😅🙃🤪😁😅😝😂😂😅😃🙃😊🤣😅😅😜😅🙃😅🤣😀😝😅😂🤣😃😅😇😂🤣😇😜😅🙃🤣😁😅😝🙃😂😅😃😅😊😅🤣😂😂😜🙃😅😁😂😝😅😂😉😃😅😊😅😅😅😃😅🙃😅😁😅😝😂😂😅😃😂😊😅😅😜😜😅😉😂😁😅😝😅😂🙃😃😅😇😄😅😅😜😅🙃😅😁😅😝😅😂😅😃😂😊😅😅🙃😜😅🙃😂😁😅😜😅😂😅😃😅😊😂😅😅😜😂🙃😅😁😂😝😅🙃😅😃😅😊😅😅😂😜😅🙃😂😁😅😝😊😂😅😄😂😊😅😅😅😜🙂🙃😅😁🙂😝😅😂😅😃😅😇😂😅😅😜😅🙃🙃😁😅😝🤣😂😅😃😅😊😅🤣😂😜😅🙃😅😁😉😝😅🙂😀😃😅😊😅😅😅😀😂🙃😅😁😅😝😊😂😅😃😇😊😅😅😅😜😅🙃😅😁😅😝😅😂🙂😃😅😊😊😅😅😜😂🙃😅😁😅😝😅😂😅😃😂😊😅😅😂😜😅🙃🙂😁😅😝😅😂😅😃😅😊🤣😅😅😜😂🙃😅😁😂😝😅🙃😅😃😅😊😅😅🤣😜😅🙃🤣😁😅🤪😅😂😅😃😅😊😅😅😅😜😂🙃😅😁😅😝😅😂😅😃😅😝😅😅😅😜😅🙃😂😁😅😝😂😂😅😃😝😊😅🤣😅😜😅🙃😅😁🤣😝😅😂😝😃😅😊😂😅😅😜😅🙃😅😁😅😝😂😂😅😃😅😊😅😅😅😜😅😊😅😁😅😝😅😂😂😃😅😊😂😅😅😜🤪🙃😅😆😅😝😅😂😅😃🤣😊😅😅🤪😜😅🙃😂😁😅😝😅🙂😄😃😅😊😅😅😅😜🤣🙃😅😁😅😝😅😂😂😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😆😄😝😅😂😅😃😅😊🤣😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅🙃😅😁😅😝🤣😂😅😃😅😊😅😅😅😀😄🙃😅😁😅😝😅😂🤣😃😅😊😅😅😅😜😅🙃😅😁😅😝😅😂🤣😃🤣😊😅😅😅😜😅🤪😁😁😉😝😅😂😅😃😅😜😆😅😂😇😅😆😀😊😀😝😝😂😅😃😅😊😅😅😅😄😂🙃😅😁🤣😝😅😂😅😃😅😊😅😅😅😀😂😉😇😁😅😝😂😂😅😃🤣😊😅😅😅😜😅🙃😊😁😇😝😅😂🙂😃😅😊😅😅😅😜🤣🙃😅😁😂😝😅😂😅😉😊😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃😅😆😊😅😅😜😅🙃😅😁😅😝😅😂😅😃😅😊😅🙃😉😜😅🙃😂😁😅😝😂😂😅😃🤣😊😅😅😅😜😂🙃😅😁🤣😝😅😂😂😃😅😊😅😅😅😜😅😉😄😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃🤣😁😅😝😅😂😅😃😅😊🙃😅😅😜😅🙃😅🙃😄😝😜😂😅😃😅😊😅🙂😝😀😂😂😀😝😂😅😀😇😁😁😂😉🙃😃🤣😂🙃😜😄😆😃😝😅😂😅😃😅🤪😊😅🙃😝🙂😁😂😊😁😄😄🙃😂😀🙂🤣😅🤪😄😅🙂😇😂😃🙃🤪😃😂😅😃😅😊😅🙂😝😀😂🤣😁🤪😀😆😁😊😂😁😀😉😁😃😂🙂🙃😜🤣😅🙃😃😆😂😉😃😅😊😅😅😅😄🙂🙃🙃😀🤣🙂🙃🤪😊😄😂😊😅😅😅😜😅😉😂😆😇😝😅😂😅😃😅😊🤣😅😅😜😅🙃😅😁😊😃😄😂😅😃😅😊😅😅🙃😜😅🙃🤣😁😅😝😅😂😅😃🙃😊🤣😅😅😜😅🙃😅😁🙃😝😅😂🤣😃😅😊🙃😅🤣😜😅🙃😅😁😅🤪😁😂😅😃🤣😊😅🤣😂😀😇🙃😅😁😅😝😅😂🙂😃😅😊😅😅😅😜😊😝😄😁😅😝😅😂😅😃😝😊😅😅🤣😜😅🙃😅😁😅😝🙃😂🤣😃😅😊😅😅😅😜😝🙃😅😁🤣😝😅😂🙃😃🤣😊😅😅😅😜😅😉😁😁😅😝🤣😂😅😄😂😇😇😅😅😜😅🙃😅😁😂😝😅😂😅😃😅😊😊🙃😄😜😅🙃😅😁😅🤪😃😂😅😃🤣😊😅😅😅😜😅🙃🙃😁🤣😝😅😂😅😃😅😇😃😅😅😜🤣🙃😅😁🙃😝🤣😂😅😃😅😊😅🤣😁😜😅🙃🤣😁😅🤪😂🙂😇😃😅😊😅😅😅😜🙃🙃😅😁😅😝😅🙃😅😁🙃😊😅😅😅😜😅🙃😅😁😅😝😂😂😅😃😅🤪🤪😅😅😜🤣🙃😅😁🤣😝😅😂😅😃😅😊😅😊🙂😜😅🙃😅😁😅😝😂😂😅😃😂😊😅😅😅🙂🤣🙃😅😁😅😝😅😂😅😃😅😊😅😅😅😜😅😉😄😁😅😝😅😂😅😃🤣😊😅😅😅😜😅🙃😅😁😅😝😅😂😅");local _l__l=(-#[[RainX is a RAT, it does not utilize SxLib like it claims and instead completely takes over your pc]]+(319+-0x7d))local _l=43 local __=_ll;local _={}_={[((-0x58+(320-0xb6))+-#[[oh you use protocrasher? name every crash you got]])]=function()local _ll,_lll,_,__l=___lll(__l_ll,__,__+_ll__);__=__+__l__;_l=(_l+(_l__l*__l__))%___;return(((__l+_l-(_l__l)+_l_*(__l__*____l))%_l_)*((____l*_ll_ll)^____l))+(((_+_l-(_l__l*____l)+_l_*(____l^_ll__))%___)*(_l_*___))+(((_lll+_l-(_l__l*_ll__)+_ll_ll)%___)*_l_)+((_ll+_l-(_l__l*__l__)+_ll_ll)%___);end,[((28888/0x5c)/157)]=function(_,_,_)local _=___lll(__l_ll,__,__);__=__+_lll;_l=(_l+(_l__l))%___;return((_+_l-(_l__l)+_ll_ll)%_l_);end,[(-#"nah Federal is my dad"+(81+-0x39))]=function()local _,_ll=___lll(__l_ll,__,__+____l);_l=(_l+(_l__l*____l))%___;__=__+____l;return(((_ll+_l-(_l__l)+_l_*(____l*__l__))%_l_)*___)+((_+_l-(_l__l*____l)+___*(____l^_ll__))%_l_);end,[((0x209a/107)+-#'Your exploit crashed and now youre crying? <a:HAHAHAHA:762019852327190598>')]=function(_l,_,__)if __ then local _=(_l/____l^(_-_ll))%____l^((__-_lll)-(_-_ll)+_lll);return _-_%_ll;else local _=____l^(_-_lll);return(_l%(_+_)>=_)and _ll or _lllll;end;end,[(0x52-77)]=function()local _l=_[(-#"i love mahmd"+(0x80+(-0xaa+55)))]();local __=_[(124/0x7c)]();local __ll=_ll;local __l=(_[(0x27-(-#'RainX more like IeatyourfuckingIPaddressX'+(0xcb+-127)))](__,_lll,_l_lll+__l__)*(____l^(_l_lll*____l)))+_l;local _l=_[(0xb0/44)](__,21,31);local _=((-_ll)^_[(544/0x88)](__,32));if(_l==_lllll)then if(__l==__llll)then return _*_lllll;else _l=_lll;__ll=__llll;end;elseif(_l==(_l_*(____l^_ll__))-_lll)then return(__l==_lllll)and(_*(_lll/__llll))or(_*(_lllll/__llll));end;return __l__l(_,_l-((___*(__l__))-_ll))*(__ll+(__l/(____l^___l_l)));end,[(-#'deobfuscation: no'+(62-0x27))]=function(__l,__ll,__ll)local __ll;if(not __l)then __l=_[((88-0x39)+-#[[Stop staring at my code weirdo]])]();if(__l==_lllll)then return'';end;end;__ll=_lll_l(__l_ll,__,__+__l-_ll);__=__+__l;local _=''for __=_lll,#__ll do _=_l_l_l(_,_l__ll((___lll(_lll_l(__ll,__,__))+_l)%___))_l=(_l+_l__l)%_l_ end return _;end}local function _lllll(...)return{...},__ll_l('#',...)end local function _lll_l()local ___l={};local __l_={};local _l={};local __ll={___l,__l_,nil,_l};local __={}local __l=((19100/0xbf)+-#"tried playing roblox once and how tf people enjoy it")local _l={[(0x22+-32)]=(function(_l)return not(#_l==_[(-#[[balls]]+(-0x3c+67))]())end),[((0x2970/204)+-52)]=(function(_l)return _[(0x45-64)]()end),[((-0x1a5e/75)+0x5e)]=(function(_l)return _[(-0x24+42)]()end),[(((-1274/0x1a)+-#"also congratulations sirhurt users for having your data leaked again!")+0x79)]=(function(_l)local __=_[(0x3e-56)]()local _l=''local _=1 for _ll=1,#__ do _=(_+__l)%___ _l=_l_l_l(_l,_l__ll((___lll(__:sub(_ll,_ll))+_)%_l_))end return _l end)};__ll[3]=_[(78+-0x4c)]();local __l=_[(104/((0x14a0/33)+-#"oh you use krnl? name everything that broke last update."))]()for _ll=1,__l do local _=_[((0xf7-176)+-69)]();local __l;local _=_l[_%(-0x1e+45)];__[_ll]=_ and _({});end;for __ll=1,_[(50+-0x31)]()do local _l=_[(95-0x5d)]();if(_[(-#'can you find what you want here?'+(92-0x38))](_l,_ll,_lll)==__llll)then local ___=_[((0xd36/89)+-#'this script is secured using socks')](_l,____l,_ll__);local __l=_[(0x27+-35)](_l,__l__,____l+__l__);local _l={_[(-#"allah save me"+(111+-0x5f))](),_[(69-0x42)](),nil,nil};local _l_={[(0x7b-123)]=function()_l[_ll_]=_[(26-0x17)]();_l[_l_l_]=_[(125+-0x7a)]();end,[((0x3412/215)+-#"SirHurt trying not to get their database leaked for 5 seconds")]=function()_l[_ll_]=_[(204/0xcc)]();end,[(0x63-97)]=function()_l[___ll]=_[((-46+0x3c)+-#[[happy is cool]])]()-(____l^_l_lll)end,[(-64+0x43)]=function()_l[___ll]=_[(-0x67+104)]()-(____l^_l_lll)_l[_____]=_[(0x1c-25)]();end};_l_[___]();if(_[(588/0x93)](__l,_lll,_ll)==_lll)then _l[_l__]=__[_l[_l_l]]end if(_[(-#[[Kitty was here you sussy baka]]+(82+-0x31))](__l,____l,____l)==_ll)then _l[_ll_l]=__[_l[____]]end if(_[(68+-0x40)](__l,_ll__,_ll__)==_lll)then _l[__ll_]=__[_l[_l_l_]]end ___l[__ll]=_l;end end;for _=_lll,_[(38-0x25)]()do __l_[_-_lll]=_lll_l();end;return __ll;end;local function _l_lll(_,__l__,_l__l)local _l=_[____l];local _l_=_[_ll__];local __=_[_ll];return(function(...)local _=_ll _*=-1 local _ll__=_;local ___=_l_;local _lllll=_lllll local __l_ll=__ll_l('#',...)-_lll;local ___lll=_l;local _l=_ll;local _ll_ll={...};local _l_=__;local __={};local _l__ll={};local __llll={};for _=0,__l_ll do if(_>=___)then _l__ll[_-___]=_ll_ll[_+_lll];else __[_]=_ll_ll[_+_ll];end;end;local _=__l_ll-___+_ll local _;local ___;while true do _=_l_[_l];___=_[((0x2fa8/100)/122)];__l=(2134386)while ___<=(94+-0x12)do __l-= __l __l=(2021188)while(0x1848/168)>=___ do __l-= __l __l=(665275)while ___<=(3258/0xb5)do __l-= __l __l=(475434)while(0x4b-67)>=___ do __l-= __l __l=(2265408)while(0x83-128)>=___ do __l-= __l __l=(2727984)while ___<=(0x49/73)do __l-= __l __l=(3932208)while(0x36-54)<___ do __l-= __l _l=_[___ll];break end while 1162==(__l)/((0xc3cd8/237))do local __l;__[_[___l]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__lll(__,__l+_ll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__[_[__l_l]][__[_[_llll]]]=__[_[_lll_]];_l=_l+_ll;_=_l_[_l];do return end;break end;break;end while 3864==(__l)/((803+-0x61))do __l=(4929976)while(-#[[clown]]+(((0x1c7-284)+-#[[Oh damn I just stepped on a shit. Nevermind its just RainX]])-0x6a))<___ do __l-= __l if(_[_l__]<__[_[_____]])then _l=_l+_lll;else _l=_[___ll];end;break end while(__l)/((1641+-0x6d))==3218 do do return __[_[__l_]]end break end;break;end break;end while(__l)/((62928/0x12))==648 do __l=(333312)while(1040/0xd0)>=___ do __l-= __l __l=(6612228)while ___>(572/0x8f)do __l-= __l __[_[_l_l]]=__[_[_ll_l]][_[___l_]];break end while 2762==(__l)/((-#"baxoplenty is fat"+(4848-(-0x1a+2463))))do local _ll=_[_ll_l];local _l=__[_ll]for _=_ll+1,_[__ll_]do _l=_l..__[_];end;__[_[_l__]]=_l;break end;break;end while 2688==(__l)/((-55+0xb3))do __l=(3186810)while ___<=(-#[[Being arrogant is like being an elephant. If someone shoots you with a bazooka, you die.]]+(8930/0x5f))do __l-= __l local __l;local ___;local ___l,___ll;local ____;local __l;__[_[__l_l]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[_llll]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_l_ll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l__];____=__[_[_ll_]];__[__l+1]=____;__[__l]=____[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__l=_[__l_]___l,___ll=_lllll(__[__l](__[__l+_lll]))_ll__=___ll+__l-_ll ___=0;for _=__l,_ll__ do ___=___+_ll;__[_]=___l[___];end;_l=_l+_ll;_=_l_[_l];__l=_[_l_l]___l={__[__l](__lll(__,__l+1,_ll__))};___=0;for _=__l,_[_l_l_]do ___=___+_ll;__[_]=___l[___];end _l=_l+_ll;_=_l_[_l];_l=_[_l_ll];break;end while 3330==(__l)/((29667/(0x1a66/218)))do __l=(5343940)while ___>(0x61-90)do __l-= __l local ___;local __l;__[_[_l_l]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[___ll]][__[_[__ll_]]];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];___=__[_[_l___]];if not ___ then _l=_l+_lll;else __[_[_l__]]=___;_l=_[___ll];end;break end while 1715==(__l)/((0x26cc4/51))do local ___=___lll[_[____]];local __l;local _ll={};__l=____ll({},{__index=function(_l,_)local _=_ll[_];return _[1][_[2]];end,__newindex=function(__,_,_l)local _=_ll[_]_[1][_[2]]=_l;end;});for __l=1,_[_lll_]do _l=_l+_lll;local _=_l_[_l];if _[(0x9d/157)]==150 then _ll[__l-1]={__,_[_llll]};else _ll[__l-1]={__l__,_[_ll_l]};end;__llll[#__llll+1]=_ll;end;__[_[__ll]]=_l_lll(___,__l,_l__l);break end;break;end break;end break;end break;end while 2598==(__l)/((-#[[RainX trippin on god (on top)]]+(0xb55c/219)))do __l=(4674278)while(-#[[Moonsec does not work with jjsploit plz help]]+(206-0x95))>=___ do __l-= __l __l=(2297448)while(112-0x66)>=___ do __l-= __l __l=(1059783)while((0xf3+-112)-122)<___ do __l-= __l local ___;local __l;__l=_[_l__]__[__l](__lll(__,__l+_lll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))break end while(__l)/(((174744/0xd8)+-#"can anyone show me a circular square"))==1371 do local __l;__[_[__ll]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l](__lll(__,__l+_lll,_[___ll]))break end;break;end while(__l)/((0x6965e/230))==1224 do __l=(127844)while(49+-0x26)>=___ do __l-= __l __[_[___l]]=__l__[_[_ll_l]];break;end while(__l)/((23312/0xbc))==1031 do __l=(5368056)while ___>(144-0x84)do __l-= __l do return end;break end while 2676==(__l)/((0x58a4e/(0xe5+-48)))do local _=_[__ll]local __l,_l=_lllll(__[_](__[_+_lll]))_ll__=_l+_-_ll local _l=0;for _=_,_ll__ do _l=_l+_ll;__[_]=__l[_l];end;break end;break;end break;end break;end while 2779==(__l)/((-0x2d+1727))do __l=(4326050)while(106-0x5b)>=___ do __l-= __l __l=(239752)while(-74+0x58)<___ do __l-= __l if(__[_[_l__]]~=_[_l_l_])then _l=_l+_lll;else _l=_[_ll_];end;break end while(__l)/(((1441+-0x61)+-#[[using free version? nice performance nerd]]))==184 do local _=_[_l_l]__[_](__[_+_lll])break end;break;end while 1550==(__l)/(((645361/0xe3)+-#'Only Donals Trump supporters uses public obfuscators'))do __l=(6016140)while ___<=(1344/0x54)do __l-= __l __[_[_l__]]=(_[_l_ll]~=0);break;end while(__l)/((0x18454/58))==3510 do __l=(3779100)while((10005/0x57)+-#'RainX is a RAT, it does not utilize SxLib like it claims and instead completely takes over your pc')<___ do __l-= __l local _={__,_};_[_ll][_[____l][_l_l]]=_[____l][_l_l_]+_[____l][_llll];break end while 975==(__l)/((7803-(-#'can you find what you want here?'+(7967-0xfa8))))do local _=_[___l]__[_]=__[_](__[_+_lll])break end;break;end break;end break;end break;end break;end while 575==(__l)/((0x2306c/124))do __l=(230412)while ___<=(0x54+-57)do __l-= __l __l=(3933600)while(-#[[weirdo above]]+(0x7c-90))>=___ do __l-= __l __l=(6644550)while(0x1090/212)>=___ do __l-= __l __l=(999750)while(0x69-(-0x74+202))<___ do __l-= __l __[_[___l]]={};break end while(__l)/((5429-((2893+-0x76)+-#"RainX moment")))==375 do local ___;local __l;__[_[_l_l]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[_l_ll]];__[__l+1]=___;__[__l]=___[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l](__lll(__,__l+_lll,_[_llll]))_l=_l+_ll;_=_l_[_l];do return end;break end;break;end while(__l)/((0xd60-1774))==4027 do __l=(377208)while ___>((104-0x3e)+-#"lua beautifier is gay")do __l-= __l local __l;__[_[___l]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_ll_l]][__[_[_lll_]]];_l=_l+_ll;_=_l_[_l];if(__[_[_l_l]]~=_[_l___])then _l=_l+_lll;else _l=_[____];end;break end while 124==(__l)/(((0x1805-3091)+-#[[wally likes kids]]))do __[_[_l_l]][_[_ll_]]=_[_____];break end;break;end break;end while(__l)/((4890-0x9ba))==1639 do __l=(8084535)while(0x74-92)>=___ do __l-= __l __l=(752732)while ___>(120-0x61)do __l-= __l local _=_[_l_l]__[_]=__[_]()break end while(__l)/((3378-0x6b8))==454 do __[_[_l_l]]=_l__l[_[___ll]];break end;break;end while 2685==(__l)/(((309060/0x65)+-#"Watch out! FBI wants to break our obfuscated code"))do __l=(7108218)while((0x38d0/144)-76)>=___ do __l-= __l local _={__,_};_[_lll][_[____l][_l__]]=_[_ll][_[____l][_____]]+_[_lll][_[____l][_ll_l]];break;end while 2886==(__l)/((-41+0x9c8))do __l=(113280)while(0x49-47)<___ do __l-= __l local __l;local ___;local _ll_,_llll;local ____;local __l;__[_[__ll]]=__[_[___ll]][_[_____]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[_l_ll]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[___ll]][_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[___ll]][_[_lll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l__];____=__[_[_ll_l]];__[__l+1]=____;__[__l]=____[_[_lll_]];_l=_l+_ll;_=_l_[_l];__l=_[___l]_ll_,_llll=_lllll(__[__l](__[__l+_lll]))_ll__=_llll+__l-_ll ___=0;for _=__l,_ll__ do ___=___+_ll;__[_]=_ll_[___];end;_l=_l+_ll;_=_l_[_l];__l=_[__ll]_ll_={__[__l](__lll(__,__l+1,_ll__))};___=0;for _=__l,_[_lll_]do ___=___+_ll;__[_]=_ll_[___];end _l=_l+_ll;_=_l_[_l];_l=_[_l_ll];break end while 1770==(__l)/((152+(-#[[so there is this thing called moonsec]]+(-0x18+-27))))do __[_[__l_]][_[___ll]]=__[_[_lll_]];break end;break;end break;end break;end break;end while(__l)/((456+-0x22))==546 do __l=(11241540)while ___<=(-#"how bored do you have to be to read this"+(159+((-131+0x39)+-#"RainX winning")))do __l-= __l __l=(14279250)while ___<=(6728/0xe8)do __l-= __l __l=(3796200)while ___>((0x1c94/118)+-#'this script is secured using socks')do __l-= __l local ___;local __l;__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))break end while(__l)/((0x8e0-1192))==3515 do local ___;local __l;__[_[_l_l]]=__l__[_[_ll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l](__lll(__,__l+_lll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];do return end;break end;break;end while(__l)/((7933-0xf8f))==3615 do __l=(109956)while ___<=(3930/0x83)do __l-= __l __[_[_l_l]]=__[_[_ll_l]][_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_l_ll]][_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_l_ll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[____]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]][_[_llll]]=_[___l_];break;end while 308==(__l)/((-#'tried playing roblox once and how tf people enjoy it'+(0x8302/82)))do __l=(6796328)while ___>(6541/0xd3)do __l-= __l __[_[_l_l]]=__[_[_ll_]]-__[_[_l_l_]];break end while 2596==(__l)/((5253-0xa4b))do local _l_=_[__l_];local _ll={};for _=1,#__llll do local _=__llll[_];for _l=0,#_ do local _=_[_l];local __l=_[1];local _l=_[2];if __l==__ and _l>=_l_ then _ll[_l]=__l[_l];_[1]=_ll;end;end;end;break end;break;end break;end break;end while(__l)/((-#"rainx is the best ever!!!"+(0x18e4-3233)))==3610 do __l=(1939136)while ___<=(-65+0x63)do __l-= __l __l=(8346780)while ___>(0x10e3/131)do __l-= __l _l__l[_[___ll]]=__[_[__l_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]={};_l=_l+_ll;_=_l_[_l];__[_[__l_l]]={};_l=_l+_ll;_=_l_[_l];_l__l[_[_ll_l]]=__[_[__l_l]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];if(__[_[_l__]]==_[__ll_])then _l=_l+_lll;else _l=_[_llll];end;break end while(__l)/((-#"RainX trippin on god (on top)"+(509066/((417-0xf2)+-#"using free version? nice performance nerd"))))==2214 do local ___;local __l;__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))break end;break;end while 2956==(__l)/(((0x2e5+-72)+-#'allah save me'))do __l=(1281800)while(0x82-95)>=___ do __l-= __l local ___;local __l;__[_[_l__]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_ll_]][_[_____]];_l=_l+_ll;_=_l_[_l];__l=_[__l_l];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]={};_l=_l+_ll;_=_l_[_l];__[_[___l]]={};_l=_l+_ll;_=_l_[_l];__[_[___l]][_[_llll]]=_[__ll_];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]][_[____]]=__[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]][_[____]]=__[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]][_[____]]=_[_lll_];_l=_l+_ll;_=_l_[_l];__[_[__l_l]][_[____]]=__[_[___l_]];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l](__lll(__,__l+_lll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];do return end;break;end while(__l)/((1318-0x29c))==1972 do __l=(3008439)while ___>(-0x20+68)do __l-= __l __[_[_l__]]=__[_[_ll_l]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_l_ll]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[___ll]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[____]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_ll_l]][_[_l___]];_l=_l+_ll;_=_l_[_l];if(_[__ll]<__[_[_l___]])then _l=_l+_lll;else _l=_[_ll_];end;break end while 2703==(__l)/((2326-0x4bd))do __[_[___l]]=__[_[____]][_[__ll_]];break end;break;end break;end break;end break;end break;end break;end while(__l)/(((-#[[deobfuscation: no]]+(0x6f616/178))-0x52c))==1654 do __l=(8958880)while ___<=(-#"cock and ball torture"+(0xb7-106))do __l-= __l __l=(2312035)while ___<=(0xcd-159)do __l-= __l __l=(3361995)while(-#'3 month of free private access be like'+(-112+0xbf))>=___ do __l-= __l __l=(139517)while ___<=(185-0x92)do __l-= __l __l=(5011230)while ___>((0xd3+-96)-0x4d)do __l-= __l __[_[_l__]]=__[_[___ll]]-__[_[_lll_]];break end while(__l)/((333396/0xa2))==2435 do _l__l[_[_ll_l]]=__[_[__ll]];break end;break;end while 133==(__l)/((-#[[RainX is fr garbageeeee ew]]+(-0x29+1116)))do __l=(2401408)while ___>(-#"just boost moonsec and get an even better obfuscator!"+(0xbb+-94))do __l-= __l __[_[___l]]=(_[_llll]~=0);_l=_l+_lll;break end while(__l)/(((-0x6c+489436)/0xee))==1168 do local ___;local __l;__[_[__ll]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[___ll]];__[__l+1]=___;__[__l]=___[_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[____]][_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_llll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[____]][__[_[_lll_]]];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_ll_l]][_[_____]];break end;break;end break;end while(__l)/((0x2019-4122))==821 do __l=(4975420)while(6278/0x92)>=___ do __l-= __l __l=(4399265)while ___>(140-0x62)do __l-= __l if not __[_[_l_l]]then _l=_l+_lll;else _l=_[_ll_];end;break end while(__l)/((-0x59+3534))==1277 do local ___;local __l;__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__lll(__,__l+_ll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[____]][__[_[_lll_]]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[_ll_l]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_llll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_ll_l]][__[_[_lll_]]];_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[_l_ll]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[____]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[____]];__[__l+1]=___;__[__l]=___[_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_l];___=__[_[_l_ll]];__[__l+1]=___;__[__l]=___[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l](__lll(__,__l+_lll,_[___ll]))_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[_[____]];__[__l+1]=___;__[__l]=___[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l](__lll(__,__l+_lll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__l=_[__ll];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[_llll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[____]];__[__l+1]=___;__[__l]=___[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_]))break end;break;end while 3713==(__l)/(((-31+(0x1b7b0/80))+-#[[stop reading the code sussy bakayero]]))do __l=(154216)while(0x83-87)>=___ do __l-= __l if(__[_[__ll]]==__[_[_l___]])then _l=_l+_lll;else _l=_[___ll];end;break;end while 148==(__l)/((2156-0x45a))do __l=(890197)while(((0xee+-57)+-#[[oh you use protocrasher? name every crash you got]])-0x57)<___ do __l-= __l local ___ll;local ___;local ___l;local __l;__[_[__ll]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[_l_ll]][_[_____]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l];___l=__[_[_ll_]];__[__l+1]=___l;__[__l]=___l[_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[____]];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_];___l=__[_[_llll]];__[__l+1]=___l;__[__l]=___l[_[___l_]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];___={__,_};___[_lll][___[____l][_l__]]=___[_ll][___[____l][_lll_]]+___[_lll][___[____l][_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_l_ll]]%_[___l_];_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];___l=_[_ll_];___ll=__[___l]for _=___l+1,_[___l_]do ___ll=___ll..__[_];end;__[_[__l_]]=___ll;_l=_l+_ll;_=_l_[_l];___={__,_};___[_lll][___[____l][__ll]]=___[_ll][___[____l][___l_]]+___[_lll][___[____l][____]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[____]]%_[__ll_];break end while(__l)/((-0x30+1099))==847 do if(__[_[__l_]]~=__[_[_l___]])then _l=_l+_lll;else _l=_[_ll_];end;break end;break;end break;end break;end break;end while 1397==(__l)/((-#"Looking for a good product to waste your money for? Well I suggest RainX"+(-46+0x6ed)))do __l=(1867940)while(-#'uwu suck it harder daddy omg im going to cum!11!!'+(0x16a8/58))>=___ do __l-= __l __l=(3463524)while ___<=(-#'This file was obfuscated using PSU Obfuscator 4.0.A'+(238-0x8b))do __l-= __l __l=(1933472)while ___>(0xb9-138)do __l-= __l if(__[_[__ll]]~=__[_[_lll_]])then _l=_l+_lll;else _l=_[_llll];end;break end while 851==(__l)/(((-0x6b+529483)/233))do if(__[_[___l]]<=_[_l_l_])then _l=_[_ll_l];else _l=_l+_lll;end;break end;break;end while(__l)/((1153+-0x55))==3243 do __l=(42312)while(-#[[lua beautifier is gay]]+(0x604/22))>=___ do __l-= __l if(_[__ll]<__[_[_l_l_]])then _l=_l+_lll;else _l=_[___ll];end;break;end while(__l)/((-#[[Moonsec does not work with jjsploit plz help]]+(386-0xdb)))==344 do __l=(4941924)while ___>(0xb6-132)do __l-= __l local ___;local __l;__[_[_l__]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_llll]][__[_[_l_l_]]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];___=__[_[___l_]];if not ___ then _l=_l+_lll;else __[_[_l_l]]=___;_l=_[___ll];end;break end while(__l)/((317592/0xc6))==3081 do __[_[___l]]=(_[_ll_]~=0);break end;break;end break;end break;end while 1583==(__l)/((63720/0x36))do __l=(354795)while(1696/0x20)>=___ do __l-= __l __l=(1159136)while ___>((0x3f0/8)+-#"Your exploit crashed and now youre crying? <a:HAHAHAHA:762019852327190598>")do __l-= __l local _=_[_l__]__[_]=__[_]()break end while(__l)/((0x1f0f-4035))==296 do local __l;local ___;local _ll_,_ll_l;local _l__;local __l;__[_[___l]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_l_ll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[____]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__l=_[__ll];_l__=__[_[___ll]];__[__l+1]=_l__;__[__l]=_l__[_[__ll_]];_l=_l+_ll;_=_l_[_l];__l=_[__ll]_ll_,_ll_l=_lllll(__[__l](__[__l+_lll]))_ll__=_ll_l+__l-_ll ___=0;for _=__l,_ll__ do ___=___+_ll;__[_]=_ll_[___];end;_l=_l+_ll;_=_l_[_l];__l=_[___l]_ll_={__[__l](__lll(__,__l+1,_ll__))};___=0;for _=__l,_[___l_]do ___=___+_ll;__[_]=_ll_[___];end _l=_l+_ll;_=_l_[_l];_l=_[___ll];break end;break;end while 3255==(__l)/((24416/(-#"mahmds bbc is kinda big no cap"+(608-0x162))))do __l=(427038)while(-41+0x5f)>=___ do __l-= __l __[_[_l__]]();break;end while(__l)/((346191/0xa7))==206 do __l=(8485160)while ___>((0x113-((212+-0x26)+-#[[weirdo above]]))+-#"Oh damn I just stepped on a shit. Nevermind its just RainX")do __l-= __l __l__[_[_llll]]=__[_[__l_]];break end while 2645==(__l)/(((3273+-0x3c)+-#"clown"))do local _l_=_[__l_l];local _ll={};for _=1,#__llll do local _=__llll[_];for _l=0,#_ do local _l=_[_l];local __l=_l[1];local _=_l[2];if __l==__ and _>=_l_ then _ll[_]=__l[_];_l[1]=_ll;end;end;end;break end;break;end break;end break;end break;end break;end while(__l)/((-#"RainX more like IeatyourfuckingIPaddressX"+(0x17b8-3084)))==3040 do __l=(3984747)while(0x9d+-91)>=___ do __l-= __l __l=(700304)while ___<=((-0x70+186)+-#'RainX winning')do __l-= __l __l=(1174644)while((-0x28+200)+-102)>=___ do __l-= __l __l=(5071113)while ___>(8607/0x97)do __l-= __l __[_[_l_l]]=#__[_[_llll]];break end while 3079==(__l)/((0x6e6+-119))do __[_[_l__]]=__l__[_[_ll_]];break end;break;end while 2412==(__l)/((-29+0x204))do __l=(2064920)while(-0x2b+102)>=___ do __l-= __l local _l=_[__l_l]__[_l](__lll(__,_l+_lll,_[___ll]))break;end while(__l)/((0x1f65-4066))==520 do __l=(5477295)while(0x35e8/230)<___ do __l-= __l if __[_[_l_l]]then _l=_l+_ll;else _l=_[_l_ll];end;break end while 2627==(__l)/((0x1090-(0x27759/75)))do local __l;__[_[___l]]=__l__[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_llll]][__[_[_l___]]];_l=_l+_ll;_=_l_[_l];if(__[_[__ll]]~=_[__ll_])then _l=_l+_lll;else _l=_[___ll];end;break end;break;end break;end break;end while 2768==(__l)/((562-0x135))do __l=(710955)while(-#[[this script is secured using socks]]+((47836-0x5da6)/246))>=___ do __l-= __l __l=(3983015)while(0x2b98/(0x1b3-255))<___ do __l-= __l __[_[__l_]][_[_ll_]]=__[_[_lll_]];break end while 997==(__l)/((-0x1c+4023))do local __l;__[_[_l_l]]=__l__[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]][__[_[_ll_]]]=__[_[_l___]];_l=_l+_ll;_=_l_[_l];if not __[_[__ll]]then _l=_l+_lll;else _l=_[_l_ll];end;break end;break;end while 1295==(__l)/((86742/0x9e))do __l=(2527910)while ___<=(218-((467-0x113)+-#"3 month of free private access be like"))do __l-= __l __[_[__l_]]=#__[_[_ll_]];break;end while(__l)/((0x4515f/(-0x6f+216)))==938 do __l=(11911395)while((-0x103a/134)+0x60)<___ do __l-= __l local ___;local __l;__l=_[__ll]__[__l](__lll(__,__l+_lll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))break end while 3147==(__l)/(((-101+0xf4e)+-#[[can you find what you want here?]]))do local _lll;local ___;local __l;__[_[__ll]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=#__[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[__l]_lll=__[__l+2];if(_lll>0)then if(___>__[__l+1])then _l=_[_ll_];else __[__l+3]=___;end elseif(___<__[__l+1])then _l=_[_llll];else __[__l+3]=___;end break end;break;end break;end break;end break;end while 3237==(__l)/((0xa03-1332))do __l=(4696320)while((275-0xa2)+-#'Need help deobfuscating? Go fuck yourself.')>=___ do __l-= __l __l=(3178890)while ___<=((-0x28+209)+-101)do __l-= __l __l=(6460809)while((0x16a4/69)+-#"baxoplenty is fat")<___ do __l-= __l __[_[___l]]=__[_[_ll_]]/_[_____];break end while 3413==(__l)/(((3910-(0x1b660/56))+-#"federal SUCKS"))do __[_[___l]]=_[_ll_];break end;break;end while 1482==(__l)/(((-74+(439461/0xc1))+-#'Oh damn I just stepped on a shit. Nevermind its just RainX'))do __l=(2283495)while((0x171c/68)+-#[[LD is discounted F]])>=___ do __l-= __l local _=_[_l_l]__[_](__[_+_lll])break;end while 3713==(__l)/((-#[[the private version should be paid, like seriously, I get 0 fps drops!]]+(0x5a9-764)))do __l=(216194)while(0x108-(0xb8e8/244))<___ do __l-= __l local _l=_[_l__]local _l_={__[_l](__lll(__,_l+1,_ll__))};local __l=0;for _=_l,_[___l_]do __l=__l+_ll;__[_]=_l_[__l];end break end while 317==(__l)/(((0x301+-66)+-#'lua beautifier is gay'))do local _l_=_[_l__];local ___=_[_l_l_];local __l=_l_+2 local _l_={__[_l_](__[_l_+1],__[__l])};for _=1,___ do __[__l+_]=_l_[_];end;local _l_=_l_[1]if _l_ then __[__l]=_l_ _l=_[_ll_];else _l=_l+_ll;end;break end;break;end break;end break;end while(__l)/((584594/0xef))==1920 do __l=(10318269)while(-#'Ok but FR Rainx be staying winning'+(0x136-203))>=___ do __l-= __l __l=(9036288)while(118+-0x2e)<___ do __l-= __l local ___;local __l;__l=_[__ll]__[__l](__lll(__,__l+_lll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[_ll_]];__[__l+1]=___;__[__l]=___[_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))break end while 2664==(__l)/(((51345/(-0x12+33))+-#"person above is gayer than i am"))do __[_[__l_]]=__[_[_llll]]/_[_____];break end;break;end while 4029==(__l)/((-0x10+2577))do __l=(7469280)while ___<=(245-0xab)do __l-= __l local ___;local __l;__[_[___l]]=(_[_ll_]~=0);_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];___=__[_[_____]];if not ___ then _l=_l+_lll;else __[_[___l]]=___;_l=_[_llll];end;break;end while 2340==(__l)/((-87+0xccf))do __l=(1976275)while ___>(-#[[cock and ball torture]]+(-0x66+198))do __l-= __l __[_[__ll]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_ll_]][_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[___ll]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[____]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[_l_ll]][_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__l__[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=(not __[_[_ll_]]);_l=_l+_ll;_=_l_[_l];__[_[_l_l]][_[_ll_]]=__[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[____]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[____]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=(not __[_[_l_ll]]);_l=_l+_ll;_=_l_[_l];__[_[__l_l]][_[_ll_l]]=__[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[____]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_l_ll]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=(not __[_[_ll_]]);_l=_l+_ll;_=_l_[_l];__[_[_l__]][_[___ll]]=__[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_llll]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[_ll_l]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__l__[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=(not __[_[_l_ll]]);_l=_l+_ll;_=_l_[_l];__[_[_l__]][_[_ll_]]=__[_[___l_]];_l=_l+_ll;_=_l_[_l];do return end;break end while(__l)/((0xe9dfe/238))==491 do local __l;__[_[___l]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[_ll_l]][__[_[_l___]]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_l_ll]];_l=_l+_ll;_=_l_[_l];_l=_[_ll_l];break end;break;end break;end break;end break;end break;end break;end break;end while 1698==(__l)/((-#'so there is this thing called moonsec'+(0x30050/152)))do __l=(4447296)while ___<=(-0x4f+194)do __l-= __l __l=(1314812)while((287-0xab)+-#[[nah Federal is my dad]])>=___ do __l-= __l __l=(1686784)while((0x12a+-107)+-0x6a)>=___ do __l-= __l __l=(632142)while ___<=((-121+0xe7)+-#'mahmds bbc is kinda big no cap')do __l-= __l __l=(4432668)while ___<=(157+-0x4f)do __l-= __l __l=(2124800)while ___>(-#"oh you use krnl? name everything that broke last update."+((16744/0x34)-0xbd))do __l-= __l local _=_[__l_]__[_]=__[_](__lll(__,_+_ll,_ll__))break end while(__l)/((2136-0x458))==2075 do local ___;local __l;__[_[_l__]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[__ll];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[_ll_]][_[___l_]];_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[___l_]];break end;break;end while(__l)/(((0x80fe3/251)+-#'nah Federal is my dad'))==2127 do __l=(5288565)while((0x77+-35)+-#[[the j]])<___ do __l-= __l __[_[_l__]]=__[_[_l_ll]];break end while 4065==(__l)/((0x3374a/162))do __[_[___l]][__[_[_l_ll]]]=__[_[_l___]];break end;break;end break;end while 261==(__l)/((4925-0x9c7))do __l=(7515648)while ___<=((-55+0xd3)+-#'Your exploit crashed and now youre crying? <a:HAHAHAHA:762019852327190598>')do __l-= __l __l=(1069684)while(274-0xc1)<___ do __l-= __l if(__[_[__l_l]]==__[_[_l_l_]])then _l=_l+_lll;else _l=_[____];end;break end while 1661==(__l)/((-#[[745595618944221236]]+(0x53f-681)))do local _l=_[__l_l];local _ll=__[_[_l_ll]];__[_l+1]=_ll;__[_l]=_ll[_[_l___]];break end;break;end while(__l)/((0xaab+-43))==2796 do __l=(1511655)while(213-0x82)>=___ do __l-= __l __[_[__l_]]=__[_[_l_ll]][__[_[__ll_]]];break;end while(__l)/((120825/0x87))==1689 do __l=(12154900)while ___>(116+-0x20)do __l-= __l __[_[__l_]]=__[_[_ll_]]%_[___l_];break end while 3085==(__l)/((27580/0x7))do local ___;local __l;__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[____]))_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[___ll]];__[__l+1]=___;__[__l]=___[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))break end;break;end break;end break;end break;end while(__l)/((0x507-688))==2816 do __l=(14970183)while ___<=(0xa32/29)do __l-= __l __l=(11392406)while ___<=((4136/0x2c)+-#'hi skid')do __l-= __l __l=(6255491)while ___>((-92+0xe3)+-#'Watch out! FBI wants to break our obfuscated code')do __l-= __l local __l;local ___;__[_[_l_l]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];___=_[_ll_l];__l=__[___]for _=___+1,_[_lll_]do __l=__l..__[_];end;__[_[___l]]=__l;_l=_l+_ll;_=_l_[_l];if __[_[__ll]]then _l=_l+_ll;else _l=_[___ll];end;break end while(__l)/((0x104f-2122))==3047 do if(__[_[__l_l]]<=_[_l_l_])then _l=_[___ll];else _l=_l+_lll;end;break end;break;end while 2966==(__l)/(((0xbc098/200)+-#'Send dudes'))do __l=(4868224)while ___<=(-#"Being arrogant is like being an elephant. If someone shoots you with a bazooka, you die."+(407-0xe7))do __l-= __l local ___;local __l;__[_[_l_l]]=__l__[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_llll]][_[_lll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[___ll]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]={};_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__l__[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]][_[_llll]]=__[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l](__lll(__,__l+_lll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];do return end;break;end while(__l)/((-#[[uwu suck it harder daddy omg im going to cum!11!!]]+(-95+0x9b0)))==2084 do __l=(2162040)while ___>(-#'Need help deobfuscating? Go fuck yourself.'+(254+-0x7b))do __l-= __l if(__[_[_l_l]]~=_[_l_l_])then _l=_l+_lll;else _l=_[_ll_];end;break end while(__l)/((0x10ce-2207))==1032 do local ___;local __l;__[_[_l_l]]=__l__[_[___ll]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_ll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[_ll_]))_l=_l+_ll;_=_l_[_l];do return end;break end;break;end break;end break;end while 4021==(__l)/((-#"tried playing roblox once and how tf people enjoy it"+(543600/0x90)))do __l=(64156)while ___<=(0xe1-(-#"how bored do you have to be to read this"+(7439/0x2b)))do __l-= __l __l=(2129904)while(217-0x7e)<___ do __l-= __l local _ll=_[___l];local _l_=__[_ll+2];local __l=__[_ll]+_l_;__[_ll]=__l;if(_l_>0)then if(__l<=__[_ll+1])then _l=_[_l_ll];__[_ll+3]=__l;end elseif(__l>=__[_ll+1])then _l=_[_l_ll];__[_ll+3]=__l;end break end while(__l)/((131040/0x82))==2113 do _l__l[_[_llll]]=__[_[_l_l]];break end;break;end while 373==(__l)/((274+(-#[[no he wasnt]]+(-0x94+57))))do __l=(6581601)while ___<=(282-(15309/0x51))do __l-= __l local ___;local __l;__l=_[__l_l]__[__l](__lll(__,__l+_lll,_[___ll]))_l=_l+_ll;_=_l_[_l];__l=_[__ll];___=__[_[_l_ll]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))break;end while 3309==(__l)/((0x78e9d/249))do __l=(134895)while ___>(-#[[3 month of free private access be like]]+(-0x61+229))do __l-= __l local ___;local __l;__l=_[__l_]__[__l](__lll(__,__l+_lll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__l=_[_l_l];___=__[_[_l_ll]];__[__l+1]=___;__[__l]=___[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))break end while(__l)/((-53+0x4ca))==115 do local _=_[__l_]__[_]=__[_](__lll(__,_+_ll,_ll__))break end;break;end break;end break;end break;end break;end while 1279==(__l)/((-91+0x45f))do __l=(3207985)while(((-0x1c+-54)+223)+-#'can anyone show me a circular square')>=___ do __l-= __l __l=(2262730)while(0x17d4/61)>=___ do __l-= __l __l=(1446774)while(0xd4-115)>=___ do __l-= __l __l=(9295450)while(0xe5-133)<___ do __l-= __l __[_[_l_l]]=_l__l[_[_l_ll]];break end while 3151==(__l)/((0xa154/(728/0x34)))do local _={__,_};_[_ll][_[____l][___l]]=_[____l][_lll_]+_[____l][_l_ll];break end;break;end while 777==(__l)/((0xea2-1884))do __l=(1403234)while ___<=(-#"mapple was here"+(320-0xcf))do __l-= __l __[_[___l]]();break;end while 1582==(__l)/((0x723-940))do __l=(5075189)while((400+-0x7e)-175)<___ do __l-= __l local _ll=_[__l_l];local __l=__[_ll]local _l_=__[_ll+2];if(_l_>0)then if(__l>__[_ll+1])then _l=_[_ll_];else __[_ll+3]=__l;end elseif(__l<__[_ll+1])then _l=_[_llll];else __[_ll+3]=__l;end break end while(__l)/((-#"RainX be winning 4evr"+(0xf5f-((0x104f-2150)+-#'Wanna hear a joke? RainX'))))==2653 do local _=_[_l_l]__[_]=__[_](__[_+_lll])break end;break;end break;end break;end while(__l)/((-#"when you tried to skid a moonsec obfuscated script"+(0x28875/51)))==706 do __l=(4117920)while ___<=((0x165-242)+-#'RainX winning')do __l-= __l __l=(159323)while(0x11c-183)<___ do __l-= __l if(__[_[___l]]==_[_lll_])then _l=_l+_lll;else _l=_[_llll];end;break end while(__l)/(((6105-0xc0e)-1530))==107 do local ___;local __l;__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__l=_[__ll];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_ll_l]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];break end;break;end while 1865==(__l)/(((-0x43+2325)+-#"when you tried to skid a moonsec obfuscated script"))do __l=(2693303)while ___<=(0x142-219)do __l-= __l do return end;break;end while 1187==(__l)/((458338/0xca))do __l=(10361295)while ___>(4680/0x2d)do __l-= __l local ___;local __l;__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__l=_[__l_l];___=__[_[____]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))break end while 3885==(__l)/((-#"SirHurt trying not to get their database leaked for 5 seconds"+(100936/0x25)))do local __l;__[_[_l_l]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__lll(__,__l+_ll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[_ll_]][__[_[_l_l_]]];_l=_l+_ll;_=_l_[_l];if(__[_[__l_]]~=_[_lll_])then _l=_l+_lll;else _l=_[____];end;break end;break;end break;end break;end break;end while(__l)/(((8227-0x1050)+-#'can anyone show me a circular square'))==799 do __l=(10436389)while ___<=(-0x2c+154)do __l-= __l __l=(2043335)while ___<=(-#'This file was obfuscated using PSU Obfuscator 4.0.A'+(-20+0xb2))do __l-= __l __l=(7712198)while ___>((-0x1a+174)+-#[[how to deobfuscate this code: you cant lol]])do __l-= __l __[_[__l_]]=__[_[___ll]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_llll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_llll]]-__[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_ll_]][_[__ll_]];_l=_l+_ll;_=_l_[_l];if(__[_[_l__]]<=_[_lll_])then _l=_[_ll_];else _l=_l+_lll;end;break end while(__l)/((0x72927/189))==3106 do local _ll=_[__l_];local __l=__[_ll]local _l_=__[_ll+2];if(_l_>0)then if(__l>__[_ll+1])then _l=_[___ll];else __[_ll+3]=__l;end elseif(__l<__[_ll+1])then _l=_[_llll];else __[_ll+3]=__l;end break end;break;end while 2765==(__l)/((-0x15+760))do __l=(5074870)while(272-0xa4)>=___ do __l-= __l local ___;local __l;__[_[___l]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_ll_]];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l](__lll(__,__l+_lll,_[___ll]))_l=_l+_ll;_=_l_[_l];do return end;break;end while 1910==(__l)/((0x9916c/236))do __l=(6471832)while(0x113-166)<___ do __l-= __l local _ll=_[_l__];local _l=__[_[_l_ll]];__[_ll+1]=_l;__[_ll]=_l[_[___l_]];break end while(__l)/((-#[[ConfusedByAttribute]]+(604224/0xc0)))==2069 do local _ll=_[____];local _l=__[_ll]for _=_ll+1,_[__ll_]do _l=_l..__[_];end;__[_[___l]]=_l;break end;break;end break;end break;end while(__l)/((0xbed+-114))==3551 do __l=(3494582)while ___<=(-#[[Oh damn I just stepped on a shit. Nevermind its just RainX]]+(0x1c8-286))do __l-= __l __l=(215600)while(0xc6+-87)<___ do __l-= __l __[_[__ll]][_[_llll]]=_[__ll_];break end while(__l)/((4021-0x80d))==110 do local ___;local __l;__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[____]))_l=_l+_ll;_=_l_[_l];__l=_[_l_l];___=__[_[____]];__[__l+1]=___;__[__l]=___[_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))break end;break;end while(__l)/(((-77+0xb10)+-#"i love mahmd"))==1274 do __l=(1893137)while ___<=(0x104-147)do __l-= __l __[_[__ll]]=(not __[_[___ll]]);break;end while 3281==(__l)/(((0x4f7-669)+-#'rainx is the best ever!!!'))do __l=(940044)while((14157/0x63)+-#'Kitty was here you sussy baka')<___ do __l-= __l local _ll=__[_[_lll_]];if not _ll then _l=_l+_lll;else __[_[_l__]]=_ll;_l=_[_l_ll];end;break end while(__l)/((5227-0xa3f))==361 do _l=_[_llll];break end;break;end break;end break;end break;end break;end break;end while(__l)/((-#[[cock]]+(0x1bd65/103)))==4032 do __l=(12207875)while(0x20fa/63)>=___ do __l-= __l __l=(1420677)while(18228/0x93)>=___ do __l-= __l __l=(5363648)while(-51+((0x2fa2/67)+-#"i love mahmd"))>=___ do __l-= __l __l=(2559384)while(340-0xdf)>=___ do __l-= __l __l=(2147229)while ___>(-#[[so there is this thing called moonsec]]+(0x18d-244))do __l-= __l local _={__,_};_[_lll][_[____l][_l_l]]=_[_ll][_[____l][___l_]]+_[_lll][_[____l][____]];break end while 567==(__l)/(((95200/0x19)+-#"nah Federal is my dad"))do local __l;__[_[_l__]]=__l__[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__[_[_l__]][__[_[_ll_]]]=__[_[_lll_]];_l=_l+_ll;_=_l_[_l];do return end;break end;break;end while(__l)/((((0xc38-1608)+-110)+-#[[wally likes kids]]))==1836 do __l=(249135)while((0x1e7-259)+-#'RainX logs your raw IPs, dont ever use it you will regret it trust me I know what I am talking about (federal)')<___ do __l-= __l local __l=_[__l_]local _l_={__[__l](__lll(__,__l+1,_ll__))};local _l=0;for _=__l,_[_l_l_]do _l=_l+_ll;__[_]=_l_[_l];end break end while(__l)/((5950/0x46))==2931 do local _l_=_[___l];local ___=_[_lll_];local __l=_l_+2 local _l_={__[_l_](__[_l_+1],__[__l])};for _=1,___ do __[__l+_]=_l_[_];end;local _l_=_l_[1]if _l_ then __[__l]=_l_ _l=_[_llll];else _l=_l+_ll;end;break end;break;end break;end while(__l)/((2789-0x585))==3898 do __l=(288706)while ___<=(322-0xc9)do __l-= __l __l=(2282812)while((-103+0x129)+-#'Your exploit crashed and now youre crying? <a:HAHAHAHA:762019852327190598>')<___ do __l-= __l if __[_[__ll]]then _l=_l+_ll;else _l=_[_ll_l];end;break end while(__l)/((705+-0x5c))==3724 do __[_[__l_]]=(not __[_[_l_ll]]);break end;break;end while 242==(__l)/((1215+-0x16))do __l=(674839)while ___<=((0x187-259)+-#'Send dudes')do __l-= __l if(__[_[__ll]]==_[__ll_])then _l=_l+_lll;else _l=_[_ll_];end;break;end while 1979==(__l)/((-0x21+374))do __l=(4542006)while ___>(0xba+-63)do __l-= __l __[_[__ll]][__[_[_ll_]]]=__[_[___l_]];break end while 2207==(__l)/(((434042/0xce)+-#'Watch out! FBI wants to break our obfuscated code'))do if not __[_[__ll]]then _l=_l+_lll;else _l=_[_ll_];end;break end;break;end break;end break;end break;end while 3671==(__l)/((17028/0x2c))do __l=(4555920)while ___<=((0x8a63/241)+-#[[LD is discounted F]])do __l-= __l __l=(2307690)while((227+-0x59)+-#[[i love mahmd]])>=___ do __l-= __l __l=(5330570)while((409-0x10a)+-#'745595618944221236')<___ do __l-= __l local ___;local __l;__[_[_l__]]=__l__[_[_llll]];_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[_[____]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[_ll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l](__lll(__,__l+_lll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];do return end;break end while(__l)/(((-41+0xae8)+-#'RainX more like IeatyourfuckingIPaddressX'))==1967 do local __l;__[_[__l_]]=__[_[_ll_l]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_ll_]][_[_____]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[___ll]][_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[___ll]]/_[_lll_];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_llll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[___ll]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[_llll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[____]]/_[_lll_];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[___ll]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_l_ll]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[___ll]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[___ll]]/_[___l_];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[____]][_[_lll_]];break end;break;end while(__l)/((-#'hi skid'+(188881/0x79)))==1485 do __l=(5037120)while ___<=(5207/0x29)do __l-= __l __[_[___l]]=__[_[_ll_l]][__[_[_l___]]];break;end while(__l)/(((-0x75+3123)+-#[[stop reading the code sussy bakayero]]))==1696 do __l=(11777608)while ___>(-#[[dn method]]+(19454/0x8e))do __l-= __l local _l=_[_l__]__[_l](__lll(__,_l+_lll,_[____]))break end while(__l)/((-0x23+3058))==3896 do local ____l;local __l__,__llll;local ___;local __l;__[_[__l_]][_[_ll_l]]=_[_____];_l=_l+_ll;_=_l_[_l];__[_[___l]][_[____]]=_[___l_];_l=_l+_ll;_=_l_[_l];__[_[__l_l]][_[_llll]]=_[_l_l_];_l=_l+_ll;_=_l_[_l];__[_[__l_]]={};_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[_[_ll_]];__[__l+1]=___;__[__l]=___[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_ll_]][_[_____]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[_ll_l]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[____];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__[_[____]][__[_[_l_l_]]];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[___ll]][_[___l_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[_l_ll]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__l_]][_[____]]=__[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[_[_l_ll]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_llll];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_llll]][_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[___ll]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[_ll_]][__[_[_l___]]];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[____]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_ll_]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]][_[_l_ll]]=__[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l];___=__[_[___ll]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__l=_[__ll]__l__,__llll=_lllll(__[__l](__lll(__,__l+1,_[____])))_ll__=__llll+__l-1 ____l=0;for _=__l,_ll__ do ____l=____l+_ll;__[_]=__l__[____l];end;_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l](__lll(__,__l+_ll,_ll__))_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l]()_l=_l+_ll;_=_l_[_l];__[_[___l]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_llll]];_l=_l+_ll;_=_l_[_l];__l=_[__ll];___=__[_[_ll_]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__l=_[___l]__l__,__llll=_lllll(__[__l](__lll(__,__l+1,_[_ll_])))_ll__=__llll+__l-1 ____l=0;for _=__l,_ll__ do ____l=____l+_ll;__[_]=__l__[____l];end;_l=_l+_ll;_=_l_[_l];__l=_[_l__]__l__,__llll=_lllll(__[__l](__lll(__,__l+_ll,_ll__)))_ll__=__llll+__l-_lll ____l=0;for _=__l,_ll__ do ____l=____l+_ll;__[_]=__l__[____l];end;_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_ll__))_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l]=__[__l]()_l=_l+_ll;_=_l_[_l];__l=_[___l];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_]))_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[____];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_llll];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__lll(__,__l+_ll,_[___ll]))_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__lll(__,__l+_ll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__[_[_l_ll]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l_l];___=__[_[_l_ll]];__[__l+1]=___;__[__l]=___[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_l__l[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_l_ll];break end;break;end break;end break;end while(__l)/(((1359+-0x50)+-#'uwu suck it harder daddy omg im going to cum!11!!'))==3704 do __l=(1521520)while ___<=(-#'impossible'+(6768/0x30))do __l-= __l __l=(11126284)while ___>(307-0xb1)do __l-= __l __[_[_l__]]={};break end while 3926==(__l)/((0xb2c+-26))do local _=_[__l_l]local __l,_l=_lllll(__[_](__[_+_lll]))_ll__=_l+_-_ll local _l=0;for _=_,_ll__ do _l=_l+_ll;__[_]=__l[_l];end;break end;break;end while 1040==(__l)/((-#"Only Donals Trump supporters uses public obfuscators"+(0xc43-1624)))do __l=(132388)while ___<=((1593240/0x55)/142)do __l-= __l local _ll=_[__ll];local _l_=__[_ll+2];local __l=__[_ll]+_l_;__[_ll]=__l;if(_l_>0)then if(__l<=__[_ll+1])then _l=_[_l_ll];__[_ll+3]=__l;end elseif(__l>=__[_ll+1])then _l=_[___ll];__[_ll+3]=__l;end break;end while 1439==(__l)/((209-0x75))do __l=(486522)while ___>(-#'Best ironbrew remake'+(0xc6+-45))do __l-= __l local _=_[__l_]local __l,_l=_lllll(__[_](__lll(__,_+_ll,_ll__)))_ll__=_l+_-_lll local _l=0;for _=_,_ll__ do _l=_l+_ll;__[_]=__l[_l];end;break end while 1611==(__l)/((-#[[wally likes kids]]+(741-((44880/0x66)+-#'RainX be the best'))))do local ___;local __l;__[_[__ll]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_ll_]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__[_[_ll_]][__[_[__ll_]]];_l=_l+_ll;_=_l_[_l];__l=_[__ll]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];___=__[_[_l_l_]];if not ___ then _l=_l+_lll;else __[_[_l_l]]=___;_l=_[____];end;break end;break;end break;end break;end break;end break;end while(__l)/((3885+-0x28))==3175 do __l=(13592326)while ___<=(24480/0xaa)do __l-= __l __l=(1383148)while ___<=(((0x6a5ba/17)+-#'when you tried to skid a moonsec obfuscated script')/184)do __l-= __l __l=(292125)while ___<=((0xac3/19)+-#"dn method")do __l-= __l __l=(2927720)while ___>(0x556e/162)do __l-= __l local _l=_[_l__]local __l,_=_lllll(__[_l](__lll(__,_l+1,_[_ll_])))_ll__=_+_l-1 local _=0;for _l=_l,_ll__ do _=_+_ll;__[_l]=__l[_];end;break end while(__l)/((5569-0xaf7))==1060 do __[_[_l_l]]=__l__[_[___ll]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=#__[_[_ll_]];_l=_l+_ll;_=_l_[_l];__l__[_[_l_ll]]=__[_[__l_l]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=__l__[_[____]];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=#__[_[___ll]];_l=_l+_ll;_=_l_[_l];__l__[_[___ll]]=__[_[__l_]];_l=_l+_ll;_=_l_[_l];do return end;break end;break;end while 75==(__l)/((7807-0xf48))do __l=(4275152)while ___<=(-58+0xc3)do __l-= __l local _l=_[__l_l]__[_l]=__[_l](__lll(__,_l+_ll,_[___ll]))break;end while 1148==(__l)/((0xefe+-114))do __l=(2435785)while(349-0xd3)<___ do __l-= __l local _ll=__[_[__ll_]];if not _ll then _l=_l+_lll;else __[_[___l]]=_ll;_l=_[_llll];end;break end while(__l)/(((0x2eb+(-0x75+40))+-#[[dn method]]))==3685 do local ___;local __l;__[_[_l__]]=_l__l[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__l=_[__l_l];___=__[_[_ll_l]];__[__l+1]=___;__[__l]=___[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_ll_l];_l=_l+_ll;_=_l_[_l];__l=_[__l_]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[___ll]][_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_ll_l]][_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_l__l[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_llll];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[___l]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[____]))_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[____]][__[_[___l_]]];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l]=__[__l](__[__l+_lll])_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_llll]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=__[_[_ll_]][_[__ll_]];_l=_l+_ll;_=_l_[_l];__[_[___l]][_[_ll_l]]=__[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__l__[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[____]][_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[___l]][_[_ll_]]=__[_[___l_]];break end;break;end break;end break;end while 871==(__l)/(((-97+0x6a1)+-#"i love mahmd"))do __l=(947703)while(12690/0x5a)>=___ do __l-= __l __l=(3399367)while ___>(-0x2b+183)do __l-= __l __[_[__l_]]=_l_lll(___lll[_[_llll]],nil,_l__l);break end while 2069==(__l)/((3381-(3548-0x712)))do local __l;__[_[___l]]=_l__l[_[____]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__[_[__l_l]]=_[_ll_];_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[___ll];_l=_l+_ll;_=_l_[_l];__l=_[_l_l]__[__l]=__[__l](__lll(__,__l+_ll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[__l_]]=_[_l_ll];_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l](__lll(__,__l+_lll,_[_llll]))break end;break;end while 309==(__l)/((6247-0xc6c))do __l=(14790816)while(342-0xc8)>=___ do __l-= __l local _=_[___l]local __l,_l=_lllll(__[_](__lll(__,_+_ll,_ll__)))_ll__=_l+_-_lll local _l=0;for _=_,_ll__ do _l=_l+_ll;__[_]=__l[_l];end;break;end while 4028==(__l)/(((7619-0xf07)+-100))do __l=(1519024)while(-#'dn method'+(-57+(449-0xf0)))<___ do __l-= __l do return __[_[__l_]]end break end while(__l)/((3584-0x732))==872 do __[_[__ll]]=_l_lll(___lll[_[___ll]],nil,_l__l);break end;break;end break;end break;end break;end while(__l)/((0x4578a/73))==3487 do __l=(53485)while(31737/0xd5)>=___ do __l-= __l __l=(523570)while ___<=(-#'why are you wasting your time here? go outside and get a partner'+(0xf8+-38))do __l-= __l __l=(1125333)while(352-0xcf)<___ do __l-= __l __[_[__l_l]]=_[____];break end while 297==(__l)/(((7742-0xf40)+-#[[uwu suck it harder daddy omg im going to cum!11!!]]))do __l__[_[_l_ll]]=__[_[_l_l]];break end;break;end while(__l)/((-#"baxoplenty is fat"+(962-0x217)))==1277 do __l=(12502534)while ___<=(322-0xaf)do __l-= __l local __ll=___lll[_[_llll]];local ___;local _ll={};___=____ll({},{__index=function(_l,_)local _=_ll[_];return _[1][_[2]];end,__newindex=function(__,_,_l)local _=_ll[_]_[1][_[2]]=_l;end;});for __l=1,_[___l_]do _l=_l+_lll;local _=_l_[_l];if _[(115/0x73)]==150 then _ll[__l-1]={__,_[_l_ll]};else _ll[__l-1]={__l__,_[_l_ll]};end;__llll[#__llll+1]=_ll;end;__[_[___l]]=_l_lll(__ll,___,_l__l);break;end while(__l)/((0x1f00-3987))==3166 do __l=(1995296)while(0x162-206)<___ do __l-= __l __[_[_l_l]]=__[_[___ll]]%_[_l___];break end while(__l)/((-#'nah Federal is my dad'+(-79+0x344)))==2711 do local _l=_[__l_]__[_l]=__[_l](__lll(__,_l+_ll,_[_ll_]))break end;break;end break;end break;end while(__l)/((-#'person below gay asf'+(87885/0x1f)))==19 do __l=(1293123)while ___<=((297+-0x60)+-#"when you tried to skid a moonsec obfuscated script")do __l-= __l __l=(3686517)while ___>(415-0x109)do __l-= __l __[_[__l_]]=(_[___ll]~=0);_l=_l+_lll;break end while(__l)/((0x149f-2670))==1413 do __[_[__l_]]=__[_[___ll]];break end;break;end while(__l)/(((2951-((0x17356/58)+-#[[RainX logs your raw IPs, dont ever use it you will regret it trust me I know what I am talking about (federal)]]))+-#'nah Federal is my dad'))==923 do __l=(459792)while((197+-0x19)+-#[[Best ironbrew remake]])>=___ do __l-= __l local ___;local __l;__[_[__ll]]=__l__[_[_l_ll]];_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[_llll]];__[__l+1]=___;__[__l]=___[_[_____]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=__[_[____]];_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l](__lll(__,__l+_lll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];do return end;break;end while(__l)/((0xa434/113))==1236 do __l=(322908)while ___>(0x169-208)do __l-= __l local _l=_[__l_]local __l,_=_lllll(__[_l](__lll(__,_l+1,_[_l_ll])))_ll__=_+_l-1 local _=0;for _l=_l,_ll__ do _=_+_ll;__[_l]=__l[_];end;break end while 213==(__l)/((-40+0x614))do local ___;local __l;__[_[__l_]]=__l__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[__l_];___=__[_[_ll_]];__[__l+1]=___;__[__l]=___[_[_lll_]];_l=_l+_ll;_=_l_[_l];__[_[_l__]]=(_[_l_ll]~=0);_l=_l+_ll;_=_l_[_l];__l=_[__l_l]__[__l](__lll(__,__l+_lll,_[_l_ll]))_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=__l__[_[_ll_]];_l=_l+_ll;_=_l_[_l];__l=_[_l__];___=__[_[_ll_]];__[__l+1]=___;__[__l]=___[_[_l___]];_l=_l+_ll;_=_l_[_l];__[_[___l]]=__[_[_ll_l]];_l=_l+_ll;_=_l_[_l];__l=_[_l__]__[__l](__lll(__,__l+_lll,_[_llll]))_l=_l+_ll;_=_l_[_l];__[_[__ll]]=__l__[_[____]];_l=_l+_ll;_=_l_[_l];__l=_[__l_l];___=__[_[___ll]];__[__l+1]=___;__[__l]=___[_[_l_l_]];_l=_l+_ll;_=_l_[_l];__[_[_l_l]]=(_[_llll]~=0);_l=_l+_ll;_=_l_[_l];__l=_[___l]__[__l](__lll(__,__l+_lll,_[_ll_l]))_l=_l+_ll;_=_l_[_l];do return end;break end;break;end break;end break;end break;end break;end break;end break;end _l+= _lll end;end);end;return _l_lll(_lll_l(),{},_ll__l())()end)_msec({[(0xc5+(-72+0xa))]='\115\116'..(function(_)return(_ and'(0x258/6)')or'\114\105'or'\120\58'end)((0x17c/76)==(798/0x85))..'\110g',[(107325/0x87)]='\108\100'..(function(_)return(_ and'(23800/0xee)')or'\101\120'or'\119\111'end)((1205/0xf1)==(-0x41+71))..'\112',[(232+-#{'nil';(function()return('BoMOi0'):find("\77")end)();44,nil;(function(_)return _ end)(),(function()return('iAOOIR'):find("\6")end)(),(function(_)return _ end)()})]=(function(_)return(_ and'(17500/0xaf)')and'\98\121'or'\100\120'end)((1090/(7630/0x23))==(-#[[balls]]+(92-0x52)))..'\116\101',[(-#[[no he wasnt]]+(0x1b7+-110))]='\99'..(function(_)return(_ and'(-#\'wally likes kids\'+(165+-0x31))')and'\90\19\157'or'\104\97'end)((-#[[this file is protected by socks]]+(0xb9-149))==(0x1a-(-#'Oh damn I just stepped on a shit. Nevermind its just RainX'+(268-0xbb))))..'\114',[(-#"how to deobfuscate this code: you cant lol"+(1267-0x285))]='\116\97'..(function(_)return(_ and'(312-0xd4)')and'\64\113'or'\98\108'end)((-0x74+122)==(0x13b/63))..'\101',[(15505/0x23)]=(function(_)return(_ and'(-#[[the j]]+(22680/0xd8))')or'\115\117'or'\78\107'end)((-#'Being arrogant is like being an elephant. If someone shoots you with a bazooka, you die.'+(0x349c/148))==(0x1132/142))..'\98',[(-#"person below gay asf"+(76804/0x5b))]='\99\111'..(function(_)return(_ and'(-#\'PSU more like ass\'+(-0x10+133))')and'\110\99'or'\110\105\103\97'end)((0x81d/67)==(0x87-104))..'\97\116',[(0x2d2+-100)]=(function(_,_l)return(_ and'(-#"person above is gayer than i am"+(0xa3c/20))')and'\48\159\158\188\10'or'\109\97'end)((0x2da/146)==(72-0x42))..'\116\104',[(-0x19+(-0x34+1427))]=(function(_,_l)return(((5168/0x98)+-#"RainX trippin on god (on top)")==(-#"when the obfuscator is too secure that the script doesnt work"+(-61+0x7d))and'\48'..'\195'or _..((not'\20\95\69'and'\90'..'\180'or _l)))or'\199\203\95'end),[(919+-#{",",(function()return('MAIMHN'):find("\73")end)(),(function()return{','}end)();109,",";nil;kernel32dII})]='\105\110'..(function(_,_l)return(_ and'(-79+0xb3)')and'\90\115\138\115\15'or'\115\101'end)((0x122/58)==(0x9d+-126))..'\114\116',[((1166+-0x59)+-#"lua beautifier is gay")]='\117\110'..(function(_,_l)return(_ and'(-#"RainX winning"+(16385/0x91))')or'\112\97'or'\20\38\154'end)((50+(-0xb-34))==(-#"RainX is fr garbageeeee ew"+(0x97+-94)))..'\99\107',[(-#{'}';87,'}',(function()return('Ao0EH0'):find("\48")end)(),87,(function(_)return _ end)();nil}+1168)]='\115\101'..(function(_)return(_ and'(0x52d0/212)')and'\110\112\99\104'or'\108\101'end)((0x24e/118)==(0x72+-83))..'\99\116',[(1332+-0x64)]='\116\111\110'..(function(_,_l)return(_ and'(0x8d+-41)')and'\117\109\98'or'\100\97\120\122'end)((-#[[baxoplenty is fat]]+(0x7f-105))==((0x2e8a/161)+-#[[also congratulations sirhurt users for having your data leaked again!]]))..'\101\114'},{[(0xb8-100)]=((getfenv))},((getfenv))()) end)()
