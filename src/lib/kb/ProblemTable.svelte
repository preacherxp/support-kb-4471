<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket, counted the desk's way (CauseTable): every report under its
  // cause by tenant (causes.ts). The ones the desk read as a one-day move (desk-notes.json) stay counted
  // there and also make their own row: no month/day flip explains them.
  let { ticket, filed = {} }: { ticket: number; filed?: Record<string, string> } = $props();

  const problemsOf = (r: Report) => [
    causeOf(r, tenantOf(r)) ?? 'unplaced',
    ...(notes.oneDayMove.includes(r.id) ? ['one-day move'] : []),
  ];

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const rows = $derived.by(() => {
    const by = new Map<string, Report[]>();
    for (const r of merged) for (const p of problemsOf(r)) by.set(p, [...(by.get(p) ?? []), r]);
    return [...by]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return { problem, count: rs.length, tenants: new Set(rs.map((r) => r.tenant)).size, first: days[0], last: days.at(-1) };
      })
      .sort((a, b) => b.count - a.count);
  });
  const twice = $derived(merged.filter((r) => problemsOf(r).length > 1).length);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as r (r.problem)}
      <tr class:open={!filed[r.problem]}>
        <td><code>{r.problem}</code></td>
        <td>{r.count}</td>
        <td>{r.tenants}</td>
        <td>{r.first} → {r.last}</td>
        <td>{#if filed[r.problem]}<a href={filed[r.problem]}>{filed[r.problem].replace(/^https:\/\/github\.com\/(.+)\/issues\//, '$1#')}</a>{:else}no ticket yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#if twice}
  <p class="note">{twice} reports count twice: under their cause by tenant, where the desk files them, and as a one-day move.</p>
{/if}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
