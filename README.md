# imgui_md (fork)

A Markdown renderer for [Dear ImGui](https://github.com/ocornut/imgui) powered by the [MD4C](https://github.com/mity/md4c) parser.

> **This is a fork** of `imgui_md` with a revised public API and safer build integration.

## Why this fork

- **No public MD4C dependency:** `imgui_md.h` no longer includes `md4c.h`. The MD4C dependency now lives **inside `imgui_md.cpp`**.
- **Optional node editor integration:** the internal hook to ImGui Node Editor is guarded by `IMGUI_MD_WITH_NODE_EDITOR` (default **off**), avoiding link errors when the editor is not present.
- **Font API update:** `get_font()` now returns **`MdSizedFont { ImFont*, float size }`** (requires **ImGui ≥ 1.92** for `ImGui::PushFont(font, size)`).

License remains MIT (same as upstream). Some ideas are inspired by [`imgui_markdown`](https://github.com/juliettef/imgui_markdown).

---

## Features

- Wrapped text
- Headers (`#`, `##`, …)
- Emphasis (italic, bold), underline, strikethrough
- Ordered & unordered lists (including nested)
- Links & images
- Horizontal rules
- Tables
- Inline HTML snippets: `<br>`, `<hr>`, `<u>`, `<div>`, `&nbsp;`
- Backslash escapes

**Table notes**
- Column widths are driven by the header row.
- Cells are left-aligned.

---

## Installation

### A) Add sources directly
Add these files to your project:
- From this fork: `imgui_md.h`, `imgui_md.cpp`
- From MD4C: `md4c.c`, `md4c.h`

The public header **does not** include `md4c.h`; consumers do **not** need MD4C include paths.  
If `md4c.h` is not in the default location, you can override it:

```bash
# Example: set a custom include path for md4c.h
g++ -DIMGUI_MD_MD4C_INCLUDE=\"path/to/md4c.h\"
````

### B) Minimal CMake example

```cmake
# imgui_md
add_library(imgui_md imgui_md.cpp)
target_include_directories(imgui_md
    PUBLIC  ${CMAKE_CURRENT_SOURCE_DIR}   # for #include "imgui_md.h"
)

# md4c (C library)
enable_language(C)
add_library(md4c STATIC ${MD4C_DIR}/src/md4c.c)
target_include_directories(md4c PUBLIC ${MD4C_DIR}/src)

target_link_libraries(imgui_md
    PUBLIC  imgui::imgui
    PRIVATE md4c
)

# Node editor hook is OFF by default
target_compile_definitions(imgui_md PUBLIC IMGUI_MD_WITH_NODE_EDITOR=0)
```

---

## Usage

```cpp
#include "imgui_md.h"

// Ensure fonts/textures are created elsewhere.
// See https://github.com/ocornut/imgui/blob/master/docs/FONTS.md
extern ImFont*     g_font_regular;
extern ImFont*     g_font_bold;
extern ImFont*     g_font_bold_large;
extern ImTextureID g_texture1;

struct MyMarkdown : imgui_md {
    // New API (ImGui >= 1.92): return font + size
    MdSizedFont get_font() const override {
        if (m_is_table_header)
            return { g_font_bold, ImGui::GetFontSize() };

        switch (m_hlevel) {
            case 0: return { m_is_strong ? g_font_bold : g_font_regular, ImGui::GetFontSize() };
            case 1: return { g_font_bold_large,                            ImGui::GetFontSize() };
            default:return { g_font_bold,                                   ImGui::GetFontSize() };
        }
    }

    void open_url() const override {
        // Platform-specific open; for example:
        // SDL_OpenURL(m_href.c_str());
    }

    bool get_image(image_info& nfo) const override {
        // Use m_img_src / m_href to resolve the image
        nfo.texture_id = g_texture1;
        nfo.size       = ImVec2(40, 20);
        nfo.uv0        = ImVec2(0, 0);
        nfo.uv1        = ImVec2(1, 1);
        nfo.col_tint   = ImVec4(1, 1, 1, 1);
        nfo.col_border = ImVec4(0, 0, 0, 0);
        return true;
    }

    void html_div(const std::string& cls, bool enter) override {
        if (cls == "red") {
            if (enter) {
                m_table_border = false;
                ImGui::PushStyleColor(ImGuiCol_Text, IM_COL32(255, 0, 0, 255));
            } else {
                ImGui::PopStyleColor();
                m_table_border = true;
            }
        }
    }
};

void RenderMarkdown(const char* text, const char* text_end) {
    static MyMarkdown md;
    md.print(text, text_end);
}
```

### Overriding callbacks with the new thin types

The public API no longer exposes MD4C types. Instead, thin wrappers are used:

```cpp
struct Hooks : imgui_md {
    void BLOCK_H(const MdBlockHDetail* d, bool enter) override {
        if (enter) m_hlevel = d->level;
    }
    void SPAN_A(const MdSpanADetail* d, bool enter) override {
        if (enter) m_href.assign(d->href.text, d->href.size);
        else       m_href.clear();
    }
};
```

---

## Public API highlights

* **Font**

  ```cpp
  struct MdSizedFont { ImFont* font; float size; };
  virtual MdSizedFont get_font() const;   // required; ImGui >= 1.92
  ```

* **Thin detail types (no MD4C headers in public API)**

  ```cpp
  struct MdAttr           { const char* text; int size; };
  struct MdBlockHDetail   { int     level; };
  struct MdBlockCodeDetail{ MdAttr  lang;  };
  struct MdSpanADetail    { MdAttr  href;  };
  struct MdSpanImgDetail  { MdAttr  src;   };
  ```

* **Representative callbacks**

  ```cpp
  // Blocks
  virtual void BLOCK_UL(bool enter);
  virtual void BLOCK_OL(bool enter);
  virtual void BLOCK_LI(bool enter);
  virtual void BLOCK_H   (const MdBlockHDetail*   d, bool enter);
  virtual void BLOCK_CODE(const MdBlockCodeDetail*d, bool enter);
  virtual void BLOCK_TABLE(bool enter);
  virtual void BLOCK_TD(bool enter);

  // Spans
  virtual void SPAN_A  (const MdSpanADetail*   d, bool enter);
  virtual void SPAN_IMG(const MdSpanImgDetail* d, bool enter);

  // Helpers for link/image
  void set_href   (bool enter, const MdAttr& a);
  void set_img_src(bool enter, const MdAttr& a);
  ```

* **Protected state you can use in overrides**

  * `m_hlevel` (current header level), `m_is_strong`, `m_is_em`, `m_is_underline`, `m_is_strike`
  * `m_is_table_header`, `m_table_border`
  * `m_href`, `m_img_src` (current link/image source during parsing)

---

## Optional node editor integration

By default the internal hook to ImGui Node Editor is **disabled**:

* Define `IMGUI_MD_WITH_NODE_EDITOR=1` if you link the node editor and want the integration enabled.
* With the default `0`, a safe stub is used and there are **no extra link dependencies**.

```cmake
target_compile_definitions(imgui_md PUBLIC IMGUI_MD_WITH_NODE_EDITOR=0)  # default
```

---

## Breaking changes (compared to original)

1. **Font API**

   * **Was:** `ImFont* get_font() const`
   * **Now:** `MdSizedFont get_font() const`
     Requires **ImGui ≥ 1.92** (uses `ImGui::PushFont(font, size)`).

2. **MD4C types are hidden**

   * Public header no longer includes `md4c.h`.
   * Callback parameters use thin wrappers (`MdBlockHDetail`, `MdSpanADetail`, …).
     Consumers do **not** need MD4C include paths.

3. **Node editor hook is opt-in**

   * Controlled by `IMGUI_MD_WITH_NODE_EDITOR` (default `0`).

---

## Compatibility

* **Dear ImGui:** **1.92+** (relies on `PushFont(font, size)`).
  On older ImGui, you would need a local fallback via `SetWindowFontScale()` (not officially supported).
* **Compiler:** C++11 or newer.
* **MD4C:** One C translation unit (`md4c.c`) plus header (`md4c.h`).
  If needed, override header path with `IMGUI_MD_MD4C_INCLUDE`.

---

## Examples

```markdown
# Table

Name &nbsp;&nbsp;&nbsp;&nbsp; | Multiline&nbsp;&nbsp;&nbsp;&nbsp;<br>header  | [Link&nbsp;](#link1)
:------|:-------------------|:--
Value-One | Long <br>explanation | 1
~~Value-Two~~ | __auto wrapped__ | 25
**etc** | [link](https://github.com/mekhontsev/imgui_md) | 3
```

```markdown
<div class="red">

This table | is inside an | HTML div
--- | --- | ---
Still | ~~renders~~ | __nicely__
Border | **is not** | visible

</div>
```

(See upstream for screenshots of similar output.)

---

## Credits

* [Dmitry Mekhontsev — imgui\_md (original)](https://github.com/mekhontsev/imgui_md)
* [Martin Mitáš — MD4C](https://github.com/mity/md4c)
* [Omar Cornut — Dear ImGui](https://github.com/ocornut/imgui)

## License

MIT, same as upstream.
