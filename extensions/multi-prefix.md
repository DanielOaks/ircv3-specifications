---
title: IRCv3 `multi-prefix` Extension
layout: spec
redirect_from:
  - /specs/extensions/multi-prefix-3.1.html
copyrights:
  -
    name: "William Pitcock"
    period: "2012"
    email: "nenolod@dereferenced.org"
---
## The multi-prefix Client Capability

When requested, the multi-prefix client capability will cause the IRC server to send
all possible prefixes which apply to a user in NAMES and WHO output.

These prefixes MUST be in order of 'rank', from highest to lowest.

Example:

    --> NAMES #tethys
    :hades.arpa 353 guest = #tethys :~&@%+aji &@Attila @+alyx +KindOne Argure
    :hades.arpa 366 guest #tethys :End of /NAMES list

    --> WHO #test
    :kenny.chatspike.net 352 guest #test grawity broken.symlink *.chatspike.net grawity H@%+ :0 Mantas M.
    :kenny.chatspike.net 315 guest #test :End of /WHO list

