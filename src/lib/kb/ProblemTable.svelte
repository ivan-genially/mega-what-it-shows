<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem found in a merged ticket. The desk's one-day-move reports are their own
  // problem: the date moves by a day, it is not flipped, so neither parser nor hint explains them.
  type Problem = 'parser' | 'hint' | 'one-day' | 'unplaced';
  let {
    ticket,
    issues = {},
    states = {},
  }: { ticket: number; issues?: Partial<Record<Problem, string>>; states?: Partial<Record<Problem, string>> } = $props();

  const LABEL: Record<Problem, string> = {
    parser: 'day and month flipped by the shared parser',
    hint: 'booking-v3 form hint asks for MM/DD',
    'one-day': 'date moves by one day',
    unplaced: 'tenant not on file',
  };

  const problemOf = (r: Report): Problem =>
    notes.oneDayMove.includes(r.id) ? 'one-day' : (causeOf(r, tenantOf(r)) ?? 'unplaced');

  const rows = $derived.by(() => {
    const all = (reports as Report[]).filter((r) => r.mergedInto === ticket);
    const unique = uniqueById(all);
    const by = new Map<Problem, Report[]>();
    for (const r of unique) by.set(problemOf(r), [...(by.get(problemOf(r)) ?? []), r]);
    const groups = [...by.entries()]
      .map(([problem, rs]) => ({
        problem,
        count: rs.length,
        tenants: new Set(rs.map((r) => r.tenant)).size,
        channels: [...new Set(rs.map((r) => r.channel))].sort().join(' · '),
        mistagged: rs.filter((r) => r.meta.intakeTag !== problem).length,
      }))
      .sort((a, b) => b.count - a.count);
    return { exported: all.length, unique: unique.length, groups };
  });
</script>

<p class="sum">
  <strong>{rows.unique}</strong> reports in #{ticket} ({rows.exported - rows.unique} rows repeated in the export) ·
  <strong>{rows.groups.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>intake tagged otherwise</th><th>ticket</th><th>state</th></tr>
  </thead>
  <tbody>
    {#each rows.groups as g (g.problem)}
      <tr>
        <td><code>{g.problem}</code> {LABEL[g.problem]}</td>
        <td>{g.count}</td>
        <td>{g.tenants}</td>
        <td>{g.channels}</td>
        <td>{g.mistagged}</td>
        <td>{#if issues[g.problem]}<a href={issues[g.problem]}>{issues[g.problem]?.split('/').pop()?.replace(/^/, '#')}</a>{:else}—{/if}</td>
        <td class:open={states[g.problem] === 'open'}>{states[g.problem] ?? '—'}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .open { color: var(--warn); font-weight: 600; }
</style>
