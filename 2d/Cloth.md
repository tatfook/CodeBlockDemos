```lua
-- settings
physics_accuracy  = 3
mouse_influence   = 20
mouse_cut         = 5
gravity           = 1200
cloth_height      = 30
cloth_width       = 50
start_y           = 20
spacing           = 7
tear_distance     = 60

local boundsx, boundsy
local mouse = {    down = false, button = "left", x = 0, y = 0, px = 0,py = 0};
local ctx

local Point = inherit(nil, "Point")
function Point:init(x, y)
    self.x      = x;
    self.y      = y;
    self.px     = x;
    self.py     = y;
    self.vx     = 0;
    self.vy     = 0;
    self.constraints = {};
    return self;
end

function Point:update(delta)
    if (mouse.down) then
        local diff_x = self.x - mouse.x;
        local diff_y = self.y - mouse.y;
        local dist = math.sqrt(diff_x * diff_x + diff_y * diff_y);

        if (mouse.button == "left") then
            if (dist < mouse_influence) then
                self.px = self.x - (mouse.x - mouse.px) * 1.8;
                self.py = self.y - (mouse.y - mouse.py) * 1.8;
            end
        elseif (dist < mouse_cut) then
            self.constraints = {}; 
        end
    end

    self:add_force(0, gravity);

    delta = delta*delta;
    local nx = self.x + ((self.x - self.px) * 0.99) + ((self.vx / 2) * delta);
    local ny = self.y + ((self.y - self.py) * 0.99) + ((self.vy / 2) * delta);

    self.px = self.x;
    self.py = self.y;

    self.x = nx;
    self.y = ny;

    self.vy = 0;
    self.vx = 0;
end

function Point:draw()
    if (#self.constraints == 0) then
        return;
    end
    for i=1, #self.constraints do
        self.constraints[i]:draw();
    end
end

function Point:resolve_constraints()
    if (self.pin_x  and self.pin_y) then
        self.x = self.pin_x;
        self.y = self.pin_y;
        return;
    end
    for i=#self.constraints, 1, -1 do
        self.constraints[i]:resolve();
    end
    if(self.x > boundsx) then
        self.x = 2 * boundsx - self.x
    elseif(1 > self.x) then 
        self.x = 2 - self.x;
    end
    if(self.y < 1) then
        self.y = 2 - self.y
    elseif(self.y > boundsy) then
        self.y = 2 * boundsy - self.y
    end
end

function Point:attach(point)
    self.constraints[#self.constraints+1] = Constraint:new():init(self, point);
end

function Point:remove_constraint(constraint)
    for i=1, #self.constraints do
        if(self.constraints[i] == constraint) then
            commonlib.removeArrayItem(self.constraints, i)
            break;
        end
    end
end

function Point:add_force(x, y)
    self.vx = self.vx + x;
    self.vy = self.vy + y;
  
    local round = 400;
    self.vx = math.floor(self.vx * round) / round;
    self.vy = math.floor(self.vy * round) / round;
end

function Point:pin(pinx, piny)
    self.pin_x = pinx;
    self.pin_y = piny;
end


local Constraint = inherit(nil, "Constraint")
function Constraint:init(p1, p2)
    self.p1     = p1;
    self.p2     = p2;
    self.length = spacing;
    return self;
end

function Constraint:resolve()
    local diff_x  = self.p1.x - self.p2.x
    local diff_y  = self.p1.y - self.p2.y
    local dist    = math.sqrt(diff_x * diff_x + diff_y * diff_y)
    local diff    = (self.length - dist) / dist

    if (dist > tear_distance) then
        self.p1:remove_constraint(self);
    end

    local px = diff_x * diff * 0.5;
    local py = diff_y * diff * 0.5;

    self.p1.x = self.p1.x + px;
    self.p1.y = self.p1.y + py;
    self.p2.x = self.p2.x - px;
    self.p2.y = self.p2.y - py;
end

function Constraint:draw()
    ctx:moveTo(self.p1.x, self.p1.y);
    ctx:lineTo(self.p2.x, self.p2.y);
end

local Cloth = inherit(nil, "Cloth")

function Cloth:ctor()
    self.points = {};
    local start_x = ctx.width / 2 - cloth_width * spacing / 2;

    for y = 0, cloth_height do
        for x = 0, cloth_width do
            local p = Point:new():init(start_x + x * spacing, start_y + y * spacing);

            if(x ~= 0) then p:attach(self.points[#self.points]) end
            if(y == 0) then p:pin(p.x, p.y) end
            if(y ~= 0) then p:attach(self.points[x + (y - 1) * (cloth_width + 1) +1]) end
            self.points[#self.points+1] = p;
        end
    end
end

function Cloth:update()
    for k=1, physics_accuracy do
        for i=1, #self.points do
            self.points[i]:resolve_constraints();
        end
    end
    for i=1, #self.points do
        self.points[i]:update(0.016);
    end
end

function Cloth:draw()
    ctx:beginPath();
    for i=1, #self.points do
        self.points[i]:draw();
    end
    ctx:stroke();
end

function start()
    window([[
        <div>Tear the cloth with your mouse</div>
        <div>Right click and drag to cut</div>
    ]], "_lt",150,30,200,50);
    local wnd = window("", "_lt",20,70,560,350);
    ctx = wnd:getContext();
        
    wnd:registerEvent("onmousedown", function(e)
        mouse.button  = e:button();
        mouse.px  = mouse.x;
        mouse.py  = mouse.y;
        mouse.x       = e:pos():x();
        mouse.y       = e:pos():y();
        mouse.down    = true;
        e:accept()
    end)
    wnd:registerEvent("onmousemove", function(e)
        mouse.px  = mouse.x;
        mouse.py  = mouse.y;
        mouse.x       = e:pos():x();
        mouse.y       = e:pos():y();
        e:accept()
    end)
    wnd:registerEvent("onmouseup", function(e)
        mouse.down = false;
        e:accept()
    end)
    
    boundsx = ctx.width - 1;
    boundsy = ctx.height - 1;
    ctx.strokeStyle = '#808080';
    local cloth = Cloth:new();
    cloth:draw();
    registerTickEvent(1, function()
        ctx:clearRect();
        cloth:update();
        cloth:draw();
    end)
end

start()
```
