# MENTOR-RL TikZ figures

The `.tex` files in this directory are reusable `tikzpicture` snippets. Their
shared palette, typography, arrows, panels, and biological glyphs live in
`mentor-rl-style.tex`.

To use them from another document, load TikZ and the same libraries as
`proposal.tex`, input the style file once in the preamble, and input a figure
inside a `\resizebox`:

```tex
\usepackage{tikz}
\usetikzlibrary{arrows.meta,backgrounds,calc,fit,positioning,shapes.geometric}
\input{path/to/mentor-rl-style}

\begin{figure}[tbp]
  \centering
  \resizebox{\linewidth}{!}{\input{path/to/three-structural-views}}
  \caption{...}
\end{figure}
```

Color is semantic across figures: green marks retained or supported biology,
orange marks inputs and decisions, blue marks deterministic/runtime operations,
purple marks learned representations or policy operations, and rose marks
noise, unsupported branches, or cautionary cases.
