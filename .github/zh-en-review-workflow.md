# zh → en Review Workflow

Use this workflow whenever an English document in `en/` is reviewed against its Chinese source in `zh/`. The English text must be faithful, natural, and professional for a Western developer, with no Chinese-English.

## Requirements

1. **No omissions.** Every fact, qualifier, parenthetical, lead-in label, cross-reference, list item, example, and consequence in the source must appear in English. Recurring gaps: dropped "see X" references, flattened lists, lost bold emphasis, and missing lead-ins such as "关键操作：" or "参数说明：".
2. **No weakened meaning.** Do not soften a hard requirement, drop a condition, or merge two distinct cases into one vague sentence.
3. **Natural English without technical loss.** Rewrite stiff or literal phrasing into professional English a Western developer would write, but never trade away a technical distinction to make a sentence flow. If natural wording and precision conflict, keep the precision and smooth only the surrounding phrasing.
4. **Consistent terminology.** Reuse the terms already established in the English docs (HUK, lifecycle stage names, bank, fuse, boot gate, and so on) and link cross-references to real anchors.
5. **Source-block correspondence.** Preserve source order and keep each heading, lead-in, paragraph, list item, table row, admonition, example, and reference as a separately identifiable English counterpart. Do not merge or summarize separate blocks without user approval. Line breaks within a paragraph may differ for natural English. Do not add content the source does not state.
6. **Evaluate feedback, don't apply it mechanically.** Prefer targeted edits and keep wording that is already faithful and natural. Check each proposed revision against the source and the surrounding technical context; reject changes that introduce ambiguity, omit an actor, or alter a constraint.
7. **Preserve technical relationships.** Make the actor, control boundary, operating state, and direction of an operation explicit where they matter. For example, distinguish system time written to an RTC from time read from it; distinguish Linux framework requests from operations performed by another processor; and distinguish suspend, resume, shutdown, and hardware power-off.
8. **Flag uncertainty without inventing behavior.** When the source appears inconsistent, omits a condition, or makes a claim that needs implementation or hardware confirmation, preserve the source's meaning and report the issue for engineering review. Do not resolve it by adding an unsupported explanation.
9. **Preserve executable examples.** Keep commands, identifiers, values, and configuration syntax intact; translate explanatory comments without changing their meaning. Check code blocks for accidental escaping or formatting changes, and distinguish syntax inspection from compilation or hardware testing.
10. **Distinguish related technical terms.** Do not substitute similar terms merely for consistency when they name different abstractions or mechanisms. For example, an OPP table and a CPUFreq frequency table are not automatically interchangeable, and a PLL-switching threshold is not necessarily a Boost marker. Preserve the source's definitions and conditions.
11. **Match claims to evidence.** Preserve the difference between a request, a successful interface write, a reported state, and independently verified hardware behavior. Do not imply that an interface readback measures hardware directly, or that a successful write proves an operation succeeded, unless the source supports it.
12. **Use portable references.** Preserve source references, but avoid copying machine-specific absolute paths as links. Where the intended source tree is clear, use source-tree-relative paths and identify the repository or tree; retain usable links and verify section anchors. Check vendor names and product terminology for established capitalization without changing case-sensitive identifiers.

## Steps

1. Read the English file and the Chinese source in full.
2. Compare in source order, mapping each source block to its English counterpart. Check each table row, list item, lead-in, parenthetical, warning, reference, and code block individually; do not use line counts as a proxy for completeness.
3. Restore missing or compressed blocks and fix weakened details and unnatural sentences. Do not rephrase text that is already faithful and natural.
4. Make a second source-to-English pass over every block, checking values, units, conditions, commands, and cross-references separately. Verify link anchors and run `git diff --check`; run other relevant validation when available.
5. Report what changed, any unresolved technical questions, and what was not verified. Claim a full completeness review only when the entire source was actually checked.
