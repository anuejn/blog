---
layout: post
title:  "A Graphical User Interface for the AXIOM recorder!"
date:   2021-08-23 18:58:36 +0200
categories: rust apertus ui
---
The AXIOM Recorder is a software package that can be used to record video from the AXIOM Beta or AXIOM micro cameras. It is a pure command line application that is able to receive data via multiple interfaces (such as USB3 or Ethernet) and write the contents in different formats to disk. It also features a flexible pipeline-processing approach that allows the user to compose processing nodes (such as a USB3 input node, a debayering node and an MP4 output node). Likewise, it is even possible to get a live preview via a “preview” node.

However, the CLI based interface of the AXIOM recorder is difficult to use outside a lab setting. For example, the recording workflow previously consisted of switching between preview and recording pipelines on the command line. To be able to use the AXIOM recorder outside the lab, a GUI shall be implemented!

## On the quest for a suitable GUI framework
Because the rest of the axiom recorder is written in `Rust` making use of `Vulkan` for GPU acceleration, it makes sense to have the UI also written in `Rust`. Additionally, an efficient data sharing mechanism (preferably zero-copy) between the `vulkan` based processing steps of the recorder and the UI framework is necessary to achieve the required performance on modest hardware.

Both traditional libraries (`QT`, `GTK`) and novel libraries written in `Rust` were considered. However, it was found that all the libraries had severe problems that make them unsuitable for building the AXIOM recorder UI:

| Framework / Library | Deficiencies                                                              |
| ------------------- | ------------------------------------------------------------------------- |
| `GTK`               | `Vulkan` support requires a minimum of GDK4 which has no `Rust` bindings. |
| `QT`                | No `Rust` bindings for `Vulkan` integration.                              |
| `Conrod`            | No support for widgets that emit custom draw calls.                       |
| `Iced`              | No support for `Vulkan` backends.                                         |
| `Druid`             | No support for `Vulkan` backends.                                         |
| `OrbTK`             | No support for widgets that emit custom draw calls.                       |
| `egui`              | Supports only primitive layout and composability of widgets.                       |
| `imGUI`             | No `Rust` bindings for `Vulkan` integration using `vulkano`; hard to make good-looking. |

## Reinvent the wheel? Introducing `narui`!
Considering the lack of suitable GUI libraries, the mission was clear: build our own UI framework: [`narui`, the “**n**ew **A**XIOM **r**ecorder **u**ser **i**nterface toolkit”](https://github.com/apertus-open-source-cinema/narui)

First, a lot of consideration went into the user facing API and UI design pattern of the UI framework.

Because the author (and many other people) had a good time with `react` it was decided that `narui` would get a declarative-style API that mimics`react`.

Especially close influence were "new style" `react` components that are just plain functions. State management should be done using a mechanism similar to `react`-hooks. This is realized in `narui` by the means of unique keys that get assigned to each `widget`. These keys are basically pointers into a data structure that is used to store state.

For more details on the concrete API, see the [`narui` readme](https://github.com/apertus-open-source-cinema/narui).


### `react` in rust? Macros to the rescue!
Since `react` relies heavily on the `jsx` syntax extension to javascript, it is not straightforward to build a UI library with a similar API in `Rust` that is still ergonomic to use.
Fortunately, `rust` allows developers to write really powerful macros that can run arbitrary code at compile time and therefore implement complex additions to the language.

These `proc-macro`s allow us to write an `rsx! {}` macro that accepts jsx-like syntax and semantics inside.
Fortunately, there is already a helper to parse xml/html like syntax: the [`syn-rsx` crate](https://github.com/stoically/syn-rsx). Additionally, the lack of keyword (and default) arguments makes it hard to implement correct semantics for the `rsx` macro since it is inherently keyword argument based (e.g. `<rect border_radius=Some(Points(20.)) />`). To implement keyword arguments, an "attribute macro", with which one has to annotate all widget functions, is used. This `proc-macro` generates a normal macro that implements keyword arguments according to the function signature. This macro is then used by the `rsx` `proc-macro`.  Quite a rube goldberg machine!

In the end, this machinery provides a convenient API to compose a hierarchical tree of widgets. Furthermore, the combination of the macros make it possible to implicitly pass a `context` struct to all widgets without any additional user-written boilerplate.

Finally, to make on demand execution of the widgets possible the macros convert each widget into a closure that can be called down the line.

### Baby Steps: Boxes, layout, input, text
Each widget in `narui` can be either
A composition of other widgets without layout / render information attached to itself  (e.g. a button or a slider)
A “primitive widget” that features layout information and optionally rendering instructions (e.g. a rectangle or text)
The “primitive widgets” are collected into their own tree that is then used to calculate the layout using `stretch`. 
[The `stretch` crate](https://github.com/vislyhq/stretch) provides a pure `Rust` flexbox layout implementation. Flexbox is a technology widely known by developers who are used to the `react` API.

Text drawing was implemented with help of the [`glyph-brush` crate](https://github.com/alexheretic/glyph-brush). `glyph-brush` generates a texture atlas containing a rasterized version of the font glyphs using a CPU font rasterizer as well as text shaping information. These are then combined to render the text as a series of textured quads`

All other shapes are drawn by tesselating them with the [CPU based tessellator `lyon`](https://github.com/nical/lyon), which produces triangle strips that are then drawn with the GPU.

Input handling is done in the most naive way possible: checking each input element for collisions on every click / mouse update.

All this effort led to the first moment of great serotonin: the first narui demo:
![button grid](https://codimd.niemo.de/uploads/upload_9089262c8e2be00e5345f338cae1192a.gif)

### Ka-Boom: Usable Widgets
After these foundations were laid, some basic but useful input widgets could be implemented. Also, basic classic UI-framework demos were possible.

A slider...
![slider demo](https://codimd.niemo.de/uploads/upload_98f1f0f72fb32c803dcf7d206f0d4f19.gif)

...and buttons
![counter demo](https://codimd.niemo.de/uploads/upload_91d7964572a2c2b740eee0001140d22e.gif)

### Here be Dragons: Delta updates
To make declarative UI competitive in performance, it is crucial that not everything is re-evaluated, re-layouted and redrawn every frame. Thankfully, narui was designed with partial re-evaluation in mind.

The most challenging part of delta updates in the context of narui is partial re-evaluation: Only these `widget` functions that depend on changed state or whose input changed should be run. To achieve that, all arguments to `widget` functions need to be stored and compared to the prior value before the actual function is run. Values that are comparable by value are compared by value, while for everything else pointer comparison is used. [This is somewhat tricky to implement in stable rust.](https://github.com/apertus-open-source-cinema/narui/blob/2084777fc270f139b8a3f70805559a14e884aa08/src/heart/all_eq.rs)

State in `narui` is stored in a tree, to which widgets can obtain references using the `context` that is passed to them.
Because a widget needs to be re-evaluated if either the state it depends on or its arguments changed, an obvious choice is to store the arguments in the normal state tree. This however comes with a cost: Since the state tree entries that are used to propagate widget arguments are only updated after the widget is evaluated, one needs `n` delta evaluation passes for a `n` deep hierarchy of changed widgets. This, however, seems like a reasonable tradeoff to make to simplify the complex problem of re-evaluation.

A lot of time was spent to implement sound re-evaluation, and it happened multiple times that some new edge case was found and large portions of `narui` (including the data structures, all the delta eval code, and the macros) needed to be rewritten.

Delta re-layout in contrast was a breeze to implement. `stretch` already supports delta re-layout and after the implementation of delta re-evaluation it was also clear what updates to set in the `stretch` layout tree.

Partial re-rendering is currently not implemented by `narui`. This is because the time spent on actual drawing is insignificant for any known `narui` use case yet. However, the tesselation and font rasterization are cached.

### Node Graphs: The end of declarative UI?!
After all the foundations of `narui` were laid, it was time to build a more complex UI.
Because it is clear that it would be desirable to have a visual representation of the (already node based) processing flow inside the recorder, a node graph was built for that "kitchen sink" test.

That process proved to be really valuable for validating the API of `narui`, both conceptually and implementation-wise. For example, it showed that the addition of the `context.post_frame` hook, which allows executing code after a frame was rendered, was necessary.

Also, the addition of `context.measure*` hooks were introduced which allow the widgets to use information from the layout pass. This proved to be invaluable for input handling in the node graph.

![node graph](https://codimd.niemo.de/uploads/upload_58d19b079cfc3e053b56017f05d34b4b.gif)


## Towards a usable GUI for the AXIOM recorder

The final part of building a GUI for the AXIOM recorder is building a GUI for the AXIOM recorder ;).

In the beginning, as a guideline for the necessary capablities `narui` minimum viable product, a sketch of the axiom recorder UI was created:
![figma sketch](https://codimd.niemo.de/uploads/upload_bbdd410156ca88e5a7bbd3cb2a211345.png)

Implementing this UI led to the exploration of a new topic, the communication between a `narui` widget tree and outside code / computation. Computations are run in a separate thread from the UI to ensure responsiveness of the UI. The computation threads can directly update UI state (which causes re-rendering). In the other direction, the UI sends events to the computation threads to manipulate the computations’ state.

 Currently, the AXIOM recorder is still in a click dummy like phase: No real recording can happen, a connection & settings dialog is missing, and the pipeline is hard coded. That said, it still validates that the UI framework
1. provides the required elements to implement the envisioned UI
2. is flexible enough to interact with the existing recorder codebase (like the `Vulkan` processing nodes)
![current state of the AXIOM recorder UI](https://codimd.niemo.de/uploads/upload_fd1cf662f283a9ff5461ce7298fe1a0f.gif)

## The End?

Google Summer of Code 2021 is done and the foundations for building the AXIOM recorder UI (and many other application UIs) is laid.
However, there are still many areas in which both `narui`, and the AXIOM recorder need to evolve:

For `narui` this is text input handling, z-ordering / draw ordering, optimized hit box testing and the implementation of a clipper, that allows the creation of things like a list view.
One further plan to improve `narui` is to abandon `stretch` for layout and implement a custom layout engine that implements a `flutter`-style layout algorithm. This would be beneficial because `stretch` is unmaintained and better performance and flexibility could be achieved. [This work is already started by (@vup).](https://github.com/apertus-open-source-cinema/narui/pull/9)

The implementation of the AXIOM recorder UI is now within reach and tracked in [this PR](https://github.com/apertus-open-source-cinema/axiom-recorder/pull/7). It is also likely that there will be some graph based video processing tool based on `narui` and the AXIOM recorder codebase. Stay tuned!

All in all, building `narui` was great fun (many thanks to @vup, who was a wonderful mentor). I deepened my understanding of rust (especially regarding proc macros) and intricacies that go into building a declarative UI framework with GPU rendering (like partial re-evaluation).

`narui` promises to be really helpful for the future of the AXIOM recorder and hopefully other applications.
