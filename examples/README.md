cc-me Examples
==============

**cc.yml** is an example definition that cc-me consumes.

To compile it into the desired cloud-config YAML files, run:

```sh
spruce merge cc.yml | spruce json | cc-me
```

(assuming `cc-me` is in your `$PATH`)

This will output two files, `lab-proto.yml` and `lab-sandbox.yml`,
each representing a different set of networking configurations in
the fictional lab.
