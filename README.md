# sdl3-ttf

SDL_ttf 3 for sysl — a font file, and text rendered out of it onto a surface or straight to a
texture.

```
dependencies {
  sdl3     { git = "github.com/sysl-lang/sdl3",     version = "0.1.0" }
  sdl3-ttf { git = "github.com/sysl-lang/sdl3-ttf", version = "0.1.0" }
}
```

```sysl
import sh.sysl.sdl3.*
import sh.sysl.sdl3_ttf.*

main()
    init(INIT_VIDEO)
    ttf_init()

    val window = create_window("text", 640, 200, 0).expect("a window")
    val renderer = window.create_renderer().expect("a renderer")
    val font = open_font("/System/Library/Fonts/Supplemental/Arial.ttf", 48.0).expect("a font")
    val label = font.texture(renderer, "Hello, sysl", rgb(240, 240, 240)).expect("a label")

    renderer.clear_to(rgb(20, 20, 30))
    renderer.copy_at(label, 40.0, 60.0)
    renderer.present()
    delay(2000)
```

## Installing

```
brew install sdl3_ttf                   # pulls sdl3 with it
sysl run prog.sysl --include-path /opt/homebrew/include --link-path /opt/homebrew/lib
```

The two flags are deliberate — see [`sdl3`](https://github.com/sysl-lang/sdl3)'s README, which also
says why this is a separate package rather than a module inside that one.

## `SDL_Color` crosses by value

The four render calls take their colours as `SDL_Color`, a four-byte struct passed **by value** in
C, and sysl does that correctly.

That is worth saying out loud because the Scala binding to this same library cannot: Scala Native
mis-marshals a small by-value struct argument, so it packs the four channels into a `uint32` by hand
and relies on an aggregate of four bytes and a `uint32` travelling in the same register — true on
SysV-AMD64 and AArch64, and a bet everywhere else. Nothing of the sort is here.

The tests prove it the only way that is worth anything. A *shaded* render fills the whole surface
with its background and draws the glyphs over it, so pixel (0, 0) is the background and nothing
else — and `TTF_RenderText_Shaded` takes two colours, so swapping them must swap that pixel. A
colour that arrived as bytes derived from an argument pointer gives some other number.

## The four qualities

| call | antialiased | background | surface |
|---|---|---|---|
| `render_solid` | no | none | 8-bit palettized |
| `render_shaded` | yes | solid, filled | 8-bit palettized |
| `render_blended` | yes | none, alpha channel | 32-bit |
| `render_lcd` | subpixel | solid, filled | 32-bit |

`render_blended` is the usual choice. `render_solid` is the cheapest and looks worst at a large
size; `render_lcd` reads better on an LCD at a small one and is wrong on anything rotated or scaled.
Each has a `_wrapped` form taking a pixel width, where zero wraps only on newlines.

## Measuring

`measure` gives the pixels a string would occupy without rendering it, and the tests assert it
agrees with the surface a render actually produces — a layout that disagreed with the renderer would
put text where the text is not.

`fit(text, max_width)` answers how much of a string lands in a width, **in bytes**: what an ellipsis
needs to know where to cut, and what an editor needs to place a cursor from a mouse position.

## Tests

```
sysl test . --include-path /opt/homebrew/include --link-path /opt/homebrew/lib
```

Fifteen tests, headless, against a real SDL3_ttf. They find a font by trying the places one lives on
macOS and on the common Linux distributions, rather than hard-coding a path — and say so plainly if
none of them is there, instead of failing forty lines later on a null handle.

## License

ISC — see [LICENSE](LICENSE).
