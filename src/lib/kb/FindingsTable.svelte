<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  let { ticket }: { ticket: number } = $props();

  // The export repeats reports where its pages overlap: count each id once.
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));

  // The desk read these as a one-day move, not a month flip. causes.ts files them under
  // `parser` by tenant; here they are their own problem.
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const parser = $derived((groups.find((g) => g.cause === 'parser')?.reports ?? []).filter((r) => !notes.oneDayMove.includes(r.id)));
  const hint = $derived(groups.find((g) => g.cause === 'hint')?.reports ?? []);

  const span = (rs: Report[]) => {
    const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
    return `${days[0]} → ${days[days.length - 1]}`;
  };

  const rows = $derived([
    { problem: 'parser reads day-first dates month first', cause: 'parser', reports: parser },
    { problem: 'booking-v3 hint says MM/DD/YYYY to day-first tenants', cause: 'hint', reports: hint },
    { problem: 'dates land one day off (not a month flip)', cause: 'one-day move', reports: oneDay },
  ]);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th></tr>
  </thead>
  <tbody>
    {#each rows as row (row.cause)}
      <tr>
        <td>{row.problem}</td>
        <td>{row.reports.length}</td>
        <td>{new Set(row.reports.map((r) => r.tenant)).size}</td>
        <td>{span(row.reports)}</td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
</style>
