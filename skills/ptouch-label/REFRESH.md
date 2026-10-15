# Refresh Procedure for ptouch-label

## On cadence
1. `ptouch-label --help`: diff the flags against the Flags table.
2. Read the README of github.com/johntrandall/ptouch-label for changed install steps, config
   keys or defaults (hi-res default, probe behaviour).
3. Check github.com/nbuchwitz/ptouch releases: if the mirror/chain/special-tape kwargs landed
   upstream, the "library gaps" paragraph is stale.

## Verification checklist
- [ ] Flags table matches `--help`
- [ ] Install and config sections match the CLI README
- [ ] Frontmatter updated (last_refreshed_at)
