# Troy

Vue and TSX in, native apps out.

Troy is a compiler. It reads Vue and TSX components, lowers them into one shared intermediate representation, and generates SwiftUI for macOS and WinUI 3 for Windows. Nothing from Troy ships inside your app. There is no webview and no runtime library, only Swift and C# source that you can open in Xcode or Visual Studio and read like code a person wrote.

> [!WARNING]
> Troy is in the design phase and there is nothing to install yet. This README describes what we are building and where the work stands. The syntax in the examples is a target, so expect it to move.

## Why

If you know web UI and want a desktop app, the usual answer is to ship a browser along with it. That works, but you pay for it in memory and startup time, and the app rarely feels at home on the platform. The alternative is to write the app twice, once in Swift and once in C#.

Troy bets on a third option: keep the component model people already know, and treat it as a source language for a compiler instead of something to interpret while the app runs.

## How it works

```
  .vue  ↘               ↗  SwiftUI  (macOS)
           Troy IR
  .tsx  ↗               ↘  WinUI 3  (Windows)
```

A frontend parses a source file and lowers it to Troy IR. The IR describes a component in terms that belong to no framework and no platform: state, layout, text, controls, events. A backend walks that IR and writes native source code.

Frontends and backends never talk to each other. A new source language costs one frontend, a new platform costs one backend, and neither has to know the other exists.

## The counter

The first milestone is the smallest app that still exercises the whole pipeline: a counter.

You write this:

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <column>
    <text>Count: {{ count }}</text>
    <row>
      <button @click="count -= 1">-</button>
      <button @click="count += 1">+</button>
    </row>
  </column>
</template>
```

Troy lowers it to IR. Printed as a tree, it looks roughly like this:

```
component Counter
  state count: int = 0

  column
    text "Count: {count}"
    row
      button "-"  onTap: count = count - 1
      button "+"  onTap: count = count + 1
```

The SwiftUI backend turns that into `Counter.swift`:

```swift
import SwiftUI

struct Counter: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("Count: \(count)")
            HStack {
                Button("-") { count -= 1 }
                Button("+") { count += 1 }
            }
        }
    }
}
```

And the WinUI 3 backend into `Counter.cs`:

```csharp
using Microsoft.UI.Xaml.Controls;

public sealed partial class Counter : UserControl
{
    private int _count = 0;
    private readonly TextBlock _text = new();

    public Counter()
    {
        var minus = new Button { Content = "-" };
        var plus = new Button { Content = "+" };
        minus.Click += (_, _) => SetCount(_count - 1);
        plus.Click += (_, _) => SetCount(_count + 1);

        Content = new StackPanel
        {
            Children =
            {
                _text,
                new StackPanel
                {
                    Orientation = Orientation.Horizontal,
                    Children = { minus, plus },
                },
            },
        };

        SetCount(_count);
    }

    private void SetCount(int value)
    {
        _count = value;
        _text.Text = $"Count: {_count}";
    }
}
```

The same component written in TSX lowers to the same IR, so it produces the same two files.

## IR v0

The first version of the IR is small on purpose. We would rather have six nodes that compile correctly on both platforms than sixty that mostly work.

| Node | What it is | SwiftUI | WinUI 3 |
| :- | :- | :- | :- |
| `state` | A reactive value owned by the component | `@State` | private field |
| `column` | Vertical stack | `VStack` | `StackPanel` |
| `row` | Horizontal stack | `HStack` | `StackPanel`, horizontal |
| `text` | Static or interpolated text | `Text` | `TextBlock` |
| `button` | Tappable control | `Button` | `Button` |
| `onTap` | Tap or click handler | action closure | `Click` event |

Everything else is out of scope for v0. Once the schema is written down we will publish it together with a support matrix that also lists what the compiler ignores, so nobody has to find out by trial and error.

## Roadmap

- [ ] Write the v0 IR schema: state, layout, events, and the list of what is ignored
- [ ] Vue frontend, `.vue` to IR
- [ ] TSX frontend, `.tsx` to IR
- [ ] SwiftUI backend
- [ ] WinUI 3 backend
- [ ] The counter compiling and running on macOS and Windows
- [ ] Documented environment setup for both platforms
- [ ] Public support matrix

## Open questions

We have not settled whether generic types, async, and classes belong in the first version of the IR. Leaving them out keeps the MVP small and the type system simple. Putting them in is what fuller TSX support and a future Angular frontend will need.

If you have designed an IR before and have opinions on this, we would like to hear them.

## Building the output

Troy generates source code, so you still need the platform toolchain to build it:

| Target | Runs on | You need |
| :- | :- | :- |
| SwiftUI | macOS | Xcode |
| WinUI 3 | Windows | .NET SDK and the Windows App SDK |

Exact versions will be pinned when the backends land.

## Contributing

It is early, which means the most useful contributions right now are opinions. Read the IR section, look at the open questions, and tell us where this is going to break. The [issue tracker](https://github.com/rafaelaugustos/troy/issues) is open.

Code contributions will make more sense once the v0 schema is merged. If you want to be around for that, watch the repo.
