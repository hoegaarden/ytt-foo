# ytt'ing strings

Often enough there is the need to run ytt against a datastructure which
is part of a yaml document, but encoded as a string.
Think of e.g. kubernetes configmaps, which hold config files that are
encoded as strings.

We can have `ytt` more or less transparantly decode those strings,
run our overlays against, and encode the result as a string again.

We can even take it a bit further and run ytt also against something that is
encoded as a string and base64 encoded.
Think of kubernetes secrets.

Have a look a the totally contrived examples in [overlay.yml](./overlay.yml),
especially have
a look at the helper functions:
- `asYaml` & `asJson`
- `asB64Yaml` & `asB64Json`
- their usage with `#@overlay/replace via=<func>`

You can run the example with
```
ytt -f input.yml -f overlay.yml
```

Note that in the input all leaf nodes are strings, so are all leaves of the
output. But we were still able to run our overlay against the sring encoded
values.

## Alternative, generic approach

In [overlay_generic.yml] we have `apply` which returns
another function to be used with `@#overlay/replace via=...`. It can take
multiple codec, which allows you to e.g. transparently first base64 decode, then
yaml decode and then run either a function or a overlay against the left side.

For more docs and examples, see the [overlay_generic.yml] file.

[overlay_generic.yml]: ./overlay_generic.yml
