+++
title = "Working with OCaml expect tests in NeoVim"
date = 2025-03-25
[taxonomies]
tags=["Programming", "OCaml", "NeoVim"]
+++

This is just a quick note and not a full-on post.
I've been playing around with inline expect tests in OCaml using the [ppx_expect](https://github.com/janestreet/ppx_expect) library.
The way these expect tests work is that they will capture `stdout` from select parts of your test function, then they will either store that as their value if you are running it for the first time, or they will compare the output against their stored value.
If the new output is different from the stored output, then `dune` will give you a nice diff and tell you that it stored a corrected version of that file.

```diff
--- a/_build/default/lib/query.ml
+++ b/_build/.sandbox/6248834ca8898cf067c96e6df4119c43/default/lib/query.ml.corrected
@@ -15,7 +15,7 @@ let print = sexp_of_t >> Sexp.to_string

 let%expect_test "parse load csv" =
   printf "%s" (print (parse "(LoadCSV blah.csv)"));
-  [%expect {| |}]
+  [%expect {| (LoadCSV blah.csv) |}]
```

If you decide you want to accept that diff, then you can run `dune promote`:

```bash
camalbrarian (main) $ dune promote
Promoting _build/default/lib/query.ml.corrected to lib/query.ml.
```

I want to read through all of the [dune docs](https://dune.readthedocs.io/en/stable/) and maybe write a post dedicated to `dune`'s features, but I love the integration between `dune` and the testing library.
This workflows feels great even if you are switching tabs/terminals to manually run `dune promote`.

But, I use NeoVim, so of course, I have to make my own keymappings so I can do all of this within my editor:

```lua
-- Run all tests and try to promote the current file
vim.keymap.set('n', '<leader>rp', ':silent !dune runtest<cr>:silent !dune promote %<cr>')
-- Run all tests and show the results
vim.keymap.set('n', '<leader>rr', ':!dune runtest<cr>')
```

These keymaps have been working great so far!
My only complaint is that `dune` does not support running inline tests for a single file (as far as I can tell).
So, you have to run all the tests in a project.
But, at least `dune promote` can take in a file parameter, so I can only promote the changes for the file I'm looking at.

Most of the time, I like to run `<leader>rr` first to view the changes in NeoVim to make sure they look good.
Then I run `<leader>rp` to run and promote.
Now that I think about it, I may just change `<leader>rp` to only promote and let it promote changes for all files.

I'm usually only working on writing one test at a time anyways.

I hope you enjoyed the NeoVim + OCaml tip.
Thanks for reading!
