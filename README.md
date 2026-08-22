# sdl3-ttf

SDL_ttf 3 for sysl — a font file, and text rendered out of it onto a surface or straight to a
texture.

```
dependencies {
  sdl3     { git = "github.com/sysl-lang/sdl3",     version = "0.3.0" }
  sdl3-ttf { git = "github.com/sysl-lang/sdl3-ttf", version = "0.3.0" }
}
```

```sysl
import sh.sysl.sdl3.*
import sh.sysl.sdl3_ttf.*
import sh.sysl.sdl3.c.INIT_VIDEO

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
sysl run prog.sysl
```

No flags — see [`sdl3`](https://github.com/sysl-lang/sdl3)'s README, which also says why this is a
separate package rather than a module inside that one. Two packages here read a header:
`sh.sysl.sdl3_ttf.c` asks the C compiler for SDL_ttf's style, hinting and alignment constants rather
than transcribing them, and `sh.sysl.sdl3.c` does the same for SDL's. Each names its library, so
pkg-config answers for both and a missing one is refused by name.

Until 0.2.1 that took three flags, and the override is now spelled `--include-path sdl3-ttf=<dir>`
rather than `sdl3_ttf=` — the name is pkg-config's now. **Needs sysl 0.0.56.**

## Two layers, and handles that own themselves

`sh.sysl.sdl3_ttf.c` holds everything that is C — the link directive, the header, the `c const`
blocks, the opaque `TTF_Font` and the thirty-five declarations. `sh.sysl.sdl3_ttf` is what an
application imports.

A `Font` is a `&T` with an `impl Drop`, so `close` is not part of this API: the font goes when the
last reference to it does. **`TTF_CloseFont` after `TTF_Quit` is a use-after-free**, which is
SDL_ttf's rule rather than this binding's — a program that calls `ttf_quit` should let its fonts go
out of scope first, and one that never calls it is fine.

The style mask stays a mask, in `c`, because several styles are true at once and that is what an
enumeration cannot say. Hinting and wrap alignment are enumerations: `Hinting.Mono`,
`WrapAlignment.Center`, each with an `Other` arm and a `Display`.

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

## One value that does not round-trip, and it is SDL_ttf's

`Hinting.LightSubpixel` sets light hinting *plus* a subpixel-positioning flag, and
`TTF_GetFontHinting` reports only the hinting half — so a font set to it answers `Hinting.Light`.
That is the library's behaviour rather than this binding's, and the tests pin it as what actually
happens, so that the missing round trip is a documented fact rather than a suspicion about the
mapping.

## Tests

```
sysl test .
```

Eighteen tests, headless, against a real SDL3_ttf. They find a font by trying the places one lives on
macOS and on the common Linux distributions, rather than hard-coding a path — and say so plainly if
none of them is there, instead of failing forty lines later on a null handle.

## License

ISC — see [LICENSE](LICENSE).
