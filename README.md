cc-me
=====

I deploy a lot of BOSH stuff into vSphere environments that
partition out a larger network (like a /20 or a /16) into smaller
"subnets" that are only isolated by convention and the occasional
firewal configuration.

![Zee Cloud!!](img/cloud.jpg)

Writing a cloud-config for this type of setup is difficult,
because BOSH doesn't allow you to specify the networking bits of
cloud-config in the affirmative; you have to reserve out the IPs
in the main network that you aren't using in the cloud-config
network.  This is tedious and error-prone, and often leads to
incorrect (and unchangeable) network partitioning.

So I wrote `cc-me`.

It takes an input JSON file (although I use `spruce merge` and
`spruce json` to get from YAML to JSON), and does all the boring
network address calculations to ensure a clean and robust
cloud-config.

Note: this isn't intended for IaaSes that allow you to officially
segment your network, like AWS or Azure.  Please use the specific
features to limit broadcast domains where it is available to you!

The input file defines a series of _environments_, each of which
will correspond to one output cloud-config file.

```yaml
---
meta:
  lab:
    range:   10.192.0.0/16
    gateway: 10.192.0.1
    cloud_properties:
      name: lab-net1

environments:
  - name: lab-proto
    networking:
      - range:   10.192.0.0/16
        gateway: 10.192.0.1
        cloud_properties:
          name: lab-net1

        limit: 10.192.0.0/24
        layout: bestfit
        networks:
          - { name: bosh,         net: /30, static: 1 }
          - { name: vault,        net: /30 }
          - { name: concourse,    net: /28, static: 3 }
          - { name: shield,       net: /29, static: 1 }
          - { name: bolo,         net: /30, static: 1 }

  - name: lab-sandbox
    networking:
      - range:   10.192.0.0/16
        gateway: 10.192.0.1
        cloud_properties:
          name: lab-net1

        limit: 10.192.1.0/24
        layout: bestfit
        networks:
          - { name: bosh,         net: /30, static: 1 }
          - { name: jumpbox,      net: /30, static: 1 }
          - { name: cf-edge,      net: /28, static: 4 }
          - { name: cf-core,      net: /27 }
          - { name: cf-runtime,   net: /25 }
          - { name: cf-services,  net: /26 }
```

(remember, I spruce merge this, then convert it to JSON, and then
pipe it into `cc-me`; it makes life easier, believe it or not!)

```
$ spruce merge example.yml | spruce json | ./cc-me
>> writing lab-proto.yml cloud-config...
>> writing lab-sandbox.yml cloud-config...
```

It couldn't be any easier.  The two cloud-configs have the
correct, non-overlapping network ranges defined, with the
appropriate reservations and static IP address allocations!

The `bestfit` layout even re-orders the networks to acheive the
most efficient IP distribution possible.  If you run this, take
note of the fact that in `lab-sandbox.yml`, the `cf-runtime`
network is actually last, after all the smaller networks, since it
eats up half of the available address space.

If you need to maintain cohesion with pre-existing cloud-configs,
you can use `layout: strict`, and cc-me will *NOT* reorder the
networks to pack the most efficiency.  If you need gaps in the
ranges, you can specify your network name as `SKIP`, and cc-me
will go through the motions of allocating the network range, but
_not_ emit the configuration to the cloud-config.

For each network you define under `networks:`, you can specify how
many static IP address you will need (if any).  If you set a
number, i.e. `static: 5`, you will get that many static IPs.  You
can also specify a percentage, which will be calculated according
to the usable IPs in the network range.  You can also specify
percentages as a fraction, like 1/5, to capture "one for every X"
semantics.

All environment-level YAML/JSON keys (aside from `networking` and
`name`) will be passed through as-is to the generated
cloud-config.  This allows you to put the entire cloud-config
under cc-me's control, without cc-me interfering.

Happy Hacking!
