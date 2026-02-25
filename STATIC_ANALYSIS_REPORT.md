# Static Analysis Report (lu5-55)

This report summarizes key issues found via:
- Makefile/build inspection
- Warning-oriented compilation attempts (`gcc -fsyntax-only`)
- Targeted grep for unsafe APIs
- Manual inspection of hotspot files

> Environment note: `cppcheck`, `clang-tidy`, and `scan-build` were not available in this container at analysis time.

## 1) Build blockers: missing dependencies and missing vendored Lua

### 1.1 Missing pkg-config deps / headers
Build expects:
- `glfw3` (`GLFW/glfw3.h`)
- `glew`
- `gl`
- `freetype2` (`ft2build.h`)

Compilation attempts fail with missing headers and pkg-config failures, e.g.:
- `fatal error: GLFW/glfw3.h: No such file or directory`
- `fatal error: lua.h: No such file or directory`
- `Package 'glfw3' not found` (pkg-config)

### 1.2 Missing `include/lua` sources
`Makefile` runs:

- `cd include/lua && make PLATFORM=linux`

But `include/lua` is empty in this checkout, and the build fails with:
- `No targets specified and no makefile found`

**Recommended fix**
- Ensure Lua sources are present (vendored or as a git submodule) under `include/lua/`.
- Document required system packages or provide a `make deps`/bootstrap step that validates them.

## 2) Memory safety: definite heap overflow in `lu5_key_callback`

**File:** `src/lu5_event_callbacks.c`

Issue:
- `malloc(key_name_length)` allocates `strlen(name)` bytes
- `sprintf(key_name, "%s", name)` writes `strlen(name) + 1` bytes including NUL terminator

This is a heap overflow and can lead to crashes or exploitation.

**Recommended fix**
- Allocate `key_name_length + 1`
- Use `snprintf` instead of `sprintf`
- Also review logic: code checks `if (name == NULL && key >= 65 && key <= 90)` after already returning when `name == NULL`.
- `lua_pushlstring(lu5.L, key_name, 1)` pushes only 1 byte; likely should push full string length.

## 3) Unsafe formatting: `sprintf` with fixed slack buffers

Occurrences:
- `src/bindings/window.c`:
  - `malloc(len + 10); sprintf(window_title, "[lu5]: %s", sketch_path);`
- `src/platform/lu5_cli_options.c`:
  - `malloc(sketch_name_len + 10); sprintf(installed_path, "/usr/bin/%s", sketch_name);`

**Risk**
- Buffer overflow if assumptions change or inputs exceed expectations.

**Recommended fix**
- Prefer:
  - `size_t n = (size_t)snprintf(NULL, 0, fmt, ...) + 1;`
  - `char *buf = malloc(n);`
  - `snprintf(buf, n, fmt, ...);`

## 4) Definite memory leak: OBJ model close does not free all allocations

**File:** `src/platform/lu5_obj.c`

Allocations:
- `model->vertices`
- `model->faces`
- `model->normals`
- `model->texcoords`

But `lu5_close_model` frees only:
- vertices, faces, model

**Recommended fix**
- Also `free(model->normals); free(model->texcoords);`
- Add allocation failure handling (and free partial allocations).

## 5) Out-of-bounds read risk in `lu5_image_crop` and missing malloc checks

**File:** `src/platform/lu5_image.c`

Issues:
- No bounds validation on `x,y,w,h` vs image dimensions; memcpy source indexing can go OOB.
- No `malloc` NULL checks for `originalPixels` and `croppedPixels`.

**Recommended fix**
- Validate crop rectangle:
  - `x >= 0`, `y >= 0`, `w > 0`, `h > 0`
  - `x + w <= image->width`
  - `y + h <= image->height`
- Check allocations and handle errors without leaking.

## 6) File I/O robustness issues

**File:** `src/platform/lu5_fs.c`

Issues:
- `lu5_write_file`: on short write, returns without `fclose(file)` (FILE* leak).
- `lu5_read_file`: does not check `ftell` / `fseek` errors; does not validate `fread` count.

**Recommended fix**
- Ensure `fclose` on all error paths.
- Check `ftell` for `-1L` and validate `fread` return count.

---
End of report.
