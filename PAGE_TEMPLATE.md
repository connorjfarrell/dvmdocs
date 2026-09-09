<!--
Page skeleton for dvmdocs. Copy into the right folder under dvmdocs/, fill in,
and add the file to dvmdocs/_toc.yml. Delete these comments.
-->

# Page Title

One or two sentences: what this is and who needs it.

```{note}
Use admonitions ({note}, {warning}, {important}, {tip}) for asides. Add the DVM usage-policy
{warning} on pages about software that transmits.
```

## Main section

Prose. Link internally with relative paths: [dvmhost](../software/dvmhost/dvmhost.md).
Cross-link glossary terms: {term}`FNE`.

### Config / commands

```yaml
# keep snippets short — link the full upstream file below
key: value
```

```bash
some-command --flag
```

## Reference table

| Field | Meaning |
|-------|---------|
| `foo` | ... |

## Source / further reading

- Upstream README: <https://github.com/DVMProject/...>
- Example config: <https://github.com/DVMProject/dvmhost/blob/master/configs/...>
- Technical note: `docs/TN.xxxx` — <https://github.com/DVMProject/dvmhost/tree/master/docs>
