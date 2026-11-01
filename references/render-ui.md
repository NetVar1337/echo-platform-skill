# Render, Menu UI, Logging reference

## Context rules

Render drawing calls are only valid **from the render thread** (`on_render(cb)`). Marked exceptions callable from any context: `render::get_fps()`, `get_frametime()`, `is_foreground()`, `get_viewport_size()` (vec2 or `bool get_viewport_size(int &out w, int &out h)`), `set_anti_aliasing(bool)`, `load_font(...)`, `load_image(...)`.

## Render — general

```cpp
float render::get_fps();
float render::get_frametime();
bool render::is_foreground();
vec2 render::get_viewport_size();
void render::set_anti_aliasing(bool enabled); // toggle freely between draw calls
```

## Fonts

```cpp
enum fonts { PROGGY, TAHOMA, TAHOMA_PX, TAHOMA_BOLD, TAHOMA_BOLD_PX, SMALLEST_PIXEL_7, TAHOMISH };
enum font_hint { NONE, AUTO, MONO };
struct font_handle { int index; }

bool render::load_font(const array<uint8> &in, font_handle &out, font_hint = AUTO); // any context
void render::update_font(font_handle font, int size);   // render context only
void render::update_font(fonts font, int size);          // render context only
```

## Images

```cpp
bool render::load_image(string data, image &out obj);    // any context; data = raw file bytes
void render::draw_image(image obj, int x, int y, int w, int h);                          // render ctx
void render::draw_image(image obj, int x, int y, int w, int h, int r, int g, int b, int a);
// load via: string d; file::read("dino.png", d); render::load_image(d, img);
```

## Text (render context)

```cpp
void render::update_font(fonts font, int size); // set current font
vec2 render::calculate_text_size(string text);  // width:x, height:y
void render::draw_text(int x, int y, string text, int r, int g, int b, int a);
void render::draw_text_outlined(...);  // with outline
void render::draw_text_shadow(...);    // 1px down+right shadow
```

## Boxes & gradients (render context)

```cpp
enum box_style { NORMAL, OUTLINED, FILLED, FILLED_OUTLINED, CORNERED, CORNERED_FILLED }; // box_style::OUTLINED
void render::draw_box(int x, int y, int w, int h, int r, int g, int b, int a, box_style style, float rounding);
// cornered styles: rounding = corner_ratio 0.05–0.45 (share of each edge)
void render::draw_gradient_box(int x, int y, int w, int h, int r1,g1,b1,a1, int r2,g2,b2,a2, bool horizontal);
```

## 3D → 2D projection (render context)

```cpp
struct bbox_2d { float x, y, width, height; bool valid; }
bbox_2d render::calculate_2d_bbox(vec3 origin, vec3 mins, vec3 maxs, matrix4x4 view_matrix);
bbox_2d render::calculate_2d_bbox_unity(vec3 origin, vec3 center, vec3 extents, matrix4x4 view_matrix); // Unity LH Y-up: center+half-extents
```

Engine-specific w2s helpers (take a view matrix/cache instead): see `engines-math.md` (`source::w2s`, `unreal::w2s`, `unity::w2s`, `iw_engine::w2s`).

## Lines & shapes (render context)

```cpp
void render::draw_line(float x1, float y1, float x2, float y2, int r, int g, int b, int a, float thickness = 1.0f);
void render::draw_line_outlined(...same...);
void render::draw_circle(float x, float y, float radius, int r, int g, int b, int a, bool filled = false, int segments = 32);
void render::draw_triangle(float x1, float y1, float x2, float y2, float x3, float y3, int r, int g, int b, int a, bool filled = false);
```

## Polygons (render context)

```cpp
void render::draw_concave_poly(array<vec2> points, int r, int g, int b, int a);
void render::draw_convex_poly(array<vec2> points, int r, int g, int b, int a, bool filled);
void render::draw_polyline(array<vec2> points, int r, int g, int b, int a, float thickness, bool closed);
void render::draw_hue_bar(int x, int y, int w, int h, int r, int g, int b, int a);        // animated, matches menu
void render::draw_hue_bar_static(int x, int y, int w, int h, int r, int g, int b, int a);
```

# Menu UI (`ui::`)

Build order: **tab → sub-tab → category → elements**. Elements attach to the most recently created category; categories to the most recent tab/sub-tab. Do all `add_*` calls in `main()` before the script starts ticking.

## Structure

```cpp
struct ui_info { bool open; float dpi_scale; float x, y, w, h; color accent; };
struct active_bind { string category; string label; };
struct color { int r, g, b, a; };

bool ui::is_open(); float ui::get_scale(); color ui::get_accent(); ui_info ui::get_info();
string ui::get_tab(); string ui::get_sub_tab();
bool ui::get_active_binds(array<active_bind> &out binds);

void ui::config_path(string folder_name); // call once with a UNIQUE name → adds a config save/load tab

enum sub_tab_side { middle, left, right };
void ui::add_tab(string label);
void ui::add_tab(string label, sub_tab_side side); // middle: sub-tab bar on top, direct categories ignored; left/right: bar on that side
enum category_pos { automatic, left_full, right_full, left_top, left_bottom, right_top, right_bottom };
void ui::add_tab(string label);
void ui::add_subtab(string label);                    // branches off most recent tab
void ui::add_category(string label);
void ui::add_category(string label, category_pos pos); // automatic = up to 2 per column; paired halves get a resize grip
```

## Elements

`config_key` is optional everywhere — when omitted a key is auto-generated from tab/sub-tab/category/label (lowercased, sanitized). Pass an explicit key when you need stability across label renames. Callbacks are optional.

```cpp
checkbox ui::add_checkbox(string label, bool default_value, string config_key = "");
checkbox ui::add_checkbox(string label, bool default_value, checkbox_callback@ cb, string config_key = "");
slider ui::add_slider(string label, int default_val, int min_val, int max_val, int step, string tag = "", string config_key = "");
color_picker ui::add_color_picker(string label, int r, int g, int b, int a, bool stick = true, string config_key = "");
// color pickers stick to the checkbox above (max 3 per checkbox); stick=false detaches
combobox ui::add_combobox(string label, array<string> options, int default_idx, string config_key = "");
combobox ui::add_combobox(string label, array<string> options, int default_idx, combobox_callback@ cb, string config_key = "");
multi_select ui::add_multi_select(string label, array<string> options, string config_key = "");
button ui::add_button(string label, button_callback@ callback = nullptr);
```

Callback funcdefs: `void cb()`, `void cb(bool val)` (checkbox), `void cb(int idx)` (combobox), `void cb()` (button).

## Get / set / visibility / style

```cpp
bool   ui::get(checkbox);
int    ui::get(slider);
color  ui::get(color_picker);
uint64 ui::get(combobox);                       // selected index
bool   ui::get(multi_select, uint64 index);     // option at index checked?

ui::set(checkbox, bool); ui::set(slider, int); ui::set(color_picker, color);
ui::set(combobox, uint64 index); ui::set(multi_select, uint64 index, bool);

ui::set_visible(<any element>, bool);
ui::add_tooltip(<element>, string text);        // all except button

enum label_style { none, warning, dangerous }; // checkbox label color: default / orange / red
ui::set_label_style(checkbox, label_style style);
```

```cpp
// multi_select example
multi_select filters = ui::add_multi_select("Filters", {"Visible", "Dormant", "Dead"});
bool vis = ui::get(filters, 0);
```

# Logging & alerts

```cpp
void alert::show(string msg);
void alert::show(string msg, int r, int g, int b);
void alert::show(string msg, int r, int g, int b, float duration);
void alert::show(string msg, int r, int g, int b, float duration, bool rainbow);
void alert::show_tag(string msg, string tag[, int r, int g, int b[, float duration[, bool rainbow]]]); // same tag updates in place
void alert::dismiss(); void alert::dismiss(string tag);

void print(? msg);                    // accepts any type, auto-converts
void console::log(? msg); void console::log(? msg, int r, int g, int b);
void console::warn(? msg); void console::error(? msg); void console::clear();
```
