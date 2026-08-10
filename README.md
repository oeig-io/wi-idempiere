# iDempiere Enhancement

The purpose of this repo is to hold the work instructions for planning and executing iDempiere changes — configuring a tenant through the Application Dictionary, writing the Groovy and SQL artifacts that carry a change, and moving that change between environments.

It matters because iDempiere is unforgiving about *where* a rule belongs: one requirement can be a callout, a model validator, or a process, and choosing wrong is expensive to undo. This README routes a need to the instruction that owns it; the skills themselves hold the detail.

## Discovering

The rest of this repo is self-describing — ask it rather than reading a list here:

```bash
ls *-tool.md                            # what tools this repo holds
rg -N '^description:' *-tool.md         # what each one is for
rg -l 'AD_InfoWindow' *-tool.md         # which tool covers a given AD table
```

## Related Documentation

- Standards for these documents: `../wi-base/WORK_INSTRUCTIONS.md`
- Host-level Incus infrastructure (storage pools, networks, profiles): `wi-incus`
- OEIG-specific iDempiere vocabulary and production database access: `wi-idempiere-oeig`

Tags: #idempiere #application-dictionary #groovy #deploy
