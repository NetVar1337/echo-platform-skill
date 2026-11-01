# Engine helpers, math, vectors, matrices, tracing, clipping reference

## Source engine

```cpp
bool source::w2s(vec3 world_pos, matrix4x4 view_matrix, vec2 &out screen_pos); // false = off-screen/behind camera
uint64 source::get_entity(uint64 entity_list_address, uint64 index);          // 0 if invalid
```

## Source 2 engine

```cpp
bool source2::w2s(vec3 world_pos, matrix4x4 view_matrix, vec2 &out screen_pos);
uint64 source2::get_controller(uint64 entity_list_address, uint64 index);
uint64 source2::get_pawn(uint64 entity_list_address, uint64 controller_address, uint64 pawn_offset);
```

## Unreal engine

```cpp
struct f_view_info { vec3 location; vec3 rotation; float fov; }; // rotation = pitch/yaw/roll
f_view_cache build_view_cache(f_view_info camera, bool vertical_fov = false); // build ONCE per frame before iterating entities
bool unreal::w2s(vec3 world_pos, f_view_cache cache, vec2 &out screen_pos);

struct u_bone_array { bool valid(); uint count(); /* indexable by raw bone index, e.g. bones[110] */ };
u_bone_array unreal::read_bones(uint64 skeletal_mesh, uint64 bone_array_offset, uint64 bone_array_cached_offset, uint64 component_offset);
u_bone_array unreal::read_bones(uint64 skeletal_mesh, uint64 bone_array_offset, uint64 component_offset); // no cached fallback (may flicker)

struct ftransform { /* rotation: quat x,y,z,w; translation; scale_3d */ };
ftransform bone_transform;
bone_transform.read(bone_address);                                  // read from memory
vec3 pos = bone_transform.transform_to_world(ftransform parent);     // local → world
```

## Unity engine

```cpp
bool unity::w2s(vec3 world_pos, matrix4x4 view_matrix, vec2 &out screen_pos);
vec3 unity::rust_transform_position(uint64 transform_address);
uint32 unity::rust_prefab_id(string name); // e.g. "assets/bundled/prefabs/autospawn/resource/ores/stone-ore.prefab"
```

## IW engine (3x3 view matrix: row0=forward, row1=right, row2=up)

```cpp
matrix3x3 view_matrix; view_matrix.read(address);
bool iw_engine::w2s(vec3 world_pos, vec3 camera_location, matrix3x3 view_matrix, float fov_x, float fov_y, vec2 &out screen_pos);
```

## Math helpers

```cpp
math::PI, math::TAU, math::RAD2DEG, math::DEG2RAD;
enum angle_range { R_180, R_360 }; // R_180 → [-180,180], R_360 → [0,360]

matrix4x4 math::multiply_matrix(matrix4x4 a, matrix4x4 b);
matrix4x4 math::rotation_matrix(vec3 rotation, vec3 origin);
vec3 math::rotated_vector(vec3 point, vec4 quaternion);

vec3 math::forward_vector(vec2 angles);  // (pitch, yaw)
vec3 math::right_vector(vec2 angles);
vec3 math::up_vector(vec2 angles);

vec2 math::vec_to_angles(vec3 direction);
vec3 math::angles_to_vec(vec2 angles);
double math::normalize_angle(double angle, angle_range range);
double math::angle_diff(double a, double b);           // shortest angular difference
double math::calculate_fov(vec2 a, vec2 b);
vec2 math::calculate_angle(vec3 from, vec3 to);        // (pitch, yaw) from→to
vec2 math::clamp_angles(vec2 angles, angle_range range); // pitch clamped to [-89, 89], yaw normalized
double math::calculate_fov(vec2 a, vec2 b);
vec2 math::smooth_angles(vec2 current, vec2 target, angle_range range,
                         double smoothing /*0 fast – 1 slow*/, double min_speed /*1–45*/, double max_speed /*1–45*/);
// clamp+normalize happens automatically; tune min/max speed to your thread rate
```

### Unreal-specific angle helpers

```cpp
vec2   math::unreal_rotation_to_radians(vec3 camera_rot);                       // (pitch, yaw) radians
double math::unreal_calculate_fov(vec3 camera_rot, vec3 camera_pos, vec3 target_pos);   // degrees, 0 = dead-on
vec2   math::unreal_calculate_angle(vec3 camera_rot, vec3 camera_pos, vec3 target_pos); // delta degrees to add
vec2   math::unreal_smooth_rotation(vec3 camera_rot, vec3 camera_pos, vec3 target_pos,
                                    double smoothing, double min_speed, double max_speed); // smoothed deltas, radians
```

## Vectors

All of vec2/vec3/vec4: `.x .y [.z .w]`, `is_empty()`, `length()`, `length_squared()`, `normalize()`, `dot(other)`, `add/subtract(other)`, `multiply(scalar)`, `distance(other)`, `lerp(other, t)`.
Memory ops (read/write directly at an address in the attached process): `read_float(addr)`, `read_double(addr)`, `write_float(addr)`, `write_double(addr)` — vec2 = 2 comps, vec3 = 3, vec4 = 4.

## Matrices

`matrix4x4`, `matrix3x4`, `matrix3x3` — index `m[row][col]`, `bool read(addr)` (float) and `bool read_double(addr)`.

## Tracing (rays & BVH)

```cpp
struct triangle { vec3 p1, p2, p3; }

bvh_tree@ tree = create_bvh_tree();        // null on alloc failure
tree.add_triangle(p1, p2, p3);
tree.add_triangles(const array<triangle>& tris);
bool tree.build();                          // false if no triangles; REQUIRED before queries
bool tree.is_occluded(vec3 from, vec3 to);  // false if not built; THREAD-SAFE for concurrent queries
// any mutation invalidates until build() again
tree.clear(); uint64 tree.triangle_count(); bool tree.is_ready();

// one-off tests (ray vs primitive within max_t):
bool trace::ray_triangle(const triangle &in tri, vec3 origin, vec3 dir, double max_t);   // Moller-Trumbore
bool trace::ray_bbox(vec3 origin, vec3 dir, vec3 box_min, vec3 box_max, double max_t);   // slab
bool trace::ray_sphere(vec3 origin, vec3 dir, vec3 center, double radius, double max_t);
bool trace::ray_capsule(vec3 origin, vec3 dir, vec3 cap_min, vec3 cap_max, double radius, double max_t);
```

## Clipping (2D polygon boolean ops — Clipper-style)

```cpp
enum clip_type { INTERSECTION, UNION, DIFFERENCE, XOR };
enum fill_rule { EVEN_ODD, NON_ZERO, POSITIVE, NEGATIVE };
enum join_type { MITER, SQUARE, BEVEL, ROUND };
enum end_type { ET_POLYGON, ET_JOINED, ET_BUTT, ET_SQUARE, ET_ROUND };

clip_paths@ p = clip_paths();
p.add_path(const array<vec2> &in points);        // polygon or polyline
array<vec2>@ p.get_path(uint index);
uint p.path_count();
p.clear(); clip_paths@ q = p.clone();
p.boolean_op(clip_paths@ clips, clip_type, fill_rule = NON_ZERO); // mutates p
p.inflate(double delta, join_type = ROUND, end_type = ET_POLYGON); // +delta outset, − inset
p.simplify(double epsilon = 0.1);
p.union_self(fill_rule = NON_ZERO);              // merge overlapping paths in-place
// draw result: render::draw_polyline(p.get_path(i), r,g,b,a, thickness, closed)
```
