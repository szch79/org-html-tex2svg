# Org HTML Export TeX2SVG

A minor mode `org-html-tex2svg` for asynchronous SVG generation during Org HTML export of LaTeX environments, primarily those cannot be rendered by MathJax.

The API can also be used for custom HTML-like backends.

**Disclaimer**: this package is intended for personal use, so I might not be responsive to issues.  Also, if anything fails, please make sure that the error is not from `org-latex-preview` (e.g., getting a message like `Creating LaTeX preview images failed (exit code 252). Please see *Org Preview Convert Output* for details`) before reporting, thanks.

## Motivation

A common practice for people using MathJax in HTML export is to put their `tikz` diagrams (or other non-MathJax-renderable environments) into an Org Babel LaTeX source block, which evaluates to an SVG during export.  This practice is detailedly demonstrated in posts like [LaTeX Source Code Blocks in Org Mode](https://orgmode.org/worg/org-contrib/babel/languages/ob-doc-LaTeX.html) and [Org Mode & LaTeX - Unwound Stack](https://www.unwoundstack.com/blog/org-mode-and-latex.html).

However, there are many drawbacks to this approach:
1. creates many boilerplates;
2. sequential evaluation and SVG generation for each Org Babel LaTeX block during export is slow;
3. requires careful management of asset paths;
4. Org Babel LaTeX blocks are not as handy, especially since it *does not* read `#+latex_header` and requires a separate `:headers` argument for the packages/macros (this [patch](https://lists.gnu.org/archive/html/emacs-orgmode/2025-12/msg00049.html) helps a bit, but still not perfect);
5. (most annoying) LaTeX fragments in source blocks *cannot* be previewed, which is disastrous for diagrams.

On the other hand, [`org-latex-preview`](https://abode.karthinks.com/org-latex-preview/) (by Tecosaur and Karthink) is a compelling package that provides blazing fast LaTeX fragment preview, with async image generation.  An unfortunate pitfall is that `org-latex-preview`'s asynchronicity breaks Org Babel LaTeX (see this [issue](https://github.com/tecosaur/org-latex-preview-todos/issues/28) and relevant Discord [discussion](https://discord.com/channels/406534637242810369/1056621127188881439/1215336716366250074)), which means that it *cannot* be used with the above practice of exporting non-MathJax-renderable LaTeX.

A natural question to ask is: *can we take the great parts from both worlds*?  The answer turns out to be affirmative.

## Usage

Before using this package, make sure that you have [`org-latex-preview`](https://abode.karthinks.com/org-latex-preview/) installed and working correctly.  It is a bit of work to get it run, so we can't do that for you.

```emacs-lisp
(use-package org-html-tex2svg
  :after org
  :vc (:url "https://github.com/szch79/org-html-tex2svg")
  :hook (org-mode . org-html-tex2svg-mode)
  :config
  ;; Like `org-latex-preview-process-default', but for export only.
  (setopt org-html-tex2svg-process-default 'dvisvgm)
  (setopt org-html-tex2svg-scale 1.3)
  ;; Put environments that need to be converted here.
  (setopt org-html-tex2svg-environments '("tikzpicture" "tikzcd" "mathpar")))
```

The `org-html-tex2svg-mode` minor mode hacks/advices Org HTML export to make it convert non-MathJax-renderable LaTeX to SVG.  `org-html-tex2svg-html-template` and `org-html-tex2svg-trigger-function` give more fine-grained control for advanced users, please consult their docstrings.

### For Export Backend Authors

For those who want to write custom export backends deriving from Org HTML export, there is a set of aliases that can be used for `:translate-alist` and `:filters-alist`:

| Symbol                              | Type       | Description                                                                  |
|-------------------------------------|------------|------------------------------------------------------------------------------|
| `org-html-tex2svg-latex-environment` | Transcoder | For `latex-environment` elements; falls back to `org-html-latex-environment` |
| `org-html-tex2svg-latex-fragment`    | Transcoder | For `latex-fragment` objects; falls back to `org-html-latex-fragment`        |
| `org-html-tex2svg-filter-parse-tree` | Filter     | Parse-tree filter; starts async SVG generation                               |
| `org-html-tex2svg-extend-transcoder` | Factory    | Extend the base transcoder with TeX2SVG conversion                           |

Example usage:

```emacs-lisp
(require 'org-html-tex2svg)
(org-export-define-derived-backend 'my-html 'html
  :translate-alist
  '((latex-environment . org-html-tex2svg-latex-environment)
    (latex-fragment . org-html-tex2svg-latex-fragment))
  :filters-alist
  '((:filter-parse-tree . org-html-tex2svg-filter-parse-tree)))
```

## Design

The `org-html-tex2svg-mode` hooks into `org-export-filter-parse-tree-functions`, and fires async SVG generation with `org-latex-preview` for target LaTeX fragments.  Meanwhile, the SVG generation process runs asynchronously in the background as the export proceeds.  When the export starts to process `latex-fragment` and `latex-environment` elements that are targeted, we will consult the cache from `org-latex-preview` - if ready, inject the SVG; otherwise, await there.

A bonus we get is that if `org-latex-preview` has already generated SVGs for the elements (in previous previews or exports), we get the cache for free.
