<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# node-libmanta

This repository is part of the Joyent Manta project.  For contribution
guidelines, issues, and general documentation, visit the main
[Manta](http://github.com/joyent/manta) project page.

This repo serves to hold any/all common code that is shared between Manta
components.  Currently a [mahi](https://mo.joyent.com/docs/mahi) client,
the indexing ring client, and some common utils exist.

# Testing

You'll need to define the morays you have for the index ring, as well as the
address of where to find Mahi (redis); for example, assuming the bellingham
lab (and you have DNS set appropriately):

```
INDEX_URLS=tcp://1.moray.bh1.joyent.us:2020,tcp://2.moray.bh1.joyent.us:2020 \
  MAHI_HOST=authcache.bh1.joyent.us make prepush
```
