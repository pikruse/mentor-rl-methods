# MENTOR-RL TikZ figures

The `.tex` files in this directory are reusable `tikzpicture` snippets. Their
shared palette, typography, arrows, panels, and biological glyphs live in
`mentor-rl-style.tex`.

The publication exports use Avenir Next, matching the other dissertation
figures. Build all seven PDFs with XeLaTeX from this directory:

```sh
make
```

The generated artwork is written to `pdf/`. The standalone exporter uses the
macOS Avenir Next system font through `fontspec`; the exported PDFs embed the
font so downstream manuscripts can continue to build with pdfLaTeX.

During figure development, load TikZ and the same libraries as `proposal.tex`,
input the style file once in the preamble, and input a figure inside a
`\resizebox`:

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

For a manuscript or dissertation build, prefer the generated PDFs so the font
and layout do not depend on the document's LaTeX engine:

```tex
\begin{figure}[tbp]
  \centering
  \includegraphics[width=\linewidth]{path/to/figure-02-three-structural-views.pdf}
  \caption{...}
\end{figure}
```

Color is semantic across figures: green marks retained or supported biology,
orange marks inputs and decisions, blue marks deterministic/runtime operations,
purple marks learned representations or policy operations, and rose marks
noise, unsupported branches, or cautionary cases.
