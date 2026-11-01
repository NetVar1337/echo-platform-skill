# Networking, JSON, Filesystem, Util reference

## Async HTTP

Non-blocking; pass a callback. Callback overloads are **only callable from `main()`**.

```cpp
struct http_response { int status_code; string body; string error; }
// funcdef: void response_callback(http_response@ response) {}

void http::get(string url, response_callback@ cb);
void http::post(string url, string body, string content_type, response_callback@ cb);
// Fire-and-forget (no callback): callable from ANY context incl. on_unload() — for webhooks/one-way notifications
void http::get(string url);
void http::post(string url, string body, string content_type);
// User-Agent: prism/1.0 · timeout 10s · redirects followed automatically
// Inline callbacks work: http::post(url, body, "application/json", function(http_response@ res) { ... });
```

## WebSocket client

Persistent connection; **all callbacks fire on a separate connection thread** — protect shared state (mutex/atomics/snapshot).

```cpp
// funcdefs: void connect_callback() {}  void message_callback(string msg) {}
//           void disconnect_callback() {}  void error_callback(string msg) {}
void ws::connect(string url, connect_callback@ on_open, message_callback@ on_message,
                 disconnect_callback@ on_disconnect, error_callback@ on_error);
void ws::reconnect();
void ws::disconnect();
bool ws::is_connected();
void ws::send(string msg);
```

## JSON

One object type (`json_obj@`) for objects and arrays.

```cpp
enum json_type { null_value, boolean, number, string, object, array };

json_obj@ obj = json_obj();          // object mode
json_obj@ arr = json::array();       // array mode
json_obj@ parsed = json::parse(string text, string &out error); // null on failure, reason in error
string s = json::stringify(json_obj@ obj, bool beautify = false);

// inspection
bool has(string key); uint length();
json_type type(string key); json_type type_at(uint index);
array<string>@ keys();

// getters — return default if key missing / type mismatch
string get_string(key); float get_float(key); double get_double(key);
int get_int(key); uint get_uint(key); int64 get_int64(key); uint64 get_uint64(key);
bool get_bool(key); json_obj@ get_obj(key);            // null if missing or not obj/array
// index versions: get_string_at(uint i) ... get_obj_at(uint i)

// setters (overwrite existing key)
void set(key, string|float|double|int|uint|int64|uint64|bool|json_obj@ value);
void set_null(key); void remove(key);

// array push (array mode only, no-op otherwise)
void push(string|float|double|int|uint|int64|uint64|bool|json_obj@ value); void push_null();
```

## Filesystem (sandboxed to the data directory)

Paths relative to the data dir · max file size 512 MB · depth 2 levels · executable extensions blocked · dirs one level deep.

```cpp
bool file::exists(string filename);
bool file::write(string filename, string data);
bool file::append(string filename, string data);
bool file::read(string filename, string &out data);
bool file::remove(string filename);
bool file::size(string filename, uint64 &out size);
bool file::write_bytes(string filename, array<uint8> data);
bool file::append_bytes(string filename, array<uint8> data);
array<uint8>@ file::read_bytes(string filename); // null on failure

bool dir::exists(string name); bool dir::create(string name); bool dir::remove(string name);
array<string>@ dir::list(string name);
```

## Util

```cpp
string util::get_username();
void util::disable_capture_protection();       // unlocks anti-capture menu element
string util::wide_to_string(array<uint8> bytes, bool stop_at_null = false);
string util::to_hex(number, bool prefix = true); // prefix prepends "0x"

uint64 util::tickcount();                        // ms since boot
string util::system_time(bool use_12h = false);  // "16:32" / "4:32 PM"
uint64 util::time_now();
double util::time_ms / time_us / time_ns(uint64 start, uint64 end);

void util::seed_random(uint64 seed); // 0 = default sequence
int    util::random_int(int min, int max);      float util::random_float(float min, float max);
double util::random_double(double min, double max); int64 util::random_i64(int64, int64); uint64 util::random_u64(uint64, uint64);

string util::hex_encode(string|array<uint8>);    // lowercase hex; "Hello" → "48656c6c6f"
string util::hex_decode(string|array<uint8>);    // "" on failure
string util::base64_encode(string|array<uint8>); // "Hello" → "SGVsbG8="
string util::base64_decode(string|array<uint8>); // "" on failure

uint64 hash::fnv1a(string str); uint32 hash::crc32(string str);

// zip: flat only (no subdirectories; directory entries skipped). Parse raw zip bytes (e.g. from http::get / file::read)
struct zip_entry { string name; string data; }
array<zip_entry>@ util::zip_parse(string data);

// string formatting — {} next arg, {0} by index, {{ }} literal braces
// {:<8} left-align w8 · {:>8} right-align (default) · {:08} zero-pad · {:.2} 2 decimals · {0:<8.2} combined
// shorthand: {.2} precision only, {0.3} index+precision · invalid placeholder → {?} · width ≤1024, precision ≤17, ≤8 args
string util::format_string(string fmt, ...);
```

## Input [WinAPI]

```cpp
void input::mouse_move(int x, int y);
bool input::mouse_press(); bool input::mouse_release(); bool input::mouse_click();
void input::mouse_pos(int &out x, int &out y);
bool input::key_active(int vk_code);   // held
bool input::key_toggled(int vk_code);  // toggled on (caps lock etc.)
void input::key_press(int vk_code); void input::key_release(int vk_code); void input::key_tap(int vk_code);
```
